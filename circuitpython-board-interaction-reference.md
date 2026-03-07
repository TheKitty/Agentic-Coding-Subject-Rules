# CircuitPython Board Interaction Reference

## Serial Port Discovery

Do not hardcode the COM port. Use `serial.tools.list_ports` to find CircuitPython devices
by Adafruit VID (0x239A) or description/manufacturer strings.

```python
from serial.tools import list_ports

def find_circuitpython_port(out):
    """Return the COM port name for a CircuitPython REPL, prompting if multiple found.

        Searches by Adafruit VID (0x239A) — the primary and most reliable check.
    Also checks 'CircuitPython' in description or 'Adafruit' in manufacturer
    as fallbacks, but Windows may report manufacturer as 'Microsoft' (generic
    CDC driver), so those secondary checks may not match. VID always does.
    If none found, falls back to prompting the user for a port name.
    If multiple found, prompts user to select.
    """
    candidates = []
    for p in list_ports.comports():
        if (p.vid == 0x239A or
                (p.description and 'CircuitPython' in p.description) or
                (p.manufacturer and 'Adafruit' in p.manufacturer)):
            candidates.append(p)

    if len(candidates) == 1:
        out.write(f"Found CircuitPython port: {candidates[0].device} "
                  f"({candidates[0].description})\n"); out.flush()
        return candidates[0].device

    if len(candidates) > 1:
        out.write("Multiple CircuitPython ports found:\n"); out.flush()
        for i, p in enumerate(candidates):
            out.write(f"  {i + 1}: {p.device} — {p.description}\n"); out.flush()
        while True:
            out.write("Select port number: "); out.flush()
            choice = input().strip()
            if choice.isdigit() and 1 <= int(choice) <= len(candidates):
                return candidates[int(choice) - 1].device
            out.write("Invalid selection.\n"); out.flush()

    # Nothing found via VID/description — ask user
    out.write("No CircuitPython port detected automatically.\n"); out.flush()
    out.write("Enter COM port (e.g. COM16): "); out.flush()
    return input().strip()
```

**Known port (this machine):** COM16 — but always discover rather than hardcode.

## Baud Rate
115200

## Sending Commands

Raw byte writes — no Unicode involved:
```python
port.write(b'\x03')   # Ctrl-C: interrupt running code
port.write(b'\x04')   # Ctrl-D: soft reboot
```
Send Ctrl-C first (0.3s delay), then Ctrl-D for clean reboot.

## Reading Serial Output

CircuitPython prints a snake emoji (🐍) in its boot header.
Valid UTF-8 but crashes the Windows console (cp1252).

### Working monitor pattern (full, with port discovery and auto-reconnect):
```python
import serial, sys, time, io
from serial.tools import list_ports

out = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8', errors='replace', line_buffering=True)

com_port = find_circuitpython_port(out)   # see above

while True:
    try:
        port = serial.Serial(com_port, 115200, timeout=1)
        out.write(f"Connected to {com_port}.\n"); out.flush()
        port.write(b'\x03'); time.sleep(0.3); port.reset_input_buffer()
        port.write(b'\x04')   # soft reboot
        while True:
            try:
                line = port.readline()
                if line:
                    out.write(line.decode('utf-8', errors='replace').rstrip() + '\n')
                    out.flush()
            except serial.SerialException:
                out.write("--- Disconnected, reconnecting... ---\n"); out.flush()
                break
        try:
            port.close()
        except Exception:
            pass
        time.sleep(2)
    except serial.SerialException as e:
        out.write(f"Cannot open {com_port}: {e}\n"); out.flush()
        time.sleep(2)
```

Key points:
- `io.TextIOWrapper` on `sys.stdout.buffer` with `encoding='utf-8'` — avoids cp1252 crash
- `decode('utf-8', errors='replace')` on incoming bytes — handles emoji safely
- Do NOT use `print()` directly to the Windows console for serial output
- Board resets cause `SerialException: ClearCommError failed (PermissionError 13)` — normal, reconnect loop handles it

## autoreload

If `supervisor.runtime.autoreload = False` is set in code.py (as in the VGA project),
copying a new file to CIRCUITPY will NOT trigger a reload. Must send Ctrl-D manually.

## CIRCUITPY Drive

Find with PowerShell: `Get-Volume | Where-Object FileSystemLabel -eq 'CIRCUITPY'`
Current drive letter: **F:** — but verify each session.
Copy code: `cp "source.py" "/f/code.py"` (adjust drive letter as needed)

## Running as Background Task (Claude Code)

Run the monitor as a background Bash task, then poll with TaskOutput:
- Output goes to a temp file; TaskOutput reads it
- The monitor survives indefinitely; use TaskStop to kill it

## Timing Note (VGA project)

All startup prints must complete BEFORE `sm.background_write()` in the VGA code.
Any serial output after DMA start shifts DMA/VSYNC phase alignment.
