# Agentic Coding AI Study Sheets

Material to provide Agentic Coding tools specific knowledge for specific tasks

---

## Reference Files

| File | Description |
|------|-------------|
| [`RP2350-PIO.md`](RP2350-PIO.md) | Full RP2350 PIO architecture and instruction set reference (Chapter 11 of the datasheet) |
| [`circuitpython-board-interaction-reference.md`](circuitpython-board-interaction-reference.md) | Full reference for interfacing with a CircuitPython board over serial — Windows 11 specific |

---

### `RP2350-PIO.md`

> **RP2350 PIO reference — compiled from Datasheet DS-2, Chapter 11 (pp. 877–961)**

Covers the full PIO subsystem for use when writing or debugging PIO programs:

- **Architecture** — 3 PIO blocks, 4 state machines each, shared 32-instruction memory, FIFOs, IRQ flags
- **RP2350 vs RP2040 changes** — new instructions (`WAIT JMPPIN`, `MOV PINDIRS`), cross-PIO IRQ (PREV/NEXT), `GPIOBASE`, simultaneous SM control
- **Programmer's model** — OSR/ISR, shift counters, stall conditions, FIFO modes, clock dividers, GPIO mapping
- **pioasm directives** — `.pio_version`, `.side_set`, `.wrap`, `.fifo`, `.mov_status`, and all other directives
- **Full instruction set** — JMP, WAIT, IN, OUT, PUSH, PULL, MOV, IRQ, SET with encoding tables and syntax
- **Key constraints and gotchas** — `set` max 31, nested loop limits, side-set during stall, cross-PIO IRQ startup phase, rp2pio pin conflicts
- **adafruit_pioasm notes** — CircuitPython-specific differences, `irq next` requirements, SM creation order for correct phase

---

### `circuitpython-board-interaction-reference.md`

> **Full CircuitPython board interaction reference — Windows 11 specific, not tested on Windows 10**

Covers everything needed to communicate with a CircuitPython board from a host PC:

- **Port discovery** — auto-detect via Adafruit VID `0x239A`; prompts user if multiple boards found
- **Serial commands** — Ctrl-C (`\x03`) to interrupt, Ctrl-D (`\x04`) for soft reboot
- **UTF-8 / emoji handling** — workaround for Windows `cp1252` crash on CircuitPython's 🐍 boot header
- **Auto-reconnect** — survives board resets that drop the USB serial connection
- **CIRCUITPY drive** — how to locate and copy files to the device
- **Timing note** — serial output must complete before DMA starts in PIO-based projects
