# Phomemo Tools Documentation

This documentation covers all tools, CUPS components, and configuration options available in the phomemo-tools package.

## Table of Contents

1. [Overview](#overview)
2. [Supported Printers](#supported-printers)
3. [Command-Line Tools](#command-line-tools)
4. [CUPS Integration](#cups-integration)
5. [Protocol Reference](#protocol-reference)
6. [macOS CUPS Support (Apple Silicon)](#macos-cups-support-apple-silicon)
7. [Troubleshooting](#troubleshooting)

---

## Overview

Phomemo-tools provides Linux/CUPS printing support for Phomemo thermal label printers. The package includes:

- **Command-line tools** for direct printing via Bluetooth or USB
- **CUPS backend** for printer discovery and connection management
- **CUPS filters** for converting raster images to printer protocol
- **PPD drivers** for printer capability definitions

All protocol information has been reverse-engineered from Android Bluetooth packet captures.

---

## Supported Printers

| Model | Resolution | Paper Width | Connection | Filter |
|-------|-----------|-------------|------------|--------|
| M02 | 203 dpi | 50mm (384 dots) | BT/USB | rastertopm02_t02 |
| M02 Pro | 300 dpi | 50mm | BT/USB | rastertopm02_t02 |
| T02 | 203 dpi | 50mm (384 dots) | BT/USB | rastertopm02_t02 |
| M110 | 203 dpi | 20-50mm (344 dots max) | BT/USB | rastertopm110 |
| M120 | 203 dpi | 20-50mm | BT/USB | rastertopm110 |
| M220 | 203 dpi | 20-70mm | BT/USB | rastertopm110 |
| M421 | 203 dpi | 40-70mm | BT/USB | rastertopm110 |
| D30 | 203 dpi | 30-40mm | BT/USB | rastertopd30 |

---

## Command-Line Tools

### phomemo-filter.py

**Location:** `tools/phomemo-filter.py`

**Purpose:** Converts images to M02-compatible printer protocol and outputs to stdout.

**Usage:**
```bash
# Print via Bluetooth (requires rfcomm connection)
tools/phomemo-filter.py image.png > /dev/rfcomm0

# Print via USB
tools/phomemo-filter.py image.png > /dev/usb/lp0

# Disable auto-rotation
tools/phomemo-filter.py --no-rotate image.png > /dev/rfcomm0
```

**Options:**
| Option | Description |
|--------|-------------|
| `--no-rotate` | Disable automatic rotation of landscape images |
| `file` | Path to the image file (PNG, JPG, etc.) |

**How it works:**
1. Opens and loads the image using PIL/Pillow
2. Auto-rotates landscape images (width > height) unless `--no-rotate` is specified
3. Resizes to 384 dots width (M02 native resolution)
4. Converts to 1-bit black & white with dithering
5. Outputs ESC/POS protocol header
6. Sends image data in blocks of up to 256 lines each
7. Outputs ESC/POS protocol footer

**Limitations:**
- Currently only fully supports M02/T02 printers
- Output goes to stdout (redirect to device or pipe)

---

### format-checker.py

**Location:** `tools/format-checker.py`

**Purpose:** Validates and reconstructs images from M02 printer protocol data. Useful for debugging and protocol analysis.

**Usage:**
```bash
# Validate protocol output from phomemo-filter
tools/phomemo-filter.py image.png | tools/format-checker.py

# Check a saved protocol file
cat protocol_dump.bin | tools/format-checker.py
```

**Output:**
- Prints block information to stdout
- Saves reconstructed image as `image-checker.png`
- Opens the reconstructed image for visual verification

**How it works:**
1. Reads binary protocol from stdin
2. Validates header bytes (ESC @, ESC a, etc.)
3. Parses image blocks and extracts pixel data
4. Reconstructs the original image
5. Validates footer sequence

---

## CUPS Integration

### Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Application   │────▶│   CUPS Daemon    │────▶│  CUPS Backend   │
│ (lp, lpr, GUI)  │     │   (cupsd)        │     │  (phomemo.py)   │
└─────────────────┘     └────────┬─────────┘     └────────┬────────┘
                                 │                        │
                        ┌────────▼─────────┐              │
                        │   CUPS Filter    │              │
                        │ (rastertopm*.py) │              │
                        └────────┬─────────┘              │
                                 │                        │
                        ┌────────▼─────────┐     ┌────────▼────────┐
                        │  Printer Data    │────▶│ Printer Device  │
                        │  (binary ESC/POS)│     │  (BT/USB)       │
                        └──────────────────┘     └─────────────────┘
```

### CUPS Backend

**File:** `cups/backend/phomemo.py`

**Purpose:** Handles printer discovery and connection for both Bluetooth and USB printers.

**Device Discovery:**

When run without arguments, the backend scans for available printers:

```bash
# Run discovery manually
/usr/lib/cups/backend/phomemo
```

**Bluetooth Discovery:**
- Uses D-Bus to query BlueZ for paired devices
- Looks for devices with names starting with "Mr.in_" or exactly "T02"
- Generates `phomemo://` URIs from MAC addresses

**USB Discovery:**
- Uses PyUSB to find printers with vendor ID 0x0493 (MAG Technology)
- Supports product IDs:
  - 0xb002: M02
  - 0x8760: M110
- Generates standard `usb://` URIs

**Connection Handling:**

When invoked by CUPS with a device URI:
- Parses the `DEVICE_URI` environment variable
- For Bluetooth: Creates RFCOMM socket connection to the printer
- Reads print data from stdin and sends to printer
- Waits for printer acknowledgment before closing

---

### CUPS Filters

All filters convert CUPS Raster format (RaS3) to printer-specific ESC/POS protocol.

#### rastertopm02_t02.py

**Location:** `cups/filter/rastertopm02_t02.py`

**Supported Printers:** M02, M02 Pro, T02

**Features:**
- Reads CUPS Raster 3 format (1796-byte header + image data)
- Converts grayscale to 1-bit black & white
- Sends images in 255-line blocks (protocol maximum)
- Supports configurable feed lines via PPD option

**Protocol Output:**
```
Header:  ESC @ (init) + ESC a (justify) + 0x1f 0x11 0x02 0x04
Blocks:  GS v 0 (raster) + mode + width + height + image data
Footer:  ESC d (feed) + 0x1f 0x11 sequences
```

---

#### rastertopm110.py

**Location:** `cups/filter/rastertopm110.py`

**Supported Printers:** M110, M120, M220, M421

**Features:**
- Speed control (1-5, where 5 is fastest)
- Density control (1-15)
- Media type selection:
  - `0x0a` - Label With Gaps
  - `0x0b` - Continuous
  - `0x26` - Label With Marks

**Protocol Output:**
```
Header:  ESC N 0x0d (speed) + ESC N 0x04 (density) + 0x1f 0x11 (media type)
Image:   GS v 0 + mode + width + height + image data
Footer:  0x1f 0xf0 sequences
```

---

#### rastertopd30.py

**Location:** `cups/filter/rastertopd30.py`

**Supported Printers:** D30

**Features:**
- Rotates image 90 degrees (D30 prints sideways)
- Single-block transmission (no chunking)
- Manual feed line padding

**Protocol Output:**
```
Header:  0x1f 0x11 0x24 0x00 + ESC @ (init)
Image:   GS v 0 + mode + width + height + rotated image data
Footer:  Null-byte padding for feed lines
```

---

### PPD Driver Files

**Location:** `cups/drv/`

PPD (PostScript Printer Description) files define printer capabilities. Source `.drv` files are compiled to `.ppd.gz` using `ppdc`.

| Driver File | Models | Resolution |
|-------------|--------|------------|
| phomemo-m02_t02.drv | M02, T02 | 203 dpi |
| phomemo-m02pro.drv | M02 Pro | 300 dpi |
| phomemo-m110.drv | M110, M120 | 203 dpi |
| phomemo-m220.drv | M220 | 203 dpi |
| phomemo-m421.drv | M421 | 203 dpi |
| phomemo-d30.drv | D30 | 203 dpi |

**Common PPD Options:**
- **MediaSize:** Paper dimensions (e.g., w50h70 = 50mm x 70mm)
- **FeedLines:** Additional paper feed after printing
- **MediaType:** (M110/M220/M421) Gap, continuous, or mark-based labels

---

## Protocol Reference

### M02/T02 Protocol (ESC/POS)

#### Header Sequence
```
1B 40        ESC @     Initialize printer
1B 61 01     ESC a 1   Center justification
1F 11 02 04            Phomemo-specific init
```

#### Image Block (max 256 lines per block)
```
1D 76 30     GS v 0    Print raster bit image
00                     Mode: 0=normal, 1=double-width, 2=double-height, 3=quad
30 00                  Bytes per line (48 = 384 dots / 8), little-endian
FF 00                  Number of lines (max 255), little-endian
[image data]           48 bytes per line, MSB first
```

#### Footer Sequence
```
1B 64 02     ESC d 2   Print and feed 2 lines
1B 64 02     ESC d 2   Print and feed 2 lines
1F 11 08               Phomemo-specific
1F 11 0E               Phomemo-specific
1F 11 07               Phomemo-specific
1F 11 09               Phomemo-specific
```

#### Special Notes
- Byte `0x0A` is reserved (interpreted as LineFeed)
- Replace `0x0A` with `0x14` in image data

---

### M110/M120/M220/M421 Protocol

#### Header Sequence
```
1B 4E 0D 05  ESC N     Print speed (01-05)
1B 4E 04 0F  ESC N     Print density (01-0F)
1F 11 0A               Media type (0A=gaps, 0B=continuous, 26=marks)
```

#### Image Block
```
1D 76 30     GS v 0    Print raster bit image
00                     Mode
2B 00                  Bytes per line (43 = 344 dots / 8)
F0 00                  Number of lines
[image data]           Variable bytes per line
```

#### Footer Sequence
```
1F F0 05 00            End print
1F F0 03 00            Finalize
```

---

## macOS CUPS Support (Apple Silicon)

### Current Status

Phomemo-tools now includes **full macOS support** including Apple Silicon (M1/M2/M3). The implementation uses a platform abstraction layer that automatically detects the operating system and uses the appropriate backend:

| Feature | Linux | macOS | Notes |
|---------|-------|-------|-------|
| USB Printing | Yes | Yes | Automatic device detection |
| USB Discovery | Yes | Yes | Via PyUSB |
| Bluetooth Discovery | Yes | Yes | Via IOBluetooth |
| Bluetooth Printing | Yes | Yes | Via IOBluetooth RFCOMM |
| CUPS Backend | Yes | Yes | Cross-platform |
| CUPS Filters | Yes | Yes | Pure Python/PIL |
| Auto-Discovery | Yes | Yes | Full support |

### Architecture

The backend uses a modular architecture with platform-specific implementations:

```
cups/backend/
├── phomemo.py          # Main CUPS backend (platform-agnostic)
├── platform.py         # Platform detection utilities
├── bluetooth/
│   ├── __init__.py     # Platform dispatcher
│   ├── base.py         # Abstract interface
│   ├── linux.py        # BlueZ/D-Bus implementation
│   └── darwin.py       # IOBluetooth implementation
└── usb/
    ├── __init__.py     # Platform dispatcher
    ├── base.py         # Abstract interface
    ├── linux.py        # Linux USB paths
    └── darwin.py       # macOS USB paths (/dev/cu.*)
```

### Quick Installation (macOS)

Use the provided installation script for automatic setup:

```bash
# Clone repository
git clone https://github.com/vivier/phomemo-tools.git
cd phomemo-tools

# Run installer
./scripts/install-macos.sh
```

The script will:
1. Install Homebrew (if needed)
2. Install Python dependencies (pillow, pyusb, pyobjc-framework-IOBluetooth)
3. Build PPD files
4. Install to `/usr/local/lib/cups/` (SIP-friendly)
5. Configure CUPS
6. Restart the CUPS service

### Manual Installation (macOS)

#### Prerequisites

```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install system dependencies
brew install python3 libusb

# Install Python dependencies
pip3 install -r requirements-macos.txt
```

#### Build and Install

```bash
cd cups
make ppds
sudo make install
```

The Makefile automatically detects macOS and uses appropriate paths:
- Backend: `/usr/local/lib/cups/backend/`
- Filters: `/usr/local/lib/cups/filter/`
- PPDs: `/Library/Printers/PPDs/Contents/Resources/Phomemo/`

#### Configure CUPS

Add to `/etc/cups/cups-files.conf`:
```
ServerBin /usr/local/lib/cups
```

Restart CUPS:
```bash
sudo launchctl stop org.cups.cupsd
sudo launchctl start org.cups.cupsd
```

### Adding a Printer

#### Via System Settings (GUI)

1. Open **System Settings → Printers & Scanners**
2. Click **+** to add a printer
3. Your Phomemo printer should appear (if paired via Bluetooth or connected via USB)
4. Select the appropriate PPD driver

#### Via Command Line

**Bluetooth:**
```bash
# Find your printer's MAC address
/usr/local/lib/cups/backend/phomemo

# Add printer
sudo lpadmin -p PhomemoM02 -E \
    -v phomemo://AABBCCDDEEFF \
    -P /Library/Printers/PPDs/Contents/Resources/Phomemo/Phomemo-M02.ppd.gz
```

**USB:**
```bash
# Find USB device
ls /dev/cu.usbmodem*

# Add printer
sudo lpadmin -p PhomemoM02 -E \
    -v serial:/dev/cu.usbmodem14201 \
    -P /Library/Printers/PPDs/Contents/Resources/Phomemo/Phomemo-M02.ppd.gz
```

### Python Dependencies

**requirements-macos.txt:**
```
pillow>=9.0.0
pyusb>=1.2.0
pyobjc-core>=9.0
pyobjc-framework-Cocoa>=9.0
pyobjc-framework-IOBluetooth>=9.0
pyobjc-framework-CoreBluetooth>=9.0
```

### IOBluetooth Implementation Details

The macOS Bluetooth implementation uses PyObjC to interface with Apple's IOBluetooth framework:

**Device Discovery:**
```python
from IOBluetooth import IOBluetoothDevice

# Get all paired devices
paired = IOBluetoothDevice.pairedDevices()
for device in paired:
    name = device.name()
    address = device.addressString()
```

**RFCOMM Connection:**
```python
device = IOBluetoothDevice.deviceWithAddressString_("AA:BB:CC:DD:EE:FF")
result, channel = device.openRFCOMMChannelSync_withChannelID_delegate_(
    None, 1, delegate
)
channel.writeSync_length_(data, len(data))
```

The implementation handles the callback-based nature of IOBluetooth by using NSRunLoop for synchronization.

### Troubleshooting (macOS)

#### Bluetooth Not Working

1. Ensure the printer is paired in System Settings → Bluetooth
2. Check PyObjC is installed: `pip3 show pyobjc-framework-IOBluetooth`
3. Test discovery: `/usr/local/lib/cups/backend/phomemo`

#### USB Not Detected

1. Check device is connected: `ls /dev/cu.usbmodem*`
2. Ensure libusb is installed: `brew install libusb`
3. Check PyUSB: `python3 -c "import usb.core; print('OK')"`

#### CUPS Filter Failure

1. Check Python dependencies: `pip3 install pillow`
2. Verify filter is executable: `ls -la /usr/local/lib/cups/filter/`
3. Check CUPS logs: `tail -f /var/log/cups/error_log`

#### SIP Blocking Installation

Use `/usr/local/lib/cups/` instead of `/usr/libexec/cups/` and configure CUPS with:
```
ServerBin /usr/local/lib/cups
```

---

## Troubleshooting

### Filter Failure Error

**Symptom:** CUPS shows "Filter Failure" when printing

**Solution:** Ensure Python dependencies are installed:
```bash
# Debian/Ubuntu
sudo apt-get install python3-pil python3-pyusb

# Fedora
sudo dnf install python3-pillow python3-pyusb

# macOS
pip3 install pillow pyusb
```

### Bluetooth Permission Denied (Linux)

**Symptom:** "Can't open Bluetooth connection: Permission denied"

**Solution (Fedora/SELinux):**
```bash
sudo semanage permissive -a cupsd_t
```

### Printer Not Discovered

**Symptom:** Printer doesn't appear in CUPS

**Solutions:**
1. Ensure printer is paired (Bluetooth) or connected (USB)
2. Check backend output: `/usr/lib/cups/backend/phomemo`
3. Verify PyUSB is installed for USB printers
4. Check D-Bus is running for Bluetooth discovery

### Image Quality Issues

**Tips:**
- Use high-contrast black & white images
- Optimal width: 384 pixels for M02/T02, 344 pixels for M110
- Avoid gradients (thermal printers use dithering)

---

## License

Phomemo-tools is licensed under the GNU General Public License v3.

Image assets in the `images/` directory are provided under a separate license - see `images/LICENSE`.
