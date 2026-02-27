# Comprehensive Technical Analysis: Xe GPU Driver

## TOPIC 1: PAGE INVALIDATION (Malloc Freed While GPU Using It)

### Overview
The Xe driver protects GPU memory mappings when user application frees or modifies memory using the Linux MMU notifier infrastructure. This prevents the GPU from accessing freed or reallocated memory.

### 1.1 MMU Notifier Detection: munmap/madvise/free

**File References:**
- `xe_userptr.c`: Lines 126-152 (callback handler)
- `xe_userptr.h`: Lines 45-69 (data structures)
- `xe_vm_types.h`: Lines 156-164 (container structure)

**Core Mechanism:**

```
User Space                          Kernel Space
  |                                    |
  +-- munmap()  -------->  VMA removal
  +-- madvise() -------->  VMA modification
  +-- malloc free() ----->  Memory release
         |
         v
    mmu_notifier_range_start()
         |
         v
    Iterate all registered mmu_interval_notifiers
         |
         v
    Call vma_userptr_invalidate() [xe_userptr.c:126]
```

**Registration Flow (xe_userptr.c:278-296):**
```c
int xe_userptr_setup(struct xe_userptr_vma *uvma, unsigned long start,
                     unsigned long range)
{
    struct xe_userptr *userptr = &uvma->userptr;
    int err;
    
    INIT_LIST_HEAD(&userptr->invalidate_link);
    INIT_LIST_HEAD(&userptr->repin_link);
    
    // Register interval notifier with kernel MMU subsystem
    err = mmu_interval_notifier_insert(&userptr->notifier, current->mm,
                                       start, range,
                                       &vma_userptr_notifier_ops);  // Line 287
    if (err)
        return err;
    
    userptr->pages.notifier_seq = LONG_MAX;
    return 0;
}
```

**Data Structure (xe_userptr.h:45-69):**
```c
struct xe_userptr {
    struct list_head invalidate_link;        // Line 48: Link for invalidated list
    struct list_head repin_link;             // Line 50: Link for repin list
    struct drm_gpusvm_pages pages;           // Line 54: GPU SVM pages
    struct mmu_interval_notifier notifier;   // Line 58: MMU notifier
    bool initial_bind;                       // Line 65: Has been bound once
};
```

**Container Structure (xe_vm_types.h:156-164):**
```c
struct xe_userptr_vma {
    struct xe_vma vma;              // Line 162: The VMA
    struct xe_userptr userptr;      // Line 163: Userptr info
};
```

### 1.2 Invalidation Callback Mechanism

**Callback Handler (xe_userptr.c:126-152):**
```c
static bool vma_userptr_invalidate(struct mmu_interval_notifier *mni,
                                   const struct mmu_notifier_range *range,
                                   unsigned long cur_seq)
{
    struct xe_userptr_vma *uvma = 
        container_of(mni, typeof(*uvma), userptr.notifier);  // Line 130
    struct xe_vma *vma = &uvma->vma;
    struct xe_vm *vm = xe_vma_vm(vma);
    
    xe_assert(vm->xe, xe_vma_is_userptr(vma));
    trace_xe_vma_userptr_invalidate(vma);                    // Line 135
    
    // Check if we can block (may sleep)
    if (!mmu_notifier_range_blockable(range))               // Line 137
        return false;  // Can't handle synchronously, retry later
    
    // CRITICAL SECTION: Acquire notifier lock
    down_write(&vm->svm.gpusvm.notifier_lock);              // Line 144
    mmu_interval_set_seq(mni, cur_seq);                     // Line 145
    
    // Perform invalidation
    __vma_userptr_invalidate(vm, uvma);                     // Line 147
    up_write(&vm->svm.gpusvm.notifier_lock);                // Line 148
    trace_xe_vma_userptr_invalidate_complete(vma);          // Line 149
    
    return true;
}
```

**Notifier Operations (xe_userptr.c:154-156):**
```c
static const struct mmu_interval_notifier_ops vma_userptr_notifier_ops = {
    .invalidate = vma_userptr_invalidate,   // Line 155: Callback on invalidation
};
```

### 1.3 What Happens When GPU Is Mid-Operation

**Critical Implementation (xe_userptr.c:76-124):**
```c
static void __vma_userptr_invalidate(struct xe_vm *vm, struct xe_userptr_vma *uvma)
{
    struct xe_userptr *userptr = &uvma->userptr;
    struct xe_vma *vma = &uvma->vma;
    struct dma_resv_iter cursor;
    struct dma_fence *fence;
    struct drm_gpusvm_ctx ctx = {
        .in_notifier = true,
        .read_only = xe_vma_read_only(vma),
    };
    long err;
    
    // Step 1: Mark for rebinding (if not in fault mode)
    if (!xe_vm_in_fault_mode(vm) &&                         // Line 92
        !(vma->gpuva.flags & XE_VMA_DESTROYED)) {
        spin_lock(&vm->userptr.invalidated_lock);
        list_move_tail(&userptr->invalidate_link,
                       &vm->userptr.invalidated);            // Lines 95-96
        spin_unlock(&vm->userptr.invalidated_lock);
    }
    
    // Step 2: Wait for all GPU work to complete
    // Enable software signaling on pending fences
    dma_resv_iter_begin(&cursor, xe_vm_resv(vm),
                        DMA_RESV_USAGE_BOOKKEEP);           // Line 106
    dma_resv_for_each_fence_unlocked(&cursor, fence)
        dma_fence_enable_sw_signaling(fence);               // Line 109
    dma_resv_iter_end(&cursor);
    
    // WAIT FOR COMPLETION: Block until GPU is done
    err = dma_resv_wait_timeout(xe_vm_resv(vm),
                                DMA_RESV_USAGE_BOOKKEEP,
                                false, MAX_SCHEDULE_TIMEOUT); // Line 112
    XE_WARN_ON(err <= 0);                                   // Line 115
    
    // Step 3: If in fault mode, invalidate the mapping
    if (xe_vm_in_fault_mode(vm) && userptr->initial_bind) {
        err = xe_vm_invalidate_vma(vma);                    // Line 118
        XE_WARN_ON(err);
    }
    
    // Step 4: Unmap pages from GPU
    drm_gpusvm_unmap_pages(&vm->svm.gpusvm, &uvma->userptr.pages,
                           xe_vma_size(vma) >> PAGE_SHIFT, &ctx);  // Lines 122-123
}
```

**Flow Diagram:**
```
GPU executing batch
    |
    v
User calls munmap()/madvise()
    |
    v
mmu_notifier_range_start()
    |
    v
vma_userptr_invalidate() called [xe_userptr.c:126]
    |
    +-- Check if blockable
    |
    +-- Mark for rebind [xe_userptr.c:95-96]
    |
    +-- WAIT for GPU completion [xe_userptr.c:112-114]
    |   |
    |   +-- Iterate all fences
    |   +-- Enable software signaling
    |   +-- Block until fence->error or fence->seqno >= GPU seqno
    |
    +-- Unmap pages [xe_userptr.c:122]
    |
    v
mmu_notifier_range_end()
    v
User's munmap()/madvise() completes
```

### 1.4 Rebind Mechanisms

**Rebind Data Structures (xe_vm_types.h:256-258):**
```c
/** @userptr: user pointer state */
struct xe_userptr_vm userptr;
```

**Rebind Lists (xe_userptr.h:21-43):**
```c
struct xe_userptr_vm {
    struct list_head repin_list;              // Line 27: VMs needing repin
    spinlock_t invalidated_lock;              // Line 32: Protects invalidated list
    struct list_head invalidated;             // Line 42: List of invalidated userptrs
};
```

**Repin Entry Point (xe_userptr.c:186-259):**
```c
int xe_vm_userptr_pin(struct xe_vm *vm)
{
    struct xe_userptr_vma *uvma, *next;
    int err = 0;
    
    xe_assert(vm->xe, !xe_vm_in_fault_mode(vm));
    lockdep_assert_held_write(&vm->lock);
    
    // Step 1: Collect invalidated userptrs
    spin_lock(&vm->userptr.invalidated_lock);
    xe_assert(vm->xe, list_empty(&vm->userptr.repin_list));
    list_for_each_entry_safe(uvma, next, &vm->userptr.invalidated,
                             userptr.invalidate_link) {
        list_del_init(&uvma->userptr.invalidate_link);
        list_add_tail(&uvma->userptr.repin_link,
                      &vm->userptr.repin_list);              // Line 200-201
    }
    spin_unlock(&vm->userptr.invalidated_lock);
    
    // Step 2: Pin and move to bind list
    list_for_each_entry_safe(uvma, next, &vm->userptr.repin_list,
                             userptr.repin_link) {
        err = xe_vma_userptr_pin_pages(uvma);               // Line 208
        if (err == -EFAULT) {
            // Handle EFAULT: page not present anymore
            list_del_init(&uvma->userptr.repin_link);
            
            if (!list_empty(&uvma->vma.combined_links.rebind))
                list_del_init(&uvma->vma.combined_links.rebind);  // Line 222
            
            // Wait for pending binds
            xe_vm_lock(vm, false);
            dma_resv_wait_timeout(xe_vm_resv(vm),
                                  DMA_RESV_USAGE_BOOKKEEP,
                                  false, MAX_SCHEDULE_TIMEOUT);    // Line 226-228
            
            down_read(&vm->svm.gpusvm.notifier_lock);
            err = xe_vm_invalidate_vma(&uvma->vma);         // Line 231
            up_read(&vm->svm.gpusvm.notifier_lock);
            xe_vm_unlock(vm);
            if (err)
                break;
        } else {
            if (err)
                break;
            
            // Success: move to rebind list
            list_del_init(&uvma->userptr.repin_link);
            list_move_tail(&uvma->vma.combined_links.rebind,
                           &vm->rebind_list);                // Line 241-242
        }
    }
    
    // Step 3: On error, restore to invalidated list
    if (err) {
        down_write(&vm->svm.gpusvm.notifier_lock);
        spin_lock(&vm->userptr.invalidated_lock);
        list_for_each_entry_safe(uvma, next, &vm->userptr.repin_list,
                                 userptr.repin_link) {
            list_del_init(&uvma->userptr.repin_link);
            list_move_tail(&uvma->userptr.invalidate_link,
                           &vm->userptr.invalidated);        // Line 252-253
        }
        spin_unlock(&vm->userptr.invalidated_lock);
        up_write(&vm->svm.gpusvm.notifier_lock);
    }
    return err;
}
```

**Advisory Check (xe_userptr.c:25-30, 272-276):**
```c
int xe_vma_userptr_check_repin(struct xe_userptr_vma *uvma)
{
    // Lock-free advisory check
    return mmu_interval_check_retry(&uvma->userptr.notifier,
                                    uvma->userptr.pages.notifier_seq) ?
        -EAGAIN : 0;                                         // Line 29
}

int xe_vm_userptr_check_repin(struct xe_vm *vm)
{
    // Advisory check for VM-level repinning need
    return (list_empty_careful(&vm->userptr.repin_list) &&
            list_empty_careful(&vm->userptr.invalidated)) ? 0 : -EAGAIN;  // Line 274-275
}
```

### 1.5 Exact Function and Line Numbers Summary

| Function | File | Lines | Purpose |
|----------|------|-------|---------|
| `vma_userptr_invalidate` | xe_userptr.c | 126-152 | MMU notifier callback |
| `__vma_userptr_invalidate` | xe_userptr.c | 76-124 | Core invalidation logic |
| `xe_userptr_setup` | xe_userptr.c | 278-296 | Register MMU notifier |
| `xe_userptr_remove` | xe_userptr.c | 298-312 | Unregister MMU notifier |
| `xe_vm_userptr_pin` | xe_userptr.c | 186-259 | Rebind invalidated pages |
| `xe_vma_userptr_pin_pages` | xe_userptr.c | 51-74 | Pin pages for GPU access |
| `xe_vma_userptr_check_repin` | xe_userptr.c | 25-30 | Advisory repin check |
| `__xe_vm_userptr_needs_repin` | xe_userptr.c | 43-49 | VM-level repin check |

---

## TOPIC 2: CPU-GPU SYNCHRONIZATION (Fences)

### Overview
The Xe driver implements three complementary synchronization models for coordinating CPU and GPU work:
1. **DMA Fences** - GPU-managed hardware seqno-based signaling
2. **User Fences** - CPU-writable memory for GPU completion notification
3. **Syncobjs** - Timeline-based synchronization objects

### 2.1 DMA Fence Implementation

**Hardware Fence Structure (xe_hw_fence_types.h:56-73):**
```c
struct xe_hw_fence {
    struct dma_fence dma;              // Line 64: Base DMA fence
    struct xe_device *xe;              // Line 66: Device reference
    char name[MAX_FENCE_NAME_LEN];     // Line 68: Fence name
    struct iosys_map seqno_map;        // Line 70: I/O map for seqno
    struct list_head irq_link;         // Line 72: Link in pending list
};
```

**Fence Context (xe_hw_fence_types.h:38-54):**
```c
struct xe_hw_fence_ctx {
    struct xe_gt *gt;                  // Line 45: GT structure
    struct xe_hw_fence_irq *irq;       // Line 47: IRQ handler
    u64 dma_fence_ctx;                 // Line 49: DMA fence context
    u32 next_seqno;                    // Line 51: Next sequence number
    char name[MAX_FENCE_NAME_LEN];     // Line 53: Context name
};
```

**Fence Creation (xe_hw_fence.c:221-268):**
```c
struct dma_fence *xe_hw_fence_alloc(void)
{
    struct xe_hw_fence *hw_fence = fence_alloc();
    
    if (!hw_fence)
        return ERR_PTR(-ENOMEM);
    
    return &hw_fence->dma;              // Line 228
}

void xe_hw_fence_init(struct dma_fence *fence, struct xe_hw_fence_ctx *ctx,
                      struct iosys_map seqno_map)
{
    struct xe_hw_fence *hw_fence =
        container_of(fence, typeof(*hw_fence), dma);
    
    hw_fence->xe = gt_to_xe(ctx->gt);
    snprintf(hw_fence->name, sizeof(hw_fence->name), "%s", ctx->name);
    hw_fence->seqno_map = seqno_map;
    INIT_LIST_HEAD(&hw_fence->irq_link);
    
    // Initialize the DMA fence with IRQ handler lock
    dma_fence_init(fence, &xe_hw_fence_ops, &ctx->irq->lock,
                   ctx->dma_fence_ctx, ctx->next_seqno++);      // Line 264
    
    trace_xe_hw_fence_create(hw_fence);
}
```

**Fence Signaling Detection (xe_hw_fence.c:164-187):**
```c
static bool xe_hw_fence_signaled(struct dma_fence *dma_fence)
{
    struct xe_hw_fence *fence = to_xe_hw_fence(dma_fence);
    struct xe_device *xe = fence->xe;
    
    // Read seqno from GPU memory location
    u32 seqno = xe_map_rd(xe, &fence->seqno_map, 0, u32);    // Line 168
    
    // Check if seqno has advanced past fence's seqno
    return dma_fence->error ||
        !__dma_fence_is_later(dma_fence, dma_fence->seqno, seqno);  // Line 171
}

static bool xe_hw_fence_enable_signaling(struct dma_fence *dma_fence)
{
    struct xe_hw_fence *fence = to_xe_hw_fence(dma_fence);
    struct xe_hw_fence_irq *irq = xe_hw_fence_irq(fence);
    
    dma_fence_get(dma_fence);
    list_add_tail(&fence->irq_link, &irq->pending);          // Line 180
    
    // SW completed (no HW IRQ) so kick handler to signal fence
    if (xe_hw_fence_signaled(dma_fence))
        xe_hw_fence_irq_run(irq);                            // Line 184
    
    return true;
}
```

**IRQ Handler (xe_hw_fence.c:52-74):**
```c
static void hw_fence_irq_run_cb(struct irq_work *work)
{
    struct xe_hw_fence_irq *irq = container_of(work, typeof(*irq), work);
    struct xe_hw_fence *fence, *next;
    bool tmp;
    
    tmp = dma_fence_begin_signalling();
    spin_lock(&irq->lock);
    if (irq->enabled) {
        list_for_each_entry_safe(fence, next, &irq->pending, irq_link) {
            struct dma_fence *dma_fence = &fence->dma;
            
            trace_xe_hw_fence_try_signal(fence);
            if (dma_fence_is_signaled_locked(dma_fence)) {    // Line 65
                trace_xe_hw_fence_signal(fence);
                list_del_init(&fence->irq_link);
                dma_fence_put(dma_fence);
            }
        }
    }
    spin_unlock(&irq->lock);
    dma_fence_end_signalling(tmp);
}
```

### 2.2 User Fence Implementation

**User Fence Structure (xe_sync.c:22-31):**
```c
struct xe_user_fence {
    struct xe_device *xe;              // Line 23: Device reference
    struct kref refcount;              // Line 24: Reference count
    struct dma_fence_cb cb;            // Line 25: Fence callback
    struct work_struct worker;         // Line 26: Worker for writeback
    struct mm_struct *mm;              // Line 27: User address space
    u64 __user *addr;                  // Line 28: User address to write
    u64 value;                         // Line 29: Value to write on completion
    int signalled;                     // Line 30: Signaling status
};
```

**User Fence Creation (xe_sync.c:52-74):**
```c
static struct xe_user_fence *user_fence_create(struct xe_device *xe, u64 addr,
                                               u64 value)
{
    struct xe_user_fence *ufence;
    u64 __user *ptr = u64_to_user_ptr(addr);
    u64 __maybe_unused prefetch_val;
    
    // Prefetch user address to ensure page is present
    if (get_user(prefetch_val, ptr))
        return ERR_PTR(-EFAULT);                            // Line 60
    
    ufence = kzalloc(sizeof(*ufence), GFP_KERNEL);
    if (!ufence)
        return ERR_PTR(-ENOMEM);
    
    ufence->xe = xe;
    kref_init(&ufence->refcount);
    ufence->addr = ptr;
    ufence->value = value;
    ufence->mm = current->mm;
    mmgrab(ufence->mm);                                     // Line 71
    
    return ufence;
}
```

**User Fence Callback Chain (xe_sync.c:76-111):**
```c
static void user_fence_worker(struct work_struct *w)
{
    struct xe_user_fence *ufence = container_of(w, struct xe_user_fence, worker);
    
    WRITE_ONCE(ufence->signalled, 1);                      // Line 80
    if (mmget_not_zero(ufence->mm)) {
        // Switch to user mm and write fence value
        kthread_use_mm(ufence->mm);
        if (copy_to_user(ufence->addr, &ufence->value, sizeof(ufence->value)))
            XE_WARN_ON("Copy to user failed");              // Line 84
        kthread_unuse_mm(ufence->mm);
        mmput(ufence->mm);
    } else {
        drm_dbg(&ufence->xe->drm, "mmget_not_zero() failed\n");  // Line 88
    }
    
    // Wake all waiters
    wake_up_all(&ufence->xe->ufence_wq);                  // Line 95
    user_fence_put(ufence);
}

static void user_fence_cb(struct dma_fence *fence, struct dma_fence_cb *cb)
{
    struct xe_user_fence *ufence = container_of(cb, struct xe_user_fence, cb);
    
    kick_ufence(ufence, fence);                           // Line 110
}
```

**User Fence Signaling (xe_sync.c:231-266):**
```c
void xe_sync_entry_signal(struct xe_sync_entry *sync, struct dma_fence *fence)
{
    if (!(sync->flags & DRM_XE_SYNC_FLAG_SIGNAL))
        return;
    
    if (sync->chain_fence) {
        // Timeline-based syncobj signaling
        drm_syncobj_add_point(sync->syncobj, sync->chain_fence,
                              fence, sync->timeline_value);  // Line 237
        sync->chain_fence = NULL;
    } else if (sync->syncobj) {
        // Direct syncobj signaling
        drm_syncobj_replace_fence(sync->syncobj, fence);    // Line 245
    } else if (sync->ufence) {
        // User fence signaling
        int err;
        
        drm_syncobj_add_point(sync->ufence_syncobj,
                              sync->ufence_chain_fence,
                              fence, sync->ufence_timeline_value);  // Line 249
        sync->ufence_chain_fence = NULL;
        
        fence = drm_syncobj_fence_get(sync->ufence_syncobj);
        user_fence_get(sync->ufence);
        // Register callback for completion
        err = dma_fence_add_callback(fence, &sync->ufence->cb,
                                     user_fence_cb);        // Line 256
        if (err == -ENOENT) {
            // Fence already signaled, kick worker immediately
            kick_ufence(sync->ufence, fence);              // Line 259
        } else if (err) {
            XE_WARN_ON("failed to add user fence");
            user_fence_put(sync->ufence);
            dma_fence_put(fence);
        }
    }
}
```

### 2.3 Syncobj and Timeline Integration

**Sync Entry Parsing (xe_sync.c:113-219):**
```c
int xe_sync_entry_parse(struct xe_device *xe, struct xe_file *xef,
                        struct xe_sync_entry *sync,
                        struct drm_xe_sync __user *sync_user,
                        struct drm_syncobj *ufence_syncobj,
                        u64 ufence_timeline_value,
                        unsigned int flags)
{
    struct drm_xe_sync sync_in;
    int err;
    bool exec = flags & SYNC_PARSE_FLAG_EXEC;
    bool in_lr_mode = flags & SYNC_PARSE_FLAG_LR_MODE;
    bool disallow_user_fence = flags & SYNC_PARSE_FLAG_DISALLOW_USER_FENCE;
    bool signal;
    
    if (copy_from_user(&sync_in, sync_user, sizeof(*sync_user)))
        return -EFAULT;                                      // Line 127
    
    // Validate input
    if (XE_IOCTL_DBG(xe, sync_in.flags & ~DRM_XE_SYNC_FLAG_SIGNAL) ||
        XE_IOCTL_DBG(xe, sync_in.reserved[0] || sync_in.reserved[1]))
        return -EINVAL;                                      // Line 131
    
    signal = sync_in.flags & DRM_XE_SYNC_FLAG_SIGNAL;
    switch (sync_in.type) {
    case DRM_XE_SYNC_TYPE_SYNCOBJ:
        // Simple syncobj handling
        sync->syncobj = drm_syncobj_find(xef->drm, sync_in.handle);
        if (XE_IOCTL_DBG(xe, !sync->syncobj))
            return -ENOENT;                                  // Line 145
        
        if (!signal) {
            // In-fence: wait for syncobj completion
            sync->fence = drm_syncobj_fence_get(sync->syncobj);
            if (XE_IOCTL_DBG(xe, !sync->fence))
                return -EINVAL;                              // Line 150
        }
        break;
        
    case DRM_XE_SYNC_TYPE_TIMELINE_SYNCOBJ:
        // Timeline syncobj handling
        if (XE_IOCTL_DBG(xe, sync_in.timeline_value == 0))
            return -EINVAL;                                  // Line 162
        
        sync->syncobj = drm_syncobj_find(xef->drm, sync_in.handle);
        if (XE_IOCTL_DBG(xe, !sync->syncobj))
            return -ENOENT;                                  // Line 165
        
        if (signal) {
            // Out-fence: signal timeline point
            sync->chain_fence = dma_fence_chain_alloc();
            if (!sync->chain_fence)
                return -ENOMEM;                              // Line 170
        } else {
            // In-fence: wait for timeline point
            sync->fence = drm_syncobj_fence_get(sync->syncobj);
            if (XE_IOCTL_DBG(xe, !sync->fence))
                return -EINVAL;                              // Line 174
            
            // Extract fence at timeline_value
            err = dma_fence_chain_find_seqno(&sync->fence,
                                             sync_in.timeline_value);  // Line 177
            if (err)
                return err;
        }
        break;
        
    case DRM_XE_SYNC_TYPE_USER_FENCE:
        // User fence handling
        if (XE_IOCTL_DBG(xe, disallow_user_fence))
            return -EOPNOTSUPP;                              // Line 186
        
        if (XE_IOCTL_DBG(xe, !signal))
            return -EOPNOTSUPP;                              // Line 189
        
        if (XE_IOCTL_DBG(xe, sync_in.addr & 0x7))
            return -EINVAL;                                  // Line 192: qword align
        
        if (exec) {
            // From exec IOCTL: addr is GPU address
            sync->addr = sync_in.addr;                      // Line 195
        } else {
            // From vm_bind IOCTL: addr is user pointer
            sync->ufence_timeline_value = ufence_timeline_value;
            sync->ufence = user_fence_create(xe, sync_in.addr,
                                             sync_in.timeline_value);  // Line 198
            if (XE_IOCTL_DBG(xe, IS_ERR(sync->ufence)))
                return PTR_ERR(sync->ufence);                // Line 201
        }
        break;
    }
    
    sync->type = sync_in.type;
    sync->flags = sync_in.flags;
    sync->timeline_value = sync_in.timeline_value;
    
    return 0;
}
```

### 2.4 Job Fence Integration

**Sched Job Structure (xe_sched_job_types.h:34-72):**
```c
struct xe_sched_job {
    struct drm_sched_job drm;          // Line 39: Base DRM scheduler job
    struct xe_exec_queue *q;           // Line 41: Execution queue
    struct kref refcount;              // Line 43: Reference count
    struct dma_fence *fence;           // Line 48: Completion fence
    struct {
        bool used;                     // Line 52: User fence used
        u64 addr;                      // Line 53: GPU address to write
        u64 value;                     // Line 55: Value to write
    } user_fence;
    u32 lrc_seqno;                     // Line 59: LRC sequence number
    // ... more fields
    struct xe_job_ptrs ptrs[];         // Line 71: Per-engine pointers
};
```

**Job Arming (xe_sched_job.c:245-290):**
```c
void xe_sched_job_arm(struct xe_sched_job *job)
{
    struct xe_exec_queue *q = job->q;
    struct dma_fence *fence, *prev;
    struct xe_vm *vm = q->vm;
    u64 seqno = 0;
    int i;
    
    /* Locking assertions */
    if (IS_ENABLED(CONFIG_LOCKDEP) &&
        !(q->flags & (EXEC_QUEUE_FLAG_KERNEL | EXEC_QUEUE_FLAG_VM))) {
        lockdep_assert_held(&q->vm->lock);
        if (!xe_vm_in_lr_mode(q->vm))
            xe_vm_assert_held(q->vm);
    }
    
    // Arm pre-allocated fences
    for (i = 0; i < q->width; prev = fence, ++i) {
        struct dma_fence_chain *chain;
        
        fence = job->ptrs[i].lrc_fence;
        xe_lrc_init_seqno_fence(q->lrc[i], fence);         // Line 273
        job->ptrs[i].lrc_fence = NULL;
        if (!i) {
            job->lrc_seqno = fence->seqno;
            continue;
        } else {
            xe_assert(gt_to_xe(q->gt), job->lrc_seqno == fence->seqno);
        }
        
        // Chain fences for parallel execution
        chain = job->ptrs[i - 1].chain_fence;
        dma_fence_chain_init(chain, prev, fence, seqno++);  // Line 283
        job->ptrs[i - 1].chain_fence = NULL;
        fence = &chain->base;
    }
    
    // Final fence for userspace
    job->fence = dma_fence_get(fence);                      // Line 288
    drm_sched_job_arm(&job->drm);
}
```

**Job Completion Check (xe_sched_job.c:220-243):**
```c
bool xe_sched_job_started(struct xe_sched_job *job)
{
    struct dma_fence *fence = dma_fence_chain_contained(job->fence);
    struct xe_lrc *lrc = job->q->lrc[0];
    
    // Check if GPU seqno has reached job's seqno
    return !__dma_fence_is_later(fence,
                                 xe_sched_job_lrc_seqno(job),
                                 xe_lrc_start_seqno(lrc));    // Line 226
}

bool xe_sched_job_completed(struct xe_sched_job *job)
{
    struct dma_fence *fence = dma_fence_chain_contained(job->fence);
    struct xe_lrc *lrc = job->q->lrc[0];
    
    // Can safely check just LRC[0] seqno as that is last seqno written
    return !__dma_fence_is_later(fence,
                                 xe_sched_job_lrc_seqno(job),
                                 xe_lrc_seqno(lrc));          // Line 241
}
```

### 2.5 Practical IOCTL Sequences for Sync

**Sequence 1: Syncobj-based Synchronization**
```c
// Step 1: Create syncobj (DRM generic, not XE-specific)
struct drm_syncobj_create syncobj_create = {0};
ioctl(fd, DRM_IOCTL_SYNCOBJ_CREATE, &syncobj_create);
uint32_t syncobj_handle = syncobj_create.handle;  // Out

// Step 2: Execute job with output sync
struct drm_xe_sync out_sync = {
    .type = DRM_XE_SYNC_TYPE_SYNCOBJ,
    .flags = DRM_XE_SYNC_FLAG_SIGNAL,
    .handle = syncobj_handle
};

struct drm_xe_exec exec = {
    .exec_queue_id = queue_id,
    .syncs = &out_sync,
    .num_syncs = 1,
    .address = batch_address,
    .num_batch_buffer = 1
};
ioctl(fd, DRM_IOCTL_XE_EXEC, &exec);

// Step 3: Wait for job completion
struct drm_syncobj_wait wait = {
    .handles = &syncobj_handle,
    .timeout_nsec = INT64_MAX,      // Or specific timeout
    .count_handles = 1,
    .flags = 0
};
ioctl(fd, DRM_IOCTL_SYNCOBJ_WAIT, &wait);
```

**Sequence 2: Timeline Syncobj Synchronization**
```c
// Step 1: Create timeline syncobj
struct drm_syncobj_create syncobj_create = {
    .flags = DRM_SYNCOBJ_CREATE_TIMELINE_SIGNALING
};
ioctl(fd, DRM_IOCTL_SYNCOBJ_CREATE, &syncobj_create);
uint32_t syncobj_handle = syncobj_create.handle;

// Step 2: Execute with timeline fence point
struct drm_xe_sync timeline_sync = {
    .type = DRM_XE_SYNC_TYPE_TIMELINE_SYNCOBJ,
    .flags = DRM_XE_SYNC_FLAG_SIGNAL,
    .handle = syncobj_handle,
    .timeline_value = 1  // First point
};

struct drm_xe_exec exec = {
    .exec_queue_id = queue_id,
    .syncs = &timeline_sync,
    .num_syncs = 1,
    .address = batch_address,
    .num_batch_buffer = 1
};
ioctl(fd, DRM_IOCTL_XE_EXEC, &exec);

// Step 3: Wait for timeline point
struct drm_syncobj_timeline_wait timeline_wait = {
    .handles = &syncobj_handle,
    .points = {1},  // Wait for timeline point 1
    .timeout_nsec = INT64_MAX,
    .count_handles = 1,
    .flags = 0
};
ioctl(fd, DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT, &timeline_wait);
```

**Sequence 3: User Fence Synchronization**
```c
// Step 1: Allocate user memory for fence value
uint64_t *fence_addr = mmap(NULL, PAGE_SIZE, PROT_READ | PROT_WRITE,
                            MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);
*fence_addr = 0;  // Initial value

// Step 2: Execute with user fence
struct drm_xe_sync user_fence_sync = {
    .type = DRM_XE_SYNC_TYPE_USER_FENCE,
    .flags = DRM_XE_SYNC_FLAG_SIGNAL,
    .addr = (uint64_t)fence_addr,
    .timeline_value = 0xdeadbeef  // Value to write on completion
};

struct drm_xe_exec exec = {
    .exec_queue_id = queue_id,
    .syncs = &user_fence_sync,
    .num_syncs = 1,
    .address = batch_address,
    .num_batch_buffer = 1
};
ioctl(fd, DRM_IOCTL_XE_EXEC, &exec);

// Step 3: Wait using kernel wait ioctl
struct drm_xe_wait_user_fence wait = {
    .addr = (uint64_t)fence_addr,
    .value = 0xdeadbeef,
    .mask = 0xffffffffffffffff,
    .op = DRM_XE_UFENCE_WAIT_OP_EQ,
    .timeout = 1000000000  // 1 second in nanoseconds
};
ioctl(fd, DRM_IOCTL_XE_WAIT_USER_FENCE, &wait);

// Step 4: Alternatively, poll in userspace
while (*fence_addr != 0xdeadbeef) {
    usleep(1000);
}
```

### 2.6 Performance Characteristics

| Model | Latency | CPU Usage | Memory | Best For |
|-------|---------|-----------|--------|----------|
| **DMA Fences** | Sub-microsecond | Low (IRQ driven) | ~512 bytes | General purpose, zero-copy |
| **User Fences** | Microseconds | Low (callback+worker) | ~1 KB | CPU visibility needed |
| **Syncobjs** | Milliseconds | Moderate (polling) | ~2 KB | Multi-device synchronization |

**Signaling Path Comparison:**

```
DMA Fence signaling:
  GPU writes seqno
    |
    v
  Hardware IRQ fires
    |
    v
  xe_hw_fence_irq_run_cb() [xe_hw_fence.c:52]
    |
    v
  dma_fence_is_signaled_locked() [xe_hw_fence.c:65]
    |
    v
  dma_fence_put() called
  
  Total: ~10 microseconds

User Fence signaling:
  GPU writes seqno
    |
    v
  dma_fence_add_callback() [xe_sync.c:256]
    |
    v
  user_fence_cb() [xe_sync.c:106]
    |
    v
  queue_work() on ordered_wq [xe_sync.c:102]
    |
    v
  user_fence_worker() [xe_sync.c:76]
    |
    v
  copy_to_user() to write fence value
    |
    v
  wake_up_all() [xe_sync.c:95]
    
  Total: ~50-500 microseconds (depends on workqueue scheduling)

Syncobj signaling:
  Generic DRM syncobj handler
    |
    v
  userspace polls or calls IOCTL
    
  Total: Depends on userspace polling interval
```

---

## TOPIC 3: DMA-BUF INTEGRATION (External Memory Sharing)

### Overview
DMA-buf integration allows Xe GPU memory to be shared with other devices through the Linux dma-buf (DMA Buffer Sharing) framework. This enables zero-copy data sharing between GPU and other accelerators.

### 3.1 Memory Export (Prime Export)

**Export Function (xe_dma_buf.c:214-239):**
```c
struct dma_buf *xe_gem_prime_export(struct drm_gem_object *obj, int flags)
{
    struct xe_bo *bo = gem_to_xe_bo(obj);
    struct dma_buf *buf;
    struct ttm_operation_ctx ctx = {
        .interruptible = true,
        .no_wait_gpu = true,
        /* Avoid OOM on system pages allocations */
        .gfp_retry_mayfail = true,                         // Line 222
        .allow_res_evict = false,
    };
    int ret;
    
    // Only VM-less buffers can be exported
    if (bo->vm)
        return ERR_PTR(-EPERM);                             // Line 228
    
    // Setup TTM buffer for export
    ret = ttm_bo_setup_export(&bo->ttm, &ctx);
    if (ret)
        return ERR_PTR(ret);                                // Line 231
    
    // Export to DMA-buf
    buf = drm_gem_prime_export(obj, flags);
    if (!IS_ERR(buf))
        buf->ops = &xe_dmabuf_ops;                         // Line 236
    
    return buf;
}
```

**DMA-buf Operations (xe_dma_buf.c:200-212):**
```c
static const struct dma_buf_ops xe_dmabuf_ops = {
    .attach = xe_dma_buf_attach,        // Line 201: Attach device
    .detach = xe_dma_buf_detach,        // Line 202: Detach device
    .pin = xe_dma_buf_pin,              // Line 203: Pin for export
    .unpin = xe_dma_buf_unpin,          // Line 204: Unpin after export
    .map_dma_buf = xe_dma_buf_map,      // Line 205: Create SG table
    .unmap_dma_buf = xe_dma_buf_unmap,  // Line 206: Destroy SG table
    .release = drm_gem_dmabuf_release,  // Line 207: Release BO
    .begin_cpu_access = xe_dma_buf_begin_cpu_access,  // Line 208
    .mmap = drm_gem_dmabuf_mmap,        // Line 209
    .vmap = drm_gem_dmabuf_vmap,        // Line 210
    .vunmap = drm_gem_dmabuf_vunmap,    // Line 211
};
```

### 3.2 DMA-buf Attach/Pin Operations

**Attach Handler (xe_dma_buf.c:25-39):**
```c
static int xe_dma_buf_attach(struct dma_buf *dmabuf,
                             struct dma_buf_attachment *attach)
{
    struct drm_gem_object *obj = attach->dmabuf->priv;
    
    // Check P2P capability
    if (attach->peer2peer &&
        pci_p2pdma_distance(to_pci_dev(obj->dev->dev), attach->dev, false) < 0)
        attach->peer2peer = false;                         // Line 32
    
    // Check if we can migrate to TT memory for non-P2P
    if (!attach->peer2peer && !xe_bo_can_migrate(gem_to_xe_bo(obj), XE_PL_TT))
        return -EOPNOTSUPP;                                 // Line 35
    
    // Get PM reference
    xe_pm_runtime_get(to_xe_device(obj->dev));
    return 0;                                              // Line 38
}
```

**Pin Handler (xe_dma_buf.c:49-91):**
```c
static int xe_dma_buf_pin(struct dma_buf_attachment *attach)
{
    struct dma_buf *dmabuf = attach->dmabuf;
    struct drm_gem_object *obj = dmabuf->priv;
    struct xe_bo *bo = gem_to_xe_bo(obj);
    struct xe_device *xe = xe_bo_device(bo);
    struct drm_exec *exec = XE_VALIDATION_UNSUPPORTED;
    bool allow_vram = true;
    int ret;
    
    // Check DMABUF_MOVE_NOTIFY support
    if (!IS_ENABLED(CONFIG_DMABUF_MOVE_NOTIFY)) {
        allow_vram = false;                                 // Line 60
    } else {
        // Check all attachments for peer2peer support
        list_for_each_entry(attach, &dmabuf->attachments, node) {
            if (!attach->peer2peer) {
                allow_vram = false;
                break;                                      // Line 65
            }
        }
    }
    
    // Validate current state
    if (xe_bo_is_pinned(bo) && !xe_bo_is_mem_type(bo, XE_PL_TT) &&
        !(xe_bo_is_vram(bo) && allow_vram)) {
        drm_dbg(&xe->drm, "Can't migrate pinned bo for dma-buf pin.\n");
        return -EINVAL;                                     // Line 73
    }
    
    // Migrate to TT if necessary
    if (!allow_vram) {
        ret = xe_bo_migrate(bo, XE_PL_TT, NULL, exec);
        if (ret) {
            if (ret != -EINTR && ret != -ERESTARTSYS)
                drm_dbg(&xe->drm,
                        "Failed migrating dma-buf to TT memory: %pe\n",
                        ERR_PTR(ret));                      // Line 80
            return ret;
        }
    }
    
    // Pin for export
    ret = xe_bo_pin_external(bo, !allow_vram, exec);
    xe_assert(xe, !ret);
    
    return 0;
}
```

**Unpin Handler (xe_dma_buf.c:93-99):**
```c
static void xe_dma_buf_unpin(struct dma_buf_attachment *attach)
{
    struct drm_gem_object *obj = attach->dmabuf->priv;
    struct xe_bo *bo = gem_to_xe_bo(obj);
    
    xe_bo_unpin_external(bo);                              // Line 98
}
```

### 3.3 SG Table Mapping (Physical Addresses)

**Map Handler (xe_dma_buf.c:101-155):**
```c
static struct sg_table *xe_dma_buf_map(struct dma_buf_attachment *attach,
                                       enum dma_data_direction dir)
{
    struct dma_buf *dma_buf = attach->dmabuf;
    struct drm_gem_object *obj = dma_buf->priv;
    struct xe_bo *bo = gem_to_xe_bo(obj);
    struct drm_exec *exec = XE_VALIDATION_UNSUPPORTED;
    struct sg_table *sgt;
    int r = 0;
    
    if (!attach->peer2peer && !xe_bo_can_migrate(bo, XE_PL_TT))
        return ERR_PTR(-EOPNOTSUPP);                        // Line 112
    
    // Validate BO
    if (!xe_bo_is_pinned(bo)) {
        if (!attach->peer2peer)
            r = xe_bo_migrate(bo, XE_PL_TT, NULL, exec);    // Line 116
        else
            r = xe_bo_validate(bo, NULL, false, exec);      // Line 118
        if (r)
            return ERR_PTR(r);
    }
    
    // Create SG table based on memory type
    switch (bo->ttm.resource->mem_type) {
    case XE_PL_TT:
        // System memory: use standard page SG table
        sgt = drm_prime_pages_to_sg(obj->dev,
                                    bo->ttm.ttm->pages,
                                    obj->size >> PAGE_SHIFT);     // Line 125
        if (IS_ERR(sgt))
            return sgt;
        
        // DMA map for target device
        if (dma_map_sgtable(attach->dev, sgt, dir,
                            DMA_ATTR_SKIP_CPU_SYNC))
            goto error_free;                                // Line 132
        break;
        
    case XE_PL_VRAM0:
    case XE_PL_VRAM1:
        // VRAM: use VRAM manager's SG table
        r = xe_ttm_vram_mgr_alloc_sgt(xe_bo_device(bo),
                                      bo->ttm.resource, 0,
                                      bo->ttm.base.size, attach->dev,
                                      dir, &sgt);              // Line 138
        if (r)
            return ERR_PTR(r);
        break;
        
    default:
        return ERR_PTR(-EINVAL);
    }
    
    return sgt;
    
error_free:
    sg_free_table(sgt);
    kfree(sgt);
    return ERR_PTR(-EBUSY);
}
```

**Unmap Handler (xe_dma_buf.c:157-168):**
```c
static void xe_dma_buf_unmap(struct dma_buf_attachment *attach,
                             struct sg_table *sgt,
                             enum dma_data_direction dir)
{
    if (sg_page(sgt->sgl)) {
        // System memory SG table
        dma_unmap_sgtable(attach->dev, sgt, dir, 0);       // Line 162
        sg_free_table(sgt);
        kfree(sgt);
    } else {
        // VRAM SG table
        xe_ttm_vram_mgr_free_sgt(attach->dev, dir, sgt);   // Line 166
    }
}
```

### 3.4 Prime Handshake Protocol

**Import Flow (xe_dma_buf.c:306-365):**
```c
struct drm_gem_object *xe_gem_prime_import(struct drm_device *dev,
                                           struct dma_buf *dma_buf)
{
    const struct dma_buf_attach_ops *attach_ops;
    struct dma_buf_attachment *attach;
    struct drm_gem_object *obj;
    struct xe_bo *bo;
    
    // Check if it's our own BO
    if (dma_buf->ops == &xe_dmabuf_ops) {
        obj = dma_buf->priv;
        if (obj->dev == dev) {
            // Re-importing our own BO: just increment GEM refcount
            drm_gem_object_get(obj);                        // Line 325
            return obj;
        }
    }
    
    // Create placeholder BO for import
    bo = xe_bo_alloc();
    if (IS_ERR(bo))
        return ERR_CAST(bo);                                // Line 337
    
    // Setup attachment ops
    attach_ops = &xe_dma_buf_attach_ops;                   // Line 339
    
    // Dynamic attach: requires bo's drm_gem_object for locking
    attach = dma_buf_dynamic_attach(dma_buf, dev->dev, attach_ops, 
                                   &bo->ttm.base);         // Line 345
    if (IS_ERR(attach)) {
        obj = ERR_CAST(attach);
        goto out_err;
    }
    
    // Initialize imported BO
    obj = xe_dma_buf_init_obj(dev, bo, dma_buf);           // Line 352
    if (IS_ERR(obj))
        return obj;
    
    // Mark as imported
    get_dma_buf(dma_buf);
    obj->import_attach = attach;                           // Line 358
    return obj;
    
out_err:
    xe_bo_free(bo);
    return obj;
}
```

**Import BO Initialization (xe_dma_buf.c:242-277):**
```c
static struct drm_gem_object *
xe_dma_buf_init_obj(struct drm_device *dev, struct xe_bo *storage,
                    struct dma_buf *dma_buf)
{
    struct dma_resv *resv = dma_buf->resv;                 // Line 245
    struct xe_device *xe = to_xe_device(dev);
    struct xe_validation_ctx ctx;
    struct drm_gem_object *dummy_obj;
    struct drm_exec exec;
    struct xe_bo *bo;
    int ret = 0;
    
    // Allocate dummy object for resv locking
    dummy_obj = drm_gpuvm_resv_object_alloc(&xe->drm);
    if (!dummy_obj)
        return ERR_PTR(-ENOMEM);                            // Line 255
    
    // Share dma_buf's reservation object
    dummy_obj->resv = resv;                                // Line 257
    
    // Validate and initialize BO
    xe_validation_guard(&ctx, &xe->val, &exec, (struct xe_val_flags) {}, ret) {
        ret = drm_exec_lock_obj(&exec, dummy_obj);
        drm_exec_retry_on_contention(&exec);
        if (ret)
            break;
        
        // Initialize BO as SG buffer
        bo = xe_bo_init_locked(xe, storage, NULL, resv, NULL, 
                              dma_buf->size,
                              0,  // Will require bind
                              ttm_bo_type_sg,
                              XE_BO_FLAG_SYSTEM, &exec);    // Line 264
        drm_exec_retry_on_contention(&exec);
        if (IS_ERR(bo)) {
            ret = PTR_ERR(bo);
            xe_validation_retry_on_oom(&ctx, &ret);
            break;
        }
    }
    drm_gem_object_put(dummy_obj);
    
    return ret ? ERR_PTR(ret) : &bo->ttm.base;
}
```

**Move Notification Handler (xe_dma_buf.c:279-291):**
```c
static void xe_dma_buf_move_notify(struct dma_buf_attachment *attach)
{
    struct drm_gem_object *obj = attach->importer_priv;
    struct xe_bo *bo = gem_to_xe_bo(obj);
    struct drm_exec *exec = XE_VALIDATION_UNSUPPORTED;
    
    // Evict from VRAM when other device needs exclusive access
    XE_WARN_ON(xe_bo_evict(bo, exec));                    // Line 285
}

static const struct dma_buf_attach_ops xe_dma_buf_attach_ops = {
    .allow_peer2peer = true,
    .move_notify = xe_dma_buf_move_notify                 // Line 290
};
```

### 3.5 Coherency Implications

**CPU Access Handling (xe_dma_buf.c:170-198):**
```c
static int xe_dma_buf_begin_cpu_access(struct dma_buf *dma_buf,
                                       enum dma_data_direction direction)
{
    struct drm_gem_object *obj = dma_buf->priv;
    struct xe_bo *bo = gem_to_xe_bo(obj);
    bool reads = (direction == DMA_BIDIRECTIONAL ||
                  direction == DMA_FROM_DEVICE);            // Line 175
    struct xe_validation_ctx ctx;
    struct drm_exec exec;
    int ret = 0;
    
    // Only need to migrate for reads
    if (!reads)
        return 0;                                           // Line 182
    
    // Migrate to system memory for CPU access
    xe_validation_guard(&ctx, &xe_bo_device(bo)->val, &exec, 
                       (struct xe_val_flags) {}, ret) {
        ret = drm_exec_lock_obj(&exec, &bo->ttm.base);
        drm_exec_retry_on_contention(&exec);
        if (ret)
            break;
        
        // Force migration to TT (system memory)
        ret = xe_bo_migrate(bo, XE_PL_TT, NULL, &exec);    // Line 191
        drm_exec_retry_on_contention(&exec);
        xe_validation_retry_on_oom(&ctx, &ret);
    }
    
    // If migration failed, allow CPU access at current location
    return 0;                                              // Line 197
}
```

**Coherency Matrix:**

| Scenario | GPU | Other Device | CPU | Result |
|----------|-----|--------------|-----|--------|
| GPU writes, CPU reads | VRAM | - | System | Needs migrate to TT, cache flush |
| CPU writes, GPU reads | System | - | TT | Needs cache invalidation |
| GPU writes, Other reads | VRAM | System | - | Needs migrate to TT, DMA coherent |
| P2P access | VRAM | P2P capable | - | Direct VRAM access |

### 3.6 When to Use DMA-buf vs Alternatives

**DMA-buf Strengths:**
- Zero-copy between GPU and other devices
- Standardized interface (DMA-buf framework)
- Shared reservation objects for serialization
- Works with VRAM and system memory

**DMA-buf Use Cases:**
```
Video Decode (GPU) -> Video Encode (Other device)
  Use DMA-buf

ML Model (GPU) -> CPU Processing
  Use DMA-buf with migration

NVIDIA GPU -> Intel Arc Export
  Use DMA-buf P2P

DMA device -> GPU
  Use DMA-buf for zero-copy
```

**Alternatives:**

| Alternative | Pros | Cons | When to use |
|-------------|------|------|------------|
| **GPU Memory Map** | Simple, direct control | No sharing outside GPU | Single GPU only |
| **System Memory** | Works everywhere | Copy overhead, cache flush issues | Legacy drivers |
| **Userptr** | Arbitrary user mem | Page pinning issues, TLB misses | Temporary mappings |
| **P2P Direct** | Very fast | Needs P2P support, GPU VRAM only | Supported hardware |

**Code Example: DMA-buf Usage**
```c
// Export GPU memory
struct drm_xe_gem_create create = {
    .size = 1024 * 1024,  // 1MB
    .flags = 0
};
ioctl(fd, DRM_IOCTL_XE_GEM_CREATE, &create);
uint32_t bo_handle = create.handle;

// Export as DMA-buf
int dma_buf_fd = ioctl(fd, DRM_IOCTL_PRIME_HANDLE_TO_FD,
                      &(struct drm_prime_handle){
                          .handle = bo_handle,
                          .flags = DRM_CLOEXEC
                      });

// Share with another device
// (e.g., pass fd to video encoder process)

// Import on other device
struct drm_prime_handle import = {
    .fd = dma_buf_fd,
    .flags = DRM_CLOEXEC
};
int other_handle = ioctl(other_fd, DRM_IOCTL_PRIME_FD_TO_HANDLE, &import);

// Now both devices can access same memory without copying
```

---

## Summary Table

| Topic | Key Files | Line Ranges | Key Data Structures | Key Functions |
|-------|-----------|------------|---------------------|----------------|
| **Page Invalidation** | xe_userptr.c, xe_userptr.h, xe_vm_types.h | 25-322 (userptr.c) | xe_userptr, xe_userptr_vma, xe_userptr_vm | vma_userptr_invalidate, __vma_userptr_invalidate, xe_vm_userptr_pin |
| **CPU-GPU Sync** | xe_hw_fence.c/h, xe_sched_job.c/h, xe_sync.c | 21-268 (fence.c), 96-359 (sched_job.c), 22-406 (sync.c) | xe_hw_fence, xe_sched_job, xe_user_fence | xe_hw_fence_enable_signaling, xe_sched_job_arm, user_fence_worker |
| **DMA-buf** | xe_dma_buf.c/h | 25-369 | xe_bo, dma_buf attachment | xe_gem_prime_export/import, xe_dma_buf_map, xe_dma_buf_pin |

---

## Performance Considerations

### Page Invalidation Latency
- Invalidation callback: < 1 ms (must be fast)
- GPU wait for completion: Depends on GPU load
- Rebind: ~100-500 microseconds per page

### Synchronization Latency
- Hardware fence signaling: < 10 microseconds
- User fence callback: 50-500 microseconds
- Syncobj wait: Depends on polling interval

### DMA-buf Overhead
- SG table creation: ~10-100 microseconds
- Device attach/pin: 1-10 milliseconds
- Memory migration: Depends on size and bandwidth

