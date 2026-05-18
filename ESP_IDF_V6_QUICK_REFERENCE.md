# ESP-IDF v6.0 Migration Quick Reference for cspot

## Critical Changes (Fix First)

### 1. I2S Driver API Overhaul
**Status:** 🔴 BLOCKING - All audio sinks affected

**Files to update:**
- `cspot/bell/main/audio-sinks/esp/InternalAudioSink.cpp`
- `cspot/bell/main/audio-sinks/esp/ES8311AudioSink.cpp`
- `cspot/bell/main/audio-sinks/esp/ES8388AudioSink.cpp`
- `cspot/bell/main/audio-sinks/esp/AC101AudioSink.cpp`
- `cspot/bell/main/audio-sinks/esp/PCM5102AudioSink.cpp`
- `cspot/bell/main/audio-sinks/esp/ES9018AudioSink.cpp`
- `cspot/bell/main/audio-sinks/esp/SPDIFAudioSink.cpp`
- `cspot/bell/main/audio-sinks/esp/BufferedAudioSink.cpp`

**Old Code Pattern:**
```cpp
#include "driver/i2s.h"
i2s_config_t i2s_config = {...};
i2s_driver_install(I2S_NUM_0, &i2s_config, 0, NULL);
i2s_set_pin(I2S_NUM_0, &pin_config);
i2s_write(I2S_NUM_0, data, len, &bytes_written, timeout);
i2s_set_clk(I2S_NUM_0, sample_rate, bits, channels);
```

**New Code Pattern (I2S Standard Mode):**
```cpp
#include "driver/i2s_std.h"

// Configure channel
i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);

// Configure standard mode with Philips slot format
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(
        I2S_DATA_BIT_WIDTH_16BIT, 
        I2S_SLOT_MODE_STEREO),
    .gpio_cfg = {
        .mclk = GPIO_NUM_0,
        .bclk = GPIO_NUM_1,
        .ws = GPIO_NUM_2,
        .dout = GPIO_NUM_3,
        .din = GPIO_NUM_-1,
    },
};

i2s_chan_handle_t tx_handle;
ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));

// Write data
ESP_ERROR_CHECK(i2s_channel_write(tx_handle, data, len, &bytes_written, timeout));

// Reconfigure sample rate
i2s_std_clk_config_t clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(48000);
ESP_ERROR_CHECK(i2s_channel_reconfig_clk(tx_handle, &clk_cfg));
```

**Key API Changes:**
| Old | New | Note |
|-----|-----|------|
| `i2s_config_t` | `i2s_chan_config_t` + `i2s_std_config_t` | Split into multiple structures |
| `i2s_driver_install()` | `i2s_new_std_tx_channel()` | Creates handle instead of installing port |
| `i2s_port_t` | `int` | Type removed |
| `I2S_NUM_0` | `I2S_NUM_0` (still works) | Now just a macro |
| `i2s_write()` | `i2s_channel_write()` | Uses handle |
| `i2s_set_clk()` | `i2s_channel_reconfig_clk()` | Uses config struct |

---

### 2. Component Dependencies
**Status:** 🟡 HIGH - Build will fail without this

**File:** `cspot/bell/main/CMakeLists.txt` (or relevant component)

**Add to REQUIRES or PRIV_REQUIRES:**
```cmake
idf_component_register(
    SRCS ${SOURCES}
    INCLUDE_DIRS "include"
    REQUIRES 
        esp_driver_i2s        # NEW - Required for I2S
        esp_driver_gpio       # NEW - For GPIO config
        esp_driver_dma        # NEW - For DMA operations
        freertos
    PRIV_REQUIRES 
        esp_hw_support
)
```

---

### 3. Deprecated Includes
**Status:** 🟡 HIGH - Compilation will fail

**Replace in all affected files:**

```cpp
// ❌ Remove these
#include "driver/i2s.h"
#include <sys/dirent.h>
#include "esp32/aes.h"
#include "esp32/sha.h"

// ✅ Add these instead
#include "driver/i2s_std.h"      // (or i2s_pdm.h, i2s_tdm.h)
#include <dirent.h>
#include "aes/esp_aes.h"
#include "sha/sha_core.h"
```

---

## High Priority Changes (Fix Second)

### 4. Wi-Fi Re-initialization
**Status:** 🟡 MEDIUM - Causes runtime error if WiFi reinits

**Issue:** Re-calling `esp_wifi_init()` now returns `ESP_ERR_INVALID_STATE` instead of `ESP_OK`

**Fix in main initialization:**
```cpp
// ❌ Old way (fails in v6.0)
esp_wifi_init(&cfg);
esp_wifi_init(&cfg);  // Returns error!

// ✅ New way
if (esp_wifi_get_init_status() == false) {
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
}
```

---

### 5. PSA Crypto Initialization
**Status:** 🟡 MEDIUM - Required for HTTPS/TLS

**Add to `app_main()` EARLY, before any networking:**
```cpp
#include "psa/crypto.h"

void app_main() {
    // Initialize PSA Crypto FIRST
    psa_status_t status = psa_crypto_init();
    if (status != PSA_SUCCESS) {
        ESP_LOGE(TAG, "PSA Crypto init failed");
        return;
    }
    
    // Now safe to use Wi-Fi and HTTPS
    esp_wifi_init(&cfg);
    // ... rest of init
}
```

---

### 6. GCC 15.1.0 Warning Fixes
**Status:** 🟡 MEDIUM - Compilation fails with warnings-as-errors

**Common issues:**
```cpp
// ❌ String too long for array
char buf[3] = "hello";

// ✅ Fix: Size includes null terminator
char buf[6] = "hello";

// ❌ Dangling reference (C++)
const int& ref = std::max(a, b);

// ✅ Fix: Use value instead
int result = std::max(a, b);

// ❌ Wrong header
#include <sys/dirent.h>
DIR *dir = opendir(".");

// ✅ Fix: Use correct header
#include <dirent.h>
DIR *dir = opendir(".");
```

---

## Medium Priority Changes

### 7. Wi-Fi Event Codes
**Status:** 🟢 LOW - Only if handling disconnect reasons

**Update event handlers:**
```cpp
// ❌ Removed codes
case WIFI_REASON_ASSOC_EXPIRE:      // Removed
case WIFI_REASON_NOT_AUTHED:        // Removed
case WIFI_REASON_NOT_ASSOCED:       // Removed

// ✅ Use new codes instead
case WIFI_REASON_AUTH_EXPIRE:
case WIFI_REASON_CLASS2_FRAME_FROM_NONAUTH_STA:
case WIFI_REASON_CLASS3_FRAME_FROM_NONASSOC_STA:
```

---

### 8. Linker Orphan Sections
**Status:** 🟢 LOW - Only if custom linker scripts

**Issue:** Linker now errors on orphan sections (previously warnings)

**Fix:**
```cmake
# Option 1: Fix the code (best)
# Option 2: Create linker fragment file:
# components/your_component/linker_fragments.txt
[sections:custom]
entries:
    .custom_section : KEEP(*)

# Option 3: Suppress (NOT recommended)
# In sdkconfig: CONFIG_COMPILER_ORPHAN_SECTIONS=warning
```

---

### 9. GPIO Wakeup API (If Using)
**Status:** 🟢 LOW - Only if implementing sleep

**Update sleep code:**
```cpp
// ❌ Removed APIs
esp_deep_sleep_enable_gpio_wakeup(BIT(GPIO_NUM_0), ESP_GPIO_WAKEUP_GPIO_LOW);
GPIO_IS_DEEP_SLEEP_WAKEUP_VALID_GPIO(GPIO_NUM_0)

// ✅ New APIs
esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown(BIT(GPIO_NUM_0), ESP_GPIO_WAKEUP_GPIO_LOW);
GPIO_IS_HP_PERIPH_PD_WAKEUP_VALID_IO(GPIO_NUM_0)
```

---

### 10. DMA Memory Allocation (If Using)
**Status:** 🟢 LOW - Only if using DMA directly

**Update code:**
```cpp
// ❌ Removed functions
void *buf = esp_dma_capable_malloc(size);
void *buf = esp_dma_capable_calloc(count, size);

// ✅ Use instead
#include "esp_heap_caps.h"
void *buf = heap_caps_malloc(size, MALLOC_CAP_DMA | MALLOC_CAP_CACHE_ALIGNED);
void *buf = heap_caps_calloc(count, size, MALLOC_CAP_DMA | MALLOC_CAP_CACHE_ALIGNED);
```

---

## Low Priority / No Action Needed

### SPIFFS & NVS - ✅ No changes needed
- SPIFFS API unchanged
- NVS API unchanged
- Existing code continues to work

### PSRAM Task Allocation - ✅ No changes needed
- Task stack allocation in PSRAM still works
- BellTask.h continues to work unchanged

### Newlib vs Picolibc - ℹ️ For awareness
- Default changed to Picolibc (smaller, faster)
- Can switch back to Newlib in menuconfig if needed
- Most code works without changes

### FreeRTOS - ✅ No changes needed
- No breaking changes for basic task/queue usage
- Event groups, semaphores, etc. unchanged

---

## Testing Checklist

- [ ] **I2S Audio**
  - [ ] Audio plays without glitches
  - [ ] Sample rate switching works
  - [ ] All audio sinks tested

- [ ] **Connectivity**
  - [ ] Wi-Fi connects and stays connected
  - [ ] Spotify API HTTPS works
  - [ ] Credentials load from storage

- [ ] **Compilation**
  - [ ] No GCC 15.1.0 warnings
  - [ ] No linker orphan section errors
  - [ ] Builds successfully to binary

---

## Migration Effort Estimate

| Task | Time | Difficulty |
|------|------|-----------|
| Update I2S driver API | 2-4 hours | HIGH |
| Update CMakeLists.txt | 30 min | LOW |
| Fix GCC warnings | 1-2 hours | MEDIUM |
| Add PSA Crypto init | 15 min | LOW |
| Test & validate | 2-4 hours | MEDIUM |
| **TOTAL** | **6-11 hours** | **MEDIUM** |

---

## File-by-File Changes Needed

### 1️⃣ `targets/esp32/main/CMakeLists.txt`
- [ ] Add `esp_driver_i2s` to REQUIRES

### 2️⃣ `cspot/bell/main/CMakeLists.txt`
- [ ] Add `esp_driver_i2s`, `esp_driver_gpio`, `esp_driver_dma` to REQUIRES

### 3️⃣ All Audio Sink Files (7 files)
```
cspot/bell/main/audio-sinks/esp/
  ├─ InternalAudioSink.cpp
  ├─ ES8311AudioSink.cpp
  ├─ ES8388AudioSink.cpp
  ├─ AC101AudioSink.cpp
  ├─ PCM5102AudioSink.cpp
  ├─ ES9018AudioSink.cpp
  ├─ SPDIFAudioSink.cpp
  └─ BufferedAudioSink.cpp
```
For each file:
- [ ] Replace `#include "driver/i2s.h"` with `#include "driver/i2s_std.h"`
- [ ] Update I2S configuration structures
- [ ] Update I2S API calls (driver_install → new_std_tx_channel, etc.)
- [ ] Update write/read calls
- [ ] Update dynamic reconfig calls

### 4️⃣ `targets/esp32/main/main.cpp`
- [ ] Add `psa_crypto_init()` call in `app_main()`
- [ ] Fix Wi-Fi re-init check
- [ ] Update any `#include <sys/dirent.h>` to `#include <dirent.h>`

---

## Resources

- **Full Migration Guide:** `ESP_IDF_V6_MIGRATION_GUIDE.md` (this directory)
- **Official Docs:** https://docs.espressif.com/projects/esp-idf/en/latest/esp32/migration-guides/release-6.x/6.0/index.html
- **I2S Driver:** https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32/api-reference/peripherals/i2s.html
- **PSA Crypto:** https://arm-software.github.io/psa-api/
- **GCC 15 Porting:** https://gcc.gnu.org/gcc-15/porting_to.html

---

**Quick Decision Tree:**

```
Does build fail?
├─ Missing esp_driver_i2s? → Add to CMakeLists.txt REQUIRES
├─ Including driver/i2s.h? → Replace with driver/i2s_std.h
└─ GCC 15 warnings? → Fix string sizes, dangling refs, includes

Does audio fail?
├─ No audio output? → Check I2S handle usage, GPIO config
├─ Dropouts/glitches? → Verify DMA buffer sizes, clock config
└─ Sample rate issues? → Check reconfig_clk() calls

Does WiFi fail?
├─ Can't connect? → Already init check needed
├─ HTTPS fails? → Call psa_crypto_init() early
└─ Credentials lost? → Check SPIFFS/NVS (should be OK)
```

---

**Last Updated:** 2024  
**For:** cspot ESP32 Audio Streaming Project  
**Scope:** ESP-IDF v5.x → v6.0 migration
