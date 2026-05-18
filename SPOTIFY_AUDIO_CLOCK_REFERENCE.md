# Spotify Audio Quality Mapping & ESP-IDF v6.0 Clock Reference

Complete technical reference for supporting Spotify audio formats on ESP32.

---

## Spotify Audio Format Specifications

### Spotify Quality Tiers

| Tier | Codec | Bitrate | Frame Rate | Bit Depth (Equiv.) | Channels | Use Case |
|------|-------|---------|-----------|------------------|----------|----------|
| **Free** | Ogg Vorbis | 96 kbps | 22.05 kHz | 12-bit equiv | Stereo | Mobile/Limited |
| **Premium** | Ogg Vorbis | 320 kbps | 44.1 kHz | 16-bit equiv | Stereo | Standard |
| **HiFi (Beta)** | FLAC | 1.4 Mbps | 44.1 kHz | 16-bit native | Stereo | Lossless |
| **HiFi Ultra** | FLAC | 2.8+ Mbps | 48 kHz | 24-bit native | Stereo | Premium Lossless |

### Quality Mapping to I2S Configuration

```
Spotify Premium (320 kbps ogg @ 44.1 kHz)
    ↓ (Decode to PCM)
16-bit stereo PCM @ 44.1 kHz
    ↓ (Can pad to 32-bit for processing headroom)
I2S output: 32-bit @ 44.1 kHz to DAC
    ↓ (DAC converts to analog)
Analog speaker output
```

### Why 32-bit Even for 16-bit Audio?

1. **Processing Headroom**: Room for digital signal processing (EQ, mixing, etc.)
2. **Precision**: Floating-point audio algorithms use 32-bit internally
3. **Future-Proof**: Supports HiFi Ultra 24-bit upgrade
4. **DAC Native**: Many audiophile DACs (ES9018) use 32-bit internally

---

## ESP-IDF v6.0 Clock Configuration

### Clock Frequency Calculations

For I2S audio transmission, the BCLK (Bit Clock) must satisfy:

```
BCLK = Sample_Rate × Bits_Per_Sample × Channels

Example 1: 44.1 kHz, 16-bit, Stereo
BCLK = 44,100 × 16 × 2 = 1,411,200 Hz = 1.4112 MHz

Example 2: 44.1 kHz, 32-bit, Stereo (our target)
BCLK = 44,100 × 32 × 2 = 2,822,400 Hz = 2.8224 MHz

Example 3: 192 kHz, 32-bit, Stereo (future support)
BCLK = 192,000 × 32 × 2 = 12,288,000 Hz = 12.288 MHz
```

### MCLK (Master Clock) Configuration

MCLK is typically a multiple of sample rate:

```
MCLK = MCLK_Multiple × Sample_Rate

For ESP-IDF v6.0, default MCLK multipliers:
- I2S_MCLK_MULTIPLE_256: MCLK = 256 × Fs (most common)
- I2S_MCLK_MULTIPLE_384: MCLK = 384 × Fs (for 24-bit, improves MCLK/BCLK ratio)
- I2S_MCLK_MULTIPLE_512: MCLK = 512 × Fs (highest precision, uses more power)
```

### Complete Clock Configuration Table

| Sample Rate | Bit Depth | Channels | BCLK | MCLK (×256) | MCLK (×384) | APLL Needed |
|-------------|-----------|----------|------|------------|------------|-------------|
| 44.1 kHz | 16-bit | Stereo | 1.41 MHz | 11.29 MHz | 16.93 MHz | No (Default) |
| 44.1 kHz | 32-bit | Stereo | 2.82 MHz | 11.29 MHz | 16.93 MHz | No (Default) |
| 48 kHz | 16-bit | Stereo | 1.54 MHz | 12.29 MHz | 18.43 MHz | No (Default) |
| 48 kHz | 32-bit | Stereo | 3.07 MHz | 12.29 MHz | 18.43 MHz | No (Default) |
| 96 kHz | 16-bit | Stereo | 3.07 MHz | 24.58 MHz | 36.86 MHz | No (Default) |
| 96 kHz | 32-bit | Stereo | 6.14 MHz | 24.58 MHz | 36.86 MHz | No (Default) |
| 192 kHz | 16-bit | Stereo | 6.14 MHz | 49.15 MHz | 73.73 MHz | Yes (APLL required) |
| 192 kHz | 32-bit | Stereo | 12.29 MHz | 49.15 MHz | 73.73 MHz | Yes (APLL required) |

**Note:** ESP32 can generate these clocks via APLL (Audio PLL). ESP-IDF v6.0 auto-selects APLL for accurate timing.

---

## Clock Source Selection in ESP-IDF v6.0

### Default Clock Behavior

```cpp
i2s_std_clk_config_t clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100);
// Automatically uses:
// - I2S_CLK_SRC_DEFAULT for base clock
// - APLL for 44.1 kHz multiples (11.2896 MHz, 22.5792 MHz, etc.)
// - Generates accurate BCLK for audio without user intervention
```

### Explicit Clock Configuration (Advanced)

```cpp
i2s_std_clk_config_t clk_cfg = {
    .sample_rate_hz = 44100,
    .clk_src = I2S_CLK_SRC_APLL,          // Use APLL for accurate audio timing
    .mclk_multiple = I2S_MCLK_MULTIPLE_256,  // Standard 256x multiplier
    .bclk_div = 8,                        // BCLK = MCLK / bclk_div
    .bipclk_div = 1,                      // Bit shift (usually 1)
};

// For different sample rates:
// 44.1 kHz: APLL = 11.2896 MHz, BCLK = 11.2896 / 4 = 2.8224 MHz ✓
// 48 kHz:   APLL = 12.288 MHz,  BCLK = 12.288 / 4 = 3.072 MHz ✓
```

### APLL Frequency Calculation for ESP32

The APLL (Audio Phase-Locked Loop) generates frequencies based on reference clock (40 MHz):

```
APLL_Freq = 40 MHz × (a + b/2^18) / 2^(1-o)

Where: a, b, o are dividers set internally by ESP-IDF

For common audio rates:
- 44.1 kHz  → APLL = 11.2896 MHz (BCLK = 2.8224 MHz for 32-bit stereo)
- 48 kHz    → APLL = 12.288 MHz  (BCLK = 3.072 MHz for 32-bit stereo)
- 96 kHz    → APLL = 24.576 MHz  (BCLK = 6.144 MHz for 32-bit stereo)
- 192 kHz   → APLL = 49.152 MHz  (BCLK = 12.288 MHz for 32-bit stereo)
```

---

## Spotify → ESP32 Audio Flow with Clocking

```
┌─────────────────────────────────────────────────────────────────┐
│ SPOTIFY DECODER                                                 │
│ Input: Ogg Vorbis @ 320 kbps, 44.1 kHz                         │
│ Output: 16-bit PCM stereo @ 44.1 kHz (176.4 KB/sec)            │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│ AUDIO BUFFER (Ring Buffer)                                      │
│ Size: 256-512 KB (0.7-1.5 seconds at 44.1 kHz)                │
│ Purpose: Handle WiFi jitter, decoder pauses                    │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│ I2S DMA FEED TASK (i2s_channel_write)                          │
│ Takes: 16-bit PCM from ring buffer                             │
│ Pads/Converts to: 32-bit PCM for DAC                          │
│ Rate: ~344 KB/sec = 44,100 samples/sec × 8 bytes              │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│ I2S PERIPHERAL (Master Clock Source)                           │
│ MCLK = 11.2896 MHz (256 × 44.1 kHz)                           │
│ BCLK = 2.8224 MHz (serial data clock)                          │
│ LRCK/WS = 44.1 kHz (left/right channel select)                │
│                                                                 │
│ Clock Timing (one 32-bit stereo frame):                       │
│   L-channel: 32 BCLK pulses                                   │
│   R-channel: 32 BCLK pulses                                   │
│   Total: 64 BCLK cycles = 22.66 microseconds                 │
└─────────────────┬───────────────────────────────────────────────┘
                  │
         LRCK (GPIO25 - alternates L/R at 44.1 kHz)
         BCLK (GPIO26 - 2.8224 MHz clock)
         DATA (GPIO13 - 32-bit serial data)
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│ DAC (PCM5102 or ES9018)                                         │
│ Input: 32-bit I2S @ 44.1 kHz                                   │
│ Output: Analog L/R stereo signal                               │
│                                                                 │
│ Clock Synchronization:                                         │
│   - DAC locks onto BCLK to serialize data                     │
│   - DAC uses LRCK to alternate left/right                     │
│   - No internal clock needed (slave mode)                      │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
         ┌─────────────────────┐
         │ Amplifier/Speaker   │
         │ Reproduces audio at │
         │ full 16-bit quality │
         │ (or 24/32-bit if    │
         │  HiFi format used)  │
         └─────────────────────┘
```

---

## I2S Timing Diagram (Philips Format, 32-bit Stereo)

```
MCLK  ──┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─
         │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │
        (Master Clock: 11.2896 MHz for 44.1 kHz)

BCLK  ──┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─
         │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │
        (Bit Clock: 2.8224 MHz, one tick per data bit)

LRCK  ──┐         ┌─────────────────────────────────┐         ┌──
        └─────────┘                                 └─────────┘
        (Low = Left channel, High = Right channel, Switches every 32 BCLK)

DATA  ──X─d31─d30─d29─...─d02─d01─d00─X─d31─d30─d29─...─d02─d01─d00─X─
         └─────────32-bit L─────────┘ └─────────32-bit R─────────┘
        (MSB first: bit 31 transmitted first)

Timing: 1 frame (L+R) = 64 BCLK cycles = 22.66 µs @ 2.8224 MHz
        Sample rate = 44,100 Hz = 1 frame per 22.68 µs ✓
```

---

## Verification: Clock Accuracy Check

### Theoretical Clock Requirements vs ESP32 APLL

For **44.1 kHz, 32-bit, Stereo**:

```
Calculated Requirements:
- BCLK: 44,100 × 32 × 2 = 2,822,400 Hz
- MCLK: 256 × 44,100 = 11,289,600 Hz

ESP32 APLL Output:
- Set MCLK = 11,289,600 Hz (APLL generates this)
- BCLK = MCLK ÷ 4 = 2,822,400 Hz ✓

ESP-IDF v6.0 Verification:
```cpp
// Measure with logic analyzer/oscilloscope:
// 1. Probe BCLK (GPIO26)
// 2. Set oscilloscope to frequency mode
// 3. Expected: 2.8224 MHz (±0.1% for audio quality)
// 4. Probe LRCK (GPIO25)
// 5. Expected: 44.1 kHz square wave (±0.1%)

// Or measure in software:
void verify_clocks(void) {
    // BCLK frequency = 2.8224 MHz
    // LRCK frequency = 44.1 kHz
    // If measured frequencies differ by >5%, reconfigure APLL
}
```

---

## Supported I2S Sample Rates in ESP-IDF v6.0

```cpp
// All these sample rates supported with automatic APLL configuration:

8000    // Telephony
11025   // Low quality
16000   // Telephony, VoIP
22050   // Low bitrate streaming
24000   // Video
32000   // Video, compressed audio
44100   // Spotify Premium, CD quality ← MAIN TARGET
48000   // Video, professional audio
64000   // Video
88200   // High-resolution audio
96000   // Professional/HD audio
176400  // Studio recording (rare)
192000  // Maximum (requires APLL in high-speed mode)
```

### Configuration for Different Rates

```cpp
// For 44.1 kHz (Spotify)
i2s_std_clk_config_t clk = I2S_STD_CLK_DEFAULT_CONFIG(44100);

// For 48 kHz (if needed for video sync)
i2s_std_clk_config_t clk = I2S_STD_CLK_DEFAULT_CONFIG(48000);

// For 96 kHz (HD audio, Spotify HiFi Ultra)
i2s_std_clk_config_t clk = I2S_STD_CLK_DEFAULT_CONFIG(96000);

// For 192 kHz (maximum, rare)
i2s_std_clk_config_t clk = I2S_STD_CLK_DEFAULT_CONFIG(192000);
```

---

## Power Consumption Considerations

At higher clock rates, power consumption increases:

| Config | MCLK | BCLK | Estimated Power | Use Case |
|--------|------|------|-----------------|----------|
| 44.1 kHz, 16-bit | 11.3 MHz | 1.4 MHz | ~50 mW (I2S only) | Spotify Premium |
| 44.1 kHz, 32-bit | 11.3 MHz | 2.8 MHz | ~60 mW (I2S only) | Spotify HiFi |
| 96 kHz, 32-bit | 24.6 MHz | 6.1 MHz | ~100 mW (I2S only) | HD Audio |
| 192 kHz, 32-bit | 49.2 MHz | 12.3 MHz | ~150 mW (I2S only) | Studio |

**Note:** Total ESP32 power ≈ 100-200 mW (CPU, WiFi, peripherals). I2S clocking adds 50-100 mW to this.

---

## APLL (Audio PLL) Configuration Reference

ESP-IDF v6.0 automatically configures APLL for standard audio sample rates. Normally, you don't need to touch this, but for reference:

```cpp
// If you need to manually configure APLL (advanced):
#include "hal/clk_tree_ll.h"

// Reconfigure APLL for 48 kHz (if 44.1 kHz is currently active):
// This is rarely needed - just call i2s_channel_reconfig_clk()

// Example: Switch from 44.1 kHz to 48 kHz
i2s_std_clk_config_t new_clk = I2S_STD_CLK_DEFAULT_CONFIG(48000);
ESP_ERROR_CHECK(i2s_channel_reconfig_clk(tx_handle, &new_clk));
// APLL is automatically adjusted to 12.288 MHz (256 × 48 kHz)
```

---

## Latency Analysis

### Total Audio Latency in System

```
WiFi Streaming → Spotify Decoder → Buffer → I2S DMA → DAC → Speaker

Latency Components:
1. WiFi network delay: 5-50 ms (variable)
2. Spotify decoder internal: ~5-10 ms
3. Ring buffer (256 KB @ 44.1 kHz): 740 ms
4. I2S DMA buffer (64 KB typical): 0.18 ms
5. DAC latency: ~1-5 ms
6. Speaker physical latency: negligible

Total: ~750 ms (ring buffer dominates for robust WiFi streaming)
For low-latency applications: Use smaller ring buffer (64 KB = 18 ms)
```

### Trade-off

- **Large buffer (256 KB)**: Handles WiFi jitter, but ~740 ms latency
- **Small buffer (64 KB)**: Low latency (~18 ms), but more dropout risk
- **Optimal for Spotify**: 256 KB ring buffer (WiFi is unreliable)

---

## Summary: Spotify Quality Support

✓ **Spotify Premium (320 kbps @ 44.1 kHz)**
- I2S: 44.1 kHz, 16-bit stereo
- DAC: PCM5102 or ES8388 (16/24-bit support)
- BCLK: 1.41 MHz
- MCLK: 11.29 MHz

✓ **Spotify HiFi (FLAC @ 44.1 kHz)**
- I2S: 44.1 kHz, 16-bit native (or 24-bit if available)
- DAC: PCM5102 or ES9018 (32-bit capable for headroom)
- BCLK: 2.82 MHz (for 32-bit output)
- MCLK: 11.29 MHz

✓ **Future-Proof (96+ kHz support)**
- I2S: Dynamic rate switching via `i2s_channel_reconfig_clk()`
- DAC: Supports up to 192 kHz (ES9018, PCM5102, TAS5711)
- Clock: APLL automatically handles all rates

**Recommended Implementation**: 32-bit I2S output at 44.1 kHz with PCM5102 DAC.
This provides excellent audio quality exceeding Spotify HiFi specifications.
