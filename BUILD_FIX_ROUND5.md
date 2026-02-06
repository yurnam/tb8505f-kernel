# Build Fix Round 5 - Stub Object Files and Additional Header Paths

## Problem
After round 4 fixes, the build progressed further but failed at the linker stage. The stub Makefiles created in earlier rounds existed but didn't actually compile any code, so the linker couldn't find the expected `built-in.o` files. Additionally, several more header files were missing from include paths.

## Analysis of Latest Build Log

### Linker Errors - Missing built-in.o Files

The linker reported it couldn't find:
```
aarch64-linux-gnu-ld: drivers/misc/mediatek/lcm/ft8201m_wxga_vdo_incell_boe/built-in.o kann nicht gefunden werden
aarch64-linux-gnu-ld: drivers/misc/mediatek/lcm/nt36523b_wxga_vdo_incell_inx/built-in.o kann nicht gefunden werden
aarch64-linux-gnu-ld: drivers/misc/mediatek/imgsensor/src/.../hi556txd_mipi_raw/built-in.o kann nicht gefunden werden
aarch64-linux-gnu-ld: drivers/misc/mediatek/imgsensor/src/.../sc500cs_mipi_raw/built-in.o kann nicht gefunden werden
aarch64-linux-gnu-ld: drivers/misc/mediatek/imgsensor/src/.../sc201cs_mipi_raw/built-in.o kann nicht gefunden werden
aarch64-linux-gnu-ld: drivers/misc/mediatek/imgsensor/src/.../gc02m1_mipi_raw/built-in.o kann nicht gefunden werden
```

**Root Cause**: The stub Makefiles contained only comments with no actual object file targets. The kernel build system expects a `built-in.o` file from every directory that's included in the build, even if it's empty.

### Missing Header Compilation Errors

Several drivers failed to compile due to missing header files:

1. **m4u_port.h** - M4U (Multimedia Memory Management Unit) driver
   - Location: `drivers/misc/mediatek/m4u/mt6761/m4u_port.h`
   - Error in: `drivers/misc/mediatek/m4u/mt6761/m4u_hw.c`

2. **mtk_leds_drv.h** - LED driver
   - Location: `drivers/misc/mediatek/leds/mtk_leds_drv.h`
   - Error in: `drivers/misc/mediatek/leds/mtk_leds_drv.c`

3. **mtk_ppm_platform.h** - PPM (Performance Power Management) driver
   - Location: `drivers/misc/mediatek/base/power/ppm_v3/src/mach/mt6761/mtk_ppm_platform.h`
   - Error in: `mtk_ppm_platform.c`, `mtk_ppm_cobra_algo.c`, `mtk_ppm_power_data.c`

4. **mtk_gpufreq.h** - GPU frequency driver
   - Location: `drivers/misc/mediatek/base/power/mt6761/mtk_gpufreq.h`
   - Error in: `drivers/misc/mediatek/base/power/mt6761/mtk_gpufreq_core.c`

5. **cmdq_engine.h** - CMDQ (Command Queue) driver
   - Location: `drivers/misc/mediatek/cmdq/v3/mt6765/cmdq_engine.h`
   - Error in: `drivers/misc/mediatek/cmdq/v3/mt6765/cmdq_mdp.c`

6. **teei_client_main.h** - TEEI (Trusted Execution Environment Interface)
   - Location: Configured via `drivers/misc/mediatek/teei/Makefile.include`
   - Error in: `drivers/misc/mediatek/base/power/spm/common/mtk_idle_select.c`

## Solutions Applied

### 1. Created Stub Source Files

For each missing driver, created a minimal C source file that compiles to an empty object:

**LCM (LCD/Display) Drivers:**
- `drivers/misc/mediatek/lcm/ft8201m_wxga_vdo_incell_boe/ft8201m_stub.c`
- `drivers/misc/mediatek/lcm/nt36523b_wxga_vdo_incell_inx/nt36523b_stub.c`

**Image Sensor Drivers:**
- `drivers/misc/mediatek/imgsensor/src/common/v1/hi556txd_mipi_raw/hi556txd_stub.c`
- `drivers/misc/mediatek/imgsensor/src/common/v1/sc500cs_mipi_raw/sc500cs_stub.c`
- `drivers/misc/mediatek/imgsensor/src/common/v1/sc201cs_mipi_raw/sc201cs_stub.c`
- `drivers/misc/mediatek/imgsensor/src/common/v1/gc02m1_mipi_raw/gc02m1_stub.c`

Each stub file contains:
```c
/*
 * Stub driver for missing [driver_name]
 */

/* Empty stub file to satisfy linker */
```

### 2. Updated Stub Makefiles

Changed from commented-out objects to actual compilation targets:

**Before:**
```makefile
# Empty stub - driver source not available
# obj-y += driver_name.o
```

**After:**
```makefile
# Stub object file to satisfy linker
obj-y += driver_stub.o
```

This ensures the build system compiles the stub and creates a `built-in.o` file.

### 3. Added Missing Header Paths

#### M4U Driver
**File**: `drivers/misc/mediatek/m4u/mt6761/Makefile`
```makefile
ccflags-y += -I$(srctree)/drivers/misc/mediatek/m4u/mt6761
```

#### LED Driver
**File**: `drivers/misc/mediatek/leds/Makefile`
```makefile
ccflags-y += -I$(srctree)/drivers/misc/mediatek/leds
```

#### PPM Driver
**File**: `drivers/misc/mediatek/base/power/ppm_v3/src/mach/mt6761/Makefile`
```makefile
-I$(PPM_ROOT_DIR)/src/mach/mt6761 \
```

#### GPU Frequency Driver
**File**: `drivers/misc/mediatek/base/power/mt6761/Makefile`
```makefile
ccflags-y += -I$(srctree)/drivers/misc/mediatek/base/power/mt6761/
```

#### CMDQ Driver
**File**: `drivers/misc/mediatek/cmdq/v3/mt6765/Makefile`
```makefile
-I$(srctree)/drivers/misc/mediatek/cmdq/v3/mt6765 \
```

#### TEEI Driver
Already configured via `drivers/misc/mediatek/teei/Makefile.include` which sets:
```makefile
ccflags-y += -I$(srctree)/drivers/misc/mediatek/teei/$(VERSION)/tz_driver/include
```

## Understanding Stub Drivers

### Why Stubs Are Needed

MediaTek's original defconfig references specific display panels and camera sensors. However, the actual driver source code for some variants is missing (likely vendor-proprietary or device-specific).

Without stubs, two options exist:
1. **Remove from defconfig** - But we don't want to modify the working config
2. **Create stubs** - Allow build to succeed without the actual drivers

### How Stubs Work

1. **Makefile declares object**: `obj-y += stub.o`
2. **Build system compiles stub.c**: Creates `stub.o`
3. **Linker creates built-in.o**: Combines all objects in directory
4. **Empty object linked in**: Doesn't add any functionality but satisfies dependencies

At runtime, these drivers simply do nothing. If the actual hardware isn't present, no harm done. If it is present, it won't work - but that's expected given the missing driver code.

## Pattern Recognition

### Common MediaTek Driver Organization

MediaTek drivers frequently:
1. **Co-locate headers with source** - Not in global include directories
2. **Use angle brackets** - `#include <header.h>` instead of `#include "header.h"`
3. **Require explicit paths** - Build system doesn't auto-add local directories
4. **Platform-specific variants** - mt6761 often maps to mt6765 subdirectories

### Build System Expectations

The kernel's Kbuild system:
1. **Expects built-in.o** - Even from directories with no functional code
2. **Searches include paths** - Only directories explicitly added with `-I`
3. **Links hierarchically** - Each subdirectory creates built-in.o, parent links them

## Files Modified Summary

**6 stub source files created:**
- 2 LCM drivers (ft8201m, nt36523b)
- 4 imgsensor drivers (hi556txd, sc500cs, sc201cs, gc02m1)

**11 Makefiles updated:**
- 6 stub Makefiles (to compile stubs)
- 5 driver Makefiles (to add header paths)

## Cumulative Progress

**Total Fixes Across All Rounds:**
- Round 1: 14 warning suppressions + DTC fix + 4 driver stubs (empty)
- Round 2: 6 warning suppressions + 1 include path
- Round 3: 6 warning suppressions + 3 include paths
- Round 4: 6 include paths
- **Round 5: 6 stub sources + 11 Makefile updates**

**Grand Total:**
- **27 warning suppressions**
- **16 include path fixes**
- **6 stub drivers with compiled objects**
- **DTC linker fix**
- **Python 2→3 migration**

## How to Build

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

## Next Steps

If build continues to fail, likely issues:
- Additional missing headers (same fix: find header, add include path)
- Additional missing driver stubs (same fix: create stub.c, update Makefile)
- Linker errors for undefined symbols (may need actual implementations or stubs)

The pattern is now well-established and future fixes follow the same approach.

## Key Insight

Vendor kernels often have incomplete driver sets because:
1. Some drivers are device-specific and not published
2. Proprietary code can't be released
3. Defconfigs are generic and reference all possible hardware

The stub approach is industry-standard for such situations - it allows compilation of a generic kernel that can boot on hardware with present drivers, while gracefully ignoring absent ones.
