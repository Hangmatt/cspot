# ESP-IDF v6.0 Migration Guide for cspot (ESP32 Audio Project)

**Date:** 2024  
**Target:** Migration from ESP-IDF 5.x to v6.0  
**Focus:** Breaking changes relevant to a 2+ year old Spotify client project

---

## Executive Summary

ESP-IDF v6.0 represents a major release with significant breaking changes. For the cspot project, the most critical updates involve:

1. **I2S Driver** - Complete API overhaul (from `driver/i2s.h` to modular drivers)
2. **Component Dependencies** - Removal of legacy driver monolithic `driver` component
3. **Build System** - Linker orphan sections now error by default
4. **System Libraries** - Switching from Newlib to Picolibc, GCC 15.1.0 upgrade
5. **Security/Crypto** - Mandatory PSA Crypto API migration from legacy mbedTLS
6. **Storage** - Changes to NVS, SPIFFS, and VFS layers
7. **Memory Management** - PSRAM handling updates and DMA changes

---

## 1. I2S Driver API Changes (CRITICAL FOR CSPOT)

### Background
The legacy I2S driver (`driver/i2s.h`) has been **completely removed** in v6.0. It was deprecated in v5.0 but still functional. The cspot project heavily uses I2S for audio output through multiple audio sinks (AC101, ES8311, ES8388, ES9018, PCM5102, SPDIF, InternalDAC).

### Current cspot Implementation (ESP-IDF 5.x)
```cpp
// Found in: cspot/bell/main/audio-sinks/esp/InternalAudioSink.cpp
#include "driver/i2s.h"

i2s_config_t i2s_config = {
    .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_TX | I2S_MODE_DAC_BUILT_IN),
    .sample_rate = (i2s_bits_per_sample_t)44100,
    .bits_per_sample = I2S_BITS_PER_SAMPLE_16BIT,
    .channel_format = I2S_CHANNEL_FMT_RIGHT_LEFT,
    .communication_format = (i2s_comm_format_t)I2S_COMM_FORMAT_STAND_I2S,
    .intr_alloc_flags = 0,
    .dma_buf_count = 6,
    .dma_buf_len = 512,
    .use_apll = true,
    .tx_desc_auto_clear = true,
    .fixed_mclk = -1
};

i2s_driver_install((i2s_port_t)0, &i2s_config, 0, NULL);
i2s_set_pin((i2s_port_t)0, &pin_config);
i2s_set_dac_mode(I2S_DAC_CHANNEL_BOTH_EN);
i2s_write((i2s_port_t)0, item, itemSize, &written, portMAX_DELAY);
i2s_set_clk((i2s_port_t)0, sampleRate, (i2s_bits_per_sample_t)bitDepth, (i2s_channel_t)channelCount);
```

### ESP-IDF v6.0 Changes

The new I2S driver is now split into three separate drivers based on mode:

| Old API | New Component | New Header |
|---------|---------------|-----------|
| `driver/i2s.h` (all modes) | `esp_driver_i2s` | `driver/i2s_std.h` (standard mode) |
| | | `driver/i2s_pdm.h` (PDM mode) |
| | | `driver/i2s_tdm.h` (TDM mode) |

### API Replacement Pattern

**Old Pattern:**
```cpp
// Configure as struct, then install
i2s_config_t config = {...};
i2s_driver_install(I2S_NUM_0, &config, 0, NULL);
i2s_set_pin(I2S_NUM_0, &pin_config);

// Write data
i2s_write(I2S_NUM_0, data, len, &bytes_written, timeout);

// Change sample rate dynamically
i2s_set_clk(I2S_NUM_0, sample_rate, bits, channels);
```

**New Pattern (I2S Standard Mode):**
```cpp
#include "driver/i2s_std.h"

// Configuration is now split into multiple structures
i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_0, I2S_ROLE_MASTER);
i2s_std_config_t std_cfg = {
    .clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(44100),
    .slot_cfg = I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_STEREO),
    .gpio_cfg = {
        .mclk = GPIO_NUM_0,
        .bclk = GPIO_NUM_1,
        .ws = GPIO_NUM_2,
        .dout = GPIO_NUM_3,
        .din = GPIO_NUM_-1,  // Set to -1 if not used
        .invert_flags = {
            .mclk_inv = false,
            .bclk_inv = false,
            .ws_inv = false,
        },
    },
};

// Allocate and initialize channel
i2s_chan_handle_t tx_handle;
ESP_ERROR_CHECK(i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &tx_handle));
ESP_ERROR_CHECK(i2s_channel_enable(tx_handle));

// Write data
ESP_ERROR_CHECK(i2s_channel_write(tx_handle, data, len, &bytes_written, timeout));

// Change sample rate (uses new config structure)
i2s_std_clk_config_t clk_cfg = I2S_STD_CLK_DEFAULT_CONFIG(48000);
ESP_ERROR_CHECK(i2s_channel_reconfig_clk(tx_handle, &clk_cfg));
```

### Key Behavioral Changes

1. **Handle-based API**: Instead of port numbers (`I2S_NUM_0`), use handles (`i2s_chan_handle_t`)
2. **Separated configs**: Clock, slot, and GPIO configs are now separate structures
3. **Type safety**: `i2s_port_t` removed; port numbers now use `int` or specific enum types
4. **Dynamic reconfiguration**: `i2s_set_clk()` replaced with `i2s_channel_reconfig_clk()`
5. **Descriptor auto-clear**: Parameter `tx_desc_auto_clear` moved to slot configuration
6. **No DAC mode in standard**: `I2S_MODE_DAC_BUILT_IN` is no longer supported in standard mode

### Built-in DAC Configuration (For InternalAudioSink)

The built-in ESP32 DAC is no longer accessible through I2S standard mode. Options:

**Option A: Use external DAC (Recommended)**
- Switch to external DAC like AC101, ES8388, or PCM5102
- This is more maintainable going forward

**Option B: Use I2S PDM mode (if supported)**
- Check if PDM mode supports DAC on target chip
- Requires different slot configuration

### Required Changes for cspot

1. **Audio Sink Files to Update:**
   - `cspot/bell/main/audio-sinks/esp/InternalAudioSink.cpp` - Remove DAC mode or refactor
   - `cspot/bell/main/audio-sinks/esp/ES8311AudioSink.cpp` - Migrate to new API
   - `cspot/bell/main/audio-sinks/esp/ES8388AudioSink.cpp` - Migrate to new API
   - `cspot/bell/main/audio-sinks/esp/AC101AudioSink.cpp` - Migrate to new API
   - `cspot/bell/main/audio-sinks/esp/PCM5102AudioSink.cpp` - Migrate to new API
   - `cspot/bell/main/audio-sinks/esp/ES9018AudioSink.cpp` - Migrate to new API
   - `cspot/bell/main/audio-sinks/esp/SPDIFAudioSink.cpp` - Migrate to new API
   - `cspot/bell/main/audio-sinks/esp/BufferedAudioSink.cpp` - Update I2S API calls

2. **Remove deprecated APIs:**
   - Remove `#include "driver/i2s.h"`
   - Remove calls to `i2s_set_adc_mode()`, `i2s_adc_enable()`, `i2s_adc_disable()` if used

3. **Update CMakeLists.txt dependencies:**
   - Add `esp_driver_i2s` component requirement in affected `CMakeLists.txt` files

---

## 2. Component Registration and Dependency Changes

### Background
The monolithic `driver` component has been deprecated. Individual drivers are now separate components with explicit dependencies.

### Changes Affecting cspot

| What Was | What Changed | How It Affects cspot |
|----------|--------------|---------------------|
| `#include "driver/i2s.h"` | Moved to `esp_driver_i2s` component, `driver/i2s_std.h`, `driver/i2s_pdm.h`, `driver/i2s_tdm.h` | Audio sink files |
| `#include "driver/gpio.h"` | Moved to `esp_driver_gpio` component | GPIO configuration in audio sinks |
| `#include "driver/spi_master.h"` | Moved to `esp_driver_spi` component | Any SPI communication |
| Legacy `driver` component | No longer includes transitive dependencies | Must explicitly add needed driver components |

### CMakeLists.txt Updates

**Old Style (ESP-IDF 5.x):**
```cmake
idf_component_register(SRCS "main.c" "audio.c"
                       INCLUDE_DIRS "include"
                       REQUIRES driver esp_wifi esp_http_client)
```

**New Style (ESP-IDF 6.0):**
```cmake
idf_component_register(SRCS "main.c" "audio.c"
                       INCLUDE_DIRS "include"
                       REQUIRES esp_driver_i2s 
                               esp_driver_gpio
                               esp_driver_spi
                               esp_wifi 
                               esp_http_client
                               esp_driver_dma)  # If using DMA for I2S
```

### Recommended Changes for cspot

**File:** `cspot/bell/main/CMakeLists.txt` (or relevant component CMakeLists.txt)

```cmake
# Add to REQUIRES or PRIV_REQUIRES:
idf_component_register(
    SRCS ${SOURCES}
    INCLUDE_DIRS "include"
    REQUIRES 
        esp_driver_i2s        # For all I2S audio sinks
        esp_driver_gpio       # For GPIO pin configuration
        esp_driver_dma        # For DMA operations
        freertos              # For tasks
    PRIV_REQUIRES 
        esp_hw_support        # For internal hardware features
)
```

---

## 3. Build System and CMake Changes

### Linker Orphan Sections Now Error

**Breaking Change**: The linker now produces **errors** (not warnings) for orphan sections.

**What are orphan sections?**
- Sections not explicitly placed in the linker script
- Unintentionally created through misconfigured code

### Impact on cspot

Check if your project has custom linker scripts or uses sections not defined in ESP-IDF's linker fragments.

**How to resolve (in order of preference):**

1. **Fix the code** - Remove unused sections
2. **Use linker fragment file** - Place sections explicitly

   Create `linker_fragments.txt` in your component directory:
   ```
   [sections:custom_section]
   entries:
       .custom_section : KEEP(*)
   ```

3. **Suppress errors** (NOT recommended):
   ```
   CONFIG_COMPILER_ORPHAN_SECTIONS=warning  # In sdkconfig
   ```

### Global Constructor Order Changed

**Old Behavior (ESP-IDF 5.x):** Constructors run in **descending** order (Xtensa-specific)  
**New Behavior (ESP-IDF 6.0):** Constructors run in **ascending** order (standard POSIX)

**If this affects cspot:**
- Check `cspot/bell/main/utilities/include/BellTask.h` for static initialization
- Use explicit constructor priorities if ordering matters

```cpp
// Use this pattern if order is critical
__attribute__((constructor(101)))
void init_first(void) { /* ... */ }

__attribute__((constructor(102)))
void init_second(void) { /* ... */ }
```

### Compiler Warnings as Errors

**Default Changed**: `CONFIG_COMPILER_DISABLE_DEFAULT_ERRORS` defaults to `N` (warnings = errors)

**GCC 15.1.0 introduces new warnings** to watch for:
- `-Wno-unterminated-string-initialization` - Character arrays initialized with string literals
- `-Wno-dangling-reference` (C++ only) - References bound to temporaries
- `-Wno-defaulted-function-deleted` (C++ only) - Defaulted functions deleted by compiler

**If compilation fails:**
```
# Option 1: Fix warnings in code (preferred)

# Option 2: In sdkconfig
CONFIG_COMPILER_DISABLE_DEFAULT_ERRORS=y
CONFIG_COMPILER_DISABLE_GCC15_WARNINGS=y
```

---

## 4. System and Memory Changes

### LibC Changed from Newlib to Picolibc

**What changed:**
- Default LibC is now **PicolibC** (a Newlib fork with optimized stdio)
- Significantly smaller binary and less stack usage

**Breaking change:**
- Cannot redefine stdin/stdout/stderr per-task
- Streams are now global and shared
- `CONFIG_LIBC_PICOLIBC_NEWLIB_COMPATIBILITY` enabled by default for compatibility

**Impact on cspot:**
- Most projects will work without changes
- If cspot does custom I/O redirection per task, it will fail
- Check for any custom stdio handling in main code

**To switch back to Newlib (if needed):**
```
menuconfig → Component config → Newlib configuration → Use Newlib
```

### Power Management and Sleep Wakeup Changes

**API Change:**
```cpp
// Old API (deprecated)
esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
if (cause == ESP_SLEEP_WAKEUP_EXT1) { /* ... */ }

// New API (recommended)
uint32_t causes = esp_sleep_get_wakeup_causes();  // Returns bitmap
if (causes & BIT(ESP_SLEEP_WAKEUP_EXT1)) { /* ... */ }
if (causes & BIT(ESP_SLEEP_WAKEUP_TIMER)) { /* ... */ }
```

**Removed APIs:**
- `esp_deep_sleep_enable_gpio_wakeup()` → Use `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown()`
- `gpio_deep_sleep_wakeup_enable()` → Use `gpio_wakeup_enable_on_hp_periph_powerdown_sleep()`

**Impact on cspot:** Low - unless implementing sleep modes for power management.

### GPIO Wakeup Updates

If cspot uses GPIO wakeup for sleep:
```cpp
// Old way
esp_deep_sleep_enable_gpio_wakeup(BIT(GPIO_NUM_0), ESP_GPIO_WAKEUP_GPIO_LOW);
GPIO_IS_DEEP_SLEEP_WAKEUP_VALID_GPIO(GPIO_NUM_0)

// New way
esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown(BIT(GPIO_NUM_0), ESP_GPIO_WAKEUP_GPIO_LOW);
GPIO_IS_HP_PERIPH_PD_WAKEUP_VALID_IO(GPIO_NUM_0)
```

---

## 5. Storage: NVS, SPIFFS, and VFS Changes

### Current cspot Usage (from `targets/esp32/main/main.cpp`)

```cpp
#include "esp_spiffs.h"
#include "nvs_flash.h"

std::string credentialsFileName = "/spiffs/authBlob.json";

void init_spiffs() {
  esp_vfs_spiffs_conf_t conf = {
      .base_path = "/spiffs",
      .partition_label = NULL,
      .max_files = 5,
      .format_if_mount_failed = true
  };
  esp_err_t ret = esp_vfs_spiffs_register(&conf);
  
  esp_spiffs_info(conf.partition_label, &total, &used);
}

void init_nvs() {
  esp_err_t ret = nvs_flash_init();
  if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());
    ret = nvs_flash_init();
  }
}
```

### ESP-IDF v6.0 Changes

#### VFS Layer Changes

**Breaking changes:**
- Deprecated UART-VFS functions removed (`esp_vfs_dev_uart_*`)
- Deprecated USB Serial JTAG-VFS functions removed (`esp_vfs_dev_usb_serial_jtag_*`)
- `esp_vfs_register_fd_range()` is now private
- Legacy VFS APIs using `esp_vfs_t` deprecated; migrate to `esp_vfs_fs_ops_t`
- **TERMIOS support disabled by default** (`CONFIG_VFS_SUPPORT_TERMIOS=n`)
- Context-less VFS function pointers deprecated; use context-aware `*_p` callbacks

**For cspot:** Most changes don't affect SPIFFS usage directly, but be aware if using UART or USB Serial JTAG for logging.

#### SPIFFS Integration

**Good news:** SPIFFS API remains stable, but uses VFS under the hood.

**Minimal changes needed:**
```cpp
// Old code works, but consider updating to new style if desired
esp_vfs_spiffs_conf_t conf = {
    .base_path = "/spiffs",
    .partition_label = NULL,
    .max_files = 5,
    .format_if_mount_failed = true
};
ESP_ERROR_CHECK(esp_vfs_spiffs_register(&conf));

// Query info (API unchanged)
size_t total = 0, used = 0;
ESP_ERROR_CHECK(esp_spiffs_info(conf.partition_label, &total, &used));

// File operations via standard POSIX calls remain unchanged
FILE *f = fopen("/spiffs/authBlob.json", "r");
```

#### NVS Compatibility

**Good news:** NVS API is largely stable. No breaking changes for basic operations.

**Code continues to work:**
```cpp
// Still valid
ESP_ERROR_CHECK(nvs_flash_init());
nvs_handle_t nvs_handle;
ESP_ERROR_CHECK(nvs_open("namespace", NVS_READWRITE, &nvs_handle));
ESP_ERROR_CHECK(nvs_set_str(nvs_handle, "key", "value"));
ESP_ERROR_CHECK(nvs_commit(nvs_handle));
nvs_close(nvs_handle);
```

#### FATFS Changes

If cspot uses SD cards with FATFS:
- Dynamic buffers enabled by default (`CONFIG_FATFS_USE_DYN_BUFFERS=y`)
- Long filename support uses heap by default (`CONFIG_FATFS_LFN_HEAP=y`)

**No action needed for cspot** - SPIFFS doesn't use FATFS.

#### Deprecated Functions Removed

**Removed:**
- `esp_vfs_fat_sdmmc_unmount()` → Use `esp_vfs_fat_sdcard_unmount()`
- Function prototype for `esp_vfs_fat_register()` changed

**Not used in cspot**, but important if adding SD card support.

### Recommended Actions for cspot

1. **No changes required** for basic SPIFFS and NVS usage
2. **Verify UART logging** works (UART-VFS functions removed)
3. **Test file I/O** after migration to ensure SPIFFS works correctly
4. **Optional:** Update to context-aware VFS callbacks if directly using VFS API (unlikely for cspot)

---

## 6. Wi-Fi and Networking Changes

### Current cspot Usage

cspot uses Wi-Fi for Spotify streaming and credential storage. Check `targets/esp32/main/main.cpp` for initialization.

### ESP-IDF v6.0 Wi-Fi Changes

#### Removed Functions and Types

| Old Function | Replacement | Impact |
|-------------|-------------|--------|
| `esp_wifi_set_ant_gpio()` | `esp_phy_set_ant_gpio()` | Antenna configuration |
| `esp_wifi_set_ant()` | `esp_phy_set_ant()` | Antenna configuration |
| `esp_wifi_config_espnow_rate()` | `esp_now_set_peer_rate_config()` | ESP-NOW rate config |
| `esp_supp_dpp_init(callback)` | `esp_supp_dpp_init(void)` | No callback parameter |
| `esp_wifi_wps_start(timeout)` | `esp_wifi_wps_start(void)` | No timeout parameter |

#### Removed Enums and Macros

```cpp
// Removed in v6.0
WIFI_AUTH_WPA3_EXT_PSK           → Use WIFI_AUTH_WPA3_PSK
WIFI_AUTH_WPA3_EXT_PSK_MIXED_MODE → Use WIFI_AUTH_WPA3_PSK
WIFI_BW_HT20                     → Use WIFI_BW20
WIFI_BW_HT40                     → Use WIFI_BW40
ESP_IF_WIFI_STA                  → Use WIFI_IF_STA (enum value)
ESP_IF_WIFI_AP                   → Use WIFI_IF_AP (enum value)
```

#### Header File Changes

```cpp
// Removed: components/esp_wifi/include/esp_interface.h
// The wifi_interface_t enum moved to:
#include "esp_wifi_types_generic.h"  // Contains WIFI_IF_STA, WIFI_IF_AP
```

#### Disconnection Reason Changes

```cpp
// Renamed in v6.0
WIFI_REASON_ASSOC_EXPIRE           → WIFI_REASON_AUTH_EXPIRE
WIFI_REASON_NOT_AUTHED             → WIFI_REASON_CLASS2_FRAME_FROM_NONAUTH_STA
WIFI_REASON_NOT_ASSOCED            → WIFI_REASON_CLASS3_FRAME_FROM_NONASSOC_STA
```

#### NAN Mode Changes (If Used)

**Significant restructure:**
```cpp
// Old API
esp_wifi_nan_start()
esp_wifi_nan_stop()
WIFI_EVENT_NAN_STARTED / WIFI_EVENT_NAN_STOPPED

// New API - Separate sync and unsync modes
esp_wifi_nan_sync_start()
esp_wifi_nan_sync_stop()
WIFI_EVENT_NAN_SYNC_STARTED / WIFI_EVENT_NAN_SYNC_STOPPED
```

**Structure changes:**
```cpp
// Old
wifi_nan_config_t nan_config = WIFI_NAN_CONFIG_DEFAULT();

// New
wifi_nan_sync_config_t nan_sync_config = WIFI_NAN_SYNC_CONFIG_DEFAULT();
```

#### Off-Channel Operations

If cspot uses action frame transmission:
```cpp
// New field required in structures
wifi_action_tx_req_t action_tx = {
    .bssid = {...},          // New field - must be set
    .frame_len = ...
};

wifi_roc_req_t roc_req = {
    .allow_broadcast = false, // New field - controls broadcast frame reception
};
```

#### Re-initialization Behavior Change

```cpp
// Old behavior (v5.x): esp_wifi_init() called twice returns ESP_OK
esp_wifi_init(&cfg);
esp_wifi_init(&cfg);  // Returns ESP_OK

// New behavior (v6.0): Second call returns ESP_ERR_INVALID_STATE
esp_wifi_init(&cfg);
esp_wifi_init(&cfg);  // Returns ESP_ERR_INVALID_STATE - ERROR!
```

**Fix for cspot:**
```cpp
// Check if already initialized
if (esp_wifi_get_init_status() == false) {
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
}
```

### Recommended Actions for cspot

1. **Update Wi-Fi initialization** to handle `ESP_ERR_INVALID_STATE` on re-init
2. **Check antenna configuration** - if used, switch from `esp_wifi` to `esp_phy` functions
3. **Update event handling** - verify disconnect reason codes are handled correctly
4. **Test Wi-Fi connectivity** thoroughly after migration

---

## 7. Security and Cryptography (mbedTLS)

### Current cspot Usage

cspot uses mbedTLS for:
- TLS connections (Spotify API)
- Certificate verification
- Possibly HTTPS client operations

### Major Changes in ESP-IDF v6.0

#### mbedTLS Upgraded to v4.0

**Breaking change:** PSA Crypto is now the **primary cryptography interface**.

**Removed deprecated headers:**
```cpp
// Removed - include replacements
#include "esp32/aes.h"        → #include "aes/esp_aes.h"
#include "esp32/sha.h"        → #include "sha/sha_core.h"
#include "esp32s2/aes.h"      → #include "aes/esp_aes.h"
#include "esp32s2/sha.h"      → #include "sha/sha_core.h"
#include "esp32s2/gcm.h"      → #include "aes/esp_aes_gcm.h"
#include "sha/sha_dma.h"      → #include "sha/sha_core.h"
#include "sha/sha_block.h"    → #include "sha/sha_core.h"
```

#### PSA Crypto Migration Required

**Critical:** `psa_crypto_init()` must be called before any cryptographic operation.

```cpp
#include "psa/crypto.h"

// MUST be called before TLS operations
psa_status_t status = psa_crypto_init();
if (status != PSA_SUCCESS) {
    ESP_LOGE(TAG, "PSA Crypto init failed: %d", status);
    return ESP_FAIL;
}

// Now safe to use mbedTLS/PSA crypto
esp_http_client_init(&config);  // Uses crypto internally
```

**Removed APIs (Cryptography):**
```cpp
// Legacy ECDSA APIs removed
esp_ecdsa_load_pubkey()              // Removed
esp_ecdsa_privkey_load_mpi()         // Removed
esp_ecdsa_privkey_load_pk_context()  // Removed
esp_ecdsa_set_pk_context()           // Removed
esp_ecdsa_tee_load_pubkey()          // Removed
esp_ecdsa_tee_set_pk_context()       // Removed
esp_aes_encrypt()                    // Removed
esp_aes_decrypt()                    // Removed

// Use PSA Crypto instead
#include "psa/crypto.h"
// PSA provides unified interface for all crypto ops
```

#### TLS 1.2 / DTLS 1.2 Compatibility

**Removed cipher suites:**
- Finite-field DHE key exchange
- RSA key exchange without forward secrecy
- Static ECDH

**Removed curves:**
- secp192r1 (P-192)
- secp224r1 (P-224)
- Any elliptic curves < 250 bits

**Impact on cspot:** Low - Spotify API likely uses modern TLS with PFS.

#### Default Configuration Changes

**New defaults (stricter security + smaller footprint):**
- `MBEDTLS_ARIA_C` disabled (must enable explicitly if needed)
- `MBEDTLS_THREADING_C` enabled (thread-safe PSA key management)
- `MBEDTLS_THREADING_PTHREAD` enabled

### Recommended Actions for cspot

1. **Add PSA Crypto initialization:**
   ```cpp
   #include "psa/crypto.h"
   
   void app_main() {
       // Initialize PSA Crypto EARLY - before any networking
       psa_status_t status = psa_crypto_init();
       ESP_ERROR_CHECK(status == PSA_SUCCESS ? ESP_OK : ESP_FAIL);
       
       // Now safe to initialize Wi-Fi and HTTP client
       esp_http_client_init(&config);
   }
   ```

2. **Verify no direct mbedTLS legacy crypto calls** - search codebase for `mbedtls_sha*`, `mbedtls_aes*`, etc.

3. **Test HTTPS connectivity** with Spotify API after migration

4. **No action needed** for standard TLS usage (handled by ESP-IDF internally)

---

## 8. PSRAM and Memory Management

### Current cspot Usage (from `cspot/bell/main/utilities/include/BellTask.h`)

```cpp
bool runOnPSRAM;  // Flag to allocate task stack in PSRAM

if (runOnPSRAM) {
    xTaskCreatePinnedToCore(taskEntryFuncPSRAM, 
                           this->TASK.c_str(), 
                           this->stackSize, 
                           this,
                           this->PRIORITY, 
                           &this->taskHandle, 
                           xPortGetCoreID());
}
```

### ESP-IDF v6.0 Changes

#### DMA Memory Allocation Changes

**Removed APIs:**
```cpp
esp_dma_capable_malloc()   // Removed
esp_dma_capable_calloc()   // Removed
```

**Replacement pattern:**
```cpp
// Old way (removed)
void *dma_buf = esp_dma_capable_malloc(size);

// New way (v6.0)
#include "esp_heap_caps.h"

void *dma_buf = heap_caps_malloc(size, MALLOC_CAP_DMA | MALLOC_CAP_CACHE_ALIGNED);
void *dma_buf2 = heap_caps_calloc(count, size, MALLOC_CAP_DMA | MALLOC_CAP_CACHE_ALIGNED);
```

**Heap capabilities flags (for reference):**
- `MALLOC_CAP_DMA` - Buffer usable by DMA
- `MALLOC_CAP_CACHE_ALIGNED` - Properly aligned for cache operations
- `MALLOC_CAP_SPIRAM` - Allocate from PSRAM
- `MALLOC_CAP_INTERNAL` - Allocate from internal RAM

#### DMA Burst Size Configuration

**Replaced in various driver configs:**
```cpp
// Old pattern (removed)
struct config_t {
    int sram_trans_align;
    int psram_trans_align;
};

// New pattern (v6.0)
struct config_t {
    size_t dma_burst_size;  // Single setting instead of two
};
```

**Affected components:**
- `async_memcpy_config_t` - Remove `sram_trans_align`, `psram_trans_align`
- `esp_lcd_i80_bus_config_t` - Use `dma_burst_size` instead
- `esp_lcd_rgb_panel_config_t` - Use `dma_burst_size` instead

#### GDMA Driver Changes

**Removed:**
- `gdma_new_channel()` - Function removed

**Replacements:**
```cpp
// Old way (removed)
gdma_channel_handle_t ch = NULL;
gdma_new_channel(&config, &ch);

// New way (v6.0)
gdma_channel_handle_t ch = NULL;
// Use type-specific function based on bus type:
gdma_new_ahb_channel(&config, &ch);   // For AHB bus devices
// or
gdma_new_axi_channel(&config, &ch);   // For AXI bus devices
```

**Other removals:**
- `GDMA_ISR_IRAM_SAFE` Kconfig option removed
- Removed `sram_trans_align`, `psram_trans_align` from `async_memcpy_config_t`

### Recommended Actions for cspot

1. **PSRAM task stack allocation** - Should continue working without changes
2. **If using I2S with DMA:**
   ```cpp
   // Update DMA buffer allocation if present
   #include "esp_heap_caps.h"
   
   // For I2S DMA buffers
   uint8_t *dma_buf = (uint8_t *)heap_caps_malloc(
       DMA_BUFFER_SIZE, 
       MALLOC_CAP_DMA | MALLOC_CAP_CACHE_ALIGNED
   );
   ```

3. **No changes needed** for basic PSRAM usage in tasks (FreeRTOS handles this internally)

---

## 9. Toolchain and Compilation Changes

### GCC Version Upgraded to 15.1.0

**Major version upgrade from GCC 14.2.0 to 15.1.0**

**New warnings introduced:** See full list in [GCC Warning Options](https://gcc.gnu.org/onlinedocs/gcc-15.1.0/gcc/Warning-Options.html)

#### Common GCC 15.1.0 Warnings for cspot

**1. `-Wno-unterminated-string-initialization`**

```cpp
// Warning: Unterminated string in char array
char config[3] = "foo";  // Array size < string length

// Fix:
char config[4] = "foo";  // Size includes null terminator
// Or disable per-item:
char config_nc[3] NONSTRING_ATTR = "foo";
```

**2. `-Wno-dangling-reference` (C++)**

```cpp
// Warning: Reference to temporary
const int& ref = std::max(a, b);  // b is temporary, destroyed immediately

// Fix:
int result = std::max(a, b);  // Use value
// Or:
const auto& ref = static_cast<const int&>(std::max(a, b));
```

**3. `-Wno-defaulted-function-deleted` (C++)**

```cpp
// Warning: Template specialization has deleted members
template<typename T>
struct Container {
    Container(const Container&&) = default;  // May be implicitly deleted
};

// Fix: Ensure all members are copyable/movable
```

**4. `-Wno-self-move` (C++)**

```cpp
// Warning: Variable moved to itself
obj = std::move(obj);  // Has no effect

// Fix: Remove unnecessary std::move
```

#### sys/dirent.h Header Change

**Breaking:** `#include <sys/dirent.h>` no longer provides function prototypes.

```cpp
// Old code (breaks in GCC 15.1.0)
#include <sys/dirent.h>
DIR *dir = opendir("path");  // Error: implicit declaration

// Fix: Include correct header
#include <dirent.h>
DIR *dir = opendir("path");  // Now works
```

### Recommended Actions for cspot

1. **Run compilation** and fix new GCC 15.1.0 warnings
2. **Priority fixes:**
   - String initialization sizes
   - Dangling references in C++ code
   - `#include <dirent.h>` instead of `<sys/dirent.h>`

3. **If compilation fails on warnings:**
   ```kconfig
   # In sdkconfig (temporary measure, not recommended)
   CONFIG_COMPILER_DISABLE_DEFAULT_ERRORS=y
   CONFIG_COMPILER_DISABLE_GCC15_WARNINGS=y
   ```

---

## 10. Provisioning and Networking

### Wi-Fi Provisioning API

**Minimal breaking changes.** If cspot uses WiFi provisioning (BLE/SoftAP), verify:
- Event handlers still work
- Manager initialization compatible

### HTTP/HTTPS Client

No major breaking changes for `esp_http_client`.

**Ensure PSA Crypto is initialized first:**
```cpp
psa_crypto_init();  // Must be called before HTTP client setup
esp_http_client_init(&config);
```

---

## 11. Migration Checklist for cspot

### Priority 1 (Critical - Must Do)

- [ ] **Update I2S driver API** - All audio sinks use new handle-based API
  - [ ] InternalAudioSink.cpp - Refactor or use external DAC
  - [ ] ES8311AudioSink.cpp - Update I2S config/API calls
  - [ ] ES8388AudioSink.cpp - Update I2S config/API calls
  - [ ] AC101AudioSink.cpp - Update I2S config/API calls
  - [ ] PCM5102AudioSink.cpp - Update I2S config/API calls
  - [ ] ES9018AudioSink.cpp - Update I2S config/API calls
  - [ ] SPDIFAudioSink.cpp - Update I2S config/API calls
  - [ ] BufferedAudioSink.cpp - Update I2S write/set_clk calls

- [ ] **Update CMakeLists.txt** - Add explicit driver component dependencies
  - [ ] Add `esp_driver_i2s` to REQUIRES
  - [ ] Add `esp_driver_gpio` if not already present
  - [ ] Add `esp_driver_dma` if using DMA

- [ ] **Verify includes** - Replace deprecated headers
  - [ ] Remove `#include "driver/i2s.h"`
  - [ ] Add `#include "driver/i2s_std.h"` (or appropriate I2S mode)
  - [ ] Check for `#include <sys/dirent.h>` → `#include <dirent.h>`

- [ ] **Fix GCC 15.1.0 compilation warnings**
  - [ ] String initialization sizes
  - [ ] Dangling references
  - [ ] Self-move warnings

- [ ] **Add PSA Crypto initialization**
  ```cpp
  psa_crypto_init();  // Call in app_main() before Wi-Fi/HTTPS
  ```

### Priority 2 (Important - Should Do)

- [ ] **Test Wi-Fi connectivity** - Verify re-init behavior works
- [ ] **Test NVS/SPIFFS** - Ensure credential storage works
- [ ] **Test audio playback** - All I2S audio output paths
- [ ] **Verify no orphan sections** - Check linker errors during build
- [ ] **Update Wi-Fi event handlers** - Handle new enum values for disconnect reasons

### Priority 3 (Nice to Have)

- [ ] Test PSRAM task allocation (should work unchanged)
- [ ] Optimize memory allocations if using DMA
- [ ] Consider switching to Picolibc if binary size matters
- [ ] Update constructor initialization if ordering is critical

---

## 12. Testing Strategy

### Test Areas for cspot

1. **Audio Playback**
   - Play a Spotify track to various audio sinks
   - Verify sample rate switching (e.g., 44.1kHz → 48kHz)
   - Check for audio dropouts or glitches

2. **Wi-Fi Connectivity**
   - Connect to known network
   - Disconnect and reconnect
   - Verify correct disconnect reason codes in logs

3. **Credential Storage**
   - Store and retrieve Spotify credentials from SPIFFS
   - Verify NVS operations work

4. **Power Management**
   - If implemented, test sleep/wake cycles
   - Verify GPIO wakeup (if used)

5. **TLS/HTTPS**
   - Test Spotify API calls over HTTPS
   - Verify certificate validation works

---

## 13. Migration Path Summary

### Step 1: Update Build Configuration
- Modify CMakeLists.txt to explicitly require driver components
- Verify no orphan section errors

### Step 2: Update I2S Driver (Largest Task)
- Rewrite I2S initialization for each audio sink
- Test audio playback with each configuration
- Handle built-in DAC removal or use external DAC

### Step 3: Update Headers and Imports
- Replace deprecated headers
- Remove old `#include "driver/i2s.h"`
- Fix dirent.h includes

### Step 4: Fix Compilation Warnings
- Address GCC 15.1.0 new warnings
- Fix string initialization, dangling references
- Enable warnings-as-errors once clean

### Step 5: Add PSA Crypto Initialization
- Call `psa_crypto_init()` in app_main()
- Ensure it's called before Wi-Fi/HTTPS operations

### Step 6: Test and Validate
- Full functional testing of audio, Wi-Fi, and storage
- Verify all audio sinks work
- Test TLS connections to Spotify API

---

## References

- [ESP-IDF v6.0 Migration Guide (Official)](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/migration-guides/release-6.x/6.0/index.html)
- [ESP-IDF I2S Driver Documentation](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32/api-reference/peripherals/i2s.html)
- [PSA Crypto API Reference](https://arm-software.github.io/psa-api/)
- [mbedTLS 4.0 Migration Guide](https://github.com/espressif/mbedtls/blob/6cc42af/docs/4.0-migration-guide.md)
- [GCC 15 Porting Guide](https://gcc.gnu.org/gcc-15/porting_to.html)

---

## Quick Reference: Breaking Change Summary

| Component | Old | New | Impact |
|-----------|-----|-----|--------|
| **I2S Driver** | `driver/i2s.h` | `driver/i2s_std.h`, etc. | **HIGH** - Major refactor needed |
| **LibC** | Newlib | Picolibc (default) | **LOW** - Works for most code |
| **Compiler** | GCC 14.2 | GCC 15.1.0 | **MEDIUM** - New warnings |
| **mbedTLS** | v3.x | v4.0 (PSA Crypto) | **MEDIUM** - Must call `psa_crypto_init()` |
| **Wi-Fi Re-init** | Returns OK | Returns ERROR | **LOW** - Easy to fix |
| **DMA Malloc** | `esp_dma_capable_malloc()` | `heap_caps_malloc()` | **LOW** - Only if used |
| **NVS/SPIFFS** | Unchanged | Unchanged | **NONE** - Fully compatible |

---

**Document Version:** 1.0  
**Last Updated:** 2024  
**For:** cspot ESP32 Audio Streaming Project  
