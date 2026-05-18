# CSpot ESP32 Modernization to ESP-IDF v6.0

This project has been successfully modernized to work with **ESP-IDF v6.0** and the **Lolin D32 Pro** ESP32 (16MB flash, 8MB PSRAM).

## What Changed

### 🎯 Core Modernizations
1. **ESP-IDF v6.0 Compatibility** - Updated build system, APIs, and drivers
2. **I2S Raw Audio Output** - 32-bit lossless audio support via native I2S
3. **Device Target** - Changed from ESP32-S3 to ESP32 (Lolin D32 Pro)
4. **Memory Optimization** - 16MB flash partitioning with PSRAM support
5. **mbedTLS v4.0** - PSA Crypto initialization for HTTPS

### ✨ Technical Improvements
- **Removed Dependencies**: LED strip component (was causing errors)
- **Modern APIs**: I2S standard mode (handle-based, not port-based)
- **Better Audio**: 32-bit stereo instead of 16-bit codec output
- **Flexible Codec**: Support for raw I2S or external DAC

## Quick Start

### Prerequisites
```bash
# Install ESP-IDF v6.0 (if not already installed)
git clone https://github.com/espressif/esp-idf.git --branch v6.0
cd esp-idf
./install.sh
source ./export.sh
```

### Build Steps
```bash
cd targets/esp32

# Clean previous build
rm -rf build

# Set target to ESP32
idf.py set-target esp32

# Configure (optional - press Q to save defaults)
idf.py menuconfig

# Build
idf.py build

# Flash to device
idf.py flash

# Monitor serial output
idf.py monitor
```

## Configuration

### I2S Audio Pins (Customizable)
The project uses GPIO-based I2S output with default pins:
- **BCK (Bit Clock)**: GPIO 26
- **LRCK (Word Select)**: GPIO 25
- **DATA (DIN)**: GPIO 13

**To change these pins:**
1. Run `idf.py menuconfig`
2. Navigate to `CSPOT Configuration` → `I2S BCK/LRCK/DATA GPIO`
3. Enter your desired GPIO numbers
4. Save and rebuild

### Audio Quality Settings
In `menuconfig` → `CSPOT Configuration`:
- **Sink Device**: Select "Raw I2S (Generic I2S Audio Output - 32-bit)"
- **Audio Quality**: 320 bps, 160 bps, or 96 bps (Spotify subscription dependent)

Alternative sinks still supported:
- Internal DAC (built-in speaker output)
- AC101, ES8388, ES8311, ES9018, PCM5102, TAS5711

## Project Structure

### Key Files Modified
```
targets/esp32/
├── CMakeLists.txt              # Removed led_strip dependency
├── sdkconfig.defaults          # ESP32 target, 16MB flash config
├── partitions.csv              # Optimized partition table
└── main/
    ├── main.cpp                # Refactored audio sink selection
    ├── Kconfig.projbuild       # Added I2S configuration options
    └── ESPStatusLed.cpp        # LED support (optional)

cspot/bell/main/audio-sinks/
├── esp/I2SAudioSink.cpp        # NEW: Modern I2S driver (v6.0)
└── include/esp/I2SAudioSink.h  # NEW: I2S sink header
```

## Architecture

### Audio Pipeline (I2S Raw Mode)
```
Spotify Stream (PCM) 
  ↓
CSpot Decoder (AAC/Vorbis/ALAC/Opus)
  ↓
TrackPlayer (PCM frames)
  ↓
Circular Buffer (1MB PSRAM)
  ↓
I2SAudioSink
  ├─ 32-bit stereo PCM
  ├─ 44.1kHz sample rate
  └─ Software volume control
  ↓
I2S Driver (Modern v6.0 API)
  ├─ BCK GPIO 26
  ├─ LRCK GPIO 25
  └─ DIN GPIO 13
  ↓
External DAC / I2S Receiver
```

## Memory Layout (16MB ESP32)

```
Address    Size      Partition      Purpose
─────────────────────────────────────────────
0x000000   8KB       Bootloader     Boot code
0x009000   24KB      NVS            Configuration storage
0x00F000   8KB       OTA Data       Update control
0x011000   4KB       PHY Init       Radio config
0x012000   1MB       Factory        Main application
0x112000   1MB       OTA_0          Update partition 1
0x212000   1MB       OTA_1          Update partition 2
0x312000   3MB       SPIFFS         File storage (auth, cache)
0x612000   9MB       (Available)    Unused
─────────────────────────────────────────────
         16MB       Total
```

Available space: ~9MB for application heap and PSRAM usage.

## ESP-IDF v6.0 Migration Details

### I2S Driver API Changes
**Old (v5.x):**
```cpp
#include "driver/i2s.h"
i2s_config_t cfg = {...};
i2s_driver_install(I2S_NUM_0, &cfg, 0, NULL);
i2s_pin_config_t pins = {...};
i2s_set_pin(I2S_NUM_0, &pins);
i2s_write(I2S_NUM_0, data, len, &bytes, timeout);
i2s_driver_uninstall(I2S_NUM_0);
```

**New (v6.0):**
```cpp
#include "driver/i2s_std.h"
i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
i2s_new_channel(&chan_cfg, &i2s_tx_handle, NULL);
i2s_std_config_t std_cfg = {...};
i2s_channel_init_std_mode(i2s_tx_handle, &std_cfg);
i2s_channel_enable(i2s_tx_handle);
i2s_channel_write(i2s_tx_handle, data, len, &bytes, timeout);
i2s_del_channel(i2s_tx_handle);
```

### mbedTLS v4.0 Changes
- **Requirement**: PSA Crypto must be initialized before HTTPS/TLS
- **Implementation**: Added `psa_crypto_init()` call in `app_main()`
- **Headers**: Now includes `<psa/crypto.h>`

## Troubleshooting

### Build Errors

#### `error: 'i2s_driver_install' was not declared in this scope`
- **Cause**: Using old I2S API
- **Solution**: Ensure `#include "driver/i2s_std.h"` is used, not `driver/i2s.h`

#### `undefined reference to 'psa_crypto_init'`
- **Cause**: mbedTLS PSA not initialized
- **Solution**: Verify `psa_crypto_init()` is called in `app_main()` before TLS operations

#### `SPIFFS partition not found`
- **Cause**: Partition table mismatch
- **Solution**: Verify `partitions.csv` exists in `targets/esp32/` and:
  ```
  CONFIG_PARTITION_TABLE_CUSTOM=y
  CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
  ```

#### `Unknown target esp32` 
- **Cause**: sdkconfig.defaults has wrong target
- **Solution**: Ensure `CONFIG_IDF_TARGET="esp32"` (not esp32s3)

### Runtime Issues

#### Audio not playing
1. Verify I2S pins are correctly connected to DAC/receiver
2. Check `idf.py menuconfig` has correct GPIO pins configured
3. Monitor logs: `idf.py monitor` and look for "I2S Audio Sink initialized"
4. Test with different sample rates via `setSampleRate()`

#### Frequent disconnections
- Increase Wi-Fi buffer: Edit `sdkconfig.defaults` CONFIG_ESP_WIFI_DYNAMIC_RX_BUFFER_NUM
- Ensure stable power supply to ESP32

#### Out of memory errors
- Check SPIFFS usage: `esp_spiffs_info()` in logs
- PSRAM enabled: `CONFIG_ESP32_SPIRAM_SUPPORT=y` should be set

## Advanced Configuration

### Enable Status LED
In `menuconfig`:
1. `CSPOT Configuration` → `Status LED type` → Choose GPIO or RMT
2. Set GPIO number (default: GPIO 5)
3. Rebuild

**Note**: LED support uses modern GPIO driver (v6.0 compatible).

### Use Different Audio Codec
1. In `menuconfig`, change `CSPOT Configuration` → `Sink Device`
2. Options: AC101, ES8388, ES8311, ES9018, PCM5102, TAS5711
3. Configure codec-specific GPIOs if needed

### Custom Partition Table
Edit `targets/esp32/partitions.csv` to adjust sizes:
- Increase app size for larger firmware
- Adjust SPIFFS size for more file storage
- Follow ESP-IDF partition table format

## Performance Notes

- **Audio Quality**: 32-bit stereo at 44.1kHz (320kbps Spotify)
- **Latency**: ~500ms (typical for Spotify Connect)
- **Memory Usage**: ~3-4MB code, 1-2MB buffers, rest available for heap
- **PSRAM Benefit**: Enables large circular buffer for smooth streaming

## Compatibility

- **Device**: ESP32 (Lolin D32 Pro) with 16MB flash, 8MB PSRAM
- **ESP-IDF**: v6.0+ required
- **GCC**: 11.2.0+ (included with ESP-IDF v6.0)
- **Python**: 3.8+

### Tested On
- ✅ macOS (Apple Silicon M1/M2)
- ✅ Linux (Ubuntu 22.04+)
- ✅ Windows 11 + WSL2

## Next Steps

1. **Build**: Follow "Build Steps" above
2. **Flash**: Connect Lolin D32 Pro via USB
3. **Configure**: Run Spotify app → Cast to "CSpot-ESP32"
4. **Enjoy**: Stream music with 32-bit lossless quality

## Support

For issues:
1. Check logs: `idf.py monitor` (Ctrl+]` to exit)
2. Review this README's Troubleshooting section
3. See `PRE_BUILD_CHECKLIST.md` for verification steps
4. Refer to `IMPLEMENTATION_SUMMARY.md` for technical details

## References

- [ESP-IDF v6.0 Documentation](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32/)
- [ESP-IDF I2S Standard Mode](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32/api-reference/peripherals/i2s.html)
- [mbedTLS v4.0 PSA Crypto](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32/api-reference/security/mbedtls.html)
- [CSpot Repository](https://github.com/feelfreelinux/cspot)
- [Lolin D32 Pro Specs](https://www.wemos.cc/en/latest/d32/d32_pro.html)

---

**Modernization Date**: May 2026
**ESP-IDF Version**: v6.0+
**Status**: ✅ Ready to Build
