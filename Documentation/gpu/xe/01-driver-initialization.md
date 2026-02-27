# GPU Driver Initialization

## Overview

This document details the Intel Xe GPU driver initialization process, from PCI probe through device readiness for user applications.

**Target Audience**: Driver developers, system engineers  
**Related Files**:
- `xe_pci.c` - PCI driver entry point
- `xe_device.c` - Device initialization
- `xe_tile.c` - Tile initialization
- `xe_gt.c` - GT (Graphics Technology) initialization
- `xe_mmio.c` - MMIO initialization
- `xe_pm.c` - Power management during probe

## 1. Initialization Phases

The initialization process is divided into distinct phases to manage dependencies and error handling:

```
PCI Probe
    ↓
[PHASE 1: Create Device Structure]
    ↓
[PHASE 2: Early Probe (MMIO, Registers)]
    ↓
[PHASE 3: Tile/GT Detection]
    ↓
[PHASE 4: Memory Management Setup]
    ↓
[PHASE 5: GT Initialization (GuC, Engines)]
    ↓
[PHASE 6: Subsystems & Interrupts]
    ↓
[PHASE 7: DRM Registration]
    ↓
Device Ready for Users
```

## 2. Phase 1: Device Creation

### Entry Point: PCI Probe

**File**: `xe_pci.c`

```c
static int xe_pci_probe(struct pci_dev *pdev, const struct pci_device_id *ent)
{
    const struct xe_device_desc *desc;
    struct xe_device *xe;
    
    // 1. Get device descriptor (capabilities, platform info)
    desc = xe_pci_device_desc(pdev, ent);
    
    // 2. Create device structure
    xe = xe_device_create(pdev, ent);
    
    // 3. Early probe
    xe_device_probe_early(xe);
    
    // 4. Full probe
    xe_device_probe(xe);
    
    // Device is now ready for users
    return 0;
}
```

### Device Structure Creation

**File**: `xe_device.c`

```c
struct xe_device *xe_device_create(struct pci_dev *pdev,
                                    const struct pci_device_id *ent)
{
    struct xe_device *xe;
    
    // Allocate device structure
    xe = devm_drm_dev_alloc(&pdev->dev, &driver, ...);
    
    // Initialize basic fields
    pdev_to_xe_device(pdev) = xe;
    xe->info = desc->info;  // Platform info
    
    // Initialize subsystems that don't require hardware
    mutex_init(&xe->lock);
    INIT_LIST_HEAD(&xe->vms);
    
    // Initialize TTM memory management
    ttm_device_init(&xe->ttm, &xe_ttm_funcs, ...);
    
    // Initialize DRM core
    drm_dev_register(&xe->drm, ...);  // Will probe drivers
    
    return xe;
}
```

**Key Structures Initialized**:

```c
struct xe_device {
    struct drm_device drm;              // DRM core
    struct pci_dev *pdev;               // PCI device
    struct ttm_device ttm;              // Memory management
    struct xe_device_desc *desc;        // Platform descriptor
    struct xe_info info;                // Capabilities
    struct xe_tile *tiles;              // Tile array
    struct xe_gt *gts[XE_MAX_GT];       // GT array
    struct xe_irq irq;                  // Interrupt info
    struct xe_uc uc;                    // Microcontroller
    int tile_count;                     // Number of tiles
    ...
};
```

## 3. Phase 2: Early Probe

**File**: `xe_device.c`

The early probe initializes hardware access and basic capabilities:

```c
int xe_device_probe_early(struct xe_device *xe)
{
    struct pci_dev *pdev = xe->pdev;
    int ret;
    
    // 1. Initialize MMIO (register access)
    ret = xe_mmio_probe_early(xe);
    if (ret)
        return ret;
    
    // 2. Map PCI memory spaces
    // Device BAR 0 → regular MMIO
    // Device BAR 1 → VRAM (discrete GPUs)
    xe->mmio.regs = ioremap(pci_resource_start(pdev, 0), ...);
    
    // 3. Initialize platform code (PCODE)
    ret = xe_pcode_probe_early(xe);
    if (ret)
        return ret;
    
    // 4. Wait for LMEM (local memory) to be ready
    if (IS_DGFX(xe)) {
        ret = wait_for_lmem_ready(xe);
        if (ret)
            return ret;
    }
    
    // 5. Read device ID, revision
    // 6. Detect platform stepping (e.g., A0, B1)
    
    // 7. Apply early workarounds
    xe_wa_init();
    
    return 0;
}
```

### MMIO Initialization

**File**: `xe_mmio.c`

```c
int xe_mmio_probe_early(struct xe_device *xe)
{
    struct pci_dev *pdev = xe->pdev;
    void __iomem *regs;
    
    // 1. Enable PCI device
    pci_enable_device(pdev);
    
    // 2. Request PCI memory regions
    pci_request_regions(pdev, ...);
    
    // 3. Map BAR 0 (registers)
    regs = ioremap(pci_resource_start(pdev, 0), 
                   pci_resource_len(pdev, 0));
    
    // 4. Initialize per-GT MMIO (may have per-tile offsets)
    // Detect GMDID (Graphics Device Identification) registers
    // Determine tile count and GT configuration
    
    // 5. Set up MMIO read/write functions
    // These handle:
    //   - Force-wake (prevent GT from sleeping)
    //   - GSI (Global Shift Index) offset for tile-specific access
    //   - Register access logging for debug
    
    return 0;
}
```

**MMIO Register Access Pattern**:

```c
// Register read with proper forcing of wake states
u32 value = xe_mmio_read32(gt, reg);

// For registers that require keeping GT awake:
u32 status;
xe_force_wake_get(gt, XE_FORCEWAKE_ALL);
status = xe_mmio_read32(gt, STATUS_REG);
xe_force_wake_put(gt, XE_FORCEWAKE_ALL);

// MMIO implementation handles:
//   - Force-wake synchronization
//   - Per-tile GSI offset
//   - Memory barriers
```

## 4. Phase 3: Tile and GT Detection

### Reading GMDID

**File**: `xe_mmio.c` or `xe_device.c`

```c
// GMDID (Graphics Device Identification) registers tell us:
// - Number of tiles
// - Number of GTs per tile
// - GT type (primary graphics or media)

void probe_tiles(struct xe_device *xe)
{
    int tile_count = 0;
    int gt_count_per_tile = 0;
    
    // Read GMDID to determine configuration
    u32 gmdid = xe_mmio_read32(gt, GMD_ID);
    
    tile_count = REG_FIELD_GET(GMD_TILE_MASK, gmdid);
    gt_count_per_tile = REG_FIELD_GET(GMD_GT_COUNT_MASK, gmdid);
    
    // Initialize tile structures
    for (int i = 0; i < tile_count; i++) {
        struct xe_tile *tile = &xe->tiles[i];
        tile->id = i;
        tile->xe = xe;
        INIT_LIST_HEAD(&tile->vms);
        // Tile-specific MMIO offset
        tile->mmio_offset = i * TILE_SIZE;
    }
    
    // Initialize GT structures
    for (int t = 0; t < tile_count; t++) {
        struct xe_tile *tile = &xe->tiles[t];
        
        // Primary GT
        struct xe_gt *gt = &tile->primary_gt;
        gt->tile = tile;
        gt->type = XE_GT_TYPE_PRIMARY;
        gt->id = t * 2 + 0;
        
        // Media GT (if present)
        if (gt_count_per_tile > 1) {
            gt = &tile->media_gt;
            gt->tile = tile;
            gt->type = XE_GT_TYPE_MEDIA;
            gt->id = t * 2 + 1;
        }
    }
}
```

**Key Output**:
- `xe->tile_count` - Number of GPU tiles
- `xe->tiles[i]` - Tile array
- `xe->tiles[i].primary_gt` - Primary GT per tile
- `xe->tiles[i].media_gt` - Media GT per tile (optional)

## 5. Phase 4: Memory Management Setup

### TTM Memory Manager Initialization

**File**: `xe_device.c`

```c
int xe_device_setup_memory(struct xe_device *xe)
{
    int ret;
    
    // 1. Initialize system memory manager (TTM)
    ret = ttm_device_init(&xe->ttm, &xe_ttm_funcs, ...);
    if (ret)
        return ret;
    
    // 2. Initialize system RAM memory manager
    ret = xe_ttm_sys_mgr_init(xe);
    if (ret)
        return ret;
    
    // 3. Initialize stolen memory manager
    // (BIOS-reserved memory for static allocations)
    ret = xe_ttm_stolen_mgr_init(xe);
    if (ret)
        return ret;
    
    // 4. For discrete GPUs, initialize VRAM manager
    if (IS_DGFX(xe)) {
        ret = xe_ttm_vram_mgr_init(xe);
        if (ret)
            return ret;
    }
    
    return 0;
}
```

### Placements Defined

Each placement defines where BOs can be allocated:

```c
struct ttm_placement default_placement = {
    .num_placement = 2,
    .placement = [
        TTM_PL_SYSTEM,      // System RAM
        TTM_PL_VRAM,        // VRAM (discrete GPUs)
    ]
};

struct ttm_placement stolen_placement = {
    .num_placement = 1,
    .placement = [
        TTM_PL_STOLEN,      // Stolen BIOS memory
    ]
};
```

**Key Files**:
- `xe_ttm_sys_mgr.c` - System memory management
- `xe_ttm_vram_mgr.c` - VRAM management
- `xe_ttm_stolen_mgr.c` - Stolen memory management

## 6. Phase 5: GT Initialization

### GT Initialization Sequence

**File**: `xe_gt.c`

```c
int xe_gt_init(struct xe_gt *gt)
{
    int ret;
    
    // 1. Initialize register access for this GT
    ret = xe_mmio_probe_tiles(gt);
    if (ret)
        return ret;
    
    // 2. Detect hardware engines
    ret = xe_hw_engine_class_init(gt);
    if (ret)
        return ret;
    // Discovers: RCS (3D), BCS (Blitter), VCS (Video), 
    //            VECS (Video Enhancement), CCS (Compute)
    
    // 3. Initialize microcontroller (GuC/HuC)
    ret = xe_uc_probe(gt);
    if (ret)
        return ret;
    // Downloads firmware, prepares shared memory
    
    // 4. Power-on/initialize GT
    ret = xe_force_wake_get(gt, XE_FORCEWAKE_ALL);
    if (ret)
        return ret;
    
    // 5. Apply register table programming (workarounds, tuning)
    ret = xe_rtp_process(gt);
    if (ret)
        return ret;
    
    // 6. Idle management
    ret = xe_gt_idle_init(gt);
    if (ret)
        return ret;
    
    // 7. Frequency management
    ret = xe_gt_freq_init(gt);
    if (ret)
        return ret;
    
    // 8. Topology discovery
    ret = xe_gt_topology_init(gt);
    if (ret)
        return ret;
    
    // 9. GT now initialized
    
    return 0;
}
```

### Microcontroller (GuC) Probe

**File**: `xe_uc.c`

```c
int xe_uc_probe(struct xe_gt *gt)
{
    struct xe_uc *uc = &gt->uc;
    int ret;
    
    // 1. Probe GuC firmware
    ret = xe_guc_init(&uc->guc);
    if (ret)
        return ret;
    // Reads firmware file, prepares for load
    
    // 2. Probe HuC firmware (optional)
    if (xe_device_has_huc(gt->xe)) {
        ret = xe_huc_init(&uc->huc);
        if (ret)
            return ret;
    }
    
    // 3. Allocate shared memory structures
    ret = xe_guc_ads_create(&uc->guc);
    if (ret)
        return ret;
    
    return 0;
}
```

### GuC Initialization

**File**: `xe_guc.c`

```c
int xe_guc_init(struct xe_guc *guc)
{
    struct xe_gt *gt = guc_to_gt(guc);
    int ret;
    
    // 1. Initialize GuC firmware structure
    ret = xe_uc_fw_init(&guc->fw);
    if (ret)
        return ret;
    
    // 2. Allocate GuC memory regions:
    //    - WOPCM (Where Out-of-band Processing is located)
    //    - PARAM block
    //    - Shared structures
    ret = xe_guc_allocate_memory(guc);
    if (ret)
        return ret;
    
    // 3. Allocate command transport (CT) structures
    ret = xe_guc_ct_init(&guc->ct);
    if (ret)
        return ret;
    
    // 4. Initialize doorbell manager
    ret = xe_guc_db_mgr_init(&guc->dbm);
    if (ret)
        return ret;
    
    // GuC will be loaded later in xe_guc_post_load_init()
    
    return 0;
}
```

**GuC Memory Layout**:

```
0x00000000 ─────────────────────────────
          │  GuC Firmware Code/Data
          │  (Downloaded from file)
xe_wopcm_size ────────────────────────────
          │  HuC Firmware (if present)
          │
XE_WOPCM_TOP ─────────────────────────────
          │  GGTT area for GuC shared structures
          │  - Descriptor (ADS)
          │  - Logs
          │  - Scheduling tables
GUC_GGTT_TOP ─────────────────────────────
          │  Available to driver
```

## 7. Phase 6: Subsystems and Interrupts

### Interrupt Installation

**File**: `xe_irq.c`

```c
int xe_irq_init(struct xe_device *xe)
{
    int ret;
    
    // 1. Detect and allocate MSI-X vectors
    ret = pci_alloc_irq_vectors(xe->pdev, 
                               min_vecs, max_vecs,
                               PCI_IRQ_MSIX);
    if (ret < 0)
        return ret;
    
    xe->irq.msix.nvec = ret;
    
    // 2. Allocate vector-to-handler mappings
    xe->irq.msix.table = devm_kcalloc(...);
    
    // 3. Register main interrupt handler
    ret = request_irq(xe->pdev->irq, xe_irq_handler, ...);
    if (ret)
        return ret;
    
    // 4. Enable interrupts at device level
    xe_mmio_write32(xe, INTR_EN_REG, INTR_ENABLE_MASK);
    
    return 0;
}
```

### Other Subsystems

```c
int xe_device_probe(struct xe_device *xe)
{
    // ... earlier phases ...
    
    // Install interrupts
    ret = xe_irq_init(xe);
    
    // Initialize GT-to-tile mapping and perform per-GT init
    for_each_gt(gt, xe, id) {
        ret = xe_gt_init(gt);
        if (ret)
            return ret;
    }
    
    // Load microcontroller and post-boot init
    ret = xe_uc_init(xe);
    if (ret)
        return ret;
    
    // Initialize page faults
    ret = xe_pagefault_init(xe);
    if (ret)
        return ret;
    
    // Initialize device coredump
    xe_devcoredump_init(xe);
    
    // Initialize power management
    ret = xe_pm_init(xe);
    if (ret)
        return ret;
    
    // Initialize display
    ret = xe_display_init(xe);
    if (ret)
        return ret;
    
    // Initialize monitoring/observability
    ret = xe_observation_init(xe);
    if (ret)
        return ret;
    
    return 0;
}
```

## 8. Phase 7: DRM Registration

### Device Registration

```c
int drm_dev_register(struct drm_device *dev, unsigned long flags)
{
    // 1. Register DRM device with subsystem
    // 2. Create device node (/dev/dri/card*, /dev/dri/renderD128)
    // 3. Call driver-specific registration callbacks
    
    // For Xe:
    // - xe_file_open is registered as drm_file_operations.open
    // - Initial userspace can now open device and submit ioctls
}
```

## 9. Detailed Component Initialization

### GGTT (Global Graphics Translation Table) Initialization

**File**: `xe_ggtt.c`

```c
int xe_ggtt_init(struct xe_ggtt *ggtt, struct xe_tile *tile)
{
    struct xe_device *xe = tile->xe;
    u64 ggtt_size;
    int ret;
    
    // 1. Determine GGTT size (platform-dependent)
    // TGL: 256MB, ADL: 512MB, MTL: 1GB, etc.
    ggtt_size = xe_ggtt_size(xe);
    
    // 2. Map GGTT in driver address space
    ggtt->regs = ioremap(ggtt_phys_base, ggtt_size);
    
    // 3. Allocate drm_mm for GGTT allocation tracking
    drm_mm_init(&ggtt->mm, 0, ggtt_size);
    
    // 4. Reserve space for firmware
    // GuC firmware needs to be accessible via GGTT
    drm_mm_reserve_node(&ggtt->mm, &fw_node);
    
    // 5. Initialize scratch pages
    // Used for unmapped address accesses
    ggtt->scratch = xe_bo_create_pinned_user(xe, ...);
    
    // GGTT is now ready for allocations
    
    return 0;
}
```

### VM (Virtual Memory) Subsystem

The VM subsystem isn't fully initialized during probe; rather, it's initialized on-demand when users create VMs via `DRM_XE_VM_CREATE`.

However, the driver does set up basic VM infrastructure during probe:

```c
// Initialize VM-related subsystems
ret = xe_pagefault_init(xe);  // Setup page fault handling

// This allows user VMs to be created after probe completes
```

## 10. Error Handling & Cleanup

### Probe Error Cleanup

The driver uses devm (device-managed resources) for automatic cleanup:

```c
int xe_pci_probe(struct pci_dev *pdev, ...)
{
    // Using devm_drm_dev_alloc ensures cleanup on error:
    xe = devm_drm_dev_alloc(&pdev->dev, &driver, ...);
    
    ret = xe_device_probe(xe);
    if (ret) {
        // devm automatically calls cleanup functions in reverse order:
        // - xe_pm_fini()
        // - xe_display_fini()
        // - xe_devcoredump_fini()
        // - xe_irq_fini()
        // - xe_mmio cleanup
        // - ...
        
        return ret;
    }
    
    return 0;
}
```

## 11. Key Data Structures Summary

| Phase | Structure | File | Purpose |
|-------|-----------|------|---------|
| 1 | `xe_device` | xe_device_types.h | Top-level device |
| 2 | `xe_mmio` | xe_mmio.h | Register access |
| 3 | `xe_tile`, `xe_gt` | xe_gt_types.h | Tile/GT hierarchy |
| 4 | `ttm_device` | TTM core | Memory management |
| 5 | `xe_guc` | xe_guc_types.h | Microcontroller |
| 6 | `xe_irq` | xe_irq.h | Interrupt handling |
| 7 | `drm_device` | DRM core | DRM registration |

## 12. Common Initialization Issues

### Issue: Firmware Loading Fails
**Symptom**: GuC initialization fails during probe  
**Common Causes**:
- Missing firmware files in `/lib/firmware/xe/`
- Wrong platform descriptor
- WOPCM size miscalculation

**Debug**:
```bash
# Check kernel logs
dmesg | grep -i guc

# Verify firmware present
ls -la /lib/firmware/xe/
```

### Issue: MMIO Access Hangs
**Symptom**: Driver hangs during register access  
**Common Causes**:
- Accessing registers without force-wake
- Incorrect MMIO offset for tile
- Device in D3cold state

**Debug**:
```c
// Always use force-wake before register access
xe_force_wake_get(gt, XE_FORCEWAKE_ALL);
u32 val = xe_mmio_read32(gt, REG);
xe_force_wake_put(gt, XE_FORCEWAKE_ALL);
```

## 13. Initialization Flow Diagram

```
xe_pci_probe()
    │
    ├─► xe_device_create()
    │   ├─► devm_drm_dev_alloc()
    │   └─► ttm_device_init()
    │
    ├─► xe_device_probe_early()
    │   ├─► xe_mmio_probe_early()
    │   ├─► xe_pcode_probe_early()
    │   ├─► wait_for_lmem_ready()
    │   └─► detect platform stepping
    │
    ├─► xe_device_probe()
    │   ├─► probe_tiles() [Read GMDID]
    │   ├─► xe_ttm_sys_mgr_init()
    │   ├─► xe_ttm_vram_mgr_init() [Discrete only]
    │   ├─► for each tile:
    │   │   └─► xe_ggtt_init()
    │   │
    │   ├─► for each GT:
    │   │   ├─► xe_gt_init()
    │   │   │   ├─► xe_hw_engine_class_init()
    │   │   │   ├─► xe_uc_probe()
    │   │   │   │   ├─► xe_guc_init()
    │   │   │   │   └─► xe_guc_ads_create()
    │   │   │   ├─► xe_force_wake_get()
    │   │   │   ├─► xe_rtp_process() [Workarounds]
    │   │   │   ├─► xe_gt_idle_init()
    │   │   │   └─► xe_gt_freq_init()
    │   │   │
    │   │   └─► xe_guc_post_load_init()
    │   │       ├─► Load GuC firmware
    │   │       ├─► Start GuC microcontroller
    │   │       └─► Initialize execution queues
    │   │
    │   ├─► xe_irq_init()
    │   ├─► xe_pagefault_init()
    │   ├─► xe_devcoredump_init()
    │   ├─► xe_pm_init()
    │   └─► xe_observation_init()
    │
    └─► drm_dev_register()
        └─► Device ready for users
```

## Summary

The Xe driver initialization is a carefully orchestrated sequence:

1. **Create** device structure and map MMIO
2. **Detect** tiles and GT configuration via GMDID
3. **Setup** memory management (TTM, GGTT)
4. **Initialize** each GT (engines, GuC, frequencies)
5. **Load** GuC firmware and start microcontroller
6. **Setup** interrupts, page faults, power management
7. **Register** with DRM subsystem
8. **Ready** for user applications

Each phase builds on previous phases, and careful error handling ensures cleanup on failure.

---

**See Also**:
- [00-architecture-overview.md](00-architecture-overview.md) - Architecture context
- [02-guc-integration.md](02-guc-integration.md) - GuC firmware details
- [03-memory-management.md](03-memory-management.md) - Memory initialization
