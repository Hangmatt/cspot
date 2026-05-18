# I2S Audio Output Research: 32-bit Lossless on ESP-IDF v6.0

**Objective:** Support high-quality Spotify audio playback (320 kbps ogg vorbis / HiFi FLAC) at 32-bit/192kHz capable specifications with pin configuration: LRCK=GPIO25, BCK=GPIO26, DATA=GPIO13.

---

## 1. ESP-IDF v6.0 Modern I2S Driver API

### API Overview

ESP-IDF v6.0 introduces a channel-based I2S driver that replaces the older monolithic API:

#### Old API (v5.x) - No Longer Recommended
```cpp
#include "driver/i2s.h"

// Single global I2S initialization
i2s_config_t cfg = {...};
i2s_driver_install(I2S_NUM_0, &cfg, 0, NULL);
i2s_set_pin(I2S_NUM_0, &pin_cfg);
i2s_write(...);
```

#### New API (v6.0) - Recommended for v6.0+
```cpp
#include "driver/i2s_std.h"

// Channel-based, supports multiple independent I2S streams
i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(...);
i2s_std_config_t std_cfg = {...};
i2s_chan_handle_t tx_handle = NULL;
i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle);
i2s_channel_enable(tx_handle);
i2s_channel_write(tx_handle, data, len, &bytes_written, timeout);
```

### Key Advantages of v6.0 API
- **Independent channels**: Multiple I2S TX/RX channels without conflicts
- **Dynamic reconfiguration**: Change sample rates without disabling (less audio dropout)
- **Better error handling**: Explicit error codes for debugging
- **PSRAM-friendly**: Optimized DMA buffer management for extended memory

---

## 2. Pin Configuration: LRCK=GPIO25, BCK=GPIO26, DATA=GPIO13

### Standard ESP-IDF v6.0 GPIO Setup

```cpp
#include "driver/gpio.h"
#include "driver/i2s_std.h"

// Recommended GPIO configuration for the specified pins
i2s_std_config_t std_cfg = {
    .gpio_cfg = {
        .mclk = GPIO_NUM_0,        // Master Clock (optional, can be -1 to disable)
        .bclk = GPIO_NUM_26,       // Bit Clock (your BCK)
        .ws = GPIO_NUM_25,         // Word Select (your LRCK)
        .dout = GPIO_NUM_13,       // Data Out (your DATA)
        .din = GPIO_NUM_-1,        // Data In (not used for TX-only)
        .invert_flags = {
            .mclk_inv = false,
            .bclk_inv = false,
            .ws_inv = false,
        },
    },
};
```

### Pin Function Mapping

| ESP32 Pin | I2S Signal | Purpose | Your Config |
|-----------|-----------|---------|-------------|
| GPIO0 | MCLK | Master Clock (optional) | Optional |
| GPIO26 | BCLK | Bit Clock for syncing data | BCK ✓ |
| GPIO25 | WS/LRCK | Left/Right Clock for channel selection | LRCK ✓ |
| GPIO13 | DOUT | Audio data output | DATA ✓ |
| GPIO-1 | DIN | Data input (not needed for TX) | N/A |

### Clock Frequency Calculations

For 44.1 kHz, 32-bit, Stereo:
- **BCLK frequency** = Sample_Rate × Slot_Bits × Channel_Count
- **BCLK** = 44,100 Hz × 32 bits × 2 = **2.8224 MHz**
- **MCLK** = 256 × Sample_Rate (APLL default)
- **MCLK** = 256 × 44,100 = **11.2896 MHz** (APLL handles this automatically)

For 96 kHz, 32-bit, Stereo:
- **BCLK** = 96,000 × 32 × 2 = **6.144 MHz**
- **MCLK** = 256 × 96,000 = **24.576 MHz**

ESP-IDF v6.0 APLL auto-configures these frequencies based on sample rate.

---

## 3. Audio Codec Options Supporting 32-bit Lossless

### Recommended Codecs for Spotify-Quality Audio

#### **1. ES9018 (Audiophile DAC) - BEST FOR QUALITY**

**Specifications:**
- Bit Depth: 32-bit native support
- Sample Rates: Up to 192 kHz
- THD+N: < -110 dB
- Signal-to-Noise: > 120 dB
- I2S Format: MSB-First (Big-Endian)
- Control: I2C interface

**Best For:** Premium audio applications, Spotify HiFi

**Configuration Example:**
```cpp
#include "driver/i2s_std.h"

// ES9018 requires MSB-first I2S format
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),  // Supports up to 192kHz
    .slot_cfg = I2S_STD_MSB_SLOT_DEFAULT_CONFIG(   // MSB format for ES9018
        I2S_DATA_BIT_WIDTH_32BIT,                   // 32-bit samples
        I2S_SLOT_MODE_STEREO                        // Stereo
    ),
    .gpio_cfg = {
        .mclk = GPIO_NUM_0,
        .bclk = GPIO_NUM_26,
        .ws = GPIO_NUM_25,
        .dout = GPIO_NUM_13,
        .din = GPIO_NUM_-1,
    },
};

// I2C control initialization (for ES9018 control registers)
i2c_config_t i2c_cfg = {
    .mode = I2C_MODE_MASTER,
    .sda_io_num = GPIO_NUM_21,
    .scl_io_num = GPIO_NUM_22,
};
```

**Integration Notes:**
- No external control required for basic audio (I2C optional for advanced features)
- Pin-strapped for I2S master slave mode operation
- Directly accepts 32-bit I2S data without intermediate codec conversion

---

#### **2. PCM5102 (TI Budget DAC) - BEST VALUE**

**Specifications:**
- Bit Depth: 16/24/32-bit support
- Sample Rates: Up to 192 kHz
- THD+N: < -100 dB
- Signal-to-Noise: > 110 dB
- I2S Format: Philips Standard (LSB-first)
- Control: Direct GPIO pin-strapping (no I2C needed)

**Best For:** Cost-conscious, still excellent quality

**Configuration Example:**
```cpp
#include "driver/i2s_std.h"

// PCM5102 uses standard Philips I2S format
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_32BIT,      // 32-bit support
        I2S_SLOT_MODE_STEREO
    ),
    .gpio_cfg = {
        .mclk = GPIO_NUM_0,            // PCM5102 uses MCLK for stable operation
        .bclk = GPIO_NUM_26,
        .ws = GPIO_NUM_25,
        .dout = GPIO_NUM_13,
        .din = GPIO_NUM_-1,
    },
};

// PCM5102 control (GPIO pin-strapping, no I2C)
// Set via GPIO during power-on:
// XMT (GPIO) = LOW  → Normal operation
// FMT (GPIO) = LOW  → I2S format
// DEL (GPIO) = LOW  → No delay
```

**Advantages:**
- No I2C communication needed (simpler PCB)
- Pin-strapping for all configuration
- Excellent compatibility with ESP32

---

#### **3. ES8388 (Integrated Codec) - GOOD COMPATIBILITY**

**Specifications:**
- Bit Depth: 16/24-bit standard (32-bit via TDM mode)
- Sample Rates: Up to 192 kHz
- Features: ADC + DAC (record + playback)
- Control: I2C interface required
- Common on: Many ESP32 dev boards

**Configuration Example:**
```cpp
#include "driver/i2s_std.h"
#include "driver/i2c_master.h"

// ES8388 uses standard Philips I2S
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_24BIT,      // 24-bit max standard
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

// I2C initialization for ES8388 control
i2c_config_t i2c_cfg = {
    .mode = I2C_MODE_MASTER,
    .sda_io_num = GPIO_NUM_21,    // Typical ESP32 I2C SDA
    .scl_io_num = GPIO_NUM_22,    // Typical ESP32 I2C SCL
};

// Register configuration for DAC output (simplified):
// Write to ES8388_DACCONTROL1 = 0x18  // 16/24-bit, I2S format
// Write to ES8388_DACPOWER = 0x3c     // Power up DAC
```

**Advantages:**
- Integrates ADC + DAC (can record and play simultaneously)
- Common on existing boards
- I2C configuration for fine control (gain, EQ, power management)

---

#### **4. TAS5711 (Amplified DAC) - FOR POWERED SPEAKERS**

**Specifications:**
- Bit Depth: 32-bit support
- Sample Rates: Up to 192 kHz
- Power: Built-in Class D amplifier (up to 2×30W)
- Control: I2C interface
- Use Case: Direct speaker connection

**Configuration Example:**
```cpp
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_32BIT,      // 32-bit support
        I2S_SLOT_MODE_STEREO
    ),
    // ... gpio_cfg as above ...
};
```

---

### Codec Comparison Table

| Feature | ES9018 | PCM5102 | ES8388 | TAS5711 |
|---------|--------|---------|--------|---------|
| **32-bit Support** | ✓ | ✓ | ✓* | ✓ |
| **Up to 192 kHz** | ✓ | ✓ | ✓ | ✓ |
| **THD+N** | < -110dB | < -100dB | ~-98dB | ~-95dB |
| **I2C Control** | Optional | No | Required | Required |
| **Form Factor** | SSOP24 | SSOP24 | LQFP48 | LQFP64 |
| **Cost** | $$ | $ | $ | $$$ |
| **Spotify HiFi Ready** | YES | YES | Partial* | YES |

\* ES8388 can do 32-bit in TDM mode, but 24-bit is standard Philips I2S

---

## 4. Buffer Sizing for 32-bit Audio with 8MB PSRAM

### Memory Calculation Framework

For **44.1 kHz, 32-bit, Stereo**:
- **Sample rate**: 44,100 samples/second
- **Bits per sample**: 32 bits × 2 channels = 64 bits = 8 bytes
- **Bytes per second**: 44,100 × 8 = **352,800 bytes/sec** ≈ **344 KB/sec**

For **96 kHz, 32-bit, Stereo**:
- **Bytes per second**: 96,000 × 8 = **768,000 bytes/sec** ≈ **750 KB/sec**

For **192 kHz, 32-bit, Stereo**:
- **Bytes per second**: 192,000 × 8 = **1,536,000 bytes/sec** ≈ **1.5 MB/sec**

### Recommended Buffer Configuration

```cpp
#include "driver/i2s_std.h"

// Channel configuration with optimized DMA buffering for PSRAM
i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(
    I2S_NUM_0,           // I2S port
    I2S_ROLE_MASTER      // ESP32 as clock source
);

// Optional: Configure DMA buffer explicitly for PSRAM
// (ESP-IDF v6.0 auto-selects based on available memory)
chan_cfg.dma_desc_num = 8;      // Number of DMA descriptors
chan_cfg.dma_frame_num = 1024;  // Samples per DMA descriptor

// For 32-bit stereo at 44.1 kHz:
// DMA buffer size per descriptor = 1024 samples × 8 bytes = 8 KB
// Total DMA memory = 8 KB × 8 descriptors = 64 KB (internal SRAM)
// Ring buffer (software): 256 KB - 1 MB (can use PSRAM if available)

i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_32BIT,
        I2S_SLOT_MODE_STEREO
    ),
    // ... gpio_cfg ...
};

i2s_chan_handle_t tx_handle = NULL;
ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
```

### Software Ring Buffer for Audio Stream

For cspot's Spotify playback (typical streaming scenario):

```cpp
#include "freertos/ringbuf.h"

// Ring buffer size: 256 KB for smooth streaming
// At 44.1 kHz, 32-bit: 256 KB ÷ 344 KB/sec ≈ 0.74 seconds of audio
// Good for network jitter (WiFi latency variations)

static RingbufHandle_t audio_ring_buffer = NULL;

esp_err_t init_audio_buffers(void) {
    // Create 256 KB ring buffer in PSRAM/SRAM (auto-selected)
    audio_ring_buffer = xRingbufferCreate(
        256 * 1024,              // 256 KB buffer
        RINGBUF_TYPE_BYTEBUF     // Byte-oriented (best for audio)
    );
    
    if (!audio_ring_buffer) {
        ESP_LOGE("AUDIO", "Failed to create ring buffer");
        return ESP_ERR_NO_MEM;
    }
    
    return ESP_OK;
}

// Feed incoming PCM data from Spotify decoder
esp_err_t buffer_pcm_frame(const uint8_t *data, size_t len) {
    BaseType_t ret = xRingbufferSend(
        audio_ring_buffer,
        (void *)data,
        len,
        pdMS_TO_TICKS(100)  // 100ms timeout if buffer full
    );
    
    if (ret != pdTRUE) {
        ESP_LOGW("AUDIO", "Ring buffer overflow (timeout)");
        return ESP_ERR_NO_MEM;
    }
    
    return ESP_OK;
}

// Background task: Read from ring buffer and feed to I2S
static void i2s_feed_task(void *arg) {
    i2s_chan_handle_t tx_handle = (i2s_chan_handle_t)arg;
    
    while (1) {
        size_t item_size;
        uint8_t *item = (uint8_t *)xRingbufferReceiveUpTo(
            audio_ring_buffer,
            &item_size,
            portMAX_DELAY,  // Block forever if buffer empty
            8192            // Max 8 KB chunks (DMA friendly)
        );
        
        if (item) {
            size_t bytes_written = 0;
            esp_err_t err = i2s_channel_write(
                tx_handle,
                item,
                item_size,
                &bytes_written,
                portMAX_DELAY
            );
            
            vRingbufferReturnItem(audio_ring_buffer, (void *)item);
            
            if (err != ESP_OK) {
                ESP_LOGE("I2S", "I2S write failed: %s", esp_err_to_name(err));
            }
        }
    }
}
```

### Buffer Size Recommendations by Use Case

| Use Case | Sample Rate | Buffer Size | Latency | Notes |
|----------|-------------|-------------|---------|-------|
| **Spotify Premium** | 44.1 kHz | 256 KB | 0.74s | WiFi streaming |
| **Spotify HiFi** | 44.1 kHz | 512 KB | 1.48s | Network resilience |
| **HD Audio** | 96 kHz | 256 KB | 0.33s | Professional |
| **Ultra HD** | 192 kHz | 512 KB | 0.33s | Studio quality |
| **Low Latency** | 44.1 kHz | 64 KB | 0.18s | Real-time monitoring |

---

## 5. ESP-IDF v6.0 I2S Data Bit Width Support

### Supported Bit Depths

ESP-IDF v6.0 I2S standard mode supports:

```cpp
typedef enum {
    I2S_DATA_BIT_WIDTH_8BIT = 8,     // 8-bit (rarely used for audio)
    I2S_DATA_BIT_WIDTH_16BIT = 16,   // 16-bit (most common, CD quality)
    I2S_DATA_BIT_WIDTH_24BIT = 24,   // 24-bit (high-resolution audio)
    I2S_DATA_BIT_WIDTH_32BIT = 32,   // 32-bit (maximum precision, professional)
} i2s_data_bit_width_t;
```

### 32-bit Configuration for Lossless Audio

```cpp
#include "driver/i2s_std.h"

// Full 32-bit I2S configuration for high-quality audio
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),  // Sample rate
    .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_32BIT,      // 32-bit samples
        I2S_SLOT_MODE_STEREO           // 2 channels (L/R)
    ),
    .gpio_cfg = {
        .mclk = GPIO_NUM_0,            // Master clock
        .bclk = GPIO_NUM_26,           // Bit clock
        .ws = GPIO_NUM_25,             // Word select (LRCK)
        .dout = GPIO_NUM_13,           // Data out
        .din = GPIO_NUM_-1,            // No input (TX-only)
        .invert_flags = {
            .mclk_inv = false,
            .bclk_inv = false,
            .ws_inv = false,
        },
    },
};

// Allocate and enable I2S channel
i2s_chan_handle_t tx_handle = NULL;
ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));
```

### Audio Quality vs Bit Depth

| Bit Depth | Use Case | Dynamic Range | Quality Level |
|-----------|----------|----------------|---------------|
| **8-bit** | Telephony | 48 dB | Very Low |
| **16-bit** | CD Audio (Spotify Premium) | 96 dB | Good |
| **24-bit** | High-Resolution Audio | 144 dB | Excellent |
| **32-bit** | Professional/Mastering | 192 dB | Studio Quality |

**For Spotify HiFi**: 24-bit is technically sufficient, but 32-bit provides headroom for DSP processing.

---

## 6. Concrete I2S Initialization Code Examples

### Example 1: Basic 32-bit I2S Setup (44.1 kHz, PCM5102)

```cpp
#include "driver/i2s_std.h"
#include "driver/gpio.h"
#include "esp_log.h"

static const char *TAG = "I2S_SETUP";

esp_err_t init_i2s_output_32bit(i2s_chan_handle_t *out_handle) {
    // Channel configuration
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(
        I2S_NUM_0,
        I2S_ROLE_MASTER
    );

    // Standard I2S configuration for 32-bit PCM5102
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

    // Allocate TX channel
    esp_err_t err = i2s_new_std_tx_channel(&chan_cfg, &std_cfg, out_handle);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Failed to allocate I2S TX channel: %s", esp_err_to_name(err));
        return err;
    }
    ESP_LOGI(TAG, "I2S TX channel allocated");

    // Enable I2S
    err = i2s_channel_enable(*out_handle);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Failed to enable I2S channel: %s", esp_err_to_name(err));
        i2s_del_channel(*out_handle);
        *out_handle = NULL;
        return err;
    }
    ESP_LOGI(TAG, "I2S enabled: 44.1 kHz, 32-bit, Stereo");

    return ESP_OK;
}

// Write audio data
esp_err_t write_audio_frame_32bit(i2s_chan_handle_t handle, 
                                  const uint8_t *pcm_data, 
                                  size_t frame_bytes) {
    size_t bytes_written = 0;
    esp_err_t err = i2s_channel_write(
        handle,
        pcm_data,
        frame_bytes,
        &bytes_written,
        portMAX_DELAY
    );
    
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "I2S write error: %s", esp_err_to_name(err));
        return err;
    }
    
    if (bytes_written != frame_bytes) {
        ESP_LOGW(TAG, "Partial I2S write: %u/%u bytes", bytes_written, frame_bytes);
    }
    
    return ESP_OK;
}

// Cleanup
esp_err_t deinit_i2s_output(i2s_chan_handle_t handle) {
    if (handle == NULL) {
        return ESP_OK;
    }
    
    esp_err_t err = i2s_channel_disable(handle);
    if (err != ESP_OK) {
        ESP_LOGW(TAG, "Failed to disable I2S: %s", esp_err_to_name(err));
    }
    
    err = i2s_del_channel(handle);
    if (err != ESP_OK) {
        ESP_LOGW(TAG, "Failed to delete I2S channel: %s", esp_err_to_name(err));
    }
    
    ESP_LOGI(TAG, "I2S cleanup complete");
    return ESP_OK;
}
```

### Example 2: Dynamic Sample Rate Switching

```cpp
#include "driver/i2s_std.h"

// Change sample rate without stopping playback (v6.0 feature)
esp_err_t change_sample_rate(i2s_chan_handle_t handle, uint32_t new_rate) {
    i2s_std_clk_config_t new_clk = I2S_STD_CLK_DEFAULT_CONFIG(new_rate);
    
    esp_err_t err = i2s_channel_reconfig_clk(handle, &new_clk);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Failed to reconfigure I2S clock: %s", esp_err_to_name(err));
        return err;
    }
    
    ESP_LOGI(TAG, "Sample rate changed to %lu Hz", new_rate);
    return ESP_OK;
}

// Example usage: Spotify switching between 44.1 kHz and 48 kHz
esp_err_t spotify_rate_switch_example(i2s_chan_handle_t handle) {
    // Switch from 44.1 kHz (typical Spotify) to 48 kHz (if needed)
    return change_sample_rate(handle, 48000);
}
```

### Example 3: ES9018 MSB-Format Setup

```cpp
#include "driver/i2s_std.h"

// ES9018 uses MSB-first I2S format (different from PCM5102)
esp_err_t init_i2s_es9018_32bit(i2s_chan_handle_t *out_handle) {
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(
        I2S_NUM_0,
        I2S_ROLE_MASTER
    );

    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
        .slot_cfg = I2S_STD_MSB_SLOT_DEFAULT_CONFIG(  // MSB format for ES9018
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

    esp_err_t err = i2s_new_std_tx_channel(&chan_cfg, &std_cfg, out_handle);
    if (err == ESP_OK) {
        err = i2s_channel_enable(*out_handle);
        if (err == ESP_OK) {
            ESP_LOGI(TAG, "ES9018 I2S initialized: MSB format, 32-bit");
        }
    }
    
    return err;
}
```

---

## 7. Custom I2S Audio Sink Implementation Pattern

### Class Structure for cspot Integration

```cpp
// i2s_audio_sink.h
#pragma once

#include <cstdint>
#include <cstddef>
#include "driver/i2s_std.h"
#include "freertos/ringbuf.h"

class I2SAudioSink {
public:
    I2SAudioSink();
    ~I2SAudioSink();
    
    // Initialize with specified sample rate and bit depth
    bool initialize(uint32_t sample_rate, uint8_t bit_depth);
    
    // Write PCM audio frames
    bool write_pcm_frames(const uint8_t *data, size_t bytes);
    
    // Update audio properties (sample rate, etc.)
    bool set_properties(uint32_t sample_rate, uint8_t bit_depth, uint8_t channels);
    
    // Cleanup resources
    void cleanup();
    
private:
    i2s_chan_handle_t tx_handle = NULL;
    RingbufHandle_t ring_buffer = NULL;
    uint32_t current_sample_rate = 44100;
    uint8_t current_bit_depth = 32;
    uint8_t current_channels = 2;
    bool is_initialized = false;
    
    void start_dma_feed_task();
    static void dma_feed_task_static(void *arg);
};
```

```cpp
// i2s_audio_sink.cpp
#include "i2s_audio_sink.h"
#include "esp_log.h"
#include "freertos/task.h"

static const char *TAG = "I2S_AUDIO_SINK";
static const size_t RING_BUFFER_SIZE = 256 * 1024;  // 256 KB

I2SAudioSink::I2SAudioSink() = default;

I2SAudioSink::~I2SAudioSink() {
    cleanup();
}

bool I2SAudioSink::initialize(uint32_t sample_rate, uint8_t bit_depth) {
    if (is_initialized) {
        ESP_LOGW(TAG, "Already initialized");
        return false;
    }
    
    current_sample_rate = sample_rate;
    current_bit_depth = bit_depth;

    // Channel configuration
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(
        I2S_NUM_0,
        I2S_ROLE_MASTER
    );

    // Convert bit_depth to I2S enum
    i2s_data_bit_width_t bit_width;
    switch (bit_depth) {
        case 16: bit_width = I2S_DATA_BIT_WIDTH_16BIT; break;
        case 24: bit_width = I2S_DATA_BIT_WIDTH_24BIT; break;
        case 32: bit_width = I2S_DATA_BIT_WIDTH_32BIT; break;
        default:
            ESP_LOGE(TAG, "Unsupported bit depth: %u", bit_depth);
            return false;
    }

    // Standard I2S configuration (Philips format)
    i2s_std_config_t std_cfg = {
        .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(sample_rate),
        .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
            bit_width,
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

    // Create I2S channel
    esp_err_t err = i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Failed to create I2S TX channel: %s", esp_err_to_name(err));
        return false;
    }

    // Enable I2S
    err = i2s_channel_enable(tx_handle);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Failed to enable I2S channel: %s", esp_err_to_name(err));
        i2s_del_channel(tx_handle);
        tx_handle = NULL;
        return false;
    }

    // Create ring buffer
    ring_buffer = xRingbufferCreate(RING_BUFFER_SIZE, RINGBUF_TYPE_BYTEBUF);
    if (!ring_buffer) {
        ESP_LOGE(TAG, "Failed to create ring buffer");
        i2s_channel_disable(tx_handle);
        i2s_del_channel(tx_handle);
        tx_handle = NULL;
        return false;
    }

    // Start DMA feed task
    start_dma_feed_task();

    is_initialized = true;
    ESP_LOGI(TAG, "I2S Audio Sink initialized: %lu Hz, %u-bit", 
             sample_rate, bit_depth);
    
    return true;
}

bool I2SAudioSink::write_pcm_frames(const uint8_t *data, size_t bytes) {
    if (!is_initialized || !ring_buffer) {
        ESP_LOGE(TAG, "I2S not initialized");
        return false;
    }

    BaseType_t ret = xRingbufferSend(ring_buffer, (void *)data, bytes, 
                                      pdMS_TO_TICKS(100));
    
    if (ret != pdTRUE) {
        ESP_LOGW(TAG, "Ring buffer overflow (ring buffer timeout)");
        return false;
    }

    return true;
}

bool I2SAudioSink::set_properties(uint32_t sample_rate, uint8_t bit_depth, 
                                  uint8_t channels) {
    if (!tx_handle) {
        ESP_LOGE(TAG, "I2S not initialized");
        return false;
    }

    // Only sample rate can be changed dynamically
    if (sample_rate != current_sample_rate) {
        i2s_std_clk_config_t clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(sample_rate);
        esp_err_t err = i2s_channel_reconfig_clk(tx_handle, &clk_cfg);
        
        if (err != ESP_OK) {
            ESP_LOGE(TAG, "Failed to reconfigure sample rate: %s", 
                     esp_err_to_name(err));
            return false;
        }
        
        current_sample_rate = sample_rate;
        ESP_LOGI(TAG, "Sample rate changed to %lu Hz", sample_rate);
    }

    // Bit depth and channels typically require re-initialization
    // For now, just log if they differ from initialization
    if (bit_depth != current_bit_depth || channels != current_channels) {
        ESP_LOGW(TAG, "Bit depth/channels change requires re-initialization");
        // In a production system, you might reinitialize the channel here
    }

    return true;
}

void I2SAudioSink::cleanup() {
    if (!is_initialized) {
        return;
    }

    // Delete ring buffer
    if (ring_buffer) {
        vRingbufferDelete(ring_buffer);
        ring_buffer = NULL;
    }

    // Disable and delete I2S channel
    if (tx_handle) {
        i2s_channel_disable(tx_handle);
        i2s_del_channel(tx_handle);
        tx_handle = NULL;
    }

    is_initialized = false;
    ESP_LOGI(TAG, "I2S Audio Sink cleaned up");
}

void I2SAudioSink::start_dma_feed_task() {
    xTaskCreatePinnedToCore(
        dma_feed_task_static,
        "i2s_dma_feed",
        4096,
        this,
        10,
        NULL,
        tskNO_AFFINITY
    );
}

void I2SAudioSink::dma_feed_task_static(void *arg) {
    I2SAudioSink *sink = static_cast<I2SAudioSink *>(arg);
    
    while (1) {
        size_t item_size;
        uint8_t *item = static_cast<uint8_t *>(
            xRingbufferReceiveUpTo(sink->ring_buffer, &item_size, 
                                   portMAX_DELAY, 8192)
        );
        
        if (item) {
            size_t bytes_written = 0;
            esp_err_t err = i2s_channel_write(sink->tx_handle, item, item_size,
                                              &bytes_written, portMAX_DELAY);
            
            vRingbufferReturnItem(sink->ring_buffer, (void *)item);
            
            if (err != ESP_OK) {
                ESP_LOGE(TAG, "I2S write failed: %s", esp_err_to_name(err));
            }
        }
    }
}
```

---

## 8. ESP-IDF v6.0 I2S Examples in the Wild

### Existing cspot Audio Sinks

The cspot repository already has several audio sink implementations in:
- `cspot/bell/main/audio-sinks/esp/`
- Current implementations use the **old ESP-IDF v5.x API** (`driver/i2s.h`)

**Key Files to Migrate:**
- `BufferedAudioSink.cpp` - Base class for all sinks
- `PCM5102AudioSink.cpp` - 32-bit capable DAC (should be your target)
- `ES9018AudioSink.cpp` - Premium audiophile DAC
- `ES8388AudioSink.cpp` - Integrated codec

---

## 9. Recommended Implementation Strategy

### For Spotify HiFi Support (32-bit Lossless):

1. **Select DAC**: PCM5102 or ES9018
   - **PCM5102**: Cost-effective, proven, widely available
   - **ES9018**: Premium quality, better SNR, audiophile-grade

2. **Migrate AudioSink Classes**:
   - Update `BufferedAudioSink` from `driver/i2s.h` to `driver/i2s_std.h`
   - Use new channel-based API for cleaner code
   - Implement 32-bit support in slot configuration

3. **Buffer Management**:
   - Keep existing ring buffer approach
   - Size for 256 KB - 1 MB depending on network jitter tolerance
   - 8 MB PSRAM provides plenty of headroom

4. **Sample Rate Flexibility**:
   - Use `i2s_channel_reconfig_clk()` for smooth rate changes
   - Support 44.1 kHz (Spotify standard), 48 kHz, and 96 kHz

5. **Testing**:
   - Verify 32-bit audio output with oscilloscope or audio analyzer
   - Check for clock stability with LRCK on GPIO25
   - Monitor PSRAM usage during playback

---

## 10. Quick Reference: v5.x to v6.0 Migration

| Operation | v5.x (Old) | v6.0 (New) |
|-----------|-----------|-----------|
| **Include** | `#include "driver/i2s.h"` | `#include "driver/i2s_std.h"` |
| **Initialize** | `i2s_driver_install()` | `i2s_new_std_tx_channel()` |
| **Set Pins** | `i2s_set_pin()` | Configured in `i2s_std_config_t.gpio_cfg` |
| **Enable** | Automatic | `i2s_channel_enable()` |
| **Write Data** | `i2s_write()` | `i2s_channel_write()` |
| **Change Rate** | `i2s_set_clk()` (stops audio) | `i2s_channel_reconfig_clk()` (seamless) |
| **Cleanup** | `i2s_driver_uninstall()` | `i2s_channel_disable()` + `i2s_del_channel()` |

---

## Summary

For **Spotify HiFi at 32-bit lossless** on ESP32 with ESP-IDF v6.0:

✓ **Modern I2S API**: Use `driver/i2s_std.h` with channel-based configuration  
✓ **Pin Configuration**: LRCK=GPIO25, BCK=GPIO26, DOUT=GPIO13  
✓ **Best DAC**: PCM5102 (value) or ES9018 (premium)  
✓ **Bit Depth**: I2S_DATA_BIT_WIDTH_32BIT  
✓ **Buffer**: 256 KB - 1 MB ring buffer (8 MB PSRAM available)  
✓ **Sample Rate**: 44.1 kHz standard, up to 192 kHz capable  

This configuration exceeds Spotify HiFi requirements and provides professional-grade audio output capability.
