# VL53LMZ ULD Driver

Ultra-Lite Driver (ULD) for the STMicroelectronics **VL53LMZ** Time-of-Flight (ToF) ranging sensor, supporting L5, L7, and L8 module types.

> **Driver version:** VL53LMZ_2.0.9  
> **Original copyright:** STMicroelectronics (2023)

## Overview

The VL53LMZ is a multi-zone ToF sensor with:
- **4×4** or **8×8** zone resolution
- Up to **4 targets per zone**
- Motion detection indicator
- Continuous and autonomous ranging modes
- I2C interface (default address: `0x52`)

This driver provides the low-level ULD API for initializing, configuring, and reading data from the VL53LMZ sensor on embedded platforms.

## File Structure

```
vl53lmz-uld-driver/
├── inc/
│   ├── vl53lmz_api.h        # Public API — data structures, macros, all function declarations
│   └── vl53lmz_buffers.h    # Firmware binary, default configuration & crosstalk calibration data
├── src/
│   └── vl53lmz_api.c        # Driver implementation
├── README.md
└── .gitignore
```

## Platform Dependencies

This driver requires a `platform.h` header that provides the hardware abstraction layer. You must implement the following functions:

| Function | Description |
|----------|-------------|
| `RdByte(VL53LMZ_Platform*, uint16_t, uint8_t*)` | Read a single byte via I2C |
| `RdMulti(VL53LMZ_Platform*, uint16_t, uint8_t*, uint32_t)` | Read multiple bytes via I2C |
| `WrByte(VL53LMZ_Platform*, uint16_t, uint8_t)` | Write a single byte via I2C |
| `WrMulti(VL53LMZ_Platform*, uint16_t, uint8_t*, uint32_t)` | Write multiple bytes via I2C |
| `WaitMs(VL53LMZ_Platform*, uint32_t)` | Blocking delay in milliseconds |
| `SwapBuffer(uint8_t*, uint16_t)` | Endian-swap buffer in place |

The `VL53LMZ_Platform` struct holds at minimum the I2C address:

```c
typedef struct {
    uint16_t address;   // I2C device address
    // Add your platform-specific fields here (e.g., I2C handle)
} VL53LMZ_Platform;
```

### Build-time Configuration

Define these macros to customize the driver:

| Macro | Description |
|-------|-------------|
| `VL53LMZ_NB_TARGET_PER_ZONE` | Number of targets per zone (default: 1, max: 4) |
| `VL53LMZ_DISABLE_AMBIENT_PER_SPAD` | Exclude ambient light data from output |
| `VL53LMZ_DISABLE_NB_SPADS_ENABLED` | Exclude SPAD count data |
| `VL53LMZ_DISABLE_NB_TARGET_DETECTED` | Exclude target count per zone |
| `VL53LMZ_DISABLE_SIGNAL_PER_SPAD` | Exclude signal rate data |
| `VL53LMZ_DISABLE_RANGE_SIGMA_MM` | Exclude range sigma data |
| `VL53LMZ_DISABLE_DISTANCE_MM` | Exclude distance data |
| `VL53LMZ_DISABLE_REFLECTANCE_PERCENT` | Exclude reflectance data |
| `VL53LMZ_DISABLE_TARGET_STATUS` | Exclude target status data |
| `VL53LMZ_DISABLE_MOTION_INDICATOR` | Exclude motion detection data |
| `VL53LMZ_USE_RAW_FORMAT` | Skip post-processing conversion to real units |

## Quick Start

```c
#include "vl53lmz_api.h"

VL53LMZ_Configuration dev;
VL53LMZ_ResultsData   results;
uint8_t is_ready, status;

// 1. Check sensor is alive
status = vl53lmz_is_alive(&dev, &is_alive);
if (!is_alive) { /* handle error */ }

// 2. Initialize (loads FW, takes ~hundreds of ms)
status = vl53lmz_init(&dev);

// 3. Configure (optional)
vl53lmz_set_resolution(&dev, VL53LMZ_RESOLUTION_8X8);
vl53lmz_set_ranging_frequency_hz(&dev, 10);
vl53lmz_set_integration_time_ms(&dev, 50);

// 4. Start ranging
status = vl53lmz_start_ranging(&dev);

// 5. Read data loop
while (1) {
    status = vl53lmz_check_data_ready(&dev, &is_ready);
    if (is_ready) {
        status = vl53lmz_get_ranging_data(&dev, &results);
        // Use results.distance_mm[], results.ambient_per_spad[], etc.
    }
    WaitMs(&dev.platform, 5);
}

// 6. Stop ranging
vl53lmz_stop_ranging(&dev);
```

## Key API Functions

| Function | Description |
|----------|-------------|
| `vl53lmz_is_alive()` | Check sensor presence via I2C |
| `vl53lmz_init()` | Full initialization — loads firmware (~hundreds of ms) |
| `vl53lmz_start_ranging()` | Create output config and start streaming |
| `vl53lmz_stop_ranging()` | Stop streaming session |
| `vl53lmz_check_data_ready()` | Poll for new measurement data |
| `vl53lmz_get_ranging_data()` | Read and parse measurement results |
| `vl53lmz_set_resolution()` | Switch between 4×4 and 8×8 modes |
| `vl53lmz_set_ranging_frequency_hz()` | Set measurement rate (1–60 Hz 4×4, 1–15 Hz 8×8) |
| `vl53lmz_set_integration_time_ms()` | Set integration time (2–1000 ms) |
| `vl53lmz_set_ranging_mode()` | Continuous or autonomous mode |
| `vl53lmz_set_power_mode()` | Sleep/wakeup for power saving |
| `vl53lmz_set_i2c_address()` | Change I2C address (for multi-sensor setups) |
| `vl53lmz_set_sharpener_percent()` | Adjust zone sharpening (0–99%) |
| `vl53lmz_set_target_order()` | Closest or strongest target reporting |
| `vl53lmz_set_external_sync_pin_enable()` | Enable sync pin for multi-device sync (L8 only) |
| `vl53lmz_set_glare_filter_cfg()` | Configure glare filter threshold and range |
| `vl53lmz_enable_internal_cp()` / `vl53lmz_disable_internal_cp()` | VCSEL charge pump control (L5/L7 only) |
| `vl53lmz_dci_read_data()` / `vl53lmz_dci_write_data()` | Direct DCI register access |
| `vl53lmz_add_output_block()` / `vl53lmz_disable_output_block()` | Customize output data blocks |
| `vl53lmz_results_extract_block()` | Extract a single data block from results |

## Supported Modules

| Module | Chip ID | Revision ID | Macro |
|--------|---------|-------------|-------|
| VL53L5CX | 0xF0 | 0x01 | `VL53LMZ_MODULE_TYPE_L5` |
| VL53L7CX | — | 0x02 | `VL53LMZ_MODULE_TYPE_L7` |
| VL53L8CX | — | 0x0C | `VL53LMZ_MODULE_TYPE_L8` |

## License

This software is copyright © 2023 STMicroelectronics. All rights reserved.

The original source headers state: *"This software is licensed under terms that can be found in the LICENSE file in the root directory of this software component. If no LICENSE file comes with this software, it is provided AS-IS."*

Please refer to STMicroelectronics' official VL53LMZ ULD distribution for the complete license terms.

## References

- [VL53L5CX Product Page](https://www.st.com/en/imaging-and-photonics-solutions/vl53l5cx.html)
- [VL53L7CX Product Page](https://www.st.com/en/imaging-and-photonics-solutions/vl53l7cx.html)
- [VL53L8CX Product Page](https://www.st.com/en/imaging-and-photonics-solutions/vl53l8cx.html)
