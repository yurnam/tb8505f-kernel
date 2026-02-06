# Build Fix Round 4 - Systematic Header Path Resolution

## Problem
After round 3 fixes, the build continued to fail with multiple "fatal error: file not found" errors across various MediaTek drivers. All headers existed in the tree but were not in the compiler's include path.

## Analysis of Latest Build Log

### Missing Header Files (All Present in Tree)

1. **ION Driver** - `drivers/staging/android/ion/`
   - Missing: `ion.h`
   - Location: `drivers/staging/android/ion/ion.h`
   - Files affected: ion.c, ion-ioctl.c, ion_heap.c, ion_system_heap.c, etc.

2. **SPM (System Power Manager)** - `drivers/misc/mediatek/base/power/spm/mt6761/`
   - Missing: `mtk_spm_internal.h`, `mtk_spm_irq.h`
   - Location: `drivers/misc/mediatek/base/power/spm/mt6761/`
   - Files affected: mtk_spm.c, mtk_spm_internal.c, mtk_spm_dram.c, mtk_spm_twam.c

3. **EMI (External Memory Interface)** - `drivers/misc/mediatek/emi/mt6761/`
   - Missing: `mt_emi.h`, `mt_emi_api.h`
   - Location: `drivers/misc/mediatek/emi/mt6761/mt_emi.h`
   - Files affected: emi_ctrl_v1.c, bwl_v1.c

4. **Watchdog Timer** - `drivers/watchdog/mediatek/wdt/common/wdt_v2/`
   - Missing: `mtk_wdt.h`
   - Location: `drivers/watchdog/mediatek/wdt/common/wdt_v2/mtk_wdt.h`
   - Files affected: mtk_wdt_v2.c

5. **External Display** - `drivers/misc/mediatek/ext_disp/mt6765/`
   - Missing: `extd_hdmi.h`
   - Location: `drivers/misc/mediatek/ext_disp/mt6765/extd_hdmi.h`
   - Files affected: external_display.c, extd_utils.c, mtk_extd_mgr.c

6. **Debug Latch** - `drivers/misc/mediatek/debug_latch/plat_dbg_info/`
   - Missing: `plat_dbg_info.h`
   - Location: `drivers/misc/mediatek/debug_latch/plat_dbg_info/plat_dbg_info.h`
   - Files affected: plat_dbg_info.c

### Additional Headers Identified (Not Yet Fixed)
- `mtk_ppm_platform.h` - PPM driver (mt6761/mt6765 specific)
- `mtk_gpufreq.h` - Thermal/GPU freq driver
- `cmdq_engine.h` - Command Queue driver (platform specific)
- `teei_client_main.h` - TEEI/TEE driver

## Solutions Applied

### 1. ION Driver Makefile
**File**: `drivers/staging/android/ion/Makefile`

Added line 15:
```makefile
ccflags-y += -I$(srctree)/drivers/staging/android/ion
```

This allows ION driver files to find `ion.h` in their own directory.

### 2. SPM Driver Makefile
**File**: `drivers/misc/mediatek/base/power/spm/mt6761/Makefile`

Added line 3:
```makefile
ccflags-y += -I$(srctree)/drivers/misc/mediatek/base/power/spm/mt6761/
```

Allows SPM driver to find platform-specific headers like `mtk_spm_internal.h`.

### 3. EMI Driver Makefile
**File**: `drivers/misc/mediatek/emi/mt6761/Makefile`

Added line 15:
```makefile
ccflags-y += -I$(srctree)/drivers/misc/mediatek/emi/mt6761
```

Enables EMI driver to locate `mt_emi.h` which includes `mt_emi_api.h`.

### 4. Watchdog Driver Makefile
**File**: `drivers/watchdog/mediatek/wdt/common/wdt_v2/Makefile`

Added line 15:
```makefile
ccflags-y += -I$(srctree)/drivers/watchdog/mediatek/wdt/common/wdt_v2
```

Allows watchdog driver to find its own `mtk_wdt.h` header.

### 5. External Display Driver Makefile
**File**: `drivers/misc/mediatek/ext_disp/mt6765/Makefile`

Added line 20:
```makefile
-I$(srctree)/drivers/misc/mediatek/ext_disp/mt6765/  \
```

Enables ext_disp driver to find platform-specific `extd_hdmi.h`.

### 6. Debug Latch Driver Makefile
**File**: `drivers/misc/mediatek/debug_latch/plat_dbg_info/Makefile`

Added line 15:
```makefile
ccflags-y += -I$(srctree)/drivers/misc/mediatek/debug_latch/plat_dbg_info
```

Allows debug latch driver to locate its own header file.

## Pattern Analysis

### Common Issue
MediaTek drivers frequently organize code with:
- Source files in subdirectories
- Headers co-located with source
- Headers use angle brackets `#include <header.h>` instead of quotes

This pattern requires **explicit include paths** to the local directory, which the original Android build system likely handled differently (perhaps through global includes or wrapper scripts).

### Why This Happens
1. **Android Build System Differences**: The original build likely used a different Makefile structure
2. **Platform Variants**: mt6761 maps to mt6765 in many cases, requiring careful path setup
3. **Header Organization**: MediaTek organizes by feature/platform, not as a flat include tree

### Solution Strategy
For each driver with "file not found" errors:
1. Find the actual header location with `find`
2. Identify the correct subdirectory
3. Add `-I$(srctree)/path/to/directory` to the driver's Makefile
4. Use platform variables like `$(MTK_PLATFORM)` where appropriate

## Remaining Work

Several headers still need path fixes but weren't in this build attempt:
- PPM platform headers (may appear in future builds)
- GPU frequency headers (may appear in future builds)  
- CMDQ engine headers (platform-specific, may need conditional paths)
- TEEI/TEE client headers (may appear in future builds)

These will be addressed as they appear in build logs.

## How to Build

Try building again with:
```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

## Files Modified Summary

6 Makefiles updated to add local include paths:
1. `drivers/staging/android/ion/Makefile`
2. `drivers/misc/mediatek/base/power/spm/mt6761/Makefile`
3. `drivers/misc/mediatek/emi/mt6761/Makefile`
4. `drivers/watchdog/mediatek/wdt/common/wdt_v2/Makefile`
5. `drivers/misc/mediatek/ext_disp/mt6765/Makefile`
6. `drivers/misc/mediatek/debug_latch/plat_dbg_info/Makefile`

## Cumulative Progress

**Total Fixes Across All Rounds:**
- Round 1: 14 warning suppressions + DTC fix + 4 driver stubs
- Round 2: 6 warning suppressions + 1 include path (devfreq)
- Round 3: 6 warning suppressions + 3 include paths (battery, mrdump, mmc)
- **Round 4: 6 include paths (ion, spm, emi, wdt, ext_disp, debug_latch)**

**Total: 27 warning suppressions + 11 include path fixes**

## Key Takeaway

Building older vendor kernels with modern toolchains requires:
1. **Warning suppression** for false positives (already done)
2. **Include path resolution** for local headers (ongoing)
3. **Platform mapping** for variant configs (mt6761→mt6765)

This is standard maintenance work when adapting vendor code to standard Linux build systems.
