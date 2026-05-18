# ESP-IDF v6.0 Migration Documentation for cspot

## 📋 Overview

This directory contains comprehensive documentation for migrating the cspot ESP32 audio project from **ESP-IDF v5.x to v6.0**. The documentation focuses on breaking changes that directly impact the cspot project, which is a 2+ year old Spotify client implementation.

### Key Facts About This Migration

- **Effort Estimate:** 6-11 hours
- **Difficulty Level:** MEDIUM
- **Critical Component:** I2S Driver (complete API overhaul)
- **Other Major Changes:** Build system, cryptography, Wi-Fi, component dependencies
- **Minimum Action Items:** I2S API updates + CMakeLists.txt fixes

---

## 📚 Documentation Files

### 1. **ESP_IDF_V6_MIGRATION_GUIDE.md** (31 KB, 1,023 lines)
**Comprehensive Deep-Dive Reference**

Complete technical documentation covering all breaking changes in ESP-IDF v6.0 with specific relevance to cspot.

**Contents:**
- Executive summary of key changes
- I2S driver API overhaul (detailed with old vs. new patterns)
- Component registration and dependency changes
- Build system changes (linker orphan sections, constructor order)
- System and memory changes (Newlib → Picolibc, power management)
- Storage layer updates (NVS, SPIFFS, VFS)
- Wi-Fi API changes and migration patterns
- Security and cryptography (mbedTLS v4.0, PSA Crypto)
- PSRAM and memory management
- Toolchain changes (GCC 15.1.0)
- Complete migration checklist
- Testing strategy
- Quick reference table

**Best For:**
- Understanding the "why" behind each change
- Deep technical reference during implementation
- Identifying all affected code areas
- Comprehensive project planning

---

### 2. **ESP_IDF_V6_QUICK_REFERENCE.md** (10 KB, 383 lines)
**Fast Lookup Guide for Busy Developers**

Quick-reference format with minimal explanation, focusing on:
- What changed
- Old code vs. new code (side-by-side)
- Files that need updating
- Priority levels (Critical, High, Medium, Low)
- Decision tree for troubleshooting

**Contents:**
- Critical changes priority list
- High-priority changes checklist
- Medium-priority changes with code snippets
- Low-priority / no-action items
- Testing checklist
- File-by-file changes needed
- Effort estimates
- Quick decision tree for common issues

**Best For:**
- Quick lookups during coding
- Identifying which files to modify
- Prioritizing work
- Troubleshooting when things break
- Team communication about what's needed

---

### 3. **I2S_MIGRATION_CODE_EXAMPLES.md** (19 KB, 715 lines)
**Complete I2S Driver Code Migration Examples**

Detailed code examples showing exact before/after patterns for I2S driver migration, the most critical change for cspot.

**Contents:**
- Generic I2S STD mode pattern with minimal example
- BufferedAudioSink.cpp complete migration (before/after)
- ES8311 audio codec migration example
- Handle management patterns (3 variations)
- Sample rate switching examples
- Complete error handling implementation
- I2S slot configuration variations (PCM, MSB-first, 24-bit, mono)
- Troubleshooting with solutions
- Copy-paste templates for quick implementation

**Included Audio Sinks Covered:**
- BufferedAudioSink (base implementation)
- ES8311 (external codec example)
- Patterns applicable to ES8388, AC101, PCM5102, ES9018, SPDIF

**Best For:**
- Implementing the I2S API migration
- Copy-paste starting templates
- Understanding handle management
- Debugging audio issues
- Learning new I2S configuration patterns

---

## 🎯 Quick Start Guide

### For a 5-Minute Overview
1. Read the **Executive Summary** in `ESP_IDF_V6_MIGRATION_GUIDE.md`
2. Check the **Critical Changes** section in `ESP_IDF_V6_QUICK_REFERENCE.md`
3. Review which audio sink files need updating

### For Implementation
1. Start with **`ESP_IDF_V6_QUICK_REFERENCE.md`** - Priority 1 (Critical)
2. Use **`I2S_MIGRATION_CODE_EXAMPLES.md`** for actual coding patterns
3. Cross-reference **`ESP_IDF_V6_MIGRATION_GUIDE.md`** for deeper understanding of specific changes

### For Troubleshooting
1. Check the **Quick Decision Tree** in `ESP_IDF_V6_QUICK_REFERENCE.md`
2. Look up specific error in **Troubleshooting** section of `I2S_MIGRATION_CODE_EXAMPLES.md`
3. Search full guide for detailed explanation

---

## 🔴 Critical Action Items (Do First)

### 1. Update I2S Driver API
**Files:** All 8 audio sink files in `cspot/bell/main/audio-sinks/esp/`

From:
```cpp
#include "driver/i2s.h"
i2s_config_t config = {...};
i2s_driver_install(I2S_NUM_0, &config, 0, NULL);
```

To:
```cpp
#include "driver/i2s_std.h"
i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(...);
i2s_std_config_t std_cfg = {...};
i2s_new_std_tx_channel(&chan_cfg, &std_cfg, &handle);
```

See **I2S_MIGRATION_CODE_EXAMPLES.md** for complete patterns.

### 2. Update CMakeLists.txt
**Files:** Build configuration files

Add to `REQUIRES`:
```cmake
esp_driver_i2s
esp_driver_gpio
esp_driver_dma
```

See **ESP_IDF_V6_QUICK_REFERENCE.md** section 2.

### 3. Fix Header Includes
**Search and replace:**
```
driver/i2s.h       → driver/i2s_std.h
sys/dirent.h       → dirent.h
esp_interface.h    → esp_wifi_types_generic.h
```

### 4. Initialize PSA Crypto
**File:** `targets/esp32/main/main.cpp`

Add early in `app_main()`:
```cpp
#include "psa/crypto.h"
psa_crypto_init();  // Call BEFORE any Wi-Fi or HTTPS
```

### 5. Fix Wi-Fi Re-initialization
**File:** `targets/esp32/main/main.cpp`

```cpp
// Check if already initialized
if (esp_wifi_get_init_status() == false) {
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
}
```

---

## 📊 Impact Summary Table

| Component | Old API | New API | Impact on cspot | Effort |
|-----------|---------|---------|-----------------|--------|
| **I2S Driver** | `driver/i2s.h` | `driver/i2s_std.h` | 🔴 HIGH | 3-4 hrs |
| **Build System** | N/A | Explicit dependencies | 🟡 MEDIUM | 30 min |
| **Compilation** | GCC 14.2 | GCC 15.1.0 | 🟡 MEDIUM | 1-2 hrs |
| **Security** | mbedTLS 3.x | mbedTLS 4.0 + PSA | 🟡 MEDIUM | 30 min |
| **Wi-Fi** | Port-based | Port-based (changed behavior) | 🟢 LOW | 15 min |
| **Memory** | Newlib | Picolibc (default) | 🟢 LOW | 0 min |
| **NVS/SPIFFS** | Legacy API | Unchanged | 🟢 NONE | 0 min |
| **PSRAM** | Direct APIs | Unchanged for tasks | 🟢 LOW | 0 min |

---

## ✅ Testing Checklist

- [ ] **Build Succeeds**
  - [ ] No compilation errors
  - [ ] No GCC 15.1.0 warnings (as errors)
  - [ ] No linker orphan section errors

- [ ] **Audio Playback**
  - [ ] Can connect to Wi-Fi
  - [ ] Can authenticate with Spotify
  - [ ] Audio plays without glitches
  - [ ] Sample rate switching works (44.1kHz ↔ 48kHz)
  - [ ] All audio sinks tested (ES8311, ES8388, AC101, etc.)

- [ ] **Storage & Credentials**
  - [ ] Credentials stored in SPIFFS successfully
  - [ ] Credentials read back correctly
  - [ ] NVS operations work as expected

- [ ] **Security & Connectivity**
  - [ ] HTTPS connections to Spotify API work
  - [ ] Certificate verification passes
  - [ ] No SSL/TLS errors in logs

---

## 📖 Document Organization by Topic

### I2S Driver (Most Critical)
1. **ESP_IDF_V6_MIGRATION_GUIDE.md** → Section 1: "I2S Driver API Changes"
2. **ESP_IDF_V6_QUICK_REFERENCE.md** → Section 1: "I2S Driver API Overhaul"
3. **I2S_MIGRATION_CODE_EXAMPLES.md** → All sections (entire document)

### Build System & Dependencies
1. **ESP_IDF_V6_MIGRATION_GUIDE.md** → Section 2: "Component Registration"
2. **ESP_IDF_V6_MIGRATION_GUIDE.md** → Section 3: "Build System Changes"
3. **ESP_IDF_V6_QUICK_REFERENCE.md** → Section 2: "Component Dependencies"

### Compilation & Toolchain
1. **ESP_IDF_V6_MIGRATION_GUIDE.md** → Section 9: "Toolchain Changes"
2. **ESP_IDF_V6_QUICK_REFERENCE.md** → Section 6: "GCC 15.1.0 Warning Fixes"

### Security & Cryptography
1. **ESP_IDF_V6_MIGRATION_GUIDE.md** → Section 7: "Security and Cryptography"
2. **ESP_IDF_V6_QUICK_REFERENCE.md** → Section 5: "PSA Crypto Initialization"

### Wi-Fi & Networking
1. **ESP_IDF_V6_MIGRATION_GUIDE.md** → Section 6: "Wi-Fi and Networking Changes"
2. **ESP_IDF_V6_QUICK_REFERENCE.md** → Section 4: "Wi-Fi Re-initialization"

### Storage (NVS, SPIFFS)
1. **ESP_IDF_V6_MIGRATION_GUIDE.md** → Section 5: "Storage: NVS, SPIFFS, VFS"
2. **ESP_IDF_V6_QUICK_REFERENCE.md** → "Low Priority" section

### Memory Management & PSRAM
1. **ESP_IDF_V6_MIGRATION_GUIDE.md** → Section 8: "PSRAM and Memory Management"
2. **ESP_IDF_V6_QUICK_REFERENCE.md** → Section 10: "DMA Memory Allocation"

---

## 🔗 External References

### Official ESP-IDF Documentation
- [ESP-IDF v6.0 Migration Guide (Official)](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/migration-guides/release-6.x/6.0/index.html)
- [ESP-IDF I2S Driver Documentation](https://docs.espressif.com/projects/esp-idf/en/v6.0/esp32/api-reference/peripherals/i2s.html)
- [ESP-IDF Build System Documentation](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/build-system.html)

### Security & Cryptography
- [PSA Crypto API Documentation](https://arm-software.github.io/psa-api/)
- [mbedTLS 4.0 Migration Guide](https://github.com/espressif/mbedtls/blob/6cc42af/docs/4.0-migration-guide.md)
- [TF-PSA-Crypto 1.0 Migration Guide](https://github.com/espressif/mbedtls/blob/6cc42af/tf-psa-crypto/docs/1.0-migration-guide.md)

### Compiler
- [GCC 15 Porting Guide](https://gcc.gnu.org/gcc-15/porting_to.html)
- [GCC 15 Warning Options](https://gcc.gnu.org/onlinedocs/gcc-15.1.0/gcc/Warning-Options.html)

---

## 📝 Files in This Directory

```
.
├── ESP_IDF_V6_MIGRATION_GUIDE.md          (Comprehensive deep-dive reference)
├── ESP_IDF_V6_QUICK_REFERENCE.md          (Fast lookup guide)
├── I2S_MIGRATION_CODE_EXAMPLES.md         (I2S API migration code samples)
└── INDEX.md                                (This file - navigation guide)
```

---

## 💡 Pro Tips

1. **Start with the Quick Reference** - Get orientation and priority
2. **Use Code Examples as Templates** - Copy-paste and adapt, don't rewrite
3. **Test Audio Early** - The I2S driver is the highest risk
4. **Build Incrementally** - Update one audio sink at a time
5. **Keep Old Code Handy** - Reference for API understanding
6. **Use the Decision Tree** - When something breaks, use the flowchart
7. **Document Your Changes** - Note what you change for team communication

---

## ❓ FAQ

**Q: How long will migration take?**  
A: 6-11 hours depending on experience level and thoroughness of testing.

**Q: Do I need to migrate everything at once?**  
A: No, but I2S driver is critical. Start there. NVS/SPIFFS can wait.

**Q: Which documentation should I read first?**  
A: Start with `ESP_IDF_V6_QUICK_REFERENCE.md` for orientation, then use specific docs as needed.

**Q: What's the biggest risk?**  
A: I2S driver API changes - all audio sinks depend on it. Test audio immediately.

**Q: Can I roll back if something goes wrong?**  
A: Yes - keep a backup branch of the old ESP-IDF version code.

**Q: Do I need to understand PSA Crypto in depth?**  
A: No, just add one initialization line. ESP-IDF handles the rest internally.

**Q: Will my SPIFFS/NVS data be lost?**  
A: No - NVS and SPIFFS APIs are unchanged. Data persists.

---

## 📞 Support & Questions

When stuck:
1. **Check the Quick Reference** - Section 2-10 covers most issues
2. **Search Code Examples** - Look for pattern matching your use case
3. **Review Migration Guide** - Detailed explanations of each change
4. **Check Official Docs** - Links provided in References section
5. **Troubleshooting Section** - `I2S_MIGRATION_CODE_EXAMPLES.md` has common solutions

---

## 📄 Document Statistics

- **Total Lines of Documentation:** 2,100+
- **Code Examples:** 50+
- **Tables and Comparisons:** 15+
- **Before/After Patterns:** 30+
- **Migration Checklists:** 2

---

**Version:** 1.0  
**Created:** 2024  
**Project:** cspot (Spotify client for ESP32)  
**Target:** ESP-IDF v5.x → v6.0 migration  
**Status:** Complete and comprehensive

---

*Last Updated: 2024*  
*For questions or clarifications, refer to the specific document sections listed above.*
