# I2S Driver Migration Code Examples for cspot

## Table of Contents
1. [Generic I2S STD Mode Pattern](#generic-i2s-std-mode-pattern)
2. [BufferedAudioSink.cpp Migration](#bufferedaudiosink-migration)
3. [ExternalDAC Audio Sinks Migration](#external-dac-audio-sinks)
4. [Handle Management Patterns](#handle-management-patterns)
5. [Sample Rate Switching](#sample-rate-switching)
6. [Error Handling](#error-handling)

---

## Generic I2S STD Mode Pattern

### Minimal Working Example

```cpp
#include "driver/i2s_std.h"

void setup_i2s_output(void) {
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(
        I2S_NUM_0,           // I2S port number
        I2S_ROLE_MASTER      // Master mode (ESP32 is clock source)
    );
    
    // Configure as standard Philips I2S format
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),  // 44.1 kHz
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
            I2S_DATA_BIT_WIDTH_16BIT,  // 16-bit samples
            I2S_SLOT_MODE_STEREO       // Stereo (2 channels)
        ),
        .gpio_cfg = {
            .mclk = GPIO_NUM_0,   // Master clock GPIO
            .bclk = GPIO_NUM_4,   // Bit clock GPIO
            .ws = GPIO_NUM_25,    // Word select (LR clock) GPIO
            .dout = GPIO_NUM_26,  // Data out GPIO
            .din = GPIO_NUM_-1,   // Data in (set to -1 if RX not used)
            .invert_flags = {
                .mclk_inv = false,
                .bclk_inv = false,
                .ws_inv = false,
            },
        },
    };
    
    // Allocate a new TX channel
    i2s_chan_handle_t tx_handle = NULL;
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
    
    // Enable the I2S channel
    ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
    
    // Now ready to write audio data
    return tx_handle;
}

void write_audio_data(i2s_chan_handle_t handle, uint8_t *data, size_t len) {
    size_t bytes_written = 0;
    ESP_ERROR_CHECK(i2s_channel_write(
        handle,
        data,
        len,
        &bytes_written,
        portMAX_DELAY
    ));
}
```

---

## BufferedAudioSink Migration

### Before (ESP-IDF 5.x)

```cpp
#include "BufferedAudioSink.h"
#include "driver/i2s.h"

BufferedAudioSink::BufferedAudioSink() {
    // Create I2S configuration
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
        .tx_desc_auto_clear = true,
        .fixed_mclk = -1
    };
    
    // Install I2S driver
    i2s_driver_install((i2s_port_t)0, &i2s_config, 0, NULL);
    
    startI2sFeed();
}

BufferedAudioSink::~BufferedAudioSink() {
    i2s_driver_uninstall((i2s_port_t)0);
}

void BufferedAudioSink::feedPcm(const std::vector<uint8_t> &data) {
    size_t written = 0;
    i2s_write((i2s_port_t)0, data.data(), data.size(), &written, portMAX_DELAY);
}

void BufferedAudioSink::setProperties(uint32_t sampleRate, uint8_t bitDepth, uint8_t channelCount) {
    i2s_set_clk(
        (i2s_port_t)0,
        sampleRate,
        (i2s_bits_per_sample_t)bitDepth,
        (i2s_channel_t)channelCount
    );
}
```

### After (ESP-IDF 6.0)

```cpp
#include "BufferedAudioSink.h"
#include "driver/i2s_std.h"

class BufferedAudioSink {
private:
    i2s_chan_handle_t tx_handle = NULL;
};

BufferedAudioSink::BufferedAudioSink() {
    // Channel configuration
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(
        I2S_NUM_0,
        I2S_ROLE_MASTER
    );
    
    // Standard I2S mode with Philips format
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
            I2S_DATA_BIT_WIDTH_16BIT,
            I2S_SLOT_MODE_STEREO
        ),
        .gpio_cfg = {
            .mclk = GPIO_NUM_0,
            .bclk = GPIO_NUM_4,
            .ws = GPIO_NUM_25,
            .dout = GPIO_NUM_26,
            .din = GPIO_NUM_-1,
            .invert_flags = {.mclk_inv = false, .bclk_inv = false, .ws_inv = false},
        },
    };
    
    // Allocate TX channel
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
    
    // Enable I2S channel
    ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
    
    startI2sFeed();
}

BufferedAudioSink::~BufferedAudioSink() {
    if (tx_handle != NULL) {
        i2s_channel_disable(tx_handle);
        i2s_del_channel(tx_handle);
        tx_handle = NULL;
    }
}

void BufferedAudioSink::feedPcm(const std::vector<uint8_t> &data) {
    size_t written = 0;
    ESP_ERROR_CHECK(i2s_channel_write(
        tx_handle,
        data.data(),
        data.size(),
        &written,
        portMAX_DELAY
    ));
}

void BufferedAudioSink::setProperties(uint32_t sampleRate, uint8_t bitDepth, uint8_t channelCount) {
    // Reconfigure clock for new sample rate
    i2s_std_clk_config_t clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(sampleRate);
    ESP_ERROR_CHECK(i2s_channel_reconfig_clk(tx_handle, &clk_cfg));
}
```

---

## External DAC Audio Sinks

### ES8311 Audio Codec (Before)

```cpp
// From: cspot/bell/main/audio-sinks/esp/ES8311AudioSink.cpp
#include "driver/i2s.h"
#include "driver/gpio.h"
#include "es8311.h"

class ES8311AudioSink : public BufferedAudioSink {
private:
    void initializeI2S();
};

void ES8311AudioSink::initializeI2S() {
    // I2S configuration
    i2s_config_t i2s_config = {
        .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_TX),
        .sample_rate = (i2s_bits_per_sample_t)44100,
        .bits_per_sample = I2S_BITS_PER_SAMPLE_16BIT,
        .channel_format = I2S_CHANNEL_FMT_RIGHT_LEFT,
        .communication_format = (i2s_comm_format_t)I2S_COMM_FORMAT_STAND_I2S,
        .intr_alloc_flags = 0,
        .dma_buf_count = 4,
        .dma_buf_len = 1024,
        .use_apll = true,
        .tx_desc_auto_clear = true,
        .fixed_mclk = 0  // Use APLL
    };
    
    // Pin configuration
    i2s_pin_config_t pin_config = {
        .bck_io_num = 4,
        .ws_io_num = 25,
        .data_out_num = 26,
        .data_in_num = 35,
    };
    
    // Install driver
    i2s_driver_install((i2s_port_t)0, &i2s_config, 0, NULL);
    i2s_set_pin((i2s_port_t)0, &pin_config);
    
    // Initialize ES8311 codec
    es8311_init();
    es8311_set_bits_per_sample(16);
    es8311_config_fmt(ES_I2S_PHILIPS);
}
```

### ES8311 Audio Codec (After)

```cpp
#include "driver/i2s_std.h"
#include "driver/gpio.h"
#include "es8311.h"

class ES8311AudioSink : public BufferedAudioSink {
private:
    i2s_chan_handle_t tx_handle = NULL;
    void initializeI2S();
};

void ES8311AudioSink::initializeI2S() {
    // Channel configuration
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(
        I2S_NUM_0,
        I2S_ROLE_MASTER
    );
    
    // Standard Philips I2S format
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
            I2S_DATA_BIT_WIDTH_16BIT,
            I2S_SLOT_MODE_STEREO
        ),
        .gpio_cfg = {
            .mclk = GPIO_NUM_0,    // or GPIO_NUM_-1 if not used
            .bclk = GPIO_NUM_4,
            .ws = GPIO_NUM_25,
            .dout = GPIO_NUM_26,
            .din = GPIO_NUM_35,
            .invert_flags = {
                .mclk_inv = false,
                .bclk_inv = false,
                .ws_inv = false,
            },
        },
    };
    
    // Allocate TX channel
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
    
    // Enable I2S channel
    ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
    
    // Initialize ES8311 codec
    es8311_init();
    es8311_set_bits_per_sample(16);
    es8311_config_fmt(ES_I2S_PHILIPS);
}

ES8311AudioSink::~ES8311AudioSink() {
    if (tx_handle != NULL) {
        i2s_channel_disable(tx_handle);
        i2s_del_channel(tx_handle);
        tx_handle = NULL;
    }
}
```

---

## Handle Management Patterns

### Pattern 1: Class Member Variable (Recommended)

```cpp
// Header file
#include "driver/i2s_std.h"

class AudioSink {
private:
    i2s_chan_handle_t tx_handle = NULL;
    void initI2S();
    void cleanupI2S();
public:
    AudioSink();
    ~AudioSink();
    void writeAudio(const uint8_t *data, size_t len);
};

// Implementation
AudioSink::AudioSink() {
    initI2S();
}

AudioSink::~AudioSink() {
    cleanupI2S();
}

void AudioSink::initI2S() {
    // ... configuration code ...
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
    ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
}

void AudioSink::cleanupI2S() {
    if (tx_handle != NULL) {
        ESP_ERROR_CHECK(i2s_channel_disable(tx_handle));
        ESP_ERROR_CHECK(i2s_del_channel(tx_handle));
        tx_handle = NULL;
    }
}

void AudioSink::writeAudio(const uint8_t *data, size_t len) {
    size_t bytes_written = 0;
    ESP_ERROR_CHECK(i2s_channel_write(tx_handle, data, len, &bytes_written, portMAX_DELAY));
}
```

### Pattern 2: Static Handle (For Single I2S Instance)

```cpp
// In main.cpp or dedicated I2S management module
static i2s_chan_handle_t g_i2s_tx_handle = NULL;

esp_err_t init_i2s_global() {
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
    i2s_std_config_t std_cfg = {
        // ... configuration ...
    };
    
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &g_i2s_tx_handle));
    return i2s_channel_enable(g_i2s_tx_handle);
}

void write_i2s_data(const uint8_t *data, size_t len) {
    size_t bytes_written = 0;
    i2s_channel_write(g_i2s_tx_handle, data, len, &bytes_written, portMAX_DELAY);
}

void cleanup_i2s_global() {
    if (g_i2s_tx_handle != NULL) {
        i2s_channel_disable(g_i2s_tx_handle);
        i2s_del_channel(g_i2s_tx_handle);
        g_i2s_tx_handle = NULL;
    }
}
```

---

## Sample Rate Switching

### Before (ESP-IDF 5.x)

```cpp
// Dynamic sample rate change during playback
void AudioSink::changeSampleRate(uint32_t new_rate) {
    i2s_set_clk(
        (i2s_port_t)0,
        new_rate,
        I2S_BITS_PER_SAMPLE_16BIT,
        I2S_CHANNEL_STEREO
    );
}
```

### After (ESP-IDF 6.0)

```cpp
void AudioSink::changeSampleRate(uint32_t new_rate) {
    // Create new clock configuration for the new sample rate
    i2s_std_clk_config_t new_clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(new_rate);
    
    // Reconfigure the I2S clock with the new settings
    ESP_ERROR_CHECK(i2s_channel_reconfig_clk(tx_handle, &new_clk_cfg));
    
    ESP_LOGI(TAG, "Sample rate changed to %lu Hz", new_rate);
}
```

### Complete Audio Properties Update Example

```cpp
void AudioSink::updateAudioProperties(
    uint32_t sample_rate,
    uint8_t bit_depth,
    uint8_t channel_count) {
    
    // Update sample rate if changed
    i2s_std_clk_config_t new_clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(sample_rate);
    ESP_ERROR_CHECK(i2s_channel_reconfig_clk(tx_handle, &new_clk_cfg));
    
    // Note: Bit depth and channel count are typically configured at initialization
    // and can't be changed dynamically. If these need to change, you would need to:
    // 1. Disable current channel
    // 2. Delete current channel
    // 3. Reconfigure and create new channel with new settings
    // This is rarely needed in typical audio playback scenarios
    
    ESP_LOGI(TAG, "Audio config: %luHz, %u-bit, %u-ch", 
             sample_rate, bit_depth, channel_count);
}
```

---

## Error Handling

### Robust Audio Initialization

```cpp
#include "driver/i2s_std.h"
#include "esp_log.h"

static const char *TAG = "I2S_AUDIO";

typedef struct {
    i2s_chan_handle_t tx_handle;
    bool is_enabled;
} i2s_context_t;

esp_err_t init_i2s_robust(i2s_context_t *ctx) {
    if (ctx == NULL) {
        ESP_LOGE(TAG, "Invalid context pointer");
        return ESP_ERR_INVALID_ARG;
    }
    
    // Channel configuration
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
    
    // Standard I2S configuration
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
            I2S_DATA_BIT_WIDTH_16BIT,
            I2S_SLOT_MODE_STEREO
        ),
        .gpio_cfg = {
            .mclk = GPIO_NUM_0,
            .bclk = GPIO_NUM_4,
            .ws = GPIO_NUM_25,
            .dout = GPIO_NUM_26,
            .din = GPIO_NUM_-1,
            .invert_flags = {false, false, false},
        },
    };
    
    // Allocate channel
    esp_err_t err = i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &ctx->tx_handle);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Failed to allocate I2S TX channel: %s", esp_err_to_name(err));
        return err;
    }
    
    ESP_LOGI(TAG, "I2S TX channel allocated");
    
    // Enable channel
    err = i2s_channel_enable(ctx->tx_handle);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Failed to enable I2S channel: %s", esp_err_to_name(err));
        i2s_del_channel(ctx->tx_handle);
        ctx->tx_handle = NULL;
        return err;
    }
    
    ESP_LOGI(TAG, "I2S channel enabled");
    ctx->is_enabled = true;
    
    return ESP_OK;
}

esp_err_t write_i2s_robust(i2s_context_t *ctx, const uint8_t *data, size_t len) {
    if (ctx == NULL || ctx->tx_handle == NULL || !ctx->is_enabled) {
        ESP_LOGE(TAG, "I2S not initialized or not enabled");
        return ESP_ERR_INVALID_STATE;
    }
    
    if (data == NULL || len == 0) {
        ESP_LOGE(TAG, "Invalid data pointer or length");
        return ESP_ERR_INVALID_ARG;
    }
    
    size_t bytes_written = 0;
    esp_err_t err = i2s_channel_write(ctx->tx_handle, data, len, &bytes_written, portMAX_DELAY);
    
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "I2S write failed: %s", esp_err_to_name(err));
        return err;
    }
    
    if (bytes_written != len) {
        ESP_LOGW(TAG, "Partial write: %u / %u bytes", bytes_written, len);
    }
    
    return ESP_OK;
}

esp_err_t cleanup_i2s_robust(i2s_context_t *ctx) {
    if (ctx == NULL || ctx->tx_handle == NULL) {
        return ESP_OK;
    }
    
    esp_err_t err = ESP_OK;
    
    if (ctx->is_enabled) {
        err = i2s_channel_disable(ctx->tx_handle);
        if (err != ESP_OK) {
            ESP_LOGW(TAG, "Failed to disable I2S channel: %s", esp_err_to_name(err));
        }
        ctx->is_enabled = false;
    }
    
    err = i2s_del_channel(ctx->tx_handle);
    if (err != ESP_OK) {
        ESP_LOGW(TAG, "Failed to delete I2S channel: %s", esp_err_to_name(err));
    }
    
    ctx->tx_handle = NULL;
    ESP_LOGI(TAG, "I2S cleanup complete");
    
    return ESP_OK;
}
```

---

## I2S Slot Configuration Variations

### PCM (3-Wire Mode)

```cpp
// For non-standard I2S (3-wire PCM mode, sometimes used by low-cost DACs)
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_PCM_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_16BIT,
        I2S_SLOT_MODE_STEREO
    ),
    // ... gpio_cfg ...
};
```

### MSB-First (Big-Endian) Mode

```cpp
// For DACs that require MSB-first format
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_MSB_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_16BIT,
        I2S_SLOT_MODE_STEREO
    ),
    // ... gpio_cfg ...
};
```

### 24-bit Depth

```cpp
// For high-resolution audio (24-bit)
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(48000),
    .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_24BIT,  // 24-bit instead of 16-bit
        I2S_SLOT_MODE_STEREO
    ),
    // ... gpio_cfg ...
};
```

### Mono Mode

```cpp
// For mono audio (single channel)
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_16BIT,
        I2S_SLOT_MODE_MONO  // Mono instead of stereo
    ),
    // ... gpio_cfg ...
};
```

---

## Migration Troubleshooting

### Problem: Compilation Error - "i2s_driver_install not found"
**Solution:** Replace `#include "driver/i2s.h"` with `#include "driver/i2s_std.h"` and use the new API.

### Problem: Audio Not Playing (Silent)
**Solution:** Check GPIO configuration and ensure I2S channel is enabled:
```cpp
// Verify handles are not NULL
assert(tx_handle != NULL);

// Check if channel is enabled
// Try writing test data
```

### Problem: Sample Rate Change Causes Audio Dropout
**Solution:** Use `i2s_channel_reconfig_clk()` without disabling/re-enabling:
```cpp
i2s_std_clk_config_t new_clk = I2S_STD_CLK_DEFAULT_CONFIG(new_rate);
ESP_ERROR_CHECK(i2s_channel_reconfig_clk(tx_handle, &new_clk));
```

### Problem: "ESP_ERR_INVALID_STATE" on i2s_channel_write()
**Solution:** Ensure channel is enabled:
```cpp
ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
```

### Problem: Memory Leak on Cleanup
**Solution:** Always call both disable and delete:
```cpp
ESP_ERROR_CHECK(i2s_channel_disable(tx_handle));
ESP_ERROR_CHECK(i2s_del_channel(tx_handle));
tx_handle = NULL;
```

---

## Quick Copy-Paste Templates

### Template 1: Basic TX-Only Audio Sink

```cpp
class AudioSink {
private:
    i2s_chan_handle_t tx_handle = NULL;
    uint32_t current_sample_rate = 44100;
    
public:
    AudioSink();
    ~AudioSink();
    
    void initialize();
    void deinitialize();
    void write(const uint8_t *data, size_t len);
    void setSampleRate(uint32_t rate);
};

AudioSink::AudioSink() = default;
AudioSink::~AudioSink() { deinitialize(); }

void AudioSink::initialize() {
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(current_sample_rate),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_STEREO),
        .gpio_cfg = {.mclk = GPIO_NUM_0, .bclk = GPIO_NUM_4, .ws = GPIO_NUM_25, .dout = GPIO_NUM_26, .din = GPIO_NUM_-1},
    };
    ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
    ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
}

void AudioSink::deinitialize() {
    if (tx_handle) { i2s_channel_disable(tx_handle); i2s_del_channel(tx_handle); tx_handle = NULL; }
}

void AudioSink::write(const uint8_t *data, size_t len) {
    size_t bytes_written = 0;
    i2s_channel_write(tx_handle, data, len, &bytes_written, portMAX_DELAY);
}

void AudioSink::setSampleRate(uint32_t rate) {
    current_sample_rate = rate;
    i2s_std_clk_config_t clk = I2S_STD_CLK_DEFAULT_CONFIG(rate);
    ESP_ERROR_CHECK(i2s_channel_reconfig_clk(tx_handle, &clk));
}
```

---

**Last Updated:** 2024  
**For:** cspot ESP32 Audio Streaming Project  
**Scope:** I2S Driver Migration Examples
