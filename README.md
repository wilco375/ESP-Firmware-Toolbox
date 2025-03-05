# ESP-Firmware-Toolkit
This toolkit is a collection of tools that can be used to dump, reverse-engineer and patch ESP32 firmware. See [this blog](https://medium.com/@wilcovanbeijnum/tutorial-hacking-and-patching-firmware-of-esp32-based-iot-devices-c12ba71a6522) for more information.

## Included tools and files
Tools:
- [mitmproxy](https://mitmproxy.org/) (installed via pip requirement)
- [esptool](https://github.com/espressif/esptool) (installed via pip requirement)
- [esp32knife](https://github.com/BlackVS/esp32knife) by [BlackVS](https://github.com/BlackVS)
- [ESP-Firmware-Patcher](https://github.com/wilco375/ESP-Firmware-Patcher) by me

Ghidra scripts:
- [SVD-Loader for Ghidra](https://github.com/leveldown-security/SVD-Loader-Ghidra) by [leveldown security](https://github.com/leveldown-security)
- [IdentifyLoggingStrings](https://gist.github.com/wilco375/0bd75cd8303b8e0c3b0189b0a0622f08) by me

Files for Ghidra analysis:
- [ESP32 ROM function labels](https://gist.github.com/jmswrnr/3095b39f8b1f3631489a5db75a275875) by [James Warner](https://github.com/jmswrnr)
- [ESP32 SVD files](https://github.com/espressif/svd)

## Requirements
- Python 3 with pip
- [Ghidra](https://github.com/NationalSecurityAgency/ghidra/releases/latest)

## Installation
### Clone the repo
```bash
git clone --recurse-submodules -j8 https://github.com/wilco375/ESP-Firmware-Toolbox.git
cd ESP-Firmware-Toolbox
```

### Python dependencies
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Ghidra scripts
1. Open a Ghidra project
2. Go to _Window_ -> _Script Manager_
3. Click _Manage Script Directories_ (list icon at the top right)
4. Click _Display file chooser to add bundles to list_ (green plus at the top right)
5. Select the `Ghidra-Scripts/IdentifyLoggingStrings` directory
6. Repeat step 4 and 5 for the `Ghidra-Scripts/SVD-Loader-Ghidra` directory

## Usage
### Inspect ESP32 behavior
#### Wi-Fi
Connect the ESP32's Wi-Fi to a Wi-Fi hotspot on your laptop/desktop.  
Network traffic can be inspected with Wireshark.  
If TLS is used, you can try to execute a MitM attack with mitmproxy.
You can [set up a transparent proxy](https://docs.mitmproxy.org/stable/howto-transparent/), and then start mitmproxy:
```bash
mitmproxy --mode transparent --showhost
```

#### Bluetooth
On Android, you can use the _Bluetooth HCI snoop_ developer option to log all Bluetooth traffic, and then generate a bugreport with ADB.  
With a rooted phone, you can directly view live Bluetooth traffic in Wireshark over ADB.

#### UART
Simply connect to the ESP32's TX port using a serial to USB converter, and use a serial monitor to view the UART log.

### Dumping the firmware
First, start the ESP32 in Bootloader mode by pulling GPIO0 to GND. While pulling to GND, power on the ESP32, and connect to the ESP32's 
TX and RX pins with a serial to USB converter.
```bash
# Identify flash size
esptool.py flash_id
# Dump flash to firmware.bin, adjust `/dev/ttyUSB0` and `0x400000` (4MB) based on
# the serial port and flash size of your ESP32
esptool.py --baud 115200 --port /dev/ttyUSB0 read_flash 0x0 0x400000 firmware.bin
```

### Analyzing the firmware
Extract partitions using esp32knife:
```bash
python3 esp32knife/esp32knife.py --chip=esp32 load_from_file firmware.bin
```
Partitions are stored in the `parsed/` directory. The ELF files in this directory can be analyzed in Ghidra. 
Factory and OTA (if available) partitions are most interesting.

Simply drag and drop the ELF files in Ghidra, and double click it. Then:
1. Cancel auto analysis
2. Run the ImportSymbolsScript.py script, and select `Ghidra-Files/ESP-ROM-Labels/ESP32_ROM_LABELS.txt`
3. Run the SVD-Loader.py script, and select `Ghidra-Files/ESP-SVD/svd/esp32.svd`
4. Run the auto analysis
5. Run the IdentifyLoggingStrings.java script

### Patching the firmware
Edit `SIG_FIND_REPLACE` in `ESP-Firmware-Patcher/main.py` with the binary code to search for and repace. Then, run the patcher:
```bash
python3 ESP-Firmware-Patcher/main.py firmware.bin
```

We can then upload the modified firmware to the ESP32:
```bash
esptool.py --baud 115200 --port /dev/ttyUSB0 --after hard_reset --chip esp32 write_flash --flash_mode dio --flash_size detect --flash_freq 40m 0x0 firmware-patched.bin 
```
