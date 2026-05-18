# I2S 32-bit Audio: Practical Working Examples for cspot

Quick reference for implementing high-quality audio output on ESP32 with ESP-IDF v6.0.

---

## Quick Start: Minimal 32-bit I2S Setup

### Copy-Paste Ready (PCM5102, 44.1 kHz, 32-bit)

```cpp
// main.c or your audio module
#include "driver/i2s_std.h"
#include "driver/gpio.h"
#include "esp_log.h"

static i2s_chan_handle_t i2s_tx_handle = NULL;

void setup_audio_output(void) {
    // Channel config
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
    
    // Standard I2S config: 32-bit, Philips format, 44.1 kHz
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
            I2S_DATA_BIT_WIDTH_32BIT,
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
    
    // Create and enable I2S channel
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &i2s_tx_handle));
    ESP_ERROR_CHECK(i2s_channel_enable(i2s_tx_handle));
    
    ESP_LOGI("AUDIO", "I2S initialized: 44.1kHz, 32-bit, Stereo");
}

void send_audio(const uint8_t *pcm_data, size_t len) {
    size_t bytes_written = 0;
    i2s_channel_write(i2s_tx_handle, (void *)pcm_data, len, &bytes_written, portMAX_DELAY);
}

void shutdown_audio(void) {
    i2s_channel_disable(i2s_tx_handle);
    i2s_del_channel(i2s_tx_handle);
}
```

---

## Preset Configurations

### Config 1: PCM5102 (Standard Philips I2S)

**Best for**: Budget-conscious, standard Philips I2S DACs  
**Codecs**: PCM5102, PCM1793, TPA9840, similar

```cpp
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_32BIT,  // 32-bit
        I2S_SLOT_MODE_STEREO
    ),
    .gpio_cfg = {
        .mclk = GPIO_NUM_0,
        .bclk = GPIO_NUM_26,
        .ws = GPIO_NUM_25,
        .dout = GPIO_NUM_13,
        .din = GPIO_NUM_-1,
        .invert_flags = {false, false, false},
    },
};
```

---

### Config 2: ES9018 (MSB-First Big-Endian I2S)

**Best for**: Premium audio, audiophile-grade sound  
**Codecs**: ES9018, ES9023, similar premium DACs

```cpp
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_MSB_SLOT_DEFAULT_CONFIG(  // Note: MSB, not Philips
        I2S_DATA_BIT_WIDTH_32BIT,  // 32-bit
        I2S_SLOT_MODE_STEREO
    ),
    .gpio_cfg = {
        .mclk = GPIO_NUM_0,
        .bclk = GPIO_NUM_26,
        .ws = GPIO_NUM_25,
        .dout = GPIO_NUM_13,
        .din = GPIO_NUM_-1,
        .invert_flags = {false, false, false},
    },
};
```

---

### Config 3: ES8388 (I2C Controlled, 24-bit Standard)

**Best for**: Integrated audio codec with recording capability  
**Note**: 24-bit standard (32-bit requires TDM mode)

```cpp
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_24BIT,  // 24-bit max for standard I2S
        I2S_SLOT_MODE_STEREO
    ),
    .gpio_cfg = {
        .mclk = GPIO_NUM_0,
        .bclk = GPIO_NUM_26,
        .ws = GPIO_NUM_25,
        .dout = GPIO_NUM_13,
        .din = GPIO_NUM_-1,
        .invert_flags = {false, false, false},
    },
};

// I2C for ES8388 control (separate from I2S)
i2c_config_t i2c_cfg = {
    .mode = I2C_MODE_MASTER,
    .sda_io_num = GPIO_NUM_21,
    .scl_io_num = GPIO_NUM_22,
    .master.clk_speed = 100000,
};
```

---

### Config 4: TAS5711 (Integrated Amplifier + DAC)

**Best for**: Direct speaker output, no separate amplifier needed  
**32-bit capable**, I2C control

```cpp
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_32BIT,
        I2S_SLOT_MODE_STEREO
    ),
    .gpio_cfg = {
        .mclk = GPIO_NUM_0,
        .bclk = GPIO_NUM_26,
        .ws = GPIO_NUM_25,
        .dout = GPIO_NUM_13,
        .din = GPIO_NUM_-1,
        .invert_flags = {false, false, false},
    },
};

// I2C for TAS5711 control (amplifier gain, etc.)
i2c_config_t i2c_cfg = {
    .mode = I2C_MODE_MASTER,
    .sda_io_num = GPIO_NUM_21,
    .scl_io_num = GPIO_NUM_22,
    .master.clk_speed = 100000,
};
```

---

## Sample Rate Presets

### 44.1 kHz (Spotify Standard)

```cpp
i2s_std_clk_config_t clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100);
// BCLK = 44,100 × 32 × 2 = 2.8224 MHz
// MCLK = 256 × 44,100 = 11.2896 MHz (APLL auto-configures)
```

### 48 kHz (Video/Pro Audio Standard)

```cpp
i2s_std_clk_config_t clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(48000);
// BCLK = 48,000 × 32 × 2 = 3.072 MHz
// MCLK = 256 × 48,000 = 12.288 MHz
```

### 96 kHz (HD Audio)

```cpp
i2s_std_clk_config_t clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(96000);
// BCLK = 96,000 × 32 × 2 = 6.144 MHz
// MCLK = 256 × 96,000 = 24.576 MHz
```

### 192 kHz (Ultra HD)

```cpp
i2s_std_clk_config_t clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(192000);
// BCLK = 192,000 × 32 × 2 = 12.288 MHz
// MCLK = 384 × 192,000 = 73.728 MHz (note: may need higher MCLK divisor)

// For 192 kHz, consider using higher MCLK multiple:
i2s_std_clk_config_t clk_cfg = {
    .sample_rate_hz = 192000,
    .clk_src = I2S_CLK_SRC_APLL,
    .mclk_multiple = I2S_MCLK_MULTIPLE_256,  // Or 384 if needed
};
```

---

## Bit Depth Presets

### 16-bit (CD Quality, Spotify Premium)

```cpp
.slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
    I2S_DATA_BIT_WIDTH_16BIT,  // 16-bit
    I2S_SLOT_MODE_STEREO
),
```

### 24-bit (High-Resolution Audio)

```cpp
.slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
    I2S_DATA_BIT_WIDTH_24BIT,  // 24-bit
    I2S_SLOT_MODE_STEREO
),
// Note: For 24-bit, MCLK should be multiple of 3:
.mclk_multiple = I2S_MCLK_MULTIPLE_384,  // Use 384 instead of 256
```

### 32-bit (Professional/Mastering)

```cpp
.slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
    I2S_DATA_BIT_WIDTH_32BIT,  // 32-bit
    I2S_SLOT_MODE_STEREO
),
```

---

## Complete AudioSink Implementation Template

### Header (audio_sink.h)

```cpp
#pragma once

#include <cstdint>
#include <cstddef>
#include "driver/i2s_std.h"

class AudioSink {
public:
    bool initialize(uint32_t sample_rate = 44100, uint8_t bit_depth = 32);
    bool write_audio(const uint8_t *data, size_t len);
    bool change_sample_rate(uint32_t new_rate);
    void shutdown();
    bool is_ready() const { return initialized_; }

private:
    i2s_chan_handle_t tx_handle_ = NULL;
    bool initialized_ = false;
};
```

### Implementation (audio_sink.cpp)

```cpp
#include "audio_sink.h"
#include "esp_log.h"

static const char *TAG = "AUDIO_SINK";

bool AudioSink::initialize(uint32_t sample_rate, uint8_t bit_depth) {
    if (initialized_) {
        ESP_LOGW(TAG, "Already initialized");
        return true;
    }

    // Convert bit_depth
    i2s_data_bit_width_t width;
    switch (bit_depth) {
        case 16: width = I2S_DATA_BIT_WIDTH_16BIT; break;
        case 24: width = I2S_DATA_BIT_WIDTH_24BIT; break;
        case 32: width = I2S_DATA_BIT_WIDTH_32BIT; break;
        default: return false;
    }

    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(sample_rate),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(width, I2S_SLOT_MODE_STEREO),
        .gpio_cfg = {
            .mclk = GPIO_NUM_0,
            .bclk = GPIO_NUM_26,
            .ws = GPIO_NUM_25,
            .dout = GPIO_NUM_13,
            .din = GPIO_NUM_-1,
        },
    };

    if (i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle_) != ESP_OK) {
        ESP_LOGE(TAG, "Failed to create I2S TX channel");
        return false;
    }

    if (i2s_channel_enable(tx_handle_) != ESP_OK) {
        ESP_LOGE(TAG, "Failed to enable I2S channel");
        i2s_del_channel(tx_handle_);
        tx_handle_ = NULL;
        return false;
    }

    initialized_ = true;
    ESP_LOGI(TAG, "Audio initialized: %lu Hz, %u-bit", sample_rate, bit_depth);
    return true;
}

bool AudioSink::write_audio(const uint8_t *data, size_t len) {
    if (!initialized_) {
        ESP_LOGE(TAG, "Not initialized");
        return false;
    }

    size_t written = 0;
    esp_err_t err = i2s_channel_write(tx_handle_, (void *)data, len, &written, portMAX_DELAY);
    
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Write failed: %s", esp_err_to_name(err));
        return false;
    }

    return written == len;
}

bool AudioSink::change_sample_rate(uint32_t new_rate) {
    if (!initialized_) {
        return false;
    }

    i2s_std_clk_config_t clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(new_rate);
    esp_err_t err = i2s_channel_reconfig_clk(tx_handle_, &clk_cfg);
    
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Rate change failed: %s", esp_err_to_name(err));
        return false;
    }

    ESP_LOGI(TAG, "Sample rate changed to %lu Hz", new_rate);
    return true;
}

void AudioSink::shutdown() {
    if (tx_handle_) {
        i2s_channel_disable(tx_handle_);
        i2s_del_channel(tx_handle_);
        tx_handle_ = NULL;
    }
    initialized_ = false;
    ESP_LOGI(TAG, "Audio shutdown complete");
}
```

---

## Integration with cspot

### For cspot's BufferedAudioSink Migration

Current (v5.x, old driver):
```cpp
#include "driver/i2s.h"

BufferedAudioSink::BufferedAudioSink() {
    i2s_config_t i2s_config = {...};
    i2s_driver_install(I2S_NUM_0, &i2s_config, 0, NULL);
    i2s_set_pin(I2S_NUM_0, &pin_config);
    startI2sFeed();
}

void BufferedAudioSink::feedPCMFrames(const uint8_t* buffer, size_t bytes) {
    i2s_write(I2S_NUM_0, buffer, bytes, &written, portMAX_DELAY);
}
```

Updated (v6.0, new driver):
```cpp
#include "driver/i2s_std.h"

class BufferedAudioSink {
private:
    i2s_chan_handle_t tx_handle = NULL;
};

BufferedAudioSink::BufferedAudioSink() {
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
            I2S_DATA_BIT_WIDTH_32BIT,
            I2S_SLOT_MODE_STEREO
        ),
        .gpio_cfg = {...},
    };
    
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
    ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
    startI2sFeed();
}

void BufferedAudioSink::feedPCMFrames(const uint8_t* buffer, size_t bytes) {
    size_t written = 0;
    ESP_ERROR_CHECK(i2s_channel_write(tx_handle, buffer, bytes, &written, portMAX_DELAY));
}

BufferedAudioSink::~BufferedAudioSink() {
    i2s_channel_disable(tx_handle);
    i2s_del_channel(tx_handle);
}
```

---

## GPIO Configuration Variations

### If you need to use different pins:

```cpp
// Alternative pin configuration
.gpio_cfg = {
    .mclk = GPIO_NUM_0,    // Or -1 to disable MCLK
    .bclk = GPIO_NUM_4,    // Your BCK pin
    .ws = GPIO_NUM_5,      // Your LRCK pin
    .dout = GPIO_NUM_18,   // Your DATA pin
    .din = GPIO_NUM_-1,    // Disable DIN for TX-only
},

// If DAC needs inverted signals (check datasheet):
.invert_flags = {
    .mclk_inv = false,
    .bclk_inv = false,     // Set true if needed
    .ws_inv = false,       // Set true if needed
},
```

---

## Troubleshooting

### Silent Audio

```cpp
// Check if channel is enabled
assert(tx_handle != NULL);

// Verify GPIO pins are correct for your DAC
// Try writing test data (pattern or silence)
uint8_t test_data[4096] = {0};  // Silence
i2s_channel_write(tx_handle, test_data, sizeof(test_data), &written, 1000);
```

### Audio Dropout on Sample Rate Change

```cpp
// OLD (causes dropout):
i2s_set_clk(I2S_NUM_0, new_rate, ...);  // v5.x

// NEW (seamless):
i2s_std_clk_config_t clk = I2S_STD_CLK_DEFAULT_CONFIG(new_rate);
i2s_channel_reconfig_clk(tx_handle, &clk);  // v6.0 - no dropout!
```

### Buffer Overflow

```cpp
// Ring buffer too small
xRingbufferCreate(256 * 1024, RINGBUF_TYPE_BYTEBUF);  // Increase to 512 KB

// Check timeout in write
BaseType_t ret = xRingbufferSend(ring_buffer, data, len, pdMS_TO_TICKS(1000));
if (ret != pdTRUE) {
    ESP_LOGW("AUDIO", "Ring buffer timeout - increase size");
}
```

---

## Verification Checklist

- [ ] Include correct header: `#include "driver/i2s_std.h"`
- [ ] GPIO pins match DAC datasheet (LRCK=25, BCK=26, DATA=13)
- [ ] I2S format matches DAC (Philips vs MSB for ES9018)
- [ ] Bit depth set to 32: `I2S_DATA_BIT_WIDTH_32BIT`
- [ ] Sample rate supported by DAC (typically 44.1 - 192 kHz)
- [ ] Channel enabled after allocation
- [ ] Channel disabled and deleted on cleanup
- [ ] Ring buffer sized for network jitter (256 KB minimum)
- [ ] PSRAM available and accessible for buffers

---

## Performance Tips

1. **Pin PSRAM allocation to I2S task**:
   ```cpp
   xTaskCreatePinnedToCore(i2s_feed_task, "i2s", 4096, NULL, 10, NULL, 0);
   ```

2. **Use larger DMA chunks** to reduce ISR overhead:
   ```cpp
   chan_cfg.dma_desc_num = 8;
   chan_cfg.dma_frame_num = 1024;  // 8 KB per transfer
   ```

3. **Monitor clock accuracy** with logic analyzer:
   - BCLK should be stable and match calculated frequency
   - LRCK should alternate at sample rate

4. **Use APLL for precise audio clocking**:
   ```cpp
   .clk_cfg.clk_src = I2S_CLK_SRC_APLL;  // Not strictly needed, but recommended
   ```
