# Page Fault Handling & GPU Memory Management

## Overview

This document details GPU page fault handling, memory protection, and virtual memory enforcement in the Intel Xe driver.

**Target Audience**: Memory protection engineers, page table specialists  
**Complexity**: HIGH  
**Key Files**:
- `xe_pagefault.c/h` - Page fault routing and handling
- `xe_guc_pagefault.c/h` - GuC page fault producer
- `xe_pt_walk.c` - Page table walking for fault resolution

## 1. Page Fault Architecture

### Two-Layer Design

```
GuC Firmware (Producer)
│
└─ Detects page fault
   ├─ Reads fault address
   ├─ Identifies VM & engine
   └─ Queues fault message
   │
   v
Host Driver (Consumer)
│
└─ Processes fault
   ├─ Finds faulting VA
   ├─ Allocates/updates page table
   ├─ Invalidates TLB
   └─ Allows GPU to retry
```

### Fault Queue

GuC maintains circular queue of pending faults:

```c
struct guc_pagefault_queue {
    // Circular buffer in GPU memory
    struct {
        u32 fault_addr;       // Faulting GPU virtual address
        u32 vm_id;            // VM context
        u32 engine_id;        // Which engine faulted
        u32 fault_type;       // Read, write, execute
    } entries[FAULT_QUEUE_SIZE];
    
    u32 head;               // Consumer reads from head
    u32 tail;               // Producer writes at tail
};
```

## 2. Fault Handler Flow

```c
static void xe_pagefault_handler(struct work_struct *work)
{
    struct xe_device *xe = container_of(work, ...);
    struct xe_pagefault_queue *queue = &xe->pagefault_queue;
    
    // 1. Get next fault from queue
    while (queue->head != queue->tail) {
        struct pagefault_entry fault = queue->entries[queue->head];
        queue->head = (queue->head + 1) % QUEUE_SIZE;
        
        // 2. Find affected VM
        struct xe_vm *vm = find_vm_by_id(fault.vm_id);
        if (!vm) {
            dev_warn(&xe->pdev->dev,
                    "Fault for unknown VM %u\n", fault.vm_id);
            continue;
        }
        
        // 3. Process fault
        process_pagefault(vm, fault.fault_addr, fault.fault_type);
    }
}

static void process_pagefault(struct xe_vm *vm,
                             u64 fault_addr,
                             u32 fault_type)
{
    // 1. Find VMA (virtual memory area) containing fault address
    struct xe_vma *vma = find_vma(vm, fault_addr);
    if (!vma) {
        // Unmapped access - fatal fault
        dev_err(&vm->xe->pdev->dev,
               "GPU page fault at unmapped VA 0x%llx\n",
               fault_addr);
        ban_vm(vm);  // Ban this VM
        return;
    }
    
    // 2. Check fault type vs VMA permissions
    if (fault_type == FAULT_TYPE_WRITE && !vma->writable) {
        // Write to read-only VMA - protect fault
        dev_warn(&vm->xe->pdev->dev,
                "Write fault on read-only VMA\n");
        ban_vm(vm);
        return;
    }
    
    // 3. Handle sparse binding (on-demand paging)
    if (vma->flags & VMA_SPARSE) {
        // Allocate missing page on fault
        allocate_page_for_vma(vm, vma, fault_addr);
    }
    
    // 4. Ensure page table present
    // (May create intermediate page tables)
    ensure_page_table_coverage(vm, fault_addr);
    
    // 5. Invalidate TLB
    // (Re-fetch translation from page table)
    xe_vm_tlb_inval_range(vm, fault_addr, fault_addr + PAGE_SIZE);
    
    // GPU can now retry instruction
}
```

## 3. Fault Types

### Hardware Fault Categories

```c
enum gpu_fault_type {
    FAULT_TYPE_READ = 0,        // Read from unmapped VA
    FAULT_TYPE_WRITE = 1,       // Write to unmapped/read-only VA
    FAULT_TYPE_EXECUTE = 2,     // Execute from non-executable VA
    FAULT_TYPE_ATOMIC = 3,      // Atomic op on unmapped VA
};
```

### Severity Classification

```
FATAL (VM banned):
├─ Access to completely unmapped VA
├─ Write to read-only memory
├─ Access to reserved VA range
└─ Multiple faults from same context

RECOVERABLE (resolved with retry):
├─ Sparse binding fault (allocate on fault)
├─ Missing intermediate page table
├─ TLB coherency issue (resolved by invalidate)
└─ Contention (shared page table modification)
```

## 4. Sparse Binding Support

Sparse binding allows allocating page tables dynamically:

```c
// User application: sparse texture
// Allocate 1GB virtual space, but use only 10MB
struct xe_vm *sparse_vm = xe_vm_create(xe);
sparse_vm->sparse_binding = true;

// Map sparse BO
struct xe_bo *sparse_bo = xe_bo_create(xe, 1_GB, SPARSE);
xe_vm_bind_sparse(sparse_vm, sparse_bo, 0, 1_GB);
// Note: No page tables allocated yet!

// GPU tries to access 0x1000
// → Page fault!
// → Driver allocates page table on fault
// → GPU retries, succeeds

// Much more memory efficient than pre-allocating all page tables
```

### Sparse Binding Handler

```c
void handle_sparse_pagefault(struct xe_vm *vm,
                            u64 fault_addr,
                            struct xe_vma *sparse_vma)
{
    struct xe_bo *sparse_bo = sparse_vma->bo;
    u64 offset_in_bo = fault_addr - sparse_vma->start;
    
    // 1. Check if BO has this offset
    if (offset_in_bo >= xe_bo_size(sparse_bo)) {
        // Fault beyond sparse BO size - fatal
        ban_vm(vm);
        return;
    }
    
    // 2. Get physical page from BO
    struct page *page = xe_bo_get_page(sparse_bo, offset_in_bo);
    if (!page) {
        // BO doesn't have this page - allocate
        page = alloc_page(GFP_KERNEL);
        set_bo_page(sparse_bo, offset_in_bo, page);
    }
    
    // 3. Ensure page table entry exists
    update_page_table_entry(vm, fault_addr, page);
    
    // 4. Invalidate TLB to allow retry
    xe_vm_tlb_inval_range(vm, fault_addr, fault_addr + PAGE_SIZE);
}
```

## 5. VM Banning

When GPU access violates memory protection:

```c
void ban_vm(struct xe_vm *vm)
{
    dev_warn(&vm->xe->pdev->dev,
            "Banning VM %u due to fault\n", vm->id);
    
    vm->banned = true;  // Mark as banned
    
    // All future submissions from this VM rejected
    // (See Issue #3 in potential-issues.md - VM ban notification missing)
    
    // Should notify userspace but doesn't currently
}

// In exec ioctl path:
if (vm->banned) {
    return -EIO;  // Reject with I/O error
}
```

## 6. Page Fault Debugging

### Common Fault Patterns

| Symptom | Cause | Solution |
|---------|-------|----------|
| "Unmapped VA fault" | Bug in batch buffer | Fix batch to only access bound BOs |
| Repeated faults | Sparse binding bug | Check sparse BO size and offset |
| Fault then hang | Ban notification missing | Implement signal to app (Issue #3) |
| Fault timeout | Fault handler stalled | Check for deadlock in page table lock |

### Debug Configuration

```bash
# Enable page fault logging
echo "1" > /sys/kernel/debug/dri/0/pagefault_log

# View fault statistics
cat /sys/kernel/debug/dri/0/fault_statistics

# Trace fault events
echo "1" > /sys/kernel/debug/tracing/events/xe/pagefault/enable
```

## 7. Fault Handler Locking

Critical for correctness but complex:

```
Fault handler must:
├─ Lock VM (prevent concurrent modifications)
├─ Lock page tables (prevent corruption)
├─ Allocate memory (may sleep)
└─ Avoid deadlock with other paths

Challenge:
└─ Other paths (job submission) also lock VM
   → Deadlock risk if taken in wrong order
```

### Lock Ordering

```c
// CORRECT order (prevents deadlock):
// 1. User fault lock
// 2. VM lock  
// 3. Page table lock
// 4. GuC lock (for TLB invalidation)

void correct_fault_locking(struct xe_vm *vm)
{
    // Take locks in order
    mutex_lock(&vm->pagefault_lock);
    
    xe_vm_lock_dma_resv(vm);
    
    xe_pt_lock_for_modification(vm->pt);
    
    // ... handle fault ...
    
    // Release in reverse order
    xe_pt_unlock(vm->pt);
    
    xe_vm_unlock_dma_resv(vm);
    
    mutex_unlock(&vm->pagefault_lock);
}
```

## 8. Known Issues

### Page Fault Handling Limited (High Priority)

**Issue**: Current implementation incomplete
**Coverage**: Basic fault detection, but limited recovery
**Missing**: Full sparse binding support, demand paging

**Status**: Being improved, but not complete

See [10-potential-issues.md](10-potential-issues.md) section 2.2

## Summary

Page fault handling enables:
1. **Sparse binding** - Allocate page tables on demand
2. **Memory protection** - Catch invalid access
3. **Flexible VA space** - Large sparse regions
4. **Dynamic allocation** - Grow page tables as needed

Key components:
- GuC firmware detects faults
- Host handler resolves (allocate page, update PT, flush TLB)
- Sparse bindings allow efficient large allocations
- VM ban prevents further damage

Still being developed with planned improvements for:
- Full sparse binding support
- Better error recovery
- Performance optimization

---

**See Also**:
- [03-memory-management.md](03-memory-management.md) - Memory basics
- [08-tlb-invalidation.md](08-tlb-invalidation.md) - TLB after fault
- [10-potential-issues.md](10-potential-issues.md) - Page fault limitations
