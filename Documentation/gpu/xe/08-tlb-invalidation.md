# TLB Invalidation & Cache Operations

## Overview

This document details TLB (Translation Lookaside Buffer) invalidation mechanisms, which are critical for maintaining GPU memory consistency after page table changes.

**Target Audience**: Memory management engineers, synchronization specialists  
**Complexity**: HIGH  
**Key Files**:
- `xe_tlb_inval.c/h` - Frontend API
- `xe_guc_tlb_inval.c/h` - Backend (GuC-based)
- `xe_tlb_inval_job.c/h` - Asynchronous job handling

## 1. Why TLB Invalidation?

### Memory Consistency Problem

```
Scenario: Update page table entry

1. Driver modifies page table
   ├─ Old: VA 0x1000 → Physical 0x100000
   └─ New: VA 0x1000 → Physical 0x200000

2. GPU still has old translation cached (in TLB)
   └─ VA 0x1000 → Physical 0x100000 (stale!)

3. GPU accesses 0x1000
   └─ Gets data from old location (0x100000)
   └─ Wrong data returned! Data corruption!

SOLUTION: Invalidate TLB entry for 0x1000
```

### When TLB Invalidation Needed

```
DRM_XE_VM_BIND       ← Bind BO to VM (write page table)
                     ← MUST invalidate TLB

DRM_XE_VM_UNBIND     ← Unbind BO from VM
                     ← MUST invalidate TLB

Memory migration     ← Move BO between VRAM/system
                     ← MUST invalidate TLB

Page table updates   ← Update PPGTTs
                     ← MUST invalidate TLB
```

## 2. TLB Invalidation Types

### GGTT Invalidation

Invalidate Global Graphics TT entries (device-level GTT):

```c
int xe_guc_tlb_inval_ggtt_all(struct xe_guc *guc)
{
    // Flushes all GGTT entries
    // Affects all VMs using GGTT
    
    // Used when GGTT structure changes
    // (GuC firmware update, display mode change)
}
```

### VM Invalidation

Invalidate all page tables in a specific VM:

```c
int xe_vm_invalidate_tlb(struct xe_vm *vm)
{
    // Flushes TLB for all contexts in this VM
    // Used when VM page tables modified extensively
    
    // More targeted than GGTT invalidation
}
```

### Range Invalidation

Invalidate specific address range in VM:

```c
int xe_vm_tlb_inval_range(struct xe_vm *vm,
                         u64 start, u64 end)
{
    // Flushes TLB entries for [start, end) in VM
    // Most selective, best for performance
    
    // Used when only subset of VM modified
}
```

## 3. Invalidation Mechanisms

### Hardware TLB Flush

Direct hardware register write:

```c
void xe_ttd_hw_flush(struct xe_device *xe)
{
    // TD (Texture Descriptor) Flush
    // Write to TD_FLUSH register
    
    u32 val = TD_FLUSH_ALL | TD_FLUSH_TILE_ALL;
    xe_mmio_write32(gt, TD_FLUSH_REG, val);
    
    // Wait for completion
    xe_mmio_wait32(gt, TD_FLUSH_REG, 
                   TD_FLUSH_VALID,
                   0, timeout_ms);
}
```

**Limitations**:
- Affects entire device
- Synchronous (blocks on wait)
- Not optimal for partial invalidations

### GuC-Based TLB Invalidation (Preferred)

Send command to GuC firmware to perform invalidation:

```c
int xe_guc_tlb_inval_range(struct xe_guc *guc,
                          struct xe_vm *vm,
                          u64 start, u64 end)
{
    struct xe_guc_tlb_inval_cmd cmd = {
        .header = GUC_TLB_INVAL_ACTION,
        .vm_guid = vm->guid,
        .start = start,
        .length = end - start,
    };
    
    // 1. Send command via CTB
    ret = xe_guc_ct_send(&guc->ct, &cmd);
    if (ret)
        return ret;
    
    // 2. Create fence to track completion
    struct dma_fence *fence = xe_tlb_inval_fence_create(
        guc, vm_guid, start, end);
    
    // 3. Return fence to caller
    *out_fence = fence;
    
    return 0;
}
```

**Advantages**:
- Can specify address range
- Per-VM or GGTT
- Asynchronous (non-blocking)
- GuC can coalesce multiple invalidations

## 4. Synchronization & Fences

### Invalidation Fence Lifecycle

```
User calls VM_BIND (modifies page tables)
    │
    v
Create TLB invalidation job
    │
    ├─ Contains: start VA, end VA, VM ID
    │
    v
Submit to GuC
    │
    ├─ GuC processes asynchronously
    │
    v
Create invalidation fence
    │
    ├─ Tracks invalidation completion
    │
    v
Subsequent GPU jobs depend on fence
    │
    ├─ GPU scheduler waits for invalidation fence
    │  before executing jobs using updated page tables
    │
    v
GuC signals invalidation complete
    │
    ├─ Fence signaled
    ├─ GPU scheduler allowed to proceed
    │
    v
GPU job executes with correct translations
```

### Fence Implementation

```c
struct xe_tlb_inval_fence {
    struct dma_fence base;
    
    // Track status
    enum {
        PENDING,
        IN_PROGRESS,
        SIGNALED,
        TIMED_OUT
    } status;
    
    // GuC command tracking
    u32 guc_seqno;           // GuC's seqno for this inval
    
    // Timeout handling
    struct delayed_work timeout_work;
    
    // VM being invalidated (for recovery)
    struct xe_vm *vm;
};
```

### Fence Signaling

GuC firmware signals completion via:

```c
// Option 1: Memory-based signaling
// GuC writes seqno to shared memory
void guc_tlb_inval_complete_handler(struct xe_guc *guc)
{
    u32 completed_seqno = read_from_shared_memory();
    
    // Find and signal all fences up to this seqno
    for (each fence in pending_list) {
        if (fence->guc_seqno <= completed_seqno)
            dma_fence_signal(&fence->base);
    }
}

// Option 2: Interrupt-based signaling
// GuC raises interrupt
irq_handler():
    └─ Read GUC_TLB_INVAL_COMPLETE interrupt
    └─ Call guc_tlb_inval_complete_handler()
```

## 5. Job-Based Invalidation

For asynchronous invalidation handling, Xe creates special jobs:

```c
struct xe_tlb_inval_job {
    struct xe_sched_job base;    // Scheduler job
    
    // Invalidation parameters
    u64 start;                   // Start address
    u64 end;                     // End address
    struct xe_vm *vm;            // VM to invalidate
    
    // Fence to wait for
    struct dma_fence *fence;
};
```

### Job Submission Flow

```c
void xe_vm_tlb_inval_job_submit(struct xe_vm *vm, u64 start, u64 end)
{
    // 1. Create invalidation job
    job = xe_tlb_inval_job_create(vm, start, end);
    
    // 2. Add dependency: VM page table modification
    //    (wait for PT updates to be visible)
    add_dependency(job, pt_update_fence);
    
    // 3. Submit to GPU scheduler
    drm_sched_job_arm(&job->base);
    drm_sched_entity_push_job(&job->base, entity);
    
    // 4. When ready, job runs:
    job_run():
        └─ Call xe_guc_tlb_inval_range()
        └─ Wait for GuC completion
}
```

## 6. MOD (Memory Ordering Dependency) Invalidation

Special invalidation type for memory coherency:

```c
// MOD (Memory Ordering Dependency) Invalidation
// Used when GPU memory ordering must be strictly enforced

int xe_guc_tlb_inval_with_mod(struct xe_guc *guc,
                              struct xe_vm *vm,
                              u64 start, u64 end)
{
    // MOD invalidation provides:
    // - Memory barrier semantics
    // - Ensures previous memory operations complete
    // - New page table entries fully visible
    
    struct cmd = {
        .action = GUC_TLB_INVAL_WITH_MOD,
        .vm_id = vm->id,
        .start = start,
        .length = end - start,
    };
    
    send_to_guc(&cmd);
}
```

**When MOD needed**:
- Cross-VM communication (rare)
- Shared GPU/CPU memory modifications
- Memory domain transitions

## 7. TLB Invalidation Timeout

### Timeout Mechanism

```c
void xe_tlb_inval_fence_init_timeout(struct xe_tlb_inval_fence *fence)
{
    // Set timeout for invalidation
    static const int TIMEOUT_MS = 100;  // 100ms should be plenty
    
    INIT_DELAYED_WORK(&fence->timeout_work,
                     xe_tlb_inval_timeout_handler);
    
    schedule_delayed_work(&fence->timeout_work,
                         msecs_to_jiffies(TIMEOUT_MS));
}

void xe_tlb_inval_timeout_handler(struct work_struct *work)
{
    struct xe_tlb_inval_fence *fence = 
        container_of(work, struct xe_tlb_inval_fence, timeout_work);
    
    if (fence->status != SIGNALED) {
        dev_err(&xe->pdev->dev,
               "TLB invalidation timeout for VM %u\n",
               fence->vm->id);
        
        // Mark as failed
        fence->status = TIMED_OUT;
        
        // Signal fence anyway (with error)
        dma_fence_set_error(&fence->base, -ETIMEDOUT);
        dma_fence_signal(&fence->base);
        
        // Trigger device recovery
        xe_device_declare_wedged(xe);
    }
}
```

## 8. Performance Considerations

### Invalidation Coalescing

GuC can merge multiple invalidations:

```
Sequential page table updates:
DRM_XE_VM_BIND(BO1, range 0x1000-0x2000)
    └─ TLB inval for 0x1000-0x2000

DRM_XE_VM_BIND(BO2, range 0x3000-0x4000)
    └─ TLB inval for 0x3000-0x4000

GuC firmware coalesces:
    └─ Single invalidation: 0x1000-0x4000
    └─ Saves multiple GuC commands
```

### Selective Invalidation

Use range invalidation when possible:

```c
// SLOW: Invalidate everything
xe_vm_invalidate_tlb(vm);

// FAST: Invalidate only what changed
xe_vm_tlb_inval_range(vm, updated_start, updated_end);
```

## 9. Debugging TLB Issues

### Symptoms of TLB Problems

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| Data corruption | Missing invalidation | Add invalidation before dependent access |
| Intermittent failures | Race in invalidation fence | Check fence signaling |
| GPU hang | Invalidation timeout | Increase timeout, check GuC status |
| Performance regression | Over-invalidation | Use range invalidation |

### Debug Commands

```bash
# Check TLB invalidation stats
cat /sys/kernel/debug/dri/0/tlb_inval_stats

# View pending invalidations
cat /sys/kernel/debug/dri/0/pending_invalidations

# Enable TLB tracing
echo "1" > /sys/kernel/debug/tracing/events/xe/tlb_inval/enable
```

## 10. L2/L3 Cache Flushing

Related to TLB invalidation: GPU caches:

```c
// TD (Texture Descriptor) and L2/L3 cache flushing
void xe_device_l2_flush(struct xe_device *xe)
{
    // Flush L2 cache
    // Ensures memory writes visible to all
    for_each_tile(tile, xe, i) {
        xe_mmio_write32(tile_gt, L2_FLUSH_REG, L2_FLUSH_ALL);
        xe_mmio_wait32(tile_gt, L2_FLUSH_REG, 0, TIMEOUT);
    }
}

// Combined: Flush + TLB Invalidate
void xe_device_flush_and_inval_tlb(struct xe_device *xe)
{
    // 1. Flush L2/L3 caches
    xe_device_l2_flush(xe);
    
    // 2. Invalidate TLB
    for_each_gt(gt, xe, id)
        xe_ttd_hw_flush(gt);
    
    // Ensures complete memory coherency
}
```

## Summary

TLB invalidation is critical for:
1. **Correctness** - Prevents data corruption from stale translations
2. **Performance** - Selective invalidation better than full flush
3. **Synchronization** - Fences ensure ordering with GPU jobs
4. **Reliability** - Timeout protection prevents hangs

Key mechanisms:
- Hardware flush (full device)
- GuC invalidation (per-VM, per-range)
- Asynchronous fences for coordination
- MOD invalidation for memory ordering

Proper TLB management is essential for driver correctness and performance.

---

**See Also**:
- [03-memory-management.md](03-memory-management.md) - Memory binding
- [02-guc-integration.md](02-guc-integration.md) - GuC command interface
- [10-potential-issues.md](10-potential-issues.md) - Known TLB issues
