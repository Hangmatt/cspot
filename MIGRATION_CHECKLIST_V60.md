# ESP-IDF v6.0 Migration Checklist: I2S 32-bit Audio Implementation

Step-by-step guide for migrating cspot's audio sink to ESP-IDF v6.0 and adding 32-bit support.

---

## Pre-Migration: Assessment

- [ ] Identify all files using `#include "driver/i2s.h"` (old API)
  ```bash
  grep -r "driver/i2s.h" cspot/bell/main/audio-sinks/
  ```

- [ ] List audio sinks to migrate:
  - [ ] `BufferedAudioSink.cpp` (base class - MUST migrate)
  - [ ] `PCM5102AudioSink.cpp` (32-bit capable)
  - [ ] `ES9018AudioSink.cpp` (premium 32-bit)
  - [ ] `ES8388AudioSink.cpp` (24-bit standard)
  - [ ] `AC101AudioSink.cpp` (24-bit)
  - [ ] `TAS5711AudioSink.cpp` (32-bit amplified)
  - [ ] `SPDIFAudioSink.cpp` (digital output)
  - [ ] `InternalAudioSink.cpp` (if applicable)

- [ ] Check current pin configuration
  ```bash
  grep -r "bck_io_num\|ws_io_num\|data_out_num" cspot/bell/main/audio-sinks/
  ```

---

## Step 1: Update BufferedAudioSink (Base Class)

### 1.1 Header Changes

**Before (v5.x):**
```cpp
#include "driver/i2s.h"
```

**After (v6.0):**
```cpp
#include "driver/i2s_std.h"
```

### 1.2 Member Variable Changes

**Before:**
```cpp
class BufferedAudioSink : public AudioSink {
private:
    // No I2S handle (global driver)
};
```

**After:**
```cpp
class BufferedAudioSink : public AudioSink {
protected:
    i2s_chan_handle_t tx_handle = NULL;
};
```

### 1.3 Constructor Changes

**Before:**
```cpp
BufferedAudioSink::BufferedAudioSink() {
    i2s_config_t i2s_config = {
        .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_TX),
        .sample_rate = 44100,
        .bits_per_sample = (i2s_bits_per_sample_t)16,
        .channel_format = I2S_CHANNEL_FMT_RIGHT_LEFT,
        .communication_format = (i2s_comm_format_t)I2S_COMM_FORMAT_STAND_I2S,
        .intr_alloc_flags = 0,
        .dma_buf_count = 6,
        .dma_buf_len = 512,
        .use_apll = true,
    };
    i2s_driver_install((i2s_port_t)0, &i2s_config, 0, NULL);
    i2s_set_pin((i2s_port_t)0, &pin_config);
    startI2sFeed();
}
```

**After:**
```cpp
BufferedAudioSink::BufferedAudioSink() {
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(
        I2S_NUM_0,
        I2S_ROLE_MASTER
    );
    
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
            I2S_DATA_BIT_WIDTH_16BIT,
            I2S_SLOT_MODE_STEREO
        ),
        .gpio_cfg = {
            .mclk = GPIO_NUM_0,
            .bclk = GPIO_NUM_26,
            .ws = GPIO_NUM_25,
            .dout = GPIO_NUM_13,
            .din = GPIO_NUM_-1,
        },
    };
    
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
    ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
    startI2sFeed();
}
```

### 1.4 Destructor Changes

**Before:**
```cpp
BufferedAudioSink::~BufferedAudioSink() {
    i2s_driver_uninstall((i2s_port_t)0);
}
```

**After:**
```cpp
BufferedAudioSink::~BufferedAudioSink() {
    if (tx_handle != NULL) {
        i2s_channel_disable(tx_handle);
        i2s_del_channel(tx_handle);
        tx_handle = NULL;
    }
}
```

### 1.5 Audio Feed Loop Changes

**Before:**
```cpp
static void i2sFeed(void* pvParameters) {
    while (true) {
        size_t itemSize;
        char* item = (char*)xRingbufferReceiveUpTo(dataBuffer, &itemSize,
                                                   portMAX_DELAY, 512);
        if (item != NULL) {
            size_t written = 0;
            while (written < itemSize) {
                i2s_write((i2s_port_t)0, item, itemSize, &written, portMAX_DELAY);
            }
            vRingbufferReturnItem(dataBuffer, (void*)item);
        }
    }
}
```

**After:**
```cpp
static void i2sFeed(void* pvParameters) {
    i2s_chan_handle_t handle = (i2s_chan_handle_t)pvParameters;
    
    while (true) {
        size_t itemSize;
        char* item = (char*)xRingbufferReceiveUpTo(dataBuffer, &itemSize,
                                                   portMAX_DELAY, 512);
        if (item != NULL) {
            size_t bytes_written = 0;
            esp_err_t err = i2s_channel_write(handle, item, itemSize, 
                                              &bytes_written, portMAX_DELAY);
            
            vRingbufferReturnItem(dataBuffer, (void*)item);
            
            if (err != ESP_OK) {
                ESP_LOGE("I2S", "Write failed: %s", esp_err_to_name(err));
            }
        }
    }
}

// In startI2sFeed():
void BufferedAudioSink::startI2sFeed(size_t buf_size) {
    dataBuffer = xRingbufferCreate(buf_size, RINGBUF_TYPE_BYTEBUF);
    xTaskCreatePinnedToCore(&i2sFeed, "i2sFeed", 4096, 
                            (void *)tx_handle,  // Pass handle as parameter
                            10, NULL, tskNO_AFFINITY);
}
```

### 1.6 Sample Rate Change

**Before:**
```cpp
bool BufferedAudioSink::setParams(uint32_t sampleRate, uint8_t channelCount,
                                  uint8_t bitDepth) {
    i2s_set_clk((i2s_port_t)0, sampleRate, (i2s_bits_per_sample_t)bitDepth,
                (i2s_channel_t)channelCount);
    return true;
}
```

**After:**
```cpp
bool BufferedAudioSink::setParams(uint32_t sampleRate, uint8_t channelCount,
                                  uint8_t bitDepth) {
    // Only sample rate can be changed dynamically
    i2s_std_clk_config_t clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(sampleRate);
    esp_err_t err = i2s_channel_reconfig_clk(tx_handle, &clk_cfg);
    
    if (err != ESP_OK) {
        ESP_LOGE("AUDIO", "Failed to reconfigure clock: %s", esp_err_to_name(err));
        return false;
    }
    
    ESP_LOGI("AUDIO", "Audio params updated: %luHz, %u-bit, %u-ch",
             sampleRate, bitDepth, channelCount);
    return true;
}
```

---

## Step 2: Add 32-bit Support to BufferedAudioSink

### 2.1 Update Constructor to Support 32-bit

```cpp
BufferedAudioSink::BufferedAudioSink(uint8_t bit_depth) {
    // Convert bit_depth parameter
    i2s_data_bit_width_t width;
    switch (bit_depth) {
        case 16: width = I2S_DATA_BIT_WIDTH_16BIT; break;
        case 24: width = I2S_DATA_BIT_WIDTH_24BIT; break;
        case 32: width = I2S_DATA_BIT_WIDTH_32BIT; break;
        default: width = I2S_DATA_BIT_WIDTH_16BIT; break;  // Fallback
    }
    
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(
        I2S_NUM_0, I2S_ROLE_MASTER
    );
    
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
            width,  // Use parameter
            I2S_SLOT_MODE_STEREO
        ),
        .gpio_cfg = {
            .mclk = GPIO_NUM_0,
            .bclk = GPIO_NUM_26,
            .ws = GPIO_NUM_25,
            .dout = GPIO_NUM_13,
            .din = GPIO_NUM_-1,
        },
    };
    
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
    ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
    startI2sFeed();
}
```

### 2.2 Header Update

```cpp
class BufferedAudioSink : public AudioSink {
public:
    BufferedAudioSink();
    BufferedAudioSink(uint8_t bit_depth);  // Add 32-bit support
    
    // ... rest of interface ...
};
```

---

## Step 3: Update Specific Audio Sinks

### 3.1 PCM5102AudioSink (32-bit DAC)

**Before:**
```cpp
PCM5102AudioSink::PCM5102AudioSink() {
    i2s_config_t i2s_config = {
        .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_TX),
        .sample_rate = 44100,
        .bits_per_sample = (i2s_bits_per_sample_t)16,  // ← Can be 32-bit
        .channel_format = I2S_CHANNEL_FMT_RIGHT_LEFT,
        .communication_format = (i2s_comm_format_t)I2S_COMM_FORMAT_I2S,
        .intr_alloc_flags = 0,
        .dma_buf_count = 8,
        .dma_buf_len = 512,
        .use_apll = true,
        .tx_desc_auto_clear = true,
        .fixed_mclk = 384 * 44100
    };

    i2s_pin_config_t pin_config = {
        .bck_io_num = 27,
        .ws_io_num = 32,
        .data_out_num = 25,
        .data_in_num = -1
    };
    i2s_driver_install((i2s_port_t)0, &i2s_config, 0, NULL);
    i2s_set_pin((i2s_port_t)0, &pin_config);
    startI2sFeed();
}
```

**After (v6.0 with 32-bit):**
```cpp
PCM5102AudioSink::PCM5102AudioSink() : BufferedAudioSink(32) {}  // 32-bit support
// Uses parent class initialization with 32-bit I2S

// Override if different GPIO pins needed:
PCM5102AudioSink::PCM5102AudioSink() {
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(
        I2S_NUM_0, I2S_ROLE_MASTER
    );
    
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
            I2S_DATA_BIT_WIDTH_32BIT,  // 32-bit support
            I2S_SLOT_MODE_STEREO
        ),
        .gpio_cfg = {
            .mclk = GPIO_NUM_0,
            .bclk = GPIO_NUM_27,      // PCM5102 specific pins
            .ws = GPIO_NUM_32,
            .dout = GPIO_NUM_25,
            .din = GPIO_NUM_-1,
            .invert_flags = {false, false, false},
        },
    };
    
    tx_handle = NULL;
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
    ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
    startI2sFeed();
}
```

### 3.2 ES9018AudioSink (MSB Format)

**Key Change:** Use `I2S_STD_MSB_SLOT_DEFAULT_CONFIG` instead of Philips

```cpp
ES9018AudioSink::ES9018AudioSink() {
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(
        I2S_NUM_0, I2S_ROLE_MASTER
    );
    
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
        .slot_cfg = I2S_STD_MSB_SLOT_DEFAULT_CONFIG(  // ← MSB, not Philips!
            I2S_DATA_BIT_WIDTH_32BIT,
            I2S_SLOT_MODE_STEREO
        ),
        .gpio_cfg = {
            .mclk = GPIO_NUM_0,
            .bclk = GPIO_NUM_27,
            .ws = GPIO_NUM_32,
            .dout = GPIO_NUM_25,
            .din = GPIO_NUM_-1,
        },
    };
    
    tx_handle = NULL;
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
    ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
    startI2sFeed();
}
```

### 3.3 ES8388AudioSink (24-bit, I2C Controlled)

```cpp
ES8388AudioSink::ES8388AudioSink() {
    // I2S configuration (unchanged from v5.x in terms of bit depth)
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(
        I2S_NUM_0, I2S_ROLE_MASTER
    );
    
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
            I2S_DATA_BIT_WIDTH_24BIT,  // ES8388 standard is 24-bit
            I2S_SLOT_MODE_STEREO
        ),
        .gpio_cfg = {
            .mclk = GPIO_NUM_0,
            .bclk = GPIO_NUM_27,
            .ws = GPIO_NUM_25,
            .dout = GPIO_NUM_26,
            .din = GPIO_NUM_-1,
        },
    };
    
    tx_handle = NULL;
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
    ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
    
    // I2C initialization (same as before)
    i2c_config_t i2c_cfg = {
        .mode = I2C_MODE_MASTER,
        .sda_io_num = 33,
        .scl_io_num = 32,
        .sda_pullup_en = GPIO_PULLUP_ENABLE,
        .scl_pullup_en = GPIO_PULLUP_ENABLE,
    };
    i2c_cfg.master.clk_speed = 100000;
    i2c_param_config(I2C_NUM_0, &i2c_cfg);
    i2c_driver_install(I2C_NUM_0, I2C_MODE_MASTER, 0, 0, 0);
    
    // ES8388 codec initialization (same as v5.x)
    initializeCodec();
    
    startI2sFeed();
}

ES8388AudioSink::~ES8388AudioSink() {
    if (tx_handle != NULL) {
        i2s_channel_disable(tx_handle);
        i2s_del_channel(tx_handle);
        tx_handle = NULL;
    }
    i2c_driver_delete(I2C_NUM_0);
}
```

---

## Step 4: Compilation and Testing

- [ ] Update CMakeLists.txt to require ESP-IDF v6.0+
  ```cmake
  set(EXTRA_COMPONENT_DIRS "$ENV{IDF_PATH}/components")
  include($ENV{IDF_PATH}/tools/cmake/version.cmake)
  
  # Verify ESP-IDF version is 6.0+
  idf_build_get_property(idf_version IDF_VERSION)
  message(STATUS "ESP-IDF version: ${idf_version}")
  ```

- [ ] Compile and check for errors
  ```bash
  idf.py build
  ```

- [ ] Fix compilation errors:
  - `error: 'i2s_driver_install' was not declared` → Update to `i2s_new_std_tx_channel`
  - `error: 'I2S_COMM_FORMAT_STAND_I2S' was not declared` → Use `I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG`
  - `error: 'I2S_MODE_MASTER' was not declared` → Already set in `I2S_CHANNEL_DEFAULT_CONFIG`

- [ ] Flash and test audio output
  ```bash
  idf.py -p /dev/ttyUSB0 flash monitor
  ```

---

## Step 5: Validation Checklist

### Audio Output Tests

- [ ] Audio plays without crackling
- [ ] Volume level is appropriate
- [ ] No audio dropout during playback
- [ ] Sample rate changes smoothly (no pops/clicks)

### 32-bit Specific Tests

- [ ] 32-bit I2S output verified with oscilloscope/logic analyzer
- [ ] BCLK frequency correct (2.8224 MHz for 44.1 kHz, 32-bit)
- [ ] LRCK frequency correct (44.1 kHz for 44.1 kHz sample rate)
- [ ] Data lines show 32-bit patterns (4 bytes per sample)

### Functionality Tests

- [ ] Play 16-bit audio (backward compatibility)
- [ ] Play 24-bit audio (if supported)
- [ ] Play 32-bit audio (new)
- [ ] Switch between 44.1 kHz and 48 kHz (dynamic rate)
- [ ] No audio interruption on rate switch

---

## Step 6: Optimization and Cleanup

### Performance Improvements

- [ ] Increase ring buffer size if audio dropout occurs
  ```cpp
  dataBuffer = xRingbufferCreate(512 * 1024, RINGBUF_TYPE_BYTEBUF);  // 512 KB
  ```

- [ ] Adjust DMA buffer if latency is critical
  ```cpp
  chan_cfg.dma_desc_num = 8;
  chan_cfg.dma_frame_num = 1024;
  ```

- [ ] Pin I2S feed task to specific core for better performance
  ```cpp
  xTaskCreatePinnedToCore(&i2sFeed, "i2sFeed", 4096, NULL, 10, NULL, 1);
  ```

### Code Cleanup

- [ ] Remove old `#include "driver/i2s.h"` references
- [ ] Remove unused I2S v5.x type definitions
- [ ] Update documentation and comments
- [ ] Add error logging for debugging

---

## Step 7: Documentation Updates

- [ ] Update README.md to document 32-bit support
- [ ] Add configuration notes for different DACs (PCM5102, ES9018, etc.)
- [ ] Document GPIO pin configuration
- [ ] Add sample rate switching usage example
- [ ] Update API documentation for BufferedAudioSink

---

## Common Migration Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Compilation fails: `i2s_driver_install undefined` | Old API used | Replace with `i2s_new_std_tx_channel()` |
| Silent audio output | Channel not enabled | Call `i2s_channel_enable()` after creation |
| Crackling/noise | Incorrect GPIO pins | Verify pins match DAC datasheet |
| Audio dropout on rate change | Using `i2s_set_clk()` | Use `i2s_channel_reconfig_clk()` instead |
| Memory crash in i2sFeed | Ring buffer timeout | Increase ring buffer size |
| APLL lock failure | Rate not supported | Use standard rates (44.1, 48, 96, 192 kHz) |

---

## Verification Commands

```bash
# Find all files still using old API
grep -r "driver/i2s.h" cspot/bell/main/audio-sinks/

# Find I2S function calls
grep -r "i2s_driver_install\|i2s_set_pin\|i2s_write\|i2s_set_clk" cspot/bell/

# Verify new API usage
grep -r "i2s_new_std_tx_channel\|i2s_channel_write\|i2s_channel_enable" cspot/bell/

# Check for 32-bit bit width
grep -r "I2S_DATA_BIT_WIDTH_32BIT" cspot/bell/
```

---

## Success Criteria

✓ All audio sinks compile without errors  
✓ 32-bit I2S output verified on oscilloscope  
✓ Spotify audio plays cleanly at full quality  
✓ No audio dropout or crackling  
✓ Sample rate switching works smoothly  
✓ PSRAM buffer usage < 20 MB  
✓ I2S clocking stable and accurate  

---

## Next Steps After Migration

1. **Profiling**: Measure power consumption and CPU usage
2. **Optimization**: Fine-tune buffer sizes for your WiFi environment
3. **Testing**: Validate with various Spotify bitrates and sample rates
4. **Documentation**: Add usage examples for different audio formats
5. **Upstream**: Consider submitting improvements back to cspot project
