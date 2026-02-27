# Intel Xe GPU Driver - Architecture Overview

## Document Purpose

This document provides a comprehensive architectural overview of the Intel Xe GPU driver, explaining the design philosophy, component relationships, and how different subsystems interact.

**Target Audience**: GPU software developers, kernel driver engineers  
**Scope**: Intel Xe GPU driver in `drivers/gpu/drm/xe/`

## 1. Driver Philosophy & Design Goals

The Intel Xe GPU driver is designed with the following principles:

### Modular Architecture
- Clear separation between device, tile, and GT (Graphics Technology) levels
- Independent subsystems for memory, scheduling, power, and security
- Firmware-driven execution through GuC microcontroller

### Hardware Abstraction
- Platform-independent upper layers
- Platform-specific handling in lower layers via WA (workarounds) and RTP (register table programming)
- Register access through MMIO with proper force-wake handling

### Multi-Tile Support
- Modern Intel GPUs can have multiple tiles (compute units)
- Each tile is an independent graphics unit with own GT, GGTT, and memory managers
- Tiles can have primary (graphics/compute) and media (video encoding/decoding) GTs

### Security & Isolation
- Per-VM address spaces with independent page tables
- SR-IOV support for virtualization
- PXP (Protected eXecution Protocol) for DRM content
- TLB invalidation for security and performance

## 2. Hierarchical Device Model

### Overall Structure

```
        PCI Device (xe_device)
                |
        ┌───────┼───────┐
        |               |
    Tile 0          Tile 1 (optional)
        |               |
    ┌───┴───┐       ┌───┴───┐
    |       |       |       |
  Primary Media  Primary Media
    GT      GT      GT      GT
    |       |       |       |
  Engines Engines Engines Engines
```

### Component Descriptions

#### **Device (xe_device)**
- Top-level structure representing the entire GPU
- Entry point for device probing and initialization
- Owns:
  - PCI device information
  - Device capabilities and features
  - Workaround information
  - TTM memory manager (unified for all tiles)
  - Interrupt handling
  - Top-level configuration (SRIOV mode, wedged state)

**Key Files**: `xe_device.c`, `xe_device.h`, `xe_device_types.h`

**Important Structures**:
```c
struct xe_device {
    struct drm_device drm;           // DRM device
    struct pci_dev *pdev;            // PCI device
    struct ttm_device ttm;           // TTM memory management
    struct xe_tile *tiles;           // Tiles array [0..tile_count-1]
    struct xe_info info;             // Device capabilities
    struct xe_irq irq;               // Interrupt handling
    struct xe_uc uc;                 // Microcontroller (GuC/HuC)
    ...
};
```

#### **Tile (xe_tile)**
- Represents one physical GPU instance on the device
- Handles:
  - VRAM management (if discrete)
  - Display output (if integrated)
  - Migration operations (VRAM <-> system memory)
  - Tile-specific memory allocation

**Key Files**: `xe_tile.c`, `xe_tile.h`

**Important Structures**:
```c
struct xe_tile {
    struct xe_device *xe;
    struct xe_gt *primary_gt;        // Primary GT (required)
    struct xe_gt *media_gt;          // Media GT (optional)
    struct xe_ggtt *ggtt;            // Global graphics translation table
    struct xe_migrate *migrate;      // Migration operations
    ...
};
```

#### **GT (Graphics Technology Unit)**
- Represents the GPU compute/graphics execution unit
- Owns:
  - Hardware engines (RCS, BCS, VCS, VECS, CCS)
  - GuC microcontroller and firmware
  - Power management (RC6, RPS)
  - Hardware monitoring
  - Interrupt handling (per-GT)

**Key Files**: `xe_gt.c`, `xe_gt.h`, `xe_gt_types.h`

**Important Structures**:
```c
struct xe_gt {
    struct xe_tile *tile;
    struct xe_device *xe;
    
    enum xe_gt_type type;            // PRIMARY or MEDIA
    u8 info.id;                      // GT ID
    
    struct xe_hw_engine *engines;    // Hardware engines array
    u32 engine_mask;                 // Bitmask of available engines
    
    struct xe_guc guc;               // GuC microcontroller
    struct xe_uc_fw fw;              // Firmware management
    struct xe_gt_idle idle;          // RC6/idle states
    struct xe_gt_freq freq;          // Frequency scaling
    ...
};
```

## 3. Major Subsystems & Their Roles

### A. Memory Management Subsystem

**Three-Level Architecture**:

```
User Process
    |
    v
Exec Queue Job with memory requirements
    |
    v
BO (Buffer Object) - TTM managed
    |
    +-----> VRAM placement (discrete GPUs)
    +-----> System memory placement
    +-----> Stolen memory placement
    |
    v
VM (Virtual Memory) Space
    |
    +-----> Page tables (xe_pt)
    +-----> VMA (Virtual Memory Area mapping)
    |
    v
GGTT (Global Graphics Translation Table)
    |
    v
GPU Virtual Address
```

**Components**:

1. **BO (Buffer Objects)** - `xe_bo.c`
   - TTM-based GPU-accessible memory allocations
   - Placements:
     - VRAM (xe_ttm_vram_mgr.c) - Local GPU memory
     - System (xe_ttm_sys_mgr.c) - System RAM
     - Stolen (xe_ttm_stolen_mgr.c) - Pre-allocated BIOS memory
   - Pinned vs unpinned objects
   - Eviction handling (xe_bo_evict.c)

2. **VM (Virtual Memory)** - `xe_vm.c`
   - Per-process GPU address space
   - User-created via DRM_XE_VM_CREATE ioctl
   - Contains:
     - Page tables (4-level or 5-level, depending on PPGTT)
     - VMA mappings (VM Area)
     - Bind/unbind operations
     - Pagefault handling

3. **GGTT (Global Graphics Translation Table)** - `xe_ggtt.c`
   - Device-level GTT for non-PPGTT access
   - Maps GGTT entries to physical memory
   - Used for:
     - GuC firmware
     - HuC firmware
     - GuC shared memory areas

**Key Flow**:
```
User submits job with buffers
    |
    v
Allocate BOs (TTM placement)
    |
    v
Map BOs into VM (create VMAs, page tables)
    |
    v
Submit to execution queue
    |
    v
GPU accesses through PPGTT → GGTT → Physical memory
```

**Related Files**:
- `xe_bo_types.h` - BO structure
- `xe_vm_types.h` - VM structure
- `xe_ggtt_types.h` - GGTT structures
- `xe_pt.c` - Page table management
- `xe_res_cursor.h` - Resource cursor for page iteration

### B. GuC (GPU Microcontroller) Integration

**Purpose**: Offload GPU scheduling and power management to firmware

**Architecture**:

```
Driver
   |
   v
GuC Command Transport (CT)
   |
   +-----> Command queue (KMD -> GuC)
   +-----> Response queue (GuC -> KMD)
   |
   v
GuC Firmware
   |
   +-----> Job scheduling
   +-----> Power conservation
   +-----> TLB invalidation
   +-----> Telemetry
   |
   v
HW Engines
```

**Key Components**:

1. **GuC Firmware Management** - `xe_guc.c`, `xe_uc_fw.c`
   - Firmware loading and initialization
   - Microcontroller state machine
   - FW logging

2. **GuC Command Transport** - `xe_guc_ct.c`
   - Bidirectional communication with GuC
   - Message queue management
   - Command/response tracking

3. **GuC Submission** - `xe_guc_submit.c`
   - Job submission to GuC
   - Doorbell notifications
   - Context pinning

4. **GuC Address Data Structures** - `xe_guc_ads.c`
   - Shared memory regions between driver and GuC
   - Configuration data structures

5. **GuC Power Conservation** - `xe_guc_pc.c`
   - Frequency scaling (RPS - Render Performance State)
   - Idle state management

**Related Files**:
- `xe_guc_types.h` - GuC structure
- `xe_guc_*.h` - GuC header files (ct, ads, submit, pc, etc.)
- `xe_guc_fwif.h` - GuC firmware interface

### C. Execution & Scheduling Subsystem

**Flow**:

```
User: DRM_XE_EXEC ioctl
    |
    v
Exec Queue (hardware engine class submission queue)
    |
    v
GPU Scheduler (drm_gpu_scheduler)
    |
    v
GuC Submission
    |
    v
HW Engines
    |
    v
GPU Execution
```

**Components**:

1. **Execution Queues** - `xe_exec_queue.c`
   - User-created or kernel-internal
   - Per-engine-class (RCS, BCS, VCS, VECS, CCS)
   - Contains:
     - Associated contexts (xe_lrc)
     - Job queue
     - GuC queue information

2. **GPU Scheduler** - `xe_gpu_scheduler.c`
   - DRM GPU scheduler integration
   - Job prioritization
   - Timeout handling
   - Preemption

3. **Jobs** - `xe_sched_job.c`
   - Work unit for GPU execution
   - Contains:
     - Batch buffer address
     - Synchronization primitives
     - Memory requirements

4. **Contexts (LRC)** - `xe_lrc.c`
   - Per-engine GPU context
   - Contains:
     - Engine state
     - Ring buffer pointers
     - Context save/restore areas

**Related Files**:
- `xe_exec_queue_types.h` - Exec queue structure
- `xe_lrc_types.h` - Logical ring context structure
- `xe_sched_job_types.h` - Job structure
- `xe_exec.c` - IOCTL implementation

### D. Power Management Subsystem

**Scope**:
- System-level suspend (S0ix, S3, S4)
- Runtime PM (D3 states)
- GT idle (RC6)
- Frequency scaling (RPS)

**Architecture**:

```
Linux PM Subsystem
    |
    ├─> System suspend → xe_pm_suspend()
    ├─> System resume → xe_pm_resume()
    ├─> Runtime suspend → xe_pm_runtime_suspend()
    └─> Runtime resume → xe_pm_runtime_resume()
    
    v
    
Device Power Management
    |
    ├─> Force-wake handling (ensure registers accessible)
    ├─> GT idle transitions (RC6)
    ├─> Frequency scaling (via GuC)
    ├─> Clock management
    └─> Temperature throttling
```

**Key Components**:

1. **PM Main Logic** - `xe_pm.c`
   - System suspend/resume
   - Runtime PM management
   - D3Cold prevention logic

2. **Force-Wake** - `xe_force_wake.c`
   - Prevents GT from entering C6+ states during register access
   - Refcounted usage

3. **GT Idle** - `xe_gt_idle.c`
   - RC6 (Render C6) entry/exit conditions
   - Idle state counters

4. **Frequency Scaling** - `xe_gt_freq.c`
   - RPS (Render Performance State) management
   - Min/max frequency configuration

**Related Files**:
- `xe_pm.h` - PM API
- `xe_force_wake.h` - Force-wake API
- `xe_hwmon.c` - Hardware monitoring

### E. Interrupt Handling & Events

**Architecture**:

```
GPU Hardware Interrupt
    |
    v
Interrupt Controller
    |
    ├─> MSI-X routing (multiple vectors)
    └─> MSTR_TILE_INTR register
    |
    v
Interrupt Handler (xe_irq.c)
    |
    ├─> GuC/HuC errors → Handle firmware crashes
    ├─> Engine resets → Handle GPU hangs
    ├─> Fence signaling → Complete jobs
    ├─> Pagefault → Handle GPU page faults
    └─> Memory interrupts → Handle memory events
    |
    v
Workqueue/Event handlers
```

**Key Components**:

1. **Interrupt Installation** - `xe_irq.c`
   - MSI-X vector allocation
   - Handler registration
   - Top-level interrupt dispatcher

2. **Tile Interrupt Handling** - `xe_irq.c`
   - Per-tile interrupt routing
   - Individual engine interrupt handling

3. **Memory-Based Interrupts** - `xe_memirq.c`
   - Memory-based IRQ signaling
   - Used in some configurations for low-latency signaling

**Related Files**:
- `xe_irq.h` - Interrupt API
- `xe_memirq.c` - Memory IRQ handling
- `xe_hw_fence.c` - Hardware fence signals

### F. TLB Invalidation

**Purpose**: Ensure GPU cache coherency after page table changes

**Flow**:

```
Driver modifies page tables
    |
    v
Need to invalidate GPU TLB
    |
    v
xe_tlb_inval_*() APIs
    |
    v
GuC TLB Invalidation (preferred)
or Hardware TLB Flush (fallback)
    |
    v
Wait for invalidation complete
```

**Invalidation Types**:
- GGTT invalidation
- VM invalidation (all contexts in VM)
- Range invalidation (specific address range)
- Invalidation with MOD-expectation

**Related Files**:
- `xe_guc_tlb_inval.c` - GuC-based TLB invalidation
- `xe_tlb_inval.c` - TLB invalidation API
- `xe_tlb_inval_job.c` - Asynchronous invalidation jobs

### G. Debug & Monitoring

**Components**:

1. **Devcoredump** - `xe_devcoredump.c`
   - GPU state capture on hang
   - Register dumps
   - GuC logs
   - Queue state

2. **GuC Logging** - `xe_guc_log.c`
   - GuC firmware log buffer management
   - Debug message collection

3. **Debug Filesystem** - `xe_debugfs.c`
   - Debugging interfaces
   - Register inspection
   - Performance counters

4. **Observability Agents** - `xe_oa.c`
   - Performance monitoring
   - Telemetry collection

5. **PMU (Performance Monitoring Unit)** - `xe_pmu.c`
   - Linux perf integration
   - Performance event sampling

**Related Files**:
- `xe_devcoredump.h` - Coredump API
- `xe_trace*.c` - Event tracing
- `xe_observation.c` - Observable entities

## 4. Execution Flow Examples

### Scenario 1: Device Initialization

```
1. xe_pci_probe() [PCI driver entry point]
   └─ xe_device_create() [Create top-level structure]
      └─ ttm_device_init() [Initialize memory management]
   
2. xe_device_probe_early() [Early initialization]
   └─ xe_mmio_probe_early() [Map registers]
   └─ xe_pcode_probe_early() [Platform code init]
   └─ wait_for_lmem_ready() [Wait for VRAM ready]
   
3. xe_device_probe() [Full probe]
   └─ detect tiles/GTs via GMDID registers
   
   For each tile:
   └─ xe_tile_init()
      └─ xe_ggtt_init() [GTT setup]
      └─ xe_vram_init() [VRAM setup, if discrete]
      
   For each GT:
   └─ xe_gt_init()
      └─ xe_uc_probe() [GuC/HuC firmware prep]
      └─ xe_hw_engine_class_init() [Engine discovery]
      └─ xe_guc_post_load_init() [GuC startup]
      
4. drm_dev_register() [Register with DRM subsystem]
```

### Scenario 2: User Submits GPU Job

```
1. User calls DRM_XE_EXEC ioctl
   └─ xe_exec() [Exec ioctl handler]
   
2. Prepare execution
   └─ Validate buffers
   └─ Lock VMs
   └─ Sync foreign BOs
   
3. Create job
   └─ xe_sched_job_create()
      └─ Create batch buffer job
      └─ Set up synchronization primitives
   
4. Submit to GPU scheduler
   └─ drm_sched_job_arm()
   └─ xe_sched_job_push() [Push to queue]
   
5. Scheduler determines readiness
   └─ Run job callback
      └─ xe_guc_submit() [Submit to GuC]
         └─ Write doorbell register
         └─ Update GuC context
   
6. GuC firmware schedules on HW
   └─ Updates context state
   └─ Notifies hardware engines
   
7. GPU executes batch
   
8. Hardware signals completion
   └─ Fence interrupt
   └─ Signal xe_hw_fence
   └─ DRM scheduler processes completion
   └─ xe_sched_job_done() [Job cleanup]
```

### Scenario 3: Memory Mapping

```
1. User calls DRM_XE_VM_CREATE
   └─ xe_vm_create() [Create VM]
   
2. User calls DRM_XE_VM_BIND
   └─ xe_vm_bind() [Map buffers into VM]
   
3. Validation
   └─ Check VM has space
   └─ Check BO is valid
   
4. Allocation
   └─ Allocate page tables if needed
   └─ Create VMA (Virtual Memory Area)
   
5. Mapping
   └─ Fill page tables with BO page entries
   
6. TLB invalidation
   └─ Invalidate GPU TLB via GuC
   
7. Complete
   └─ Return mapped virtual address to user
```

## 5. Supported Platforms

The driver supports multiple Intel GPU architectures:

| Generation | Platforms | Type | Notes |
|-----------|-----------|------|-------|
| **Xe HPG** | DG1, DG2 | Discrete | First Xe architecture |
| **Xe HPC** | PVC | Discrete | Compute-focused |
| **Xe-LP** | TGL, RKL | Integrated | Legacy, pre-GuC |
| **Xe-LPG** | ADL | Integrated | GuC-enabled integrated |
| **Xe-LPG+** | MTL, LNL | Integrated | Media engines, multi-tile |
| **Xe2** | BMG, PTL | Upcoming | Next-generation |

Platform information is stored in `xe_device_desc` structures (defined in `xe_pci.c`).

## 6. Key Data Structures

### Device-Level Structures

```
xe_device          - Top-level GPU device
├─ xe_device_types.h - Device structure definition
├─ xe_info          - Device capabilities
└─ xe_device_desc   - Platform descriptor

xe_tile            - Physical GPU tile
├─ xe_tile_types.h  - Tile structure definition
└─ xe_gt[0..n]      - Graphics technology units

xe_gt              - GPU compute/graphics unit
├─ xe_gt_types.h    - GT structure definition
├─ xe_hw_engine[]   - Hardware engines
├─ xe_guc           - Microcontroller
└─ xe_gt_idle       - Idle state management
```

### Memory Management Structures

```
xe_bo              - Buffer object
├─ xe_bo_types.h    - BO structure
├─ TTM BO           - TTM base object
└─ placement[]      - Allowed placements

xe_vm              - Virtual memory space
├─ xe_vm_types.h    - VM structure
├─ xe_pt_types.h    - Page table entries
└─ xe_vma[]         - VM areas

xe_ggtt            - Global graphics TT
├─ xe_ggtt_types.h  - GGTT structure
└─ xe_ggtt_node[]   - Allocated entries
```

### Execution Structures

```
xe_exec_queue      - Submission queue
├─ xe_exec_queue_types.h
├─ xe_lrc[]         - Logical ring contexts
└─ drm_sched_entity - Scheduler entity

xe_sched_job       - GPU job
├─ xe_sched_job_types.h
├─ xe_hw_fence      - Hardware fence
└─ xe_sync_entry[]  - Synchronization

xe_guc             - Microcontroller
├─ xe_guc_types.h
├─ xe_guc_ct        - Command transport
└─ xe_guc_ads       - Address structures
```

## 7. Important Patterns & Conventions

### Force-Wake Pattern
When accessing registers, GT must be in awake state:

```c
xe_force_wake_get(gt, XE_FORCEWAKE_ALL);
// Access registers
val = xe_mmio_read32(gt, reg);
xe_force_wake_put(gt, XE_FORCEWAKE_ALL);
```

### VM Locking Pattern
VMs have complex locking requirements:

```c
xe_vm_lock_dma_resv(vm);
// Modify VM bindings
xe_vm_unlock_dma_resv(vm);
```

### MMIO Access Pattern
Register access goes through gt for proper MMIO handling:

```c
u32 val = xe_mmio_read32(gt, reg);
xe_mmio_write32(gt, reg, new_val);
```

### Job Submission Pattern
Jobs follow a standard flow:

```c
job = xe_sched_job_create(exec_queue, ...);
xe_sched_job_push(job);  // Submit to GPU scheduler
```

## 8. Common Development Scenarios

### When Adding GPU Support
1. Create device descriptor in `xe_pci.c`
2. Define platform workarounds in `xe_wa.c`
3. Add register table programming in `xe_rtp*.c`
4. Update capability checks in `xe_device_types.h`

### When Modifying Scheduling
1. Change `xe_guc_submit.c` for GuC interaction
2. Update `xe_sched_job.c` for job structure
3. Modify `xe_exec_queue.c` for queue management
4. Update `xe_gpu_scheduler.c` for scheduling policy

### When Changing Memory Layout
1. Update page table structures (`xe_pt.c`)
2. Modify PPGTT initialization (`xe_vm.c`)
3. Update TLB invalidation if needed (`xe_guc_tlb_inval.c`)
4. Test through `xe_pte_encode`

## Summary

The Intel Xe GPU driver is a modular, layered architecture that cleanly separates:
- **Device/Tile/GT hierarchy** for multi-instance support
- **Memory management** through GGTT→VM→BO layers
- **Execution** through GuC-driven job submission
- **Power management** through firmware and hardware cooperation
- **Debug** through comprehensive observability

Understanding these layers and their interactions is key to effective development and debugging of the Xe driver.

---

**See Also**:
- [01-driver-initialization.md](01-driver-initialization.md) - Detailed initialization flow
- [02-guc-integration.md](02-guc-integration.md) - GuC firmware details
- [03-memory-management.md](03-memory-management.md) - Memory subsystem details
