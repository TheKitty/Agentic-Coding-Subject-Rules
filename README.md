# Agentic Coding AI Study Sheets

Material to provide Agentic Coding tools specific knowledge for specific tasks

---

## Reference Files

| File | Description |
|------|-------------|
| [`circuitpython-board-interaction-reference.md`](circuitpython-board-interaction-reference.md) | Full reference for interfacing with a CircuitPython board over serial — Windows 11 specific |

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
