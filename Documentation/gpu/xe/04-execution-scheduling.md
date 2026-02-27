# Execution & Scheduling

## Overview

This document details the GPU job execution model and scheduling infrastructure in the Intel Xe driver.

**Target Audience**: GPU scheduler developers, job submission engineers  
**Key Files**:
- `xe_exec_queue.c/h` - Execution queues
- `xe_gpu_scheduler.c/h` - GPU scheduler
- `xe_guc_submit.c/h` - GuC-based submission
- `xe_sched_job.c/h` - Scheduler jobs
- `xe_lrc.c/h` - Logical ring context

## 1. Execution Model Overview

### Components

```
┌──────────────────────────────────────┐
│  User Application (DRM Client)       │
└──────────┬───────────────────────────┘
           │
           v
┌──────────────────────────────────────┐
│  Execution Queue                     │
│  (Per-engine-class submission)       │
└──────────┬───────────────────────────┘
           │
           v
┌──────────────────────────────────────┐
│  GPU Scheduler                       │
│  (DRM GPU Scheduler)                 │
└──────────┬───────────────────────────┘
           │
           v
┌──────────────────────────────────────┐
│  GuC Submission                      │
│  (Batch buffer submission)           │
└──────────┬───────────────────────────┘
           │
           v
┌──────────────────────────────────────┐
│  Hardware Engine                     │
│  (RCS, BCS, VCS, VECS, CCS)          │
└──────────┬───────────────────────────┘
           │
           v
         GPU
```

### Key Concepts

**Execution Queue (Exec Queue)**
- User-created or kernel-internal
- Represents submission capability to specific engine class
- Each queue has associated contexts (xe_lrc)
- Jobs submitted to queues run serially

**Scheduler Job**
- Represents one unit of work (batch buffer)
- Can depend on other jobs (fence dependencies)
- Tracked by GPU scheduler
- Submitted to GuC for execution

**Logical Ring Context (LRC)**
- GPU context state for one engine
- Contains:
  - Engine registers
  - Command ring pointers
  - Context save/restore data
- Per-exec-queue

**Hardware Engines**
- RCS (Render/3D) - Graphics and compute
- BCS (Blitter) - Memory copy engine
- VCS (Video) - Video encode/decode
- VECS (Video Enhancement) - Video effects
- CCS (Compute) - Dedicated compute

## 2. Execution Queue Lifecycle

### Creation

```c
// User creates exec queue
struct xe_exec_queue *xe_exec_queue_create(
    struct xe_vm *vm,
    u32 engine_class,
    u32 priority)
{
    struct xe_exec_queue *q;
    
    // 1. Allocate exec queue
    q = kzalloc(sizeof(*q));
    
    // 2. Set parameters
    q->vm = vm;
    q->engine_class = engine_class;
    q->priority = priority;
    
    // 3. Allocate context (LRC)
    q->lrc = xe_lrc_create(vm, engine_class);
    
    // 4. Register with GuC (if not user-created)
    if (!user_queue)
        xe_guc_context_register(guc, q->lrc);
    
    // 5. Initialize scheduler entity
    drm_sched_entity_init(&q->entity, ...);
    
    return q;
}
```

### Usage

```c
// User submits jobs to queue
user_submits_job(exec_queue, batch_buffer, buffers, ...);
    └─ Driver creates job
    └─ Associates with exec queue
    └─ Submits to GPU scheduler
```

### Destruction

```c
// When exec queue no longer needed
xe_exec_queue_destroy(q);
    └─ Flush remaining jobs
    └─ Deregister from GuC
    └─ Free context
    └─ Free exec queue
```

## 3. Scheduler Job Lifecycle

### Job Creation

```c
// User submits work (DRM_XE_EXEC ioctl)
struct xe_sched_job *xe_sched_job_create(
    struct xe_exec_queue *q,
    struct xe_exec_queue_binding *bindings[],
    struct xe_sync_entry *syncs[])
{
    struct xe_sched_job *job;
    
    // 1. Allocate job structure
    job = kzalloc(sizeof(*job));
    
    // 2. Set job parameters
    job->q = q;
    job->lrc = q->lrc;
    job->batch_addr = batch_addr;
    job->batch_len = batch_len;
    
    // 3. Add fence dependencies
    for (each sync)
        add_fence_dependency(job, sync->fence);
    
    // 4. Prepare memory (lock buffers, validate addresses)
    xe_exec_queue_prepare_memops(q, bindings);
    
    // 5. Create GPU fence (will signal on completion)
    job->gpu_fence = xe_hw_fence_create(job);
    
    return job;
}
```

### Job Submission

```c
// Scheduler runs job when ready
void xe_sched_job_push(struct xe_sched_job *job)
{
    // 1. Arm job (prepare for execution)
    drm_sched_job_arm(&job->base);
    
    // 2. Push to entity queue
    drm_sched_entity_push_job(&job->base, &job->q->entity);
    
    // GPU Scheduler will:
    // - Monitor dependencies
    // - Call job->ops->run() when ready
    // - Which calls xe_guc_submit(job)
}
```

### Job Execution

```c
// Called by GPU scheduler
int xe_sched_job_run(struct drm_sched_job *sched_job)
{
    struct xe_sched_job *job = to_xe_sched_job(sched_job);
    struct xe_guc *guc = guc_from_job(job);
    
    // 1. Submit to GuC
    int ret = xe_guc_submit(guc, job);
    if (ret)
        return ret;
    
    // 2. Wait for hardware to acknowledge
    // 3. Job now executing on GPU
    
    return 0;
}
```

### Job Completion

```c
// Hardware signals completion
interrupt_handler():
    └─ Identifies completed job
    └─ Signals job's fence
    └─ GPU scheduler is notified

xe_sched_job_done(job):
    ├─ Release locked buffers
    ├─ Free GPU fence
    ├─ Update timing statistics
    └─ Free job structure
```

## 4. GPU Scheduler Integration

The Xe driver uses the Linux DRM GPU Scheduler (drm_gpu_scheduler).

### Scheduler Features

- **Priority Levels**: Multiple job priorities for scheduling
- **Fence Dependencies**: Jobs can wait for other jobs
- **Timeout Detection**: Hangs detected and recovered
- **Load Balancing**: Distribute jobs across engines
- **Workqueue Integration**: Uses Linux workqueues

### Scheduler Architecture

```
Per-Exec-Queue:
┌─────────────────────────────────┐
│ Scheduler Entity                │
│  - Job queue                    │
│  - Priority level               │
│  - Dependency tracking          │
└──────────┬──────────────────────┘
           │
           v
Scheduler Runqueue (selects which job to run)
           │
           ├─ Pick job from highest priority entity
           ├─ Check dependencies satisfied
           ├─ Call job->ops->run()
           └─ Monitor for timeout
```

## 5. Priority & Preemption

### Priority Levels

```c
enum xe_sched_priority {
    XE_SCHED_PRIORITY_LOW = 0,      // Background work
    XE_SCHED_PRIORITY_NORMAL = 1,   // Default user work
    XE_SCHED_PRIORITY_HIGH = 2,     // Important user work
    XE_SCHED_PRIORITY_KERNEL = 3,   // System critical
};
```

### Priority Scheduling

```
Scheduler prioritizes jobs:

HIGH         ████████ 80% of bandwidth
NORMAL       ████     15% of bandwidth  
LOW          ██       5% of bandwidth
KERNEL      Used only when needed
```

### Preemption

When high-priority job arrives:

```c
// Current job executing
execute_low_priority_job()

// High priority arrives
preempt_current_job():
    ├─ Save current context
    ├─ Load new context
    ├─ Execute high priority job
    ├─ Restore previous context
    └─ Resume previous job
```

## 6. Synchronization & Dependencies

### Fence Types

**Hardware Fence (Seqno)**
- GPU writes completion value to memory
- Drivers polls or waits on interrupt

**Software Fence**
- DMA fence (Linux fence infrastructure)
- Signaled in software after GPU completes

### Dependency Tracking

```c
struct xe_sync_entry {
    enum {
        XE_SYNC_SYNCOBJ,        // Sync object (Vulkan-style)
        XE_SYNC_TIMELINE_SYNCOBJ, // Timeline sync object
        XE_SYNC_DMA_BUF,        // DMA buffer fence
        XE_SYNC_USER_FENCE      // User-supplied fence
    } type;
    
    union {
        struct drm_syncobj *syncobj;
        struct dma_fence *fence;
        u64 user_fence_addr;    // User-space fence address
    };
};
```

## 7. Context & Batch Management

### Logical Ring Context (LRC)

```c
struct xe_lrc {
    // Context state
    struct xe_bo *bo;           // BO holding context save area
    
    // Ring buffer
    struct xe_ring *ring;       // Command ring
    u32 ring_head;              // Where GPU read from
    u32 ring_tail;              // Where we write to
    
    // GuC interface
    u32 guc_id;                 // GuC's ID for this context
    bool guc_registered;        // Registered with GuC?
    
    // Timestamps
    u64 timestamp;              // Context switch timestamp
};
```

### Batch Buffer Submission

```
Batch Buffer Format:
┌─────────────────┐
│  Header         │  (MI_BATCH_BUFFER_START command)
├─────────────────┤
│  GPU Commands   │  (3D rendering, compute, etc.)
├─────────────────┤
│  Fence Signal   │  (MI_FLUSH_DW, MI_STORE_DATA_IMM)
├─────────────────┤
│  Tail Padding   │  (Required by some HW)
└─────────────────┘

Driver fills batch with:
1. User commands (from UMD)
2. Memory validation (TLB invalidation if needed)
3. Completion signaling
```

## 8. Engine Classes & Discovery

### Engine Discovery

```c
void xe_hw_engine_class_init(struct xe_gt *gt)
{
    // Read ENGINE_CLASS_DISCOVERY_INFO
    // Determine which engines are present
    
    for each physical engine {
        enum xe_engine_class class = get_engine_class(engine);
        int instance = get_engine_instance(engine);
        
        // Create entry in gt->engines[]
        gt->engines[class][instance].class = class;
        gt->engines[class][instance].instance = instance;
    }
}
```

### Engine Capabilities

```
RCS (Render/3D):
  ├─ 3D graphics
  ├─ Compute (GPGPU)
  └─ Video decode (on some platforms)

BCS (Blitter):
  ├─ Memory copy
  ├─ Tile/untile
  └─ Compression operations

VCS (Video):
  ├─ Video encode
  ├─ Video decode
  └─ Transcoding

VECS (Video Enhancement):
  ├─ Deinterlacing
  ├─ Scaling
  └─ Color enhancement

CCS (Compute):
  ├─ Dedicated compute
  └─ RayTracing (on some platforms)
```

## 9. Common Submission Issues

### Issue: Batch Never Executes
**Causes**:
- Wrong exec queue class (submitted RCS job to BCS engine)
- Missing memory bindings (BO not mapped to VM)
- Incorrect priority (can't get GPU time)

**Debug**:
```bash
# Check exec queue
cat /sys/kernel/debug/dri/0/exec_queues

# Check job state
cat /sys/kernel/debug/dri/0/sched_jobs

# Check GuC logs
cat /sys/kernel/debug/dri/0/guc_logs
```

### Issue: Timeout/Job Hangs
**Causes**:
- Infinite loop in batch buffer
- Waiting on unavailable memory
- GPU page fault
- Deadlock in batch execution

**Debug**:
```bash
# Enable scheduler timeout logging
echo "1" > /proc/sys/kernel/sched_debug

# GPU hang detected - look for timeout messages
dmesg | grep -i "timeout\|hang"
```

## 10. Performance Considerations

### Job Submission Overhead
- Memory validation: ~1-10μs
- GuC doorbell write: ~1μs
- Scheduler overhead: ~10-100ns

### Scheduler Latency
- Low priority: Can wait seconds
- Normal priority: ~1-10ms typical
- High priority: <1ms typical
- Kernel priority: Immediate (if engine available)

### Optimization Tips
1. Batch jobs together (reduce submission overhead)
2. Use appropriate priorities
3. Avoid fine-grained synchronization
4. Prefer asynchronous submission

## Summary

The Xe execution model provides:
- **Flexible queuing** via execution queues
- **Priority scheduling** via GPU scheduler
- **Dependency management** via fences
- **Hardware abstraction** via GuC

Understanding this layer is critical for:
- Debugging job submission issues
- Understanding performance characteristics
- Implementing new scheduler features

---

**See Also**:
- [00-architecture-overview.md](00-architecture-overview.md) - Architecture context
- [02-guc-integration.md](02-guc-integration.md) - GuC submission details
- [03-memory-management.md](03-memory-management.md) - Memory in execution
