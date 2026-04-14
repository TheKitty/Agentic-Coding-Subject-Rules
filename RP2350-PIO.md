# RP2350 PIO Behavioral Notes for CircuitPython

Empirically discovered PIO behaviors on RP2350B running CircuitPython 10.0.3.
These supplement the RP2350 datasheet and are specific to the `rp2pio` +
`adafruit_pioasm` CircuitPython stack.

**Sources:**
- RP2350 datasheet (Raspberry Pi, rev 2) — §11.7 PIO registers, §12.6 DMA registers
- `circuitpython/ports/raspberrypi/common-hal/rp2pio/StateMachine.c` — `background_write()` implementation, DMA channel claim, software loop restart, `wrap_target`/`wrap` kwargs at line 426
- `circuitpython/ports/raspberrypi/bindings/rp2pio/__init__.c` — Python binding layer
- PicoVGA `vga.cpp` (Miroslav Nemecek, github.com/Panda381/PicoVGA) — CB+data DMA chain pattern
- Empirical hardware testing on Adafruit Metro RP2350 running CircuitPython 10.0.3

---

## `.wrap_target` is IGNORED by adafruit_pioasm (WRAP_BOTTOM = 0 always)

**Discovery date:** 2026-04-09
**Confirmed by:** 2x frequency NOP test on Metro RP2350

`adafruit_pioasm.assemble()` does **not** extract `.wrap_target` / `.wrap` from
assembly source and pass them to `rp2pio.StateMachine`. The StateMachine
constructor defaults `wrap_target=0, wrap=-1` (which becomes `program_len-1`),
so the hardware wrap covers the entire program. **Every instruction executes
on every wrap cycle** — there is no "run once" preamble from the directive
alone.

**Confirmed by reading rp2pio StateMachine.c (2026-04-14):** The `wrap_target`
and `wrap` are kwargs to the Python `StateMachine()` constructor and ARE
honored via `sm_config_set_wrap()` at line 426. To get wrap behavior, pass
them explicitly:

```python
sm = rp2pio.StateMachine(prog, ..., wrap_target=1, wrap=2)  # wraps 1..2
```

The `.wrap_target` / `.wrap` directives in the `.pio` source are silently
discarded by `adafruit_pioasm.assemble()` — they must be translated to kwargs
by the caller.

### Evidence

A 2-instruction program at 2x frequency was tested:

```python
prog = adafruit_pioasm.assemble("""
.program test
    nop
.wrap_target
    out pins, 2
.wrap
""")
sm = rp2pio.StateMachine(prog, frequency=28_636_360, ...)
```

- If `.wrap_target` were respected: WRAP_BOTTOM=1, only `out pins, 2` loops.
  At 28.6 MHz with a 1-instruction loop, fsc = 28.6M / 4 = 7.16 MHz. **No
  color** (wrong frequency for NTSC).

- If `.wrap_target` is ignored: WRAP_BOTTOM=0, both instructions loop. At
  28.6 MHz with a 2-instruction loop, fsc = 28.6M / 2 / 4 = 3.58 MHz.
  **Color appeared on TV** — confirming 2-instruction loop.

Previous test: adding `nop` before `.wrap_target` at 1x frequency (14.3 MHz)
killed color — consistent with 2-instruction loop halving fsc to 1.79 MHz.

### Implications

1. **No one-time initialization in PIO programs.** Any instruction placed before
   `.wrap_target` executes every cycle, not just once at startup.

2. **`wait 1 irq 0` in a multi-instruction program executes every cycle**, not
   just once. This blocks the SM after every byte if the IRQ isn't continuously
   set.

3. **To use N-instruction programs at the correct output rate**, set SM frequency
   to N times the desired byte rate. The effective byte rate = SM_HZ / N.

4. **IRQ-based sync requires per-byte IRQ firing.** Since `wait` can't be
   "one-shot," any SM with `wait 1 irq 0` + `out pins, 2` needs the IRQ set
   on every cycle by another SM.

### Workaround for correct fsc with 2-instruction programs

Set SM frequency to 2x the desired byte rate:

```python
SM_HZ = 28_636_360   # 2x the 14,318,180 byte rate
# 2-instruction loop: irq 0 + out pins 2 (or wait + out)
# Effective byte rate = 28,636,360 / 2 = 14,318,180
# fsc = 14,318,180 / 4 = 3,579,545 Hz (exact NTSC)
```

---

## `.pio_version 1` required for IRQ instructions

**Discovery date:** 2026-04-08 (previous session)

Without `.pio_version 1` in the assembly source, `irq` instructions silently
fail. Observed: sync-detect SM never fired its LED diagnostic when `.pio_version
1` was omitted. Adding the directive fixed it.

Always include `.pio_version 1` when using any `irq` or `wait 1 irq` instruction.

---

## `irq 0` stalls if flag already set

From the RP2350 PIO spec (not CircuitPython-specific): the `irq 0` instruction
**stalls the SM** if IRQ flag 0 is already set. It does not silently overwrite.
The SM blocks until another SM (or CPU) clears the flag.

This is important for the `irq 0` + `wait 1 irq 0` sync pattern:
- SM_A: `irq 0` (sets flag) + `out pins, 2`
- SM_B: `wait 1 irq 0` (clears flag) + `out pins, 2`

If SM_B is stalled (empty FIFO), it can't clear the flag. SM_A then stalls at
its next `irq 0`. Both SMs deadlock until SM_B's DMA fills its FIFO.

Once both are running, the handshake works: SM_A sets, SM_B clears, both
proceed, every cycle.

---

## `memorymap.AddressRange` — byte index fails, slice works

**Discovery date:** 2026-04-09

On RP2350 CircuitPython, `memorymap.AddressRange` for IO registers:

```python
m = memorymap.AddressRange(start=0x50400000, length=256)

m[0]       # RuntimeError: Unable to access unaligned IO register
m[0:4]     # OK — returns bytearray(4)
bytes(m)   # TypeError: not iterable
```

Use slice access for reading/writing 32-bit registers:

```python
def read32(base_map, offset):
    return int.from_bytes(base_map[offset:offset+4], 'little')

def write32(base_map, offset, val):
    base_map[offset:offset+4] = val.to_bytes(4, 'little')
```

---

## PIO register addresses (RP2350B)

| Peripheral | Base Address |
|------------|-------------|
| DMA        | 0x50000000  |
| PIO0       | 0x50200000  |
| PIO1       | 0x50300000  |
| PIO2       | 0x50400000  |

Atomic access aliases (same as RP2040):
- XOR: base + 0x1000
- SET: base + 0x2000
- CLR: base + 0x3000

### PIO CTRL register (offset 0x000) — confirmed from RP2350 datasheet §11.7

| Bits    | Field                       | Description |
|---------|----------------------------|-------------|
| [3:0]   | SM_ENABLE                   | Enable SMs 0–3 (RW) |
| [7:4]   | SM_RESTART                  | Restart SMs — clears shift counters, ISR, delay counter, waiting-on-IRQ, stalled instr (SC, self-clearing) |
| [11:8]  | CLKDIV_RESTART              | Restart clock dividers (SC). Writing multiple 1 bits restarts multiple dividers **in precise lockstep** |
| [19:16] | PREV_PIO_MASK               | SM mask for lower-numbered neighbor PIO block (RP2350 PIO v1 only) |
| [23:20] | NEXT_PIO_MASK               | SM mask for higher-numbered neighbor PIO block (RP2350 PIO v1 only) |
| [24]    | NEXTPREV_SM_ENABLE          | Apply ENABLE to NEXT/PREV masks — cross-block atomic start in ONE write |
| [25]    | NEXTPREV_SM_DISABLE         | Apply DISABLE to NEXT/PREV masks |
| [26]    | NEXTPREV_CLKDIV_RESTART     | Apply CLKDIV_RESTART to NEXT/PREV masks |

**Key datasheet quote:** "setting/clearing SM_ENABLE does not stop the clock divider from running, so once multiple state machines' clocks are synchronised, it is safe to disable/re-enable a state machine, whilst keeping the clock dividers in sync."

**Cross-block sync (RP2350 PIO v1 feature):** A single write to PIO_N CTRL with NEXT_PIO_MASK or PREV_PIO_MASK plus NEXTPREV_SM_ENABLE atomically starts SMs in neighboring PIO blocks on the same cycle. No MMIO ping-pong needed.

### PIO TX FIFO addresses

`PIO_BASE + 0x010 + SM_NUM * 4`

Example: PIO2 SM1 TXF = 0x50400014

### DMA register layout — confirmed from RP2350 datasheet §12.6.10

Each channel occupies 0x40 bytes starting at DMA_BASE = 0x50000000:

| Offset (within channel) | Name | Notes |
|-------|------|-------|
| `+0x00` | READ_ADDR | — |
| `+0x04` | WRITE_ADDR | — |
| `+0x08` | TRANS_COUNT | — |
| `+0x0C` | CTRL_TRIG | Writing this **triggers start**. Bit 0 = EN, Bit 26 = BUSY (RO), Bits 22:17 = TREQ_SEL, Bits 16:13 = CHAIN_TO, Bits 3:2 = DATA_SIZE |
| `+0x10` | AL1_CTRL | Alias for CTRL — **does NOT trigger** |
| `+0x14` | AL1_READ_ADDR | Alias — does NOT trigger |
| `+0x18` | AL1_WRITE_ADDR | Alias — does NOT trigger |
| `+0x1C` | AL1_TRANS_COUNT_TRIG | **Triggers start** |
| `+0x3C` | AL3_READ_ADDR_TRIG | **Triggers start** — useful for restarting a loop |

Channel N base = DMA_BASE + (0x40 × N).

### DMA global registers (at DMA_BASE)

| Offset | Name | Purpose |
|--------|------|---------|
| 0x430  | INTF0 | Force interrupt |
| 0x450  | **MULTI_CHAN_TRIGGER** | Bits 0–15 trigger channels 0–15 simultaneously — writing ONE word starts multiple channels on the same cycle |
| 0x464  | CHAN_ABORT | Abort channels (bitmap) — required to safely stop a stuck channel |

**⚠ Correction from earlier note:** MULTI_CHAN_TRIGGER is at **0x450**, not 0x430.

### CHAN_ABORT procedure (datasheet §12.6.8.3)

To safely stop active DMA channels (e.g., before resetting READ_ADDR for realignment):

1. Clear `EN` bit AND disable `CHAIN_TO` for all channels to be aborted (per erratum RP2350-E5 — without this, chained aborts can re-trigger themselves)
2. Write bitmap to `CHAN_ABORT`
3. Poll `CHAN_ABORT` until bits clear
4. Verify `BUSY` bit (CTRL.26) is low before restart

### Strategy for fixing DMA byte-level alignment

With all channels stopped via CHAN_ABORT:
1. Stop both PIO SMs (CTRL_CLR of SM_ENABLE bits) — FIFOs stop draining
2. CHAN_ABORT both DMA channels, poll until clear
3. Reset READ_ADDR of both channels via `AL1_READ_ADDR` (non-trigger alias)
4. Reset TRANS_COUNT via `AL1_CTRL` (also non-trigger)
5. Reset PIO via CTRL write: `CLKDIV_RESTART | SM_RESTART` bits set (single atomic write → dividers synchronize)
6. Write `MULTI_CHAN_TRIGGER` with both channel bits set → both DMA channels start on same cycle
7. Write PIO CTRL_SET with SM_ENABLE bits → both SMs start on same cycle

Steps 6 and 7 guarantee byte-level and sample-level alignment. The dividers stay synced across SM_ENABLE toggle (datasheet guarantee above).

---

## `background_write(loop=)` uses SOFTWARE loop, not hardware DMA chain

**Discovery date:** 2026-04-14 (from reading rp2pio StateMachine.c)
**File:** `common_hal_rp2pio_statemachine_background_write()` at StateMachine.c:1166

### What it actually does

`background_write()` claims **exactly ONE** DMA channel per SM via
`dma_claim_unused_channel(false)`. There is no hardware chain. The "looping"
behavior is implemented in software:

1. `dma_channel_configure(channel, ..., buf, len, false)` — configure
2. `dma_start_channel_mask(1u << channel)` — start
3. On channel completion, DMA IRQ fires
4. `rp2pio_statemachine_dma_complete_write()` runs (IRQ context, C code)
5. Sets new `READ_ADDR` and `TRANS_COUNT` via `dma_channel_set_read_addr()` +
   `dma_channel_set_trans_count()` (the second call has trigger=true)

Between step 3 (channel idle) and step 5 (channel triggered again), several
microseconds pass — IRQ latency + function-call overhead + MMIO access. **This
is the source of the per-boot phase variation.**

### Implications for composite color alignment

- With two SMs, each has its own DMA channel and its own independent IRQ
  latency. Each restart gap is different.
- At the end of every buffer loop (every ~3ms for a 262-line NTSC frame),
  the two SMs drift relative to each other.
- The MMIO `CLKDIV_RESTART` fix realigns them once, but drift accumulates at
  every loop boundary.

### Real fix: bypass background_write() and use hardware-chained DMA pairs

**Canonical PicoVGA / Pico SDK pattern (confirmed from PicoVGA vga.cpp):**

Two DMA channels per SM:

**Data channel (to PIO):**
- `WRITE_ADDR` = `&PIO->txf[sm]` (no increment)
- `READ_ADDR` = scanline buffer (increment)
- `TREQ_SEL` = `pio_get_dreq(PIO, sm, true)` — paced by PIO FIFO
- `CHAIN_TO` = CB (control block) channel
- `IRQ_QUIET` + `HIGH_PRIORITY` typical

**Control block (CB) channel:**
- Read from a small "control pairs" array: `u32 ctrl[] = {trans_count, read_addr, 0, 0}` (null-terminated for end-of-chain IRQ, or repeated for loop)
- Write to `&dma_hw->ch[data_channel].al3_transfer_count` (offset **+0x38**)
- `WRITE_RING` with size 3 bits (8-byte window) — this is the clever part
  - Writes alternate between +0x38 (`AL3_TRANS_COUNT`) and +0x3c (`AL3_READ_ADDR_TRIG`) due to the 8-byte ring wrap
  - The second write (to `AL3_READ_ADDR_TRIG`) automatically triggers the data channel to start with the new read pointer and count
- Transfer 2 u32s per activation

**Sequence:**
1. Data channel feeds PIO, paced by DREQ
2. Data channel completes → `CHAIN_TO` fires CB
3. CB writes new TRANS_COUNT then new READ_ADDR_TRIG → data channel starts again
4. CB's read pointer is also ringed (or just a 2-entry loop) so it cycles forever
5. Zero IRQs, zero CPU involvement, gap = 1 bus cycle (deterministic)

For a simple single-buffer loop: `CtrlBuf = [len, buf_addr]` with CB read ring = size 3 → CB re-reads the same 2 words every completion, re-triggering the same buffer indefinitely.

**AL3 alias layout per channel (critical):**

| Offset | Register |
|--------|----------|
| +0x30  | AL3_CTRL |
| +0x34  | AL3_WRITE_ADDR |
| +0x38  | AL3_TRANS_COUNT |
| +0x3C  | AL3_READ_ADDR_TRIG (writing this triggers the channel) |

The 8-byte write ring at +0x38 means the last two words (TRANS_COUNT then READ_ADDR_TRIG) are written in order — the final trigger happens atomically with the last CB transfer.

Alternative: `RING_SIZE` (CTRL_TRIG bits 11:8). With a power-of-2-aligned
buffer, READ_ADDR wraps automatically. But TRANS_COUNT still needs to be set
huge or self-chained, because RING only affects the address, not the counter.

Either approach requires direct MMIO DMA configuration (not
`background_write()`). Channel numbers aren't exposed via public CircuitPython
API — must be claimed independently via memorymap writes before
`background_write()` is called, or `background_write()` must be replaced
entirely.

### No public API to retrieve the DMA channel number

`rp2pio_statemachine_obj_t` stores the channel in a file-static C array
`_sm_dma_plus_one_write[pio_index][sm]`. Not accessible from Python.

**Workaround: scan DMA channels by TREQ_SEL.**

After calling `sm.background_write(loop=buf)`, the active channel will have:
- `CTRL_TRIG.EN` (bit 0) = 1
- `CTRL_TRIG.TREQ_SEL` (bits 22:17) = the PIO TX DREQ number

RP2350 DREQ numbers (from datasheet Table, §12.6):

| DREQ | Source        |
|------|---------------|
| 0    | PIO0 SM0 TX   |
| 1    | PIO0 SM1 TX   |
| 2    | PIO0 SM2 TX   |
| 3    | PIO0 SM3 TX   |
| 4    | PIO0 SM0 RX   |
| 8    | PIO1 SM0 TX   |
| 9    | PIO1 SM1 TX   |
| 16   | PIO2 SM0 TX   |
| 17   | PIO2 SM1 TX   |
| 18   | PIO2 SM2 TX   |
| 19   | PIO2 SM3 TX   |
| 52   | HSTX          |

To find the channel used by PIO2 SM0 (luma) and SM1 (chroma):

```python
DMA_BASE = 0x50000000
dma_map = memorymap.AddressRange(start=DMA_BASE, length=0x480)

def find_dma_channel(dreq_target):
    for ch in range(16):
        ctrl = int.from_bytes(dma_map[(ch*0x40)+0x0c : (ch*0x40)+0x10], 'little')
        en   = ctrl & 0x1
        treq = (ctrl >> 17) & 0x3f
        if en and treq == dreq_target:
            return ch
    return -1

ch_luma   = find_dma_channel(16)  # PIO2 SM0 TX
ch_chroma = find_dma_channel(17)  # PIO2 SM1 TX
```

### `wrap_target` and `wrap` ARE exposed as Python kwargs

Confirmed from `StateMachine.c` line 421–426: both parameters flow through
`sm_config_set_wrap()`. The shared-bindings Python layer passes them as
keyword arguments to the constructor:

```python
sm = rp2pio.StateMachine(prog, ..., wrap_target=1, wrap=2)
```

`adafruit_pioasm.assemble()` does NOT extract these from the `.pio` source
automatically — they must be passed explicitly. The `.wrap_target` / `.wrap`
assembly directives are silently discarded by the assembler.

---

## SM block assignment observed (rp2pio)

With two small programs (2 instructions each), `rp2pio` placed both on PIO2
as SM0 and SM1. PIO0 had SM0+SM2 enabled (likely system use — USB/serial).

This means block-local `irq 0` works between the two video SMs (both on PIO2).

To force SMs onto different blocks, pad the first program to 32 instructions
(fills the block). See `project_pio_cross_block_irq.md` in memory.

---

## Simultaneous SM restart via MMIO

To restart both SMs on the same clock edge (fixing sample-level phase alignment):

```python
import memorymap

PIO2_BASE = 0x50400000
pio2_set = memorymap.AddressRange(start=PIO2_BASE + 0x2000, length=4)
pio2_clr = memorymap.AddressRange(start=PIO2_BASE + 0x3000, length=4)

SM_ENABLE = 0x03    # SM0 + SM1
SM_RESTART = 0x30   # SM0 + SM1 restart bits
CLK_RESTART = 0x0300  # SM0 + SM1 clock divider restart

# Stop both SMs
pio2_clr[0:4] = SM_ENABLE.to_bytes(4, 'little')

# Restart (flush FIFOs, reset PCs) + resync clock dividers
pio2_set[0:4] = (SM_RESTART | CLK_RESTART).to_bytes(4, 'little')

# Re-enable both simultaneously
pio2_set[0:4] = SM_ENABLE.to_bytes(4, 'little')
```

**SM_RESTART does NOT flush the TX/RX FIFOs.** Only the SM's internal state
(PC, OSR, ISR, scratch X/Y, delay counter) is reset. Data already in the FIFO
is preserved and will be consumed as soon as the SM re-enables. This is
important: DMA can continue feeding the FIFO during the restart sequence,
and the SM picks up from wherever DMA left off.

**Wait for FIFOs to fill before CLKDIV_RESTART.** After calling
`sm.background_write(loop=buf)` on both SMs, add a short sleep (~50 ms) before
the CLKDIV_RESTART sequence:

```python
sm_luma.background_write(loop=luma_buf)
sm_chroma.background_write(loop=chroma_buf)
time.sleep(0.05)   # let DMA fill both FIFOs before restarting SMs
_pio2_clr_bits(SM_ENABLE)
_pio2_set_bits(SM_RESTART | CLK_RESTART)
_pio2_set_bits(SM_ENABLE)
```

Without the sleep, SM0 may be re-enabled with an empty FIFO. It stalls on
`out pins, 2`, never executes `irq 0`, and SM1 deadlocks at `wait 1 irq 0`
indefinitely. The sleep ensures both FIFOs have data before the SMs start.

**Limitation:** This fixes sample-phase alignment between SMs, but does NOT
reset DMA read pointers. The byte-level offset (which buffer position each SM
starts reading from) still depends on where DMA was when the restart occurred.

### Simpler single-write variant (datasheet-verified)

Because CLKDIV_RESTART + SM_RESTART are self-clearing (SC), and SM_ENABLE is
independent of the divider, all three can happen in one atomic write:

```python
# Single write: reset state, resync dividers, disable SMs — all atomic
val = (0x03 << 8) | (0x03 << 4)  # CLKDIV_RESTART | SM_RESTART for SM0+SM1
write32(pio2_set, 0, val)
# Then a separate write to enable (leaves dividers synced per datasheet)
write32(pio2_set, 0, 0x03)  # SM_ENABLE for SM0+SM1
```

This is functionally equivalent to the three-write sequence above but shorter.
The datasheet guarantees: "once multiple state machines' clocks are
synchronised, it is safe to disable/re-enable a state machine, whilst keeping
the clock dividers in sync."

### Cross-block atomic start (PIO v1 feature)

If the two SMs ever end up on different PIO blocks (e.g., SM_A on PIO1, SM_B on
PIO2), use the NEXT_PIO_MASK field to start them from one write to PIO1 CTRL:

```python
# From PIO1 CTRL: enable local SM_A (bit 0) AND SM_B on PIO2 (NEXT block)
# NEXT_PIO_MASK = 0b0010 << 20 (SM1 on PIO2), NEXTPREV_SM_ENABLE = 1 << 24
val = (1 << 0) | (0b0010 << 20) | (1 << 24)
write32(pio1_set, 0, val)
```

This is more robust than MMIO ping-pong: both SMs start on the exact same
cycle, no propagation delay.
