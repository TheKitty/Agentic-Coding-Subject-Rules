# Agentic Coding AI Study Sheets

Material to provide Agentic Coding tools specific knowledge for specific tasks

---

## Reference Files

| File | Description |
|------|-------------|
| [`circuitpython-board-interaction-reference.md`](circuitpython-board-interaction-reference.md) | Full reference for interfacing with a CircuitPython board over serial |
| [`circuitpython-serial.md`](circuitpython-serial.md) | Condensed CircuitPython serial monitor cheat sheet for AI memory context |

---

### `circuitpython-board-interaction-reference.md`

> **Full CircuitPython board interaction reference**

Covers everything needed to communicate with a CircuitPython board from a host PC:

- **Port discovery** — auto-detect via Adafruit VID `0x239A`; prompts user if multiple boards found
- **Serial commands** — Ctrl-C (`\x03`) to interrupt, Ctrl-D (`\x04`) for soft reboot
- **UTF-8 / emoji handling** — workaround for Windows `cp1252` crash on CircuitPython's 🐍 boot header
- **Auto-reconnect** — survives board resets that drop the USB serial connection
- **CIRCUITPY drive** — how to locate and copy files to the device
- **Timing note** — serial output must complete before DMA starts in PIO-based projects

---

### `circuitpython-serial.md`

> **Condensed cheat sheet for AI memory / session context**

A compact version of the board interaction reference, formatted for use in AI persistent memory files. Contains ready-to-paste code snippets for:

- Port discovery function
- Full monitor loop with auto-reconnect
- UTF-8 output wrapping
- Key facts (baud rate, drive letter, autoreload behaviour)
