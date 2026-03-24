# D75Link

A D-STAR reflector client for the Kenwood TH-D75 radio. Connects via Bluetooth to the radio in DV Gateway Mode and bridges voice to D-STAR reflectors (REF, XRF, DCS) over the internet.

The radio handles all audio encoding and decoding with its built-in AMBE chip. d75link bridges the radio's MMDVM serial protocol over Bluetooth to reflector UDP packets, with a retro command center terminal interface.

Copyright (c) 2026 Fabian Lozano, VE4ELB. All rights reserved.

## System Requirements

- Kenwood TH-D75A or TH-D75E radio
- macOS 12 or later, or Linux with BlueZ
- Bluetooth Classic support on the host computer

## Installation

### macOS

Download the latest release and place the `d75link` binary in your PATH:

```bash
chmod +x d75link
sudo mv d75link /usr/local/bin/
```

### Linux

Download the Linux release for your architecture (x86_64 or ARM):

```bash
chmod +x d75link
sudo mv d75link /usr/local/bin/
```

Linux requires the BlueZ Bluetooth library. Install it with:

```bash
# Debian / Ubuntu / Raspberry Pi OS
sudo apt install libbluetooth-dev

# Fedora / RHEL
sudo dnf install bluez-libs-devel
```

## Setting Up Host Files

d75link needs D-STAR host files to resolve reflector names to IP addresses. Download them from Pi-Star:

```bash
mkdir -p ~/.config/d75link/hosts
cd ~/.config/d75link/hosts
curl -o dplus.txt https://www.pistar.uk/downloads/DPlus_Hosts.txt
curl -o dextra.txt https://www.pistar.uk/downloads/DExtra_Hosts.txt
curl -o dcs.txt https://www.pistar.uk/downloads/DCS_Hosts.txt
```

Update these files periodically as reflectors come and go.

## Radio Setup

### Step 1: Enable DV Gateway Mode

On the TH-D75, press MENU and navigate to:

- Menu 650 (DV GW): Set to "Reflector TERM Mode"

This puts the radio in Gateway Mode where it communicates via Bluetooth using the MMDVM serial protocol instead of transmitting over RF.

### Step 2: Enable Bluetooth on the Radio

- Menu 985 (Bluetooth): Set to ON

The radio will become discoverable for pairing.

### Step 3: Pair the Radio

#### macOS

1. Open System Settings (or System Preferences) and go to Bluetooth
2. The radio appears as "TH-D75" in the device list
3. Click "Connect" or "Pair"
4. Accept the pairing on both the radio and the computer
5. Once paired, note the MAC address — on macOS you can find it by Option-clicking the Bluetooth icon in the menu bar, or by running this command in Terminal:

```bash
system_profiler SPBluetoothDataType | grep -A 5 "TH-D75"
```

Look for the "Address" field (e.g., `28:3C:90:77:11:E0`).

#### Linux

1. Make sure the Bluetooth service is running:

```bash
sudo systemctl start bluetooth
```

2. Use `bluetoothctl` to scan, pair, and trust the radio:

```bash
bluetoothctl
[bluetooth]# power on
[bluetooth]# scan on
```

3. Wait for the radio to appear (look for "TH-D75"), then pair and trust it:

```bash
[bluetooth]# pair AA:BB:CC:DD:EE:FF
[bluetooth]# trust AA:BB:CC:DD:EE:FF
[bluetooth]# quit
```

Replace `AA:BB:CC:DD:EE:FF` with the address shown during scanning. This is the Bluetooth MAC address you will use with d75link.

#### Raspberry Pi

Same as Linux above. If Bluetooth is not working, ensure the Bluetooth firmware is installed:

```bash
sudo apt install bluetooth bluez
sudo systemctl enable bluetooth
sudo systemctl start bluetooth
```

Then use `bluetoothctl` as described above.

### Step 4: Find the Bluetooth MAC Address

If you already paired but don't remember the address:

**macOS:**
```bash
system_profiler SPBluetoothDataType | grep -A 5 "TH-D75"
```

**Linux / Raspberry Pi:**
```bash
bluetoothctl devices | grep TH-D75
```

The output shows the MAC address in `AA:BB:CC:DD:EE:FF` format. This is what you pass to d75link with the `--address` flag.

### Notes

- The radio's RF transmitter is disabled in Gateway Mode. All voice goes through Bluetooth to the internet.
- The radio must be in DV Gateway Mode (Menu 650) before connecting. If you pair while in normal mode, the Bluetooth connection will succeed but d75link will not receive MMDVM data.
- On macOS, the first Bluetooth connection after the radio powers on sometimes fails. d75link retries automatically with exponential backoff.
- The TH-D75 uses RFCOMM channel 2 for the data connection. d75link handles this automatically.

## Quick Start

```bash
d75link \
    --callsign YOURCALL \
    --address AA:BB:CC:DD:EE:FF \
    --reflector REF001 \
    --module C \
    --hosts-dir ~/.config/d75link/hosts
```

Replace `YOURCALL` with your amateur radio callsign and `AA:BB:CC:DD:EE:FF` with your radio's Bluetooth MAC address. The terminal interface launches automatically.

## Terminal Interface

d75link displays a retro command center interface with:

- Status bar showing your callsign, radio connection, reflector link status, QSO count, uptime, and clock
- Large callsign display that shows the active station in block letters (green for incoming, red for your transmission)
- Animated signal meter during voice activity with duration counter
- Last-heard station list with live-updating timestamps
- Scrolling activity log
- Link status indicator (LINKED / UNLINKED) with reflector name

### Keyboard Commands

Press these keys at any time:

- L: Link to a reflector. Type the reflector name and module (e.g., `XRF757 A`) then press Enter. Press Escape to cancel.
- U: Unlink from the current reflector.
- E: Display echo test instructions.
- Q: Quit the gateway with a clean shutdown.

## Controlling from the Radio

Set the URCALL (destination) field on the TH-D75 to send commands without touching the computer:

- Link to a reflector: Set URCALL to the reflector name (6 characters), module letter, and L. Example: `REF001CL` links to REF001 module C.
- Unlink: Set URCALL to seven spaces followed by U.
- Echo test: Set URCALL to seven spaces followed by E. Press PTT and speak, then release. The gateway records your voice and plays it back through the radio.

After setting URCALL and pressing PTT briefly, the command is processed and the radio returns to normal operation.

## Echo Test

The echo test verifies your full audio path without needing another station:

1. Set URCALL on the radio to seven spaces followed by the letter E
2. Press PTT, speak for a few seconds, then release
3. Your voice plays back through the radio speaker

If the echo sounds clear, your TX and RX audio paths are working correctly.

## Configuration File

Create a configuration file at `~/.config/d75link/config` to avoid typing options every time:

```
callsign = VE4ELB
address = 28:3C:90:77:11:E0
reflector = REF001
module = C
hosts_dir = ~/.config/d75link/hosts
```

Then run with just:

```bash
d75link
```

Command line options override values from the configuration file.

## Command Line Reference

```
d75link [options]

Required (unless set in config file):
  -c, --callsign CALL    Amateur radio callsign
  -a, --address ADDR     Bluetooth MAC address of TH-D75
  -r, --reflector NAME   Initial reflector (e.g., REF001, XRF757, DCS001)
  -m, --module MOD       Reflector module letter (A-Z)

Optional:
  -f, --config FILE      Config file (default: ~/.config/d75link/config)
  -d, --hosts-dir DIR    Host files directory (default: ./hosts)
  -t, --tui              Force terminal interface on
  -v, --verbose          Enable verbose logging (not recommended during use)
  -h, --help             Show help
```

## Reflector Types

d75link supports all three D-STAR reflector protocols. The correct protocol is selected automatically based on the reflector name:

- REF reflectors (e.g., REF001) use the DPlus protocol on port 20001. Registration at https://regist.dstargateway.org is required to transmit.
- XRF reflectors (e.g., XRF757) use the DExtra protocol on port 30001. No registration required.
- DCS reflectors (e.g., DCS001) use the DCS protocol on port 30051. No registration required.

## Recommended Reflectors

- REF001 module C: Active worldwide with stations from the US, Europe, and Asia.
- XRF757 module A: QuadNet, one of the busiest reflectors worldwide. Good for finding someone to talk to at any time of day.
- XRF103 module A: Canadian D-STAR network.
- DCS001 module A: Popular DCS reflector.

Module B is the standard voice chat module on most reflectors. Module Z on some DCS reflectors provides an echo test service.

## D-STAR Registration

To transmit on REF (DPlus) reflectors, your callsign must be registered in the D-STAR trust system. Register at https://regist.dstargateway.org with your callsign. Registration is free and typically approved within 24 hours.

XRF and DCS reflectors do not require registration.

## Running as a Service

For unattended operation on Linux (e.g., on a Raspberry Pi), d75link can run as a systemd service. When standard input is not a terminal, the interface is disabled and the gateway runs in headless mode with log output to the system journal.

Create a service file at `/etc/systemd/system/d75link.service`:

```ini
[Unit]
Description=D-STAR Gateway
After=bluetooth.target network-online.target

[Service]
ExecStart=/usr/local/bin/d75link --config /etc/d75link/config
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable d75link
sudo systemctl start d75link
```

## Troubleshooting

### Radio does not connect via Bluetooth

Verify that DV Gateway Mode is enabled in Menu 650 (Reflector TERM Mode) and Bluetooth is enabled in Menu 985. The TH-D75 uses RFCOMM channel 2 for the data connection. If the connection fails, try unpairing and re-pairing the radio. On macOS, the first connection attempt after the radio powers on may fail; the gateway retries automatically.

### Can hear others but nobody hears me

Register your callsign at https://regist.dstargateway.org if you are using REF reflectors. Try the echo test to verify your audio path works. If the echo test sounds good, try an XRF or DCS reflector which do not require registration. If you recently registered, allow up to 24 hours for propagation.

### Audio sounds choppy or has dropouts

Do not use the `--verbose` flag during normal operation. Verbose logging adds overhead between voice frames. The gateway paces voice frames at precise 20ms intervals in both directions; additional CPU load from logging can introduce jitter.

### Reflector shows NAK or connection refused

Verify the reflector name is spelled correctly and the module letter is valid (A-Z). Re-download the host files as reflector IP addresses change over time. Some reflectors may be temporarily offline or at capacity.

### Terminal display is corrupted

Ensure your terminal supports UTF-8 and 256 colors. The interface requires a minimum terminal size of 50 columns by 16 rows. Resize your terminal window if the display appears broken. On macOS, Terminal.app and iTerm2 both work well.

## Support

For support, bug reports, or feature requests, contact VE4ELB at ve4elb@gmail.com.
