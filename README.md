# Kyocera ECOSYS PA4500x — CUPS Setup Guide

How to add a Kyocera ECOSYS PA4500x as a network printer in CUPS on Linux, with a PPD that actually renders PostScript instead of printing the source code as plain text.

This guide is written around the **PA4500x** but the same approach works for other Kyocera ECOSYS / FS / TASKalfa devices that speak KPDL.

## Tested on

- **Printer:** Kyocera ECOSYS PA4500x (B&W LED, 45 ppm, network model)
- **Firmware:** `2025.10.29 C0T_S000.005.114`
- **Reported languages** (from `@PJL INFO CONFIG`): `LINEP`, `IBMPRO`, `ESCP`, `PCL`, `PCLXL`, `KPDL`
- **Print server:** Ubuntu 22.04.5 LTS, CUPS 2.4.1
- **Transport:** raw socket on TCP/9100 (`socket://<ip>:9100`)

## Why this guide exists

If you add a Kyocera ECOSYS to CUPS using the **Generic PostScript** PPD (`drv:///sample.drv/generic.ppd`) over a raw socket (`socket://<ip>:9100`), the printer happily eats the job and the page counter increments — but what comes out of the tray is the literal PostScript source:

```
%!PS-Adobe-3.0
%%HiResBoundingBox: 0 0 596.00 842.00
%%Creator: GPL Ghostscript
```

The printer received bytes, didn't recognize them as a known page description language, and fell back to printing them as text.

The fix is to ship a PPD that prepends a PJL header telling the printer "the next stream is KPDL (Kyocera PostScript)". CUPS injects that header for you if the PPD declares the right `*JCL*` directives. Generic PostScript PPD doesn't.

## Requirements

- A Linux host (tested on Ubuntu 22.04) with CUPS running
- Network reachability to the printer on TCP/9100 (raw socket) or TCP/631 (IPP) or TCP/515 (LPD)
- Root or sudo access on the print server

## Setup

### 1. Install the PPD

Copy [`kyocera-ecosys-kpdl.ppd`](kyocera-ecosys-kpdl.ppd) into the CUPS model directory:

```bash
sudo cp kyocera-ecosys-kpdl.ppd /usr/share/cups/model/
sudo systemctl restart cups
```

Verify CUPS picked it up:

```bash
lpinfo -m | grep -i kyocera-ecosys
# expected: kyocera-ecosys-kpdl.ppd Kyocera ECOSYS PA4500x KPDL
```

### 2. Add the printer

Replace `PRINTER_IP` with the device's IP address and pick any queue name you like (here: `kyocera_pa4500x`):

```bash
sudo lpadmin \
  -p kyocera_pa4500x \
  -E \
  -v socket://PRINTER_IP:9100 \
  -P /usr/share/cups/model/kyocera-ecosys-kpdl.ppd \
  -D "Kyocera ECOSYS PA4500x" \
  -L "office"
```

Confirm:

```bash
lpstat -v kyocera_pa4500x
# device for kyocera_pa4500x: socket://PRINTER_IP:9100
lpstat -p kyocera_pa4500x
# printer kyocera_pa4500x is idle.
```

### 3. Print a test page

```bash
lp -d kyocera_pa4500x /usr/share/cups/data/testprint
```

You should get the standard CUPS test page (CUPS logo, calibration bars, color/grayscale strips), **not** raw PostScript source.

## What the PPD does

The relevant lines:

```ppd
*Protocols: PJL
*JCLBegin: "<1B>%-12345X@PJL JOB<0A>"
*JCLToPSInterpreter: "@PJL ENTER LANGUAGE = KPDL<0A>"
*JCLEnd: "<1B>%-12345X@PJL EOJ<0A><1B>%-12345X"
```

CUPS uses these to wrap each PostScript job with the Universal Exit Language (UEL) escape sequence and a `@PJL ENTER LANGUAGE = KPDL` directive, so the printer's firmware switches into PostScript-interpreting mode before the `%!PS-Adobe-3.0` payload arrives.

For PCL-only printers, swap `KPDL` for `PCL` or `PCLXL`. The languages a given device supports are listed in the response to `@PJL INFO CONFIG` — see the diagnostics section below.

## Diagnostics

### Query the printer over PJL

PA4500x and most Kyocera devices answer PJL queries on the same TCP/9100 port used for printing. This works even when the device's web UI is locked down:

```bash
python3 - <<'PY'
import socket, time
s = socket.socket(); s.settimeout(5)
s.connect(("PRINTER_IP", 9100))
s.sendall(b"\x1b%-12345X@PJL INFO ID\r\n@PJL INFO STATUS\r\n@PJL INFO PAGECOUNT\r\n@PJL INFO CONFIG\r\n\x1b%-12345X\r\n")
time.sleep(2)
buf = b""
try:
    while True:
        chunk = s.recv(4096)
        if not chunk: break
        buf += chunk
        if len(buf) > 16384: break
except socket.timeout: pass
print(buf.decode("utf-8", "ignore"))
s.close()
PY
```

Useful pieces of the response:

- `@PJL INFO ID` — exact model string (e.g. `"ECOSYS PA4500x"`)
- `@PJL INFO STATUS` — `ONLINE=TRUE`, `DISPLAY="..."` (current LCD message), error codes
- `@PJL INFO PAGECOUNT` — lifetime page counter; **the most reliable way to verify a job actually rendered** (CUPS marking a job "completed" only means CUPS handed bytes to the backend, not that the printer drew anything)
- `@PJL INFO CONFIG` — supported `LANGUAGES` list, paper trays, capacities

### Capture what CUPS is actually sending

If you suspect the JCL wrapper isn't being applied, redirect CUPS output to a file and inspect the bytes:

```bash
# enable file devices
echo "FileDevice Yes" | sudo tee -a /etc/cups/cups-files.conf
sudo systemctl restart cups

# create a capture queue using the same PPD
sudo lpadmin -p _capture_ -E \
  -v file:///tmp/capture.bin \
  -P /usr/share/cups/model/kyocera-ecosys-kpdl.ppd
sudo cupsenable _capture_
sudo cupsaccept _capture_

# send a job
lp -d _capture_ /usr/share/cups/data/testprint
sleep 3

# look at the first ~200 bytes — you should see UEL + @PJL ENTER LANGUAGE = KPDL
xxd /tmp/capture.bin | head -15

# clean up
sudo lpadmin -x _capture_
sudo sed -i '/^FileDevice Yes$/d' /etc/cups/cups-files.conf
sudo systemctl restart cups
```

Expected start of the capture:

```
1b25 2d31 3233 3435 5840 504a 4c0a 4050   .%-12345X@PJL.@P
4a4c 204a 4f42 ...                         JL JOB ...
... @PJL ENTER LANGUAGE = KPDL ...
%!PS-Adobe-3.0 ...
```

If you see `%!PS-Adobe-3.0` at byte 0 with no PJL preamble, the PPD's JCL directives aren't being applied — double-check that the queue is actually using this PPD (`grep -E '^\*ModelName|^\*JCL' /etc/cups/ppd/<queue>.ppd`).

### Symptom → cause cheatsheet

| Symptom | Likely cause |
|---|---|
| Printer prints raw `%!PS-Adobe-3.0...` text on paper | No `@PJL ENTER LANGUAGE` wrapper — wrong PPD (Generic PostScript) |
| CUPS job stuck in queue forever, "Connected to printer" | Driverless / IPP-Everywhere PPD generated against a printer whose IPP service doesn't return attributes — switch to `socket://...:9100` |
| Job marked completed in CUPS but pagecount doesn't increment | Printer in deep sleep — wake it with a PJL status query first, or just retry; if pagecount still flat, the data was silently dropped (wrong language) |
| `Filter failed` in `lpstat -l` | PPD's `*cupsFilter` chain references a binary that isn't installed, or the PPD itself is malformed |
| Printer LCD shows "Load paper into cassette 1" and nothing prints | Physical: tray empty. CUPS jobs queue up and flush all at once when paper is reloaded |

## Notes on transport

- `socket://<ip>:9100` (HP JetDirect, raw TCP) — simplest, what this guide uses. The printer auto-detects the language but only once it sees a PJL header.
- `lpd://<ip>/lp` — also works on this device. Slightly more chatty than raw socket, no real advantage here.
- `ipp://<ip>/ipp/print` — the device accepts IPP connections but on the firmware version tested (2025.10.29) didn't return printer attributes to `Get-Printer-Attributes`, which prevents IPP-Everywhere driverless setup from generating a usable PPD. Use socket+this PPD instead.

## License

The PPD file is provided as-is, no warranty. Adapt freely.
