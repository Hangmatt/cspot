# Pre-Build Verification Checklist

## Configuration Files ✅
- [x] `targets/esp32/CMakeLists.txt` - Led_strip removed
- [x] `targets/esp32/sdkconfig.defaults` - ESP32 target (16MB flash, PSRAM)
- [x] `targets/esp32/partitions.csv` - Optimized partition table created
- [x] `targets/esp32/main/Kconfig.projbuild` - I2S configuration options added

## Code Changes ✅
- [x] `targets/esp32/main/main.cpp` - Audio sink selection refactored
- [x] `targets/esp32/main/main.cpp` - PSA crypto initialization added
- [x] `cspot/bell/main/audio-sinks/esp/I2SAudioSink.cpp` - Modern I2S driver
- [x] `cspot/bell/main/audio-sinks/include/esp/I2SAudioSink.h` - I2S sink header

## Environmental Setup ✅
- [x] Git submodules initialized (bell + dependencies)
- [x] ESP-IDF v6.0 compatible structures

## Key Modernizations
### I2S Driver (v5.x → v6.0)
- Old: `#include "driver/i2s.h"`
- New: `#include "driver/i2s_std.h"`
- Old: `i2s_driver_install()` + `i2s_set_pin()`
- New: `i2s_new_channel()` + `i2s_channel_init_std_mode()`

### mbedTLS (v3.x → v4.0)
- Added: `#include <psa/crypto.h>`
- Added: `psa_crypto_init()` in app_main
- PSA Crypto is now the primary API in v4.0

### Build System
- Removed: `led_strip` component dependency
- Kept: `protocol_examples_common` for Wi-Fi
- Added: Automatic I2S component inclusion

## Audio Configuration
**Default Settings:**
- Sink: I2S Raw (GPIO-based output)
- BCK Pin: 26
- LRCK Pin: 25
- DATA Pin: 13
- Sample Rate: 44.1kHz
- Bit Depth: 32-bit (lossless capable)
- Channels: Stereo

## Memory Layout (16MB ESP32)
```
0x00000 - 0x08000   : Bootloader
0x09000 - 0x0f000   : NVS (24KB)
0x0f000 - 0x11000   : OTA data (8KB)
0x11000 - 0x12000   : PHY init (4KB)
0x12000 - 0x112000  : Factory/App (1MB)
0x112000 - 0x212000 : OTA_0 (1MB)
0x212000 - 0x312000 : OTA_1 (1MB)
0x312000 - 0x612000 : SPIFFS (3MB)
Remaining: ~9MB available
```

## Build Commands (Next Steps)
```bash
cd targets/esp32

# Clean build
rm -rf build

# Set target to ESP32
idf.py set-target esp32

# Configuration (optional - verify I2S selected)
idf.py menuconfig

# Build
idf.py build

# Flash
idf.py flash

# Monitor
idf.py monitor
```

## Troubleshooting

### If I2S Driver Build Error
- Verify ESP-IDF v6.0 is installed: `idf.py --version`
- Check CONFIG_IDF_TARGET="esp32" in sdkconfig.defaults

### If mbedTLS Errors
- PSA crypto must be initialized before any HTTPS/TLS operations
- Verify `psa_crypto_init()` is called in app_main

### If Partition Table Error
- Verify `CONFIG_PARTITION_TABLE_CUSTOM=y` is set
- Check filename: `CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"`

### If Audio Sink Compile Error
- Bell library must be fully initialized (check submodule status)
- I2SAudioSink.cpp requires ESP-IDF v6.0 I2S headers

## Expected Build Output
```
[100%] Built target cspot
Generating binary image
esptool.py app-flash -o ... build/cspot-esp32.bin
...
Bin size: 912545 bytes
```

## Validation After Build
1. Binary file exists: `build/cspot-esp32.bin`
2. Partition binary: `build/partition_table/partitions.bin`
3. Bootloader binary: `build/bootloader/bootloader.bin`

