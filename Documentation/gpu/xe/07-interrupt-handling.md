# Interrupt Handling & Event Routing

## Overview

This document details the interrupt handling architecture in the Intel Xe GPU driver, covering MSI-X initialization, interrupt routing through device/tile/GT hierarchy, and hardware error categorization.

**Target Audience**: Interrupt/event system developers, firmware engineers  
**Complexity**: HIGH  
**Key Files**:
- `xe_irq.c/h` - Main interrupt handling
- `xe_memirq.c/h` - Memory-based interrupts
- `xe_hw_error.c` - Hardware error categorization

## 1. Interrupt Architecture Overview

### Interrupt Flow

```
GPU Hardware Event
    │
    ├─ Assertion on interrupt line
    │
    v
MSI-X Vector Selection
    │
    ├─ Determines handler/GT assignment
    │
    v
Interrupt Handler (xe_irq_handler)
    │
    ├─ Read MSTR_TILE_INTR register
    ├─ Identify which tile(s) signaled
    │
    v
Per-Tile Interrupt Processing
    │
    ├─ Read tile-level interrupt status
    ├─ Route to per-GT handlers
    ├─ Route to per-engine handlers
    │
    v
Per-Engine/Subsystem Handlers
    │
    ├─ Fence completion
    ├─ Hardware error
    ├─ Engine reset
    ├─ Page fault
    └─ Other events
    │
    v
Deferred Work (Workqueue)
    │
    └─ Process in context where sleeping allowed
```

## 2. MSI-X Vector Initialization

### MSI-X Architecture

```c
struct xe_irq {
    struct {
        unsigned int nvec;              // Number of vectors
        struct msix_entry *entries;     // Vector array
        
        // Vector to handler mappings
        struct {
            int (*handler)(int irq, void *arg);
            void *arg;
        } table[max_vectors];
    } msix;
};
```

### Vector Allocation

```c
int xe_irq_msix_init(struct xe_device *xe)
{
    struct pci_dev *pdev = xe->pdev;
    int min_vectors = 1;        // At minimum, 1 for whole device
    int max_vectors = 4;        // One per GT (2 tiles × 2 GTs max)
    
    // 1. Request MSI-X vectors from PCI subsystem
    int ret = pci_alloc_irq_vectors(pdev,
                                     min_vectors,
                                     max_vectors,
                                     PCI_IRQ_MSIX);
    if (ret < 0)
        return ret;
    
    // ret = actual number of vectors allocated
    xe->irq.msix.nvec = ret;
    
    // 2. Allocate mapping table
    xe->irq.msix.table = devm_kcalloc(
        &pdev->dev,
        xe->irq.msix.nvec,
        sizeof(*xe->irq.msix.table),
        GFP_KERNEL);
    
    // 3. Request interrupts
    for (int i = 0; i < xe->irq.msix.nvec; i++) {
        ret = request_irq(pci_irq_vector(pdev, i),
                         xe_irq_handler,
                         0,
                         "xe",
                         xe);
        if (ret)
            return ret;
    }
    
    // 4. Enable interrupts at device level
    u32 intr_enable = 0;
    for_each_gt(gt, xe, id)
        intr_enable |= GT_INTR_ENABLE_MASK(gt);
    
    xe_mmio_write32(xe_root_mmio_gt(xe), INTR_EN, intr_enable);
    
    return 0;
}
```

### Vector Assignment Strategy

```
Multi-Vector Setup (if 4 vectors available):
┌────────────────────────────────────────┐
│ Vector 0: Tile 0 Primary GT            │
│ Vector 1: Tile 0 Media GT              │
│ Vector 2: Tile 1 Primary GT            │
│ Vector 3: Tile 1 Media GT              │
└────────────────────────────────────────┘

Single-Vector Setup (fallback):
┌────────────────────────────────────────┐
│ Vector 0: All GT interrupts            │
│          (All GTs share one vector)    │
└────────────────────────────────────────┘
```

## 3. Interrupt Handler Dispatch

### Main Interrupt Handler

```c
static irqreturn_t xe_irq_handler(int irq, void *arg)
{
    struct xe_device *xe = arg;
    irqreturn_t ret = IRQ_NONE;
    
    // 1. Identify which tile(s) have pending interrupts
    //    (Master tile interrupt status register)
    u32 master_ctl = xe_mmio_read32(
        xe_root_mmio_gt(xe),
        MSTR_TILE_INTR);
    
    if (master_ctl == 0)
        return IRQ_NONE;  // Spurious interrupt
    
    // 2. Process each signaling tile
    for_each_tile(tile, xe, i) {
        if (!(master_ctl & TILE_INTR_BIT(i)))
            continue;
        
        // 3. Process this tile
        xe_tile_irq_handler(tile);
        ret = IRQ_HANDLED;
    }
    
    return ret;
}
```

### Tile-Level Interrupt Processing

```c
static void xe_tile_irq_handler(struct xe_tile *tile)
{
    // Read tile-level interrupt status registers
    u32 tile_ctl = xe_mmio_read32(gt, TILE_INTR);
    
    // Route to per-GT handlers
    for_each_gt_on_tile(gt, tile, id) {
        if (!(tile_ctl & GT_INTR_BIT(id)))
            continue;
        
        xe_gt_irq_handler(gt);
    }
    
    // Route to display interrupts (if applicable)
    if (tile_ctl & DISPLAY_INTR_BIT)
        xe_display_irq_handler(tile);
}
```

### GT-Level Interrupt Processing

```c
static void xe_gt_irq_handler(struct xe_gt *gt)
{
    u32 gt_ctl = xe_mmio_read32(gt, GT_INTR);
    
    // Hardware errors (often fatal)
    if (gt_ctl & GT_HW_ERROR_BIT) {
        xe_hw_error_handler(gt);
    }
    
    // Engine completion/faults
    for (int i = 0; i < num_engines; i++) {
        if (!(gt_ctl & ENGINE_INTR_BIT(i)))
            continue;
        
        xe_engine_irq_handler(&gt->engines[i]);
    }
    
    // GuC/HuC events
    if (gt_ctl & GUC_INTR_BIT)
        xe_guc_irq_handler(&gt->guc);
    
    // Page faults
    if (gt_ctl & PAGEFAULT_INTR_BIT)
        queue_work(xe->wq, &xe->pagefault_work);
}
```

## 4. Hardware Error Handling

### Error Categories

```c
enum xe_hw_error_class {
    XE_HW_ERROR_CORRECTABLE = 0,    // Single-bit ECC errors
    XE_HW_ERROR_NONFATAL = 1,       // Recoverable GPU fault
    XE_HW_ERROR_FATAL = 2,          // Unrecoverable, GPU hang
};
```

### Hardware Error Processing

```c
void xe_hw_error_handler(struct xe_gt *gt)
{
    struct xe_device *xe = gt->xe;
    
    // 1. Read error register
    u32 err_status = xe_mmio_read32(gt, HW_ERROR_STATUS);
    
    // 2. Categorize error
    enum xe_hw_error_class class;
    u32 error_code = FIELD_GET(ERROR_CODE_MASK, err_status);
    
    class = categorize_hw_error(error_code);
    
    // 3. Take appropriate action
    switch (class) {
    case XE_HW_ERROR_CORRECTABLE:
        // Log and continue
        dev_warn_once(&xe->pdev->dev,
                     "Correctable HW error: %u\n",
                     error_code);
        break;
        
    case XE_HW_ERROR_NONFATAL:
        // Trigger GT reset
        xe_gt_reset(gt);
        break;
        
    case XE_HW_ERROR_FATAL:
        // Mark device as wedged, initiate recovery
        dev_err(&xe->pdev->dev,
               "Fatal HW error: %u, device wedged\n",
               error_code);
        xe_device_declare_wedged(xe);
        // May trigger devcoredump
        break;
    }
    
    // 4. Clear error register (CSME may require acknowledgment)
    xe_mmio_write32(gt, HW_ERROR_STATUS, err_status);
}
```

## 5. Memory-Based Interrupts (MemIRQ)

### Purpose

Traditional MSI-X requires special PCI device support. MemIRQ allows signaling via memory writes, useful in certain configurations (SR-IOV VFs, some platforms).

### MemIRQ Architecture

```c
struct xe_memirq {
    struct xe_bo *bo;           // BO containing IRQ values
    void *mapping;              // Kernel mapping of BO
    
    // Pending interrupt bits
    atomic_t pending;
    
    // Work to process pending
    struct work_struct work;
};
```

### MemIRQ Handler

```c
void xe_memirq_handler(struct xe_device *xe)
{
    struct xe_memirq *memirq = &xe->memirq;
    u32 pending;
    
    // 1. Read pending interrupts from BO
    pending = atomic_read(&memirq->pending);
    if (!pending)
        return;
    
    // Clear pending bits
    atomic_set(&memirq->pending, 0);
    
    // 2. Process like normal interrupt
    if (pending & TILE0_INTR_BIT)
        xe_tile_irq_handler(&xe->tiles[0]);
    
    if (pending & TILE1_INTR_BIT)
        xe_tile_irq_handler(&xe->tiles[1]);
}
```

## 6. Fence Signaling from Interrupts

### Completion Interrupt Flow

```
Hardware completes job
    │
    v
Completion interrupt signaled
    │
    v
Interrupt handler reads completion seqno
    │
    v
Find matching fence/job
    │
    v
Signal fence
    │
    ├─ GPU scheduler notified
    ├─ Userspace poll returns
    └─ DMA fence callbacks invoked
```

### Implementation

```c
static void xe_engine_irq_handler(struct xe_hw_engine *engine)
{
    u32 last_seqno = xe_mmio_read32(engine->gt, SEQNO_REG);
    
    // 1. Find all completed jobs
    for (each job in queue) {
        if (job->seqno > last_seqno)
            break;  // Not completed
        
        // 2. Signal fence
        dma_fence_signal(&job->fence);
        
        // 3. GPU scheduler processes completion
        drm_sched_job_done(&job->sched_job);
    }
}
```

## 7. Common Interrupt Issues

### Issue: Missed Completions

**Symptom**: Jobs not completing, GPU appears hung  
**Cause**: Race between job completion and interrupt enable

**Solution**:
```c
// Always use:
//   1. Set completion seqno in batch
//   2. Memory barrier
//   3. Check for completion before enabling interrupt
u32 last_seqno = read_seqno();
enable_completion_interrupt();
u32 current_seqno = read_seqno();

// If current > last, interrupt already arrived
if (current_seqno > last_seqno)
    handle_completion_manually();
```

### Issue: Interrupt Storms

**Symptom**: CPU load 100%, log flooded with interrupts  
**Cause**: Interrupt level not being cleared

**Solution**:
- Ensure interrupt status register is properly cleared
- Use interrupt masking during handler execution
- Implement bottom-half handler with proper synchronization

## 8. Debugging Interrupts

### Sysfs/Debugfs

```bash
# Check interrupt statistics
cat /sys/kernel/debug/dri/0/interrupt_info

# View pending interrupts
cat /sys/kernel/debug/dri/0/pending_interrupts

# Check GT interrupt state
cat /sys/kernel/debug/dri/0/gt_interrupt_state
```

### Kernel Logging

```bash
# Enable interrupt tracing
echo "1" > /sys/kernel/debug/tracing/events/xe/xe_irq_handler/enable

# View traced interrupts
cat /sys/kernel/debug/tracing/trace
```

### Common Debug Pattern

```c
// Add in handler to trace specific interrupts
if (unlikely(debug_interrupt)) {
    dev_dbg(&xe->pdev->dev,
           "IRQ: tile=%u gt=%u status=0x%x\n",
           tile->id, gt->info.id, gt_ctl);
}
```

## Summary

Interrupt handling in Xe:
1. **MSI-X vectors** routed to per-GT handlers
2. **Hierarchical dispatch** from device → tile → GT → engine
3. **Hardware errors** categorized and handled appropriately
4. **Fence signaling** via interrupt or memory-based IRQ
5. **Complex synchronization** between interrupt and normal paths

Understanding interrupt handling is critical for:
- Debugging job completion issues
- Understanding system latency
- Implementing new event types
- Performance optimization

---

**See Also**:
- [04-execution-scheduling.md](04-execution-scheduling.md) - Job completion via fences
- [10-potential-issues.md](10-potential-issues.md) - Known interrupt issues
- [02-guc-integration.md](02-guc-integration.md) - GuC interrupt handling
