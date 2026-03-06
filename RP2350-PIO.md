# RP2350 PIO Reference — Chapter 11

Source: RP2350 Datasheet DS-2, Chapter 11 (pp. 877–961)

---

## 1. Architecture Overview

RP2350 contains **3 PIO blocks** (vs 2 on RP2040). Each PIO block has:
- **4 state machines** (12 total across chip)
- **32-instruction shared instruction memory** (write-only from system; all 4 SMs read simultaneously)
- **8 IRQ flags** per PIO block
- **Independent connections** to bus fabric, GPIO, and interrupt controller

Each state machine has:
- Two 32-bit shift registers: **OSR** (output) and **ISR** (input)
- Two 32-bit scratch registers: **X** and **Y**
- 4×32-bit TX FIFO + 4×32-bit RX FIFO (joinable to 8×32 one-direction)
- Fractional clock divider (16 integer, 8 fractional bits)
- Flexible GPIO mapping (up to 30 user GPIOs on RP2350)
- DMA DREQ interface

**All instructions execute in exactly 1 clock cycle** (unless stalling). Each instruction may add up to 31 delay cycles.

---

## 2. RP2350-Specific Changes from RP2040

### New Registers / Controls
- `DBG_CFGINFO.VERSION` = 1 on RP2350 (was 0 on RP2040) — use for runtime feature detection
- `GPIOBASE` — selects which 32 of the 30+ GPIOs a PIO block can access
- `CTRL.NEXT_PIO_MASK` / `CTRL.PREV_PIO_MASK` — apply CTRL operations to neighbouring PIO blocks simultaneously:
  - `CTRL.NEXTPREV_SM_DISABLE` — stop SMs in multiple PIO blocks simultaneously
  - `CTRL.NEXTPREV_SM_ENABLE` — start SMs in multiple PIO blocks simultaneously
  - `CTRL.NEXTPREV_CLKDIV_RESTART` — synchronise clock dividers across PIO blocks
- `SM0_SHIFTCTRL.IN_COUNT` — masks unneeded IN-mapped pins to zero (useful for `MOV x, PINS`)
- `IRQ0_INTE` / `IRQ1_INTE` — now expose all 8 SM IRQ flags to system interrupts (was only lower 4)
- `RXF0_PUTGET0..3` — random read/write access to RX FIFO internal storage registers

### New Instruction Features
- `WAIT JMPPIN` — wait on the pin indexed by `PINCTRL_JMP_PIN` plus an offset (0–3)
- `MOV PINDIRS, src` — change direction of all OUT-mapped pins in one instruction (not on PIO v0)
- `MOV x, STATUS` — can now include SM IRQ flags as source (allows branching on IRQ flags)
- IRQ instruction extended — can set/clear/observe IRQ flags from **different PIO blocks** (PREV/NEXT modes); no delay penalty for cross-PIO IRQ
- `FJOIN_RX_GET` FIFO mode — new `MOV osr, rxfifo[y/index]` instruction (random read from RX FIFO)
- `FJOIN_RX_PUT` FIFO mode — new `MOV rxfifo[y/index], isr` instruction (random write to RX FIFO)

### General Improvements
- 3 PIO blocks instead of 2 (12 SMs total)
- Improved GPIO input/output delay and skew
- Reduced DMA DREQ latency by 1 cycle vs RP2040

---

## 3. Programmer's Model

### 3.1 Registers

| Register | Width | Description |
|----------|-------|-------------|
| PC | 5-bit | Program counter. Wraps at 31. Updated each cycle. |
| X | 32-bit | Scratch register. Used as loop counter, data, or branch condition. |
| Y | 32-bit | Scratch register. Same uses as X. |
| OSR | 32-bit | Output Shift Register. Loaded from TX FIFO via PULL/autopull. Shifted out by OUT. |
| ISR | 32-bit | Input Shift Register. Shifted in by IN. Drained to RX FIFO via PUSH/autopush. |
| Shift counter (OSR) | 6-bit saturating | Tracks bits shifted out since last PULL/autopull. Reset to 0 on PULL; 32 at reset. |
| Shift counter (ISR) | 6-bit saturating | Tracks bits shifted in since last PUSH/autopush. Reset to 0 on PUSH. |

### 3.2 Shift Counter Rules
- `OUT` instruction: OSR shift counter += bit count (saturates at 32)
- `IN` instruction: ISR shift counter += bit count (saturates at 32)
- `PULL` / autopull: OSR counter → 0
- `PUSH` / autopush: ISR counter → 0
- `MOV OSR, x`: OSR shift counter → 0 (treated as full)
- `MOV ISR, x`: ISR shift counter → 0 (treated as empty)
- `OUT ISR, n`: ISR shift counter → n

### 3.3 FIFOs
- Default: 4-entry TX + 4-entry RX per SM
- Joinable (SHIFTCTRL_FJOIN): 8-entry TX-only or 8-entry RX-only
- DMA DREQ signals generated from FIFO state
- **Changing FJOIN discards FIFO contents**
- New RP2350 FIFO modes: FJOIN_RX_PUT, FJOIN_RX_GET (for use as status/config registers)

### 3.4 Stalling
SM stalls (PC does not advance) on:
- `WAIT` condition not met
- Blocking `PULL` with TX FIFO empty
- Blocking `PUSH` with RX FIFO full
- `IRQ WAIT` — set IRQ, waiting for acknowledgement
- `OUT` when autopull enabled and OSR at threshold, TX FIFO empty
- `IN` when autopush enabled, ISR at threshold, RX FIFO full

**Side-set always takes effect on the first cycle of the instruction, even if the instruction stalls.**
Delay cycles do not begin until after the stall clears.

### 3.5 IRQ Flags
- 8 flags per PIO block, visible to all 4 SMs in that block
- All 8 now routable to system IRQ lines (RP2350; was only lower 4 on RP2040)
- SMs interact via `IRQ` (set/clear) and `WAIT IRQ` (wait-and-clear)
- Cross-PIO communication via PREV/NEXT modes — no delay penalty
- Cross-PIO IRQs between Non-secure and Secure PIO blocks are disabled

### 3.6 Inter-SM Interactions
- SMs share instruction memory (1-write, 4-read register file)
- SMs cannot share data directly, but synchronise via IRQ flags
- Higher-numbered SMs take priority when multiple SMs write the same GPIO
- `CTRL.NEXTPREV_*` allows simultaneous enable/disable/clkdiv-restart across PIO blocks

---

## 4. PIO Assembler (pioasm)

### 4.1 Directives

```
.program <name>
```
Start a program. Name must be alphanumeric/underscore, not starting with digit.

```
.pio_version <version>
```
0 or RP2040 for RP2040 compatibility; 1 or RP2350 for RP2350 features. Default is 1 (RP2350).

```
.define (PUBLIC) <symbol> <value>
```
Define integer constant. PUBLIC exposes it to user code.

```
.origin <offset>
```
Force program to load at specific instruction memory offset (0–31).

```
.side_set <count> (opt) (pindirs)
```
Reserve `count` MSBs of delay/side-set field for side-set. `opt` makes side-set optional (costs 1 extra bit). `pindirs` drives pin directions instead of levels. Max count = 5 (leaves 0 delay bits).

```
.wrap_target
.wrap
```
Mark start and end of hardware wrap loop (free, 0-cycle jump). Defaults: wrap_target = start of program, wrap = after last instruction.

```
.out <count> (left|right) (auto) (<threshold>)
```
Configure OUT: bit count, shift direction, autopull enable, autopull threshold.

```
.in <count> (left|right) (auto) (<threshold>)
```
Configure IN: bit count, shift direction, autopush enable, autopush threshold. On PIO v0, count must be 32.

```
.set <count>
```
Configure number of SET bits.

```
.fifo <config>
```
Configure FIFO mode: `txrx` (default), `tx`, `rx`, `txput`*, `txget`*, `putget`*. (* RP2350 only)

```
.mov_status rxfifo < <n>
.mov_status txfifo < <n>
.mov_status irq (prev|next) set <n>
```
Configure source for `MOV x, STATUS`. IRQ option requires PIO v1 (RP2350).

```
.clock_div <divider>
```
Set SM clock divider (floating point). Affects default SM config.

```
.word <value>
```
Store raw 16-bit instruction value.

```
.lang_opt <lang> <name> <option>
```
Language-generator-specific options.

### 4.2 Instruction Syntax Pattern
```
<instruction> (side <side_set_value>) ([<delay_value>])
```
- `side <value>`: required if `.side_set N` (non-opt); optional if `.side_set N opt`; invalid if no `.side_set`
- `[<delay>]`: 0–31 minus SIDESET_COUNT bits. If side_set enabled, delay bits = 5 - SIDESET_COUNT (minus 1 more if opt)
- Keywords and directives are **case-insensitive**; commas optional

### 4.3 Pseudo-instructions
```
nop      ; assembles to: mov y, y
```

### 4.4 Values and Expressions
Integer, hex (0xN), binary (0bN), symbol, label, or parenthesised expression.
Operators: `+`, `-`, `*`, `/`, `<<`, `>>`, `-` (negate), `::` (bit-reverse)

---

## 5. Instruction Set

**All instructions are 16 bits. All execute in 1 cycle (unless stalling).**

Delay/side-set field (bits 12:8): MSBs used for side-set data, LSBs for delay cycles.

### 5.1 Instruction Encoding Summary

| Instruction | [15:13] | Notes |
|-------------|---------|-------|
| JMP         | 000     | |
| WAIT        | 001     | |
| IN          | 010     | |
| OUT         | 011     | |
| PUSH        | 100     | bit 7 = 0, bit 6 = 0 |
| MOV (to RX) | 100     | bit 7 = 0, bit 4 = 1, bit 3 = 0 (RP2350) |
| PULL        | 100     | bit 7 = 1, bit 6 = 0 |
| MOV (frm RX)| 100     | bit 7 = 1, bit 4 = 1, bit 3 = 0 (RP2350) |
| MOV         | 101     | |
| IRQ         | 110     | |
| SET         | 111     | |

---

### 5.2 JMP

**Encoding:** `000 [delay/ss] [condition 3b] [address 5b]`

**Operation:** Set PC to Address if Condition true; otherwise no-op. Delay always takes effect.

**Conditions:**
| Code | Mnemonic | Branch if |
|------|----------|-----------|
| 000  | (none)   | Always |
| 001  | !x       | X == 0 |
| 010  | x--      | X != 0 (then decrement X) |
| 011  | !y       | Y == 0 |
| 100  | y--      | Y != 0 (then decrement Y) |
| 101  | x!=y     | X != Y |
| 110  | pin      | JMP_PIN GPIO is high |
| 111  | !osre    | OSR not empty (shift count < PULL_THRESH) |

**Notes:**
- `JMP X--` and `JMP Y--`: X/Y **always** decremented regardless of branch. Branch taken if value was non-zero **before** decrement.
- `JMP PIN`: uses `EXECCTRL_JMP_PIN` (independent of IN mapping)
- `!OSRE`: uses same threshold as autopull
- Address is **absolute** in PIO instruction memory; SDK adjusts for load offset

**Syntax:**
```
jmp (<cond>) <target>
```

---

### 5.3 WAIT

**Encoding:** `001 [delay/ss] [polarity 1b] [source 2b] [index 5b]`

**Operation:** Stall until condition met. Delay begins **after** wait clears.

**Sources:**
| Code | Source   | Description |
|------|----------|-------------|
| 00   | gpio     | Absolute GPIO index (not affected by IN mapping) |
| 01   | pin      | Input pin at `PINCTRL_IN_BASE + Index` (mod 32) |
| 10   | irq      | PIO IRQ flag selected by Index |
| 11   | jmppin   | GPIO at `PINCTRL_JMP_PIN + Index` (0–3, mod 32) — **RP2350 only** |

**WAIT IRQ behaviour:**
- Polarity=1: IRQ flag is **cleared** when condition met
- IRQ index modes (2 MSBs of index field):
  - `00`: direct index into this PIO's IRQ flags (0–7)
  - `01` (PREV): IRQ flag in next-lower-numbered PIO block
  - `10` (REL): index = (index + SM_ID) mod 4; bit 2 unaffected — for SM synchronisation
  - `11` (NEXT): IRQ flag in next-higher-numbered PIO block

> **CAUTION:** `WAIT 1 IRQ x` should not be used with IRQ flags wired to system interrupts (race condition).

**Syntax:**
```
wait <polarity> gpio <gpio_num>
wait <polarity> pin <pin_num>
wait <polarity> irq (prev|next) <irq_num> (rel)
wait <polarity> jmppin (+ <pin_offset>)
```

---

### 5.4 IN

**Encoding:** `010 [delay/ss] [source 3b] [bit_count 5b]`

**Operation:** Shift `bit_count` bits from Source into ISR. ISR shift counter += bit_count. If autopush enabled and threshold reached, push ISR to RX FIFO (stalls if FIFO full).

**Sources:**
| Code | Source |
|------|--------|
| 000  | PINS (from IN pin mapping, LSB = IN_BASE) |
| 001  | X |
| 010  | Y |
| 011  | NULL (all zeroes) |
| 110  | ISR |
| 111  | OSR |

**Notes:**
- Bit count: 1–32; 32 encoded as `00000`
- Shift direction: configured by `SHIFTCTRL_IN_SHIFTDIR` (left or right)
- IN always uses least-significant `bit_count` bits of source
- Input data bit order is NOT dependent on shift direction
- `IN NULL, n` useful for aligning data within ISR

**Syntax:**
```
in <source>, <bit_count>
```

---

### 5.5 OUT

**Encoding:** `011 [delay/ss] [destination 3b] [bit_count 5b]`

**Operation:** Shift `bit_count` bits from OSR to Destination. OSR shift counter += bit_count. If autopull enabled and threshold reached, refill OSR from TX FIFO simultaneously (stalls if FIFO empty).

**Destinations:**
| Code | Destination |
|------|-------------|
| 000  | PINS (OUT pin mapping) |
| 001  | X |
| 010  | Y |
| 011  | NULL (discard) |
| 100  | PINDIRS (OUT pin mapping) |
| 101  | PC (unconditional jump) |
| 110  | ISR (also sets ISR shift counter to bit_count) |
| 111  | EXEC (execute shifted-out data as instruction next cycle) |

**Notes:**
- Bit count: 1–32; 32 encoded as `00000`
- Shift direction: configured by `SHIFTCTRL_OUT_SHIFTDIR`
- `OUT EXEC`: delay on the OUT is ignored; executee may add delay
- `OUT PC`: unconditional jump to shifted-out address

**Syntax:**
```
out <destination>, <bit_count>
```

---

### 5.6 PUSH

**Encoding:** `100 [delay/ss] 0 [IfFull 1b] [Block 1b] 00000`

**Operation:** Push ISR contents to RX FIFO as 32-bit word. Clear ISR to 0.

**Flags:**
- `IfFull`: if 1, no-op unless ISR shift count >= PUSH_THRESH
- `Block`: if 1, stall if RX FIFO full; if 0, continue (data lost, FDEBUG_RXSTALL set)

**Notes:**
- Block is default in pioasm
- Undefined when FJOIN_RX_PUT or FJOIN_RX_GET set — use `MOV rxfifo[y/n], isr` instead

**Syntax:**
```
push (iffull) (block|noblock)
```

---

### 5.7 PULL

**Encoding:** `100 [delay/ss] 1 [IfEmpty 1b] [Block 1b] 00000`

**Operation:** Load 32-bit word from TX FIFO into OSR.

**Flags:**
- `IfEmpty`: if 1, no-op unless OSR shift count >= PULL_THRESH
- `Block`: if 1, stall if TX FIFO empty; if 0, copy X to OSR instead

**Notes:**
- Non-blocking PULL from empty FIFO = `MOV OSR, X`
- When autopull enabled, PULL is a no-op if OSR is full (acts as fence)
- `PULL IFEMPTY`: controlled stall point, same threshold as autopull

**Syntax:**
```
pull (ifempty) (block|noblock)
```

---

### 5.8 MOV (to RX FIFO) — RP2350 only

**Encoding:** `100 [delay/ss] 0 0 0 1 [IdxI 1b] 0 [Index 2b]`

**Operation:** Write ISR to selected RX FIFO entry. Requires `SHIFTCTRL_FJOIN_RX_PUT`. Indexed by Y register (IdxI=0) or immediate Index (IdxI=1). Useful for status registers readable by system.

**Syntax:**
```
mov rxfifo[y], isr
mov rxfifo[<index>], isr    ; index 0-3
```

---

### 5.9 MOV (from RX FIFO) — RP2350 only

**Encoding:** `100 [delay/ss] 1 0 0 1 [IdxI 1b] 0 [Index 2b]`

**Operation:** Read selected RX FIFO entry into OSR. Requires `SHIFTCTRL_FJOIN_RX_GET`. Useful for configuration registers writable by system.

**Syntax:**
```
mov osr, rxfifo[y]
mov osr, rxfifo[<index>]    ; index 0-3
```

---

### 5.10 MOV

**Encoding:** `101 [delay/ss] [destination 3b] [op 2b] [source 3b]`

**Operation:** Copy data from Source to Destination, optionally with transformation.

**Destinations:**
| Code | Destination |
|------|-------------|
| 000  | PINS (OUT pin mapping) |
| 001  | X |
| 010  | Y |
| 011  | PINDIRS (OUT pin mapping) — **RP2350 only** |
| 100  | EXEC (execute as instruction next cycle) |
| 101  | PC (unconditional jump) |
| 110  | ISR (ISR shift counter reset to 0) |
| 111  | OSR (OSR shift counter reset to 0) |

**Sources:**
| Code | Source |
|------|--------|
| 000  | PINS (IN pin mapping, masked by IN_COUNT) |
| 001  | X |
| 010  | Y |
| 011  | NULL (all zeroes) |
| 101  | STATUS (all-ones or all-zeroes based on EXECCTRL_STATUS_SEL) |
| 110  | ISR |
| 111  | OSR |

**Operations:**
| Code | Operation |
|------|-----------|
| 00   | None |
| 01   | Invert (bitwise NOT) |
| 10   | Bit-reverse |
| 11   | Reserved |

**Notes:**
- `MOV EXEC` / `MOV PC`: same as OUT EXEC / OUT PC; MOV delay ignored
- `MOV dst, PINS`: uses IN pin mapping; masked to IN_COUNT width (RP2350)
- `MOV PINDIRS, src`: not supported on PIO v0 (RP2040)
- `MOV x, STATUS`: STATUS = all-ones if condition true (FIFO level or IRQ flag), else all-zeroes
- Useful patterns: `MOV x, ~null` → x = 0xFFFFFFFF; `MOV pins, ~null` → all OUT pins high

**Syntax:**
```
mov <destination>, (! | ~ | ::) <source>
```
`!` and `~` are both bitwise NOT; `::` is bit-reverse.

---

### 5.11 IRQ

**Encoding:** `110 [delay/ss] 0 [Clr 1b] [Wait 1b] [IdxMode 2b] [Index 3b]`

**Operation:** Set or clear IRQ flag.

**Flags:**
- `Clr`: if 1, clear the flag; Wait bit has no effect
- `Wait`: if 1, stall until flag is cleared by external agent (after setting it)
- `IdxMode` (RP2350):
  - `00`: direct index (0–7) into this PIO's flags
  - `01` (PREV): target next-lower-numbered PIO block's flags
  - `10` (REL): (index + SM_ID) mod 4; bit 2 unaffected
  - `11` (NEXT): target next-higher-numbered PIO block's flags

**Notes:**
- All 8 IRQ flags routable to system IRQ lines via IRQ0_INTE / IRQ1_INTE
- Cross-PIO IRQ (PREV/NEXT): no delay penalty; clock dividers must match and be synchronised
- If Wait set, delay begins after wait period
- **`irq next/prev` requires `.pio_version 1` directive** — adafruit_pioasm raises `RuntimeError: irq next requires .pio_version 1` without it
- **Cross-PIO IRQ allocation is deterministic when SM0 fills its PIO block:** if SM0 uses all 32 instruction slots, rp2pio cannot fit SM1 on the same block and places it on the next block. `irq next 0` from SM0 then reliably targets SM1's block. This makes cross-PIO IRQ the correct approach when SM0 is full.

**Syntax:**
```
irq (prev|next) <irq_num> (rel)          ; set, no wait
irq (prev|next) set <irq_num> (rel)      ; set, no wait
irq (prev|next) nowait <irq_num> (rel)   ; set, no wait
irq (prev|next) wait <irq_num> (rel)     ; set, then wait
irq (prev|next) clear <irq_num> (rel)    ; clear
```
`rel`: actual IRQ# = (irq_num[1:0] + sm_num[1:0]) mod 4, combined with irq_num[2]

---

### 5.12 SET

**Encoding:** `111 [delay/ss] [destination 3b] [data 5b]`

**Operation:** Write 5-bit immediate Data to Destination.

**Destinations:**
| Code | Destination |
|------|-------------|
| 000  | PINS (SET pin mapping) |
| 001  | X (5 LSBs set to Data; upper 27 bits cleared) |
| 010  | Y (5 LSBs set to Data; upper 27 bits cleared) |
| 100  | PINDIRS (SET pin mapping) |

**Notes:**
- Data range: 0–31 (5-bit immediate)
- SET and OUT pin mappings are independent (may overlap)
- SET useful for loop counters (max 31 iterations directly; nest X/Y for longer loops)

**Syntax:**
```
set <destination>, <value>    ; value 0-31
```

---

## 6. Functional Details

### 6.1 Side-set
- Drives up to 5 GPIOs **concurrently** with any instruction
- Takes effect on the **first cycle** of the instruction (even if instruction stalls)
- Side-set pin mapping is independent of OUT/SET mappings (may overlap; side-set takes priority)
- Configuration:
  - `PINCTRL_SIDESET_COUNT`: number of MSBs in delay/side-set field used for side-set (0–5)
  - `PINCTRL_SIDESET_BASE`: GPIO mapped to LSB of side-set data
  - `EXECCTRL_SIDE_EN`: if set, MSB of side-set field is an enable (costs 1 extra bit)
  - `EXECCTRL_SIDE_PINDIR`: if set, side-set drives PINDIRS instead of GPIO levels
- With `SIDESET_COUNT = N`, delay bits available = 5 - N (minus 1 more if opt/SIDE_EN)
- With `SIDESET_COUNT = 5`, no delay cycles available

### 6.2 Program Wrapping
After each instruction, PC update logic:
1. If JMP and condition true → PC = target
2. Else if PC == `EXECCTRL_WRAP_TOP` → PC = `EXECCTRL_WRAP_BOTTOM`
3. Else → PC += 1 (wraps 31 → 0)

`.wrap_target` / `.wrap` = 0-cycle unconditional jump (replaces explicit `jmp` at loop end).
WRAP_BOTTOM and WRAP_TOP are **absolute** addresses; adjust for load offset.

### 6.3 FIFO Joining (SHIFTCTRL_FJOIN)
| Mode | TX entries | RX entries |
|------|-----------|-----------|
| Default (txrx) | 4 | 4 |
| tx | 8 | 0 (TXFULL=TXEMPTY=1) |
| rx | 0 (RXFULL=RXEMPTY=1) | 8 |
| txput (RP2350) | 4 | 4 (PUT-mode) |
| txget (RP2350) | 4 | 4 (GET-mode) |
| putget (RP2350) | 0 | 0 (both PUT and GET, SM random access only) |

8-entry FIFO sufficient for 1 word/clock through system DMA without contention.
**Changing FJOIN discards FIFO contents.**

### 6.4 Autopush and Autopull
- Enabled per SM: `SHIFTCTRL_AUTOPUSH` / `SHIFTCTRL_AUTOPULL`
- Threshold: `SHIFTCTRL_PUSH_THRESH` / `SHIFTCTRL_PULL_THRESH` (1–32; 32 encoded as 0)

**Autopush** (on IN reaching threshold):
- If RX FIFO full → stall
- Else → push ISR to RX FIFO, clear ISR, clear ISR counter
- All happens in 1 cycle unless stalling

**Autopull** (on OUT reaching threshold, or between OUTs):
- If TX FIFO empty → stall (at OUT)
- Else → refill OSR from TX FIFO, clear OSR counter
- Can refill simultaneously with shifting out last data
- When autopull enabled, `PULL` becomes a no-op if OSR full (fence behaviour)
- Avoid `MOV` from/to OSR when autopull enabled (nondeterministic)
- **Do not enable autopush when FJOIN_RX_PUT or FJOIN_RX_GET is set**

### 6.5 Clock Dividers
- Format: 16-bit integer + 8-bit fractional (first-order delta-sigma for fractional part)
- Divisor range: 1 to 65536, in steps of 1/256
- Integer divisor = 1: SM runs every system clock cycle (full speed)
- Integer divisor = 0: same as 65536
- Fractional division introduces jitter; for fast async serial use even divisions or multiples of 1 Mbaud
- Multiple SMs can vary independently while running identical programs
- `CLKDIV_RESTART` (in CTRL) resets divider phase — use for synchronising multiple SMs

### 6.6 GPIO Mapping
Four independent pin ranges per SM, each with configurable base and count:

| Range | Instruction | Config registers |
|-------|------------|-----------------|
| OUT   | OUT PINS, OUT PINDIRS | PINCTRL_OUT_BASE, PINCTRL_OUT_COUNT |
| SET   | SET PINS, SET PINDIRS | PINCTRL_SET_BASE, PINCTRL_SET_COUNT |
| IN    | IN PINS, MOV x PINS   | PINCTRL_IN_BASE, SHIFTCTRL_IN_COUNT |
| SIDE  | side-set              | PINCTRL_SIDESET_BASE, PINCTRL_SIDESET_COUNT |

- All four ranges can cover any of the 30 GPIOs on RP2350 and can overlap
- On simultaneous write to same GPIO: **side-set > OUT/SET** (same SM); **higher SM# wins** (different SMs)
- IN: GPIO bus is right-rotated by IN_BASE; LSB = IN_BASE, higher bits = higher GPIOs (wrapping at 31)
- `WAIT GPIO n`: uses **absolute** GPIO number, not affected by IN mapping
- Each GPIO input has a 2-flipflop synchroniser (2-cycle latency); bypass per-GPIO via `INPUT_SYNC_BYPASS`

### 6.7 Forced and EXEC'd Instructions
SM can execute instructions from 3 sources besides instruction memory:
- `MOV EXEC, src` — executes source register contents as instruction (next cycle)
- `OUT EXEC` — executes data shifted from OSR as instruction (next cycle)
- `SMx_INSTR` register — system writes instruction for immediate execution

For both EXEC forms: EXEC instruction takes 1 cycle; executee runs next cycle; delay on EXEC ignored; executee may add delay.

---

## 7. Key Constraints and Gotchas

| Constraint | Detail |
|-----------|--------|
| Instruction memory | 32 instructions max per PIO block (shared by all 4 SMs) |
| `set` immediate | 5-bit: values 0–31 only |
| `in`/`out` bit count | 1–32 (32 encoded as 00000) |
| Delay field | 5 bits minus SIDESET_COUNT bits (minus 1 more if SIDE_EN) |
| X/Y via `set` | Only sets lower 5 bits; upper 27 bits cleared to 0 |
| X/Y via `mov x, ~null` | Sets all 32 bits to 1 (0xFFFFFFFF) |
| `JMP X--`/`JMP Y--` | Always decrements; branches only if pre-decrement value != 0 |
| Nested loops | Max 2 levels (X outer, Y inner); max range 32 per level via `set` |
| IRQ flags | 8 per PIO block; lower 4 to system IRQ on RP2040; all 8 on RP2350 |
| Cross-PIO IRQ | PREV/NEXT modes on RP2350 only; requires synchronised clock dividers |
| `WAIT GPIO n` | Absolute GPIO; ignores IN mapping |
| `WAIT PIN n` | Relative to IN_BASE; n = offset into IN-mapped pins |
| FJOIN + PUSH | Undefined; use PUT instruction instead |
| Autopull + MOV OSR | Nondeterministic |
| Side-set on stall | Side-set fires on first cycle; delay does not begin until stall clears |
| WRAP addresses | Absolute in instruction memory; adjust for load offset |
| `MOV PINDIRS` | RP2350 only (PIO v1) |
| JMPPIN for WAIT | RP2350 only (PIO v1) |
| IN_COUNT masking | RP2350 only; affects `MOV x, PINS` to limit to IN-mapped width |
| rp2pio pin claims | In CircuitPython, each pin can only be claimed by **one SM** — even read-only `first_in_pin` conflicts with another SM's sideset/out pin (`ValueError: Dx in use`). Use IRQ-based sync instead of `wait gpio N` to avoid needing `first_in_pin` on a pin already owned by another SM. |
| Cross-PIO IRQ startup phase | When SM0 signals SM1 via `irq next 0`, **create SM1 before SM0**. SM1 executes its setup instructions (e.g. `set x` / `set y`) in ~ns, then stalls at `wait 1 irq 0`. SM0's very first line-end IRQ becomes SM1's line 1, giving correct phase from frame 1. If SM0 starts first, SM1 starts counting at an arbitrary line offset → IRQ-timed signal fires at the wrong phase → all-black display. |
| `irq next/prev` assembler | Requires `.pio_version 1` directive in the program; adafruit_pioasm raises `RuntimeError` without it. RP2350 (PIO version 1) only — not available on RP2040. |

---

## 8. adafruit_pioasm Notes (CircuitPython)

`adafruit_pioasm.assemble()` accepts standard pioasm syntax. Key differences from SDK pioasm:
- **`.pio_version 1` required** for any RP2350-only features (`irq next/prev`, `wait jmppin`, `mov pindirs`). Omitting it causes `RuntimeError: irq next requires .pio_version 1` at assemble time.
- Assembles to Python `bytes` object (array of 16-bit words)
- `nop` → `mov y, y`
- `wait gpio N` requires the pin to be within the SM's declared pin range in rp2pio (use `first_in_pin` or `wait pin N` with `first_in_pin` instead)
- `wait pin N` is relative to `first_in_pin` (IN_BASE)
- **`irq 0` (direct) is block-local** — only visible to SMs on the same PIO block. If SM0 fills its block (32/32 instructions), rp2pio places SM1 on the next block, which cannot see SM0's direct IRQs. SM1 hangs silently at `wait 1 irq 0` with no error.
- **`irq next 0` (cross-PIO)** targets the next-higher-numbered PIO block. When SM0 fills PIO0, SM1 always lands on PIO1, making `irq next 0` the correct and deterministic choice. Requires `.pio_version 1`.
- **SM creation order determines signal phase:** create the waiting SM (SM1) **before** the driving SM (SM0). SM1 reaches `wait 1 irq 0` within ~2 instructions (~40 ns at 50 MHz). SM0 then starts and its first line-end IRQ is SM1's count start — correct phase from frame 1. Creating SM0 first causes SM1 to start counting at an arbitrary line, shifting the timed signal into the wrong part of the frame.
- `irq rel` — SM-relative IRQ for same-program multi-SM synchronisation

---

*Reference compiled from RP2350 Datasheet DS-2 Chapter 11, pp. 877–961.*
