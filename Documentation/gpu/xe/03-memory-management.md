# Memory Management Subsystem

## Overview

The Intel Xe GPU driver implements a sophisticated three-level memory management system that handles GPU-accessible allocations, virtual address spaces, and translation tables. This document details the design and operation of this subsystem.

**Target Audience**: GPU driver developers, memory system engineers  
**Key Files**:
- `xe_bo.c/h` - Buffer objects (TTM-based)
- `xe_vm.c/h` - Virtual memory and address spaces
- `xe_pt.c/h` - Page table management
- `xe_ggtt.c/h` - Global graphics translation table
- `xe_ttm_*.c` - TTM memory managers (VRAM, system, stolen)

## 1. Memory Management Architecture

### Three-Level Hierarchy

```
User Process
    |
    v
Application Memory Allocations
    │
    ├─── BO (Buffer Object) ────────> Physical Memory
    │    (Managed by TTM)             ├─ VRAM (Discrete)
    │    ├─ Placement                 ├─ System RAM
    │    ├─ Eviction                  └─ Stolen Memory
    │    └─ Pinning
    │
    v
VM (Virtual Memory Space)
    │
    ├─── VMA (Virtual Memory Area) ─> Page Tables
    │    ├─ Bind/Unbind              ├─ 4-Level (Skylake)
    │    ├─ Priority                 ├─ 5-Level (Tiger Lake+)
    │    └─ Prefetch Hints           └─ Automatic allocation
    │
    v
GGTT (Global Graphics Translation Table)
    │
    ├─── GGTT Entry ────────────────> Physical Address
    │    ├─ Allocation               (1GB-4GB region)
    │    └─ Mapping
    │
    v
GPU Virtual Address Space
    │
    v
GPU Hardware Access
```

### Design Principles

1. **Per-VM Address Spaces**: Each VM is isolated with its own page tables
2. **TTM Integration**: Buffer objects managed through Linux TTM subsystem
3. **Lazy Binding**: Page tables only allocated when needed
4. **Efficient Eviction**: BOs can be moved between placements
5. **Security Isolation**: Page table invalidation prevents unauthorized access

## 2. Buffer Objects (BO) - Layer 1

### Purpose & Role

Buffer Objects are GPU-accessible memory allocations. They represent the physical memory that holds actual data.

**Typical BOs**:
- User command buffers (batch buffers)
- Textures and framebuffers
- Compute workload data
- Synchronization primitives
- GuC firmware and shared memory

### BO Types & Placements

```c
struct xe_bo {
    struct ttm_buffer_object ttm_bo;    // TTM base object
    
    // Placement (where this BO can live)
    struct ttm_placement placement;
    
    // Flag: pinned (never evict) vs dynamic (can evict)
    bool pinned;
    
    // Virtual address mapping info
    struct xe_vma *vma;                 // VMAs using this BO
    
    // Eviction & migration info
    struct list_head pinned_link;       // For pinned tracking
    struct drm_gem_object gem;          // GEM object for userspace
    
    // For VRAM BOs: backup object for eviction
    struct xe_bo *backup;               // System RAM copy
};
```

### Placement Options

```
┌─────────────────────────────────────────┐
│        Placement Decision Tree            │
└─────────────────────────────────────────┘
        │
        ├─ Pinned BO?
        │  ├─ Yes → Must stay in original placement
        │  └─ No → Can migrate between placements
        │
        ├─ Discrete GPU?
        │  ├─ Yes → Options: VRAM, System RAM
        │  │       Prefer VRAM for performance
        │  └─ No → Options: System RAM only
        │
        └─ Size / Frequency of Access?
           ├─ Large, hot data → VRAM
           └─ Small, cold data → System RAM

Actual placements defined in xe_bo_create():
TTM_PL_SYSTEM          (System RAM)
TTM_PL_VRAM            (GPU VRAM)
TTM_PL_STOLEN          (Stolen BIOS memory)
```

### BO Allocation

```c
struct xe_bo *xe_bo_create(struct xe_device *xe, 
                           struct xe_tile *tile,
                           struct xe_vm *vm,
                           size_t size,
                           enum ttm_placement_flags flags)
{
    struct xe_bo *bo;
    int ret;
    
    // 1. Allocate xe_bo structure
    bo = kzalloc(sizeof(*bo));
    
    // 2. Initialize GEM object
    drm_gem_private_object_init(&xe->drm, &bo->gem, size);
    
    // 3. Create TTM BO with placement preferences
    // For discrete GPU:
    struct ttm_place places[] = {
        { .mem_type = TTM_PL_VRAM },      // Prefer VRAM
        { .mem_type = TTM_PL_SYSTEM },    // Fall back to system
    };
    
    // For integrated GPU:
    struct ttm_place places[] = {
        { .mem_type = TTM_PL_SYSTEM },    // Only option
    };
    
    bo->placement.placement = places;
    bo->placement.num_placement = sizeof(places) / sizeof(places[0]);
    
    // 4. Allocate via TTM
    ret = ttm_bo_init(&xe->ttm, &bo->ttm_bo, ...);
    if (ret)
        goto error;
    
    // 5. Initialization successful
    return bo;
    
error:
    kfree(bo);
    return NULL;
}
```

### BO Lifecycle

```
xe_bo_create()
    ├─ Allocate BO structure
    ├─ Find placement using TTM
    ├─ Allocate physical memory
    └─ Return handle to user
    │
    v
User Process:
    ├─ Write data to BO
    ├─ Map BO into VM (via DRM_XE_VM_BIND)
    ├─ Submit jobs using BO
    └─ GPU accesses BO
    │
    v
Optional: evict_bo()
    ├─ Memory pressure → need space
    ├─ Move BO to backup (System RAM)
    ├─ Free VRAM space
    └─ Later restore when needed
    │
    v
xe_bo_put() / xe_bo_destroy()
    ├─ Remove all VMAs referencing this BO
    ├─ Deallocate physical memory
    ├─ Free BO structure
    └─ Return resources to system
```

## 3. Virtual Memory (VM) - Layer 2

### Purpose & Role

A VM represents a GPU address space. Each user process typically has one VM, and the VM's page tables define which physical memory is accessible at which virtual addresses.

```c
struct xe_vm {
    // Base structure for DRM VM bind
    struct drm_gpuvm base;
    
    // Device reference
    struct xe_device *xe;
    
    // Page tables
    struct xe_pt *pt;               // Root page table
    u32 pt_levels;                  // Page table depth (4 or 5)
    
    // Virtual memory areas
    struct list_head vmas;          // All VMAs in this VM
    
    // Locking (complex due to recursion avoidance)
    struct mutex lock;              // VM structure lock
    struct dma_resv resv;           // DMA reservation
    
    // Memory management
    struct drm_mm mm;               // VA allocation tracking
    u64 size;                       // Total VA space
    
    // Flags
    enum xe_vm_type type;           // COMPUTE or GRAPHICS
    bool scratch;                   // Has scratch pages
    
    // Pagefault handling
    struct work_struct pagefault_work;
    u32 pagefault_pending;
};
```

### VM Types

```
XE_VM_TYPE_GRAPHICS
├─ Traditional graphics workloads
├─ Can use sparse binding
└─ Allows access to GGTT space

XE_VM_TYPE_COMPUTE
├─ Compute workloads
├─ Full binding required
└─ No GGTT access
```

### Address Space Layout

```
Virtual Address Space (per VM)
┌────────────────────────────────────┐
│ 0x00000000                         │
│                                    │
│  User-Allocated Virtual Addresses  │
│  (via VMA bind operations)         │
│                                    │
│  ...                               │
│                                    │
│ 0xFFFFFFFF (32-bit)               │
│ or                                 │
│ 0xFFFFFFFFFFFFFFFF (48/57-bit)    │
└────────────────────────────────────┘

Page Table Levels (Example: 5-level):
┌─────────────────────────────────────┐
│ PML5E (Page Map Level 5 Entry)      │ Root PT
├─────────────────────────────────────┤
│ PML4E (Page Map Level 4 Entry)      │ L4 PT
├─────────────────────────────────────┤
│ PDPE (Page Directory Pointer Entry) │ L3 PT
├─────────────────────────────────────┤
│ PDE (Page Directory Entry)          │ L2 PT
├─────────────────────────────────────┤
│ PTE (Page Table Entry)              │ L1 PT
├─────────────────────────────────────┤
│ Physical Address (4KB Pages)        │ Data
└─────────────────────────────────────┘
```

### VM Creation

```c
int xe_vm_create(struct xe_device *xe, int type, struct xe_vm **out)
{
    struct xe_vm *vm;
    int ret;
    
    // 1. Allocate VM structure
    vm = kzalloc(sizeof(*vm));
    
    // 2. Initialize GPUVM base
    drm_gpuvm_init(&vm->base, "xe", ...);
    
    // 3. Initialize locking
    mutex_init(&vm->lock);
    dma_resv_init(&vm->resv);
    
    // 4. Initialize VA space allocator
    drm_mm_init(&vm->mm, 0, VM_SIZE);
    
    // 5. Allocate root page table
    ret = xe_pt_create(vm, 0);  // 0 = root level
    if (ret)
        goto error;
    
    // 6. Allocate scratch pages (access to unmapped addresses)
    ret = xe_pt_scratch_create(vm);
    if (ret)
        goto error;
    
    // VM ready for bindings
    *out = vm;
    return 0;
    
error:
    xe_vm_destroy(vm);
    return ret;
}
```

### Virtual Memory Area (VMA)

A VMA maps a BO into a VM at a specific virtual address:

```c
struct xe_vma {
    struct drm_gpuva base;      // Base GPUVA
    
    struct xe_vm *vm;           // Parent VM
    struct xe_bo *bo;           // Referenced BO
    
    u64 start;                  // Virtual start address
    u64 end;                    // Virtual end address
    
    enum xe_vma_type type;      // NORMAL, USERPTR, etc.
    
    // Page table references
    struct xe_pt_entry pte;     // Root page table entry
    
    // Binding state
    enum {
        VMA_UNBOUND,
        VMA_BINDING,
        VMA_BOUND,
        VMA_UNBINDING
    } state;
};
```

### Binding Process (Map BO into VM)

```
User: DRM_XE_VM_BIND ioctl
    │
    v
xe_vm_bind()
    │
    ├─ Validate BO and VM
    │
    ├─ Check address space available
    │
    ├─ Create VMA structure
    │
    ├─ For each PT level needed:
    │  └─ Allocate intermediate page tables
    │     (if not already present)
    │
    ├─ Fill in page table entries:
    │  └─ For each BO page:
    │     ├─ Get physical address
    │     ├─ Encode as PTE (including flags)
    │     └─ Write to appropriate PT level
    │
    ├─ Invalidate GPU TLB
    │  └─ Send invalidation command to GuC
    │  └─ Wait for completion
    │
    └─ Mark VMA as BOUND
```

**Related Files**:
- `xe_vm.c` - VM management
- `xe_vm_types.h` - VM structures
- `xe_pt.c` - Page table operations
- `xe_pt_walk.c` - PT traversal

## 4. Global Graphics Translation Table (GGTT) - Layer 3

### Purpose

The GGTT is a device-level GTT that maps GPU virtual addresses for non-PPGTT (Per-Process GTT) access. Unlike PPGTT which is per-VM, GGTT is global to the device/tile.

### GGTT Uses

```
GGTT is used for:

1. GuC Firmware
   └─ GuC code and data sections in WOPCM
   
2. GuC Shared Memory
   └─ Command transport buffers
   └─ Address data structures
   └─ Logging buffers
   
3. HuC Firmware (if present)
   └─ HuC code and data
   
4. Other privileged structures
   └─ Display/video engine access
   └─ Some platform-specific allocations
```

### GGTT Structure

```c
struct xe_ggtt {
    // Device reference
    struct xe_device *xe;
    struct xe_tile *tile;
    
    // Memory mapping
    void __iomem *regs;              // GGTT registers
    u64 size;                        // GGTT size (256MB - 4GB)
    
    // Allocation tracking
    struct drm_mm mm;                // drm_mm for allocations
    struct list_head nodes;          // Allocated GGTT regions
    
    // Scratch pages
    struct xe_bo *scratch;           // Default for unmapped access
};
```

### GGTT Entry (PTE) Format

```
64-bit GGTT PTE:
┌────────────────────────────────────┐
│ [63:52] Physical Page Address[41:20]│  (40 bits of physical address)
├────────────────────────────────────┤
│ [51:32] Physical Page Address[19:0] │  (continuation of phys addr)
├────────────────────────────────────┤
│ [11:8]  Memory Type                 │  (Cached, Uncached, etc.)
│ [7]     Valid / Present             │  (Entry is valid)
│ [6]     Write Enable                │  (GPU can write)
│ [5:0]   Reserved                    │
└────────────────────────────────────┘

Typical valid entry:
- Bits [63:12] = Physical page address >> 12
- Bit [7] = 1 (present/valid)
- Other bits = platform-specific flags
```

### GGTT Allocation

```c
u64 xe_ggtt_allocate(struct xe_ggtt *ggtt, struct xe_bo *bo, u64 size)
{
    struct xe_ggtt_node *node;
    u64 offset;
    
    // 1. Allocate region in GGTT address space
    offset = drm_mm_insert_node(&ggtt->mm, size);
    if (!offset)
        return -ENOSPC;
    
    // 2. Create allocation tracker
    node = kzalloc(sizeof(*node));
    node->bo = bo;
    node->size = size;
    node->offset = offset;
    
    // 3. Get physical page from BO
    struct page *page = xe_bo_get_page(bo, 0);
    dma_addr_t phys = page_to_phys(page);
    
    // 4. Write GGTT PTEs
    for (u64 i = 0; i < size; i += PAGE_SIZE) {
        u64 pte = phys | GGTT_VALID | GGTT_CACHE_WRITEBACK;
        writeq(pte, ggtt->regs + (offset + i) / PAGE_SIZE * 8);
        
        // Get next page from BO
        page = xe_bo_get_page(bo, i + PAGE_SIZE);
        phys = page_to_phys(page);
    }
    
    // 5. GGTT entry now maps to BO
    return offset;
}
```

## 5. Memory Placement & Eviction

### TTM Placement Algorithm

When a BO is created, TTM finds the best memory location:

```c
int ttm_bo_validate(struct ttm_buffer_object *bo)
{
    struct ttm_place *candidate;
    int best_placement = -1;
    
    // For each possible placement in order
    for (int i = 0; i < bo->placement.num_placement; i++) {
        candidate = &bo->placement.placement[i];
        
        // Check if placement has space
        if (ttm_mem_type_manager_check(&bo->bdev, candidate->mem_type)) {
            // This placement works
            best_placement = i;
            break;
        }
    }
    
    if (best_placement == -1) {
        // Need to evict something
        ttm_evict_for_space(bo);
        best_placement = 0;  // Retry first placement
    }
    
    // Place BO in chosen location
    bo->mem = &bo->bdev->mem_type_manager[best_placement];
    
    return 0;
}
```

### Eviction

When memory is tight, TTM evicts BOs to make space:

```
Eviction Flow:

1. GPU memory pressure detected
   └─ Low VRAM available

2. Find candidate BO to evict
   └─ Must not be pinned
   └─ Least recently used
   └─ Low priority

3. Create backup in system RAM
   └─ allocate_backup_bo()
   └─ Copy VRAM → System RAM

4. Update page tables
   └─ Remap virtual addresses to backup

5. Free VRAM
   └─ Return to VRAM manager

6. When BO needed again
   └─ Copy System RAM → VRAM
   └─ Update page tables
   └─ Resume execution
```

## 6. Memory Access Synchronization

### DMA Reservation

BOs use DMA reservations to synchronize access:

```c
struct xe_bo {
    struct dma_resv *resv;           // Reservation object
};
```

Before GPU accesses a BO:

```c
int xe_bo_lock_vm_and_signaling(struct xe_bo *bo, struct xe_vm *vm)
{
    // 1. Lock the reservation
    dma_resv_lock(bo->resv, NULL);
    
    // 2. Add GPU fence to reservation
    //    (GPU will signal when done accessing)
    dma_resv_add_fence(bo->resv, gpu_fence, DMA_RESV_USAGE_WRITE);
    
    // 3. Unlock
    dma_resv_unlock(bo->resv);
    
    return 0;
}
```

This prevents:
- CPU from freeing BO while GPU accesses it
- Another GPU job from conflicting access
- Display from scanning BO while it's being written

## 7. Memory Debugging & Monitoring

### Sysfs Memory Stats

```bash
cat /sys/class/drm/card0/device/mem_info_vram_total
cat /sys/class/drm/card0/device/mem_info_vram_used
cat /sys/class/drm/card0/device/mem_info_vram_available
```

### Debugfs BO Inspection

```bash
# List all BOs
cat /sys/kernel/debug/dri/0/bos

# Detailed BO information
cat /sys/kernel/debug/dri/0/bo_info
```

### Memory Tracing

```bash
# Enable memory allocation tracing
echo "1" > /proc/sys/kernel/gpu_trace_memory

# View memory allocation events
cat /sys/kernel/debug/tracing/trace
```

## 8. Common Memory Issues & Solutions

| Issue | Symptom | Cause | Solution |
|-------|---------|-------|----------|
| OOM | "Out of memory" errors | Excessive BO allocation | Reduce BO size/count, enable eviction |
| Memory leak | Memory grows over time | BOs not freed | Check bo_put() cleanup paths |
| Stale TLB | GPU accesses wrong data | Missing TLB invalidation | Ensure invalidation after bind |
| Eviction hang | GPU hangs on memory pressure | Deadlock during eviction | Review locking in eviction path |
| Page fault | "GPU page fault" error | Unmapped VA access | Check VM binding completeness |

## Summary

The Xe memory management system provides:

1. **Buffer Objects (BO)** - Physical GPU memory allocations
   - TTM-based with flexible placement
   - Automatic eviction and migration
   - Pinning support for critical allocations

2. **Virtual Memory (VM)** - Per-process GPU address spaces
   - 4/5-level hierarchical page tables
   - Lazy page table allocation
   - Efficient sparse binding

3. **GGTT** - Global device-level translation table
   - Firmware and shared structure mapping
   - Per-tile resource

This three-layer design provides security (per-VM isolation), flexibility (BO placement), and efficiency (lazy allocation, automatic eviction).

---

**See Also**:
- [00-architecture-overview.md](00-architecture-overview.md) - Architecture context
- [01-driver-initialization.md](01-driver-initialization.md) - Memory initialization
- [03-execution-scheduling.md](04-execution-scheduling.md) - Memory in job submission
