# Phomemo Tools - macOS USB Support (Test Build)

This directory contains experimental macOS support for Phomemo thermal printers via USB connection.

## Status

**This is a test build** for evaluating macOS USB support. Features:

| Feature | Status |
|---------|--------|
| USB Connection | Experimental |
| Bluetooth | Not Supported |
| CUPS Integration | Experimental |
| Direct Printing | Supported |

## Supported Printers

- Phomemo M02, M02 Pro, T02
- Phomemo M110, M120, M220, M421
- Phomemo D30

## Requirements

### System
- macOS 10.15 (Catalina) or later
- Python 3.8 or later

### Dependencies

Install these before proceeding:

```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install libusb (required for PyUSB)
brew install libusb

# Install Python packages
pip3 install Pillow pyusb
```

## Installation

### Option 1: Using Makefile

```bash
# Check dependencies
make check

# Install drivers (requires sudo)
sudo make install
```

### Option 2: Using install script

```bash
sudo ./install.sh
```

### Option 3: Manual Installation

```bash
# Create directories
sudo mkdir -p /usr/local/libexec/cups/filter
sudo mkdir -p /usr/local/libexec/cups/backend

# Install filters
sudo cp ../cups/filter/rastertopm02_t02.py /usr/local/libexec/cups/filter/rastertopm02_t02
sudo cp ../cups/filter/rastertopm110.py /usr/local/libexec/cups/filter/rastertopm110
sudo cp ../cups/filter/rastertopd30.py /usr/local/libexec/cups/filter/rastertopd30
sudo chmod 755 /usr/local/libexec/cups/filter/rastertopm*
sudo chmod 755 /usr/local/libexec/cups/filter/rastertopd30

# Install backend
sudo cp backend/phomemo-usb.py /usr/local/libexec/cups/backend/phomemo
sudo chmod 755 /usr/local/libexec/cups/backend/phomemo

# Restart CUPS
sudo launchctl stop org.cups.cupsd
sudo launchctl start org.cups.cupsd
```

## Usage

### Direct Printing (Recommended for Testing)

The simplest way to test is using direct USB printing:

```bash
# Find your printer's USB device
ls /dev/cu.usbmodem*

# Print an image directly
python3 ../tools/phomemo-filter.py image.png > /dev/cu.usbmodemXXXX
```

### CUPS Printing

After installation:

1. Open **System Preferences > Printers & Scanners**
2. Click **+** to add a printer
3. Select your Phomemo printer from the USB list
4. Choose the appropriate driver (PPD)

### Test USB Detection

```bash
# Run the backend in discovery mode
python3 backend/phomemo-usb.py
```

This should list any connected Phomemo USB printers.

## Troubleshooting

### "No module named 'usb'"

Install PyUSB:
```bash
pip3 install pyusb
```

### "No backend available"

Install libusb:
```bash
brew install libusb
```

### USB device not found

1. Check the printer is connected and powered on
2. Try a different USB port
3. On Apple Silicon Macs, check **System Preferences > Security & Privacy** for USB permissions
4. List USB devices: `system_profiler SPUSBDataType | grep -A 10 Phomemo`

### Permission denied

The CUPS backend needs to run as root. Ensure it's installed with mode 755:
```bash
ls -la /usr/local/libexec/cups/backend/phomemo
```

### CUPS not finding the printer

1. Check CUPS error log: `tail -f /var/log/cups/error_log`
2. Restart CUPS: `sudo launchctl stop org.cups.cupsd && sudo launchctl start org.cups.cupsd`
3. Check backend is executable: `sudo /usr/local/libexec/cups/backend/phomemo`

## Uninstallation

```bash
sudo make uninstall
```

Or manually:
```bash
sudo rm /usr/local/libexec/cups/filter/rastertopm*
sudo rm /usr/local/libexec/cups/filter/rastertopd30
sudo rm /usr/local/libexec/cups/backend/phomemo
sudo rm -rf /Library/Printers/PPDs/Contents/Resources/Phomemo
sudo rm -rf /usr/local/share/phomemo
```

## Known Limitations

1. **Bluetooth not supported**: macOS uses IOBluetooth framework which requires different implementation
2. **USB hot-plug**: CUPS may not detect newly connected printers automatically; restart CUPS if needed
3. **System Integrity Protection**: On some systems, you may need to disable SIP to install to system directories

## Feedback

This is a test build. Please report any issues or feedback to help improve macOS support.
