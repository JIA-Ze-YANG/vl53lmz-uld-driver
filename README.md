# VL53LMZ ULD Driver

Ultra-Lite Driver (ULD) for the STMicroelectronics **VL53LMZ** Time-of-Flight (ToF) ranging sensor, supporting L5, L7, and L8 module types.

> **Driver version:** VL53LMZ_2.0.9  
> **Original copyright:** STMicroelectronics (2023)

---

##  项目介绍 / Introduction

**VL53LMZ ULD Driver** 是 STMicroelectronics（意法半导体）VL53LMZ 系列 ToF 激光测距传感器的超轻量驱动库（Ultra-Lite Driver），专为嵌入式 MCU 平台移植和二次开发而整理。

本仓库基于[**逐飞开源库（Seekfree CYT4BB Opensource Library）**](https://github.com/JIA-Ze-YANG/sixuanyi)开发，提供了 **VL53L8（VL53L8CX）的软件 I2C 驱动例程**以及相关烧录固件数据（`vl53lmz_buffers.h` 中的固件二进制与校准参数）。去除了特定平台的耦合依赖，**仅需实现 6 个平台抽象函数**即可适配任意 MCU（STM32、ESP32、Infineon CYT4BB7 等），方便快速集成到各类嵌入式项目中。

> ⚠️ **使用注意**：由于软件 I2C 采用阻塞式读取数据，在与 IMU、其他环境传感器、姿态传感器等**多传感器并用**时，建议将 VL53L8 的数据读取放在**主循环**中执行，避免阻塞其他传感器的实时数据采集。

###  传感器简介

VL53LMZ 系列是 ST 推出的**多区域（Multi-Zone）ToF 测距传感器**，采用 940nm VCSEL 激光，基于 FlightSense™ 飞行时间技术，可在任何光照条件下实现精确测距。目前已验证支持以下模块：

| 型号 | 分辨率 | 最大距离 | 适用场景 |
|------|--------|----------|----------|
| VL53L5CX | 4×4 / 8×8 | ~4m | 多区域测距、物体检测 |
| VL53L7CX | 4×4 / 8×8 | ~3.5m | 低功耗人体存在检测 |
| VL53L8CX | 4×4 / 8×8 | ~4m | 增强型，支持同步引脚 |

### ✨ 主要特性

- **多区域测距**：支持 4×4（16 区）或 8×8（64 区）分辨率
- **多目标检测**：每区最多 4 个目标
- **运动检测**：内置运动指示器（Motion Indicator）
- **双模式**：连续模式 / 自主模式（可设精确积分时间）
- **低功耗**：支持睡眠 / 唤醒模式
- **I2C 通信**：默认地址 0x52，可配置多传感器级联
- **眩光过滤**（Glare Filter）：减少盖板玻璃反射干扰
- **平台无关**：仅需实现 6 个 HAL 函数，无 RTOS 依赖

###  硬件连接

```
VL53LMZ 模块           MCU
┌──────────┐        ┌──────┐
│  VDD     │───────▶│ 3.3V │
│  GND     │────────│ GND  │
│  SDA     │────────│ SDA  │
│  SCL     │────────│ SCL  │
│  LPn     │────────│ GPIO │ (可选：多传感器时用于切换)
│  INT      │────────│ GPIO │ (可选：中断引脚)
└──────────┘        └──────┘
```

>  ⚠️ 注意：VL53L5CX/L7CX 需要 **AVDD = 3.3V** 且 IO 电平匹配。部分模块上已经有 LDO 和电平转换电路，请以实际模块原理图为准。

###  移植说明

本仓库**不依赖任何特定 MCU 平台**。要适配你的硬件，只需创建一个 `platform.h` 文件，实现以下 6 个函数即可。详见下方 [Platform Dependencies](#platform-dependencies) 章节。

---

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
