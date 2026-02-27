# GPU Reset & Error Recovery

## Overview

This document details GPU error handling, reset mechanisms, and recovery procedures in the Intel Xe driver.

**Target Audience**: Error recovery engineers, firmware integration specialists  
**Complexity**: VERY HIGH  
**Key Files**:
- `xe_hw_error.c` - Hardware error categorization
- `xe_guc_capture.c` - Error state capture
- `xe_devcoredump.c` - Coredump generation

## 1. Error Classification

### Hardware Error Types

```c
enum xe_hw_error_class {
    XE_HW_ERROR_CORRECTABLE = 0,    // Single-bit errors, can ignore
    XE_HW_ERROR_NONFATAL = 1,       // Recoverable, requires GT reset
    XE_HW_ERROR_FATAL = 2,          // Unrecoverable, device wedged
};

// Examples by class:

CORRECTABLE:
  - Single-bit ECC error (memory)
  - Instruction cache parity error
  - → Action: Log, continue execution

NONFATAL:
  - GPU hang (watchdog timeout)
  - Unhandled page fault
  - Engine stall (instruction stream hang)
  → Action: Reset GT, retry jobs

FATAL:
  - Double-bit ECC error
  - FIFO overflow
  - Firmware crash
  → Action: Mark device wedged, trigger full recovery
```

### Error Sources

```
Hardware Error Detection:
│
├─ Hardware Watchdog
│  ├─ Timeout on engine execution
│  └─ Signals if job takes too long
│
├─ Firmware Detection (GuC)
│  ├─ Instruction stream validation
│  ├─ Memory access violations
│  └─ Priority inversion detection
│
├─ ECC/Parity Checking
│  ├─ Memory error detection
│  ├─ Instruction cache parity
│  └─ Bus parity errors
│
└─ Manual Injection (for testing)
   └─ Fault injection via sysfs
```

## 2. GT Reset Sequence

### Reset Levels

```
Level 1: Soft Reset (per-engine)
├─ Reset one engine without affecting others
├─ Fastest recovery
├─ Used for single-engine hangs

Level 2: GT Reset (per-GT)
├─ Reset entire GT (all engines on that GT)
├─ Medium recovery time
├─ Used when multiple engines hang

Level 3: Tile Reset
├─ Reset entire tile (all GTs)
├─ Slower, more complete recovery
├─ Used for tile-wide failures

Level 4: Full Device Reset
├─ Reset entire device
├─ Slowest, last resort
├─ Used for critical failures
```

### GT Reset Implementation

```c
int xe_gt_reset(struct xe_gt *gt)
{
    struct xe_device *xe = gt->xe;
    int ret;
    
    dev_warn(&xe->pdev->dev,
            "GT reset initiated for GT %u\n", gt->info.id);
    
    // 1. Abort pending jobs on this GT
    abort_all_jobs_on_gt(gt);
    
    // 2. Disable GT operations
    xe_force_wake_get(gt, XE_FORCEWAKE_ALL);
    
    // 3. Hold reset
    xe_mmio_write32(gt, GT_RESET_REG,
                   GT_RESET_ASSERT);
    
    // 4. Wait for reset complete
    ret = xe_mmio_wait32(gt, GT_STATUS_REG,
                        GT_RESET_COMPLETE,
                        GT_RESET_COMPLETE,
                        100); // 100ms timeout
    if (ret) {
        dev_err(&xe->pdev->dev,
               "GT reset timeout\n");
        return ret;
    }
    
    // 5. Release reset
    xe_mmio_write32(gt, GT_RESET_REG, 0);
    
    // 6. Wait for GT ready
    ret = xe_mmio_wait32(gt, GT_STATUS_REG,
                        GT_READY,
                        GT_READY,
                        100);
    if (ret) {
        dev_err(&xe->pdev->dev,
               "GT failed to come ready after reset\n");
        return ret;
    }
    
    // 7. Reinitialize GT
    ret = xe_gt_reinit(gt);
    if (ret)
        return ret;
    
    // 8. Release force-wake
    xe_force_wake_put(gt, XE_FORCEWAKE_ALL);
    
    dev_info(&xe->pdev->dev,
            "GT reset complete\n");
    
    return 0;
}
```

### Job Abort Process

```c
void abort_all_jobs_on_gt(struct xe_gt *gt)
{
    // 1. Identify all jobs running/queued on this GT
    for (each exec_queue on gt) {
        // 2. Abort GPU scheduler
        drm_sched_stop(&eq->gpu_sched, NULL);
        
        // 3. Mark jobs as failed
        for (each job in eq->job_list) {
            dma_fence_set_error(&job->fence, -EIO);
            dma_fence_signal(&job->fence);
        }
        
        // 4. Reset scheduler (allows new jobs)
        drm_sched_start(&eq->gpu_sched);
    }
    
    // 5. Notify userspace
    send_signal_to_affected_clients(SIGIO);
}
```

## 3. Wedged Device State

When device cannot be recovered, mark as "wedged" (broken):

```c
enum xe_device_wedge_mode {
    XE_WEDGE_NEVER = 0,      // Never wedge
    XE_WEDGE_ON_FAILURE = 1, // Default: wedge on unrecoverable error
    XE_WEDGE_ALWAYS = 2,     // Always wedge (for testing)
};

void xe_device_declare_wedged(struct xe_device *xe)
{
    atomic_set(&xe->wedged.flag, 1);
    
    // Prevent new job submissions
    // All ioctl submissions will be rejected
    
    // Notify userspace via sysfs
    sysfs_notify(&xe->pdev->dev.kobj, NULL, "wedged");
}

bool xe_device_is_wedged(struct xe_device *xe)
{
    return atomic_read(&xe->wedged.flag);
}

// In IOCTL path:
int xe_exec(struct xe_file *xef, ...)
{
    if (xe_device_is_wedged(xe)) {
        return -EIO;  // "Device not operational"
    }
    // ... proceed with execution
}
```

## 4. GuC Error Handling

### GuC Error Types

```c
enum guc_error_type {
    GUC_ERROR_NONE = 0,
    GUC_ERROR_EXCEEDED_LIP = 1,        // Lost interrupt packet
    GUC_ERROR_MEMORY_ACCESS_FAULT = 2, // GuC memory fault
    GUC_ERROR_PREEMPT_TIMEOUT = 3,     // Preemption timeout
    GUC_ERROR_RESPONSE_TIMEOUT = 4,    // Response timeout
    GUC_ERROR_ENGINE_TIMEOUT = 5,      // Engine timeout
};
```

### GuC Reset Procedure

```c
int xe_guc_reset(struct xe_guc *guc)
{
    struct xe_gt *gt = guc_to_gt(guc);
    int ret;
    
    dev_warn(&xe->pdev->dev,
            "GuC reset initiated\n");
    
    // 1. Stop GuC
    xe_guc_stop(guc);
    
    // 2. Abort all GuC submissions
    for (each execution context) {
        // Tell GuC to deregister context
        send_guc_action(GUC_ACTION_DEREGISTER_CONTEXT);
    }
    
    // 3. Reset GT (contains GuC)
    ret = xe_gt_reset(gt);
    if (ret)
        return ret;
    
    // 4. Reload GuC firmware
    ret = xe_guc_post_load_init(guc, gt);
    if (ret) {
        dev_err(&xe->pdev->dev,
               "Failed to reinit GuC\n");
        return ret;
    }
    
    // 5. Re-register execution contexts
    for (each execution context) {
        send_guc_action(GUC_ACTION_REGISTER_CONTEXT);
    }
    
    dev_info(&xe->pdev->dev,
            "GuC reset complete\n");
    
    return 0;
}
```

## 5. Error State Capture (Devcoredump)

### What Gets Captured

```c
struct xe_devcoredump {
    // Snapshot parameters
    u32 timestamp;          // When captured
    u32 guc_timestamp;      // GuC time
    
    // Hardware state
    struct {
        u32 *regs;          // Register snapshot (100KB+)
        struct xe_lrc *contexts;  // Context state
        struct {
            u64 page_dir;
            u32 *page_tables;
        } page_tables;
    } hw;
    
    // GuC state
    struct {
        u8 *log_buf;        // GuC firmware log (MB+)
        struct xe_guc_tlb_inval_fence *pending_tlb;
        u32 engine_activity[num_engines];
    } guc;
    
    // Job state
    struct {
        struct xe_sched_job *hung_job;
        u32 job_queue_snapshot[num_queues];
    } jobs;
    
    // Configuration
    struct xe_device_config *config;
};
```

### Coredump Capture Flow

```c
void xe_devcoredump_capture(struct xe_device *xe,
                           const char *reason)
{
    struct xe_devcoredump *coredump;
    
    // Only capture if not already capturing
    if (atomic_cmpxchg(&xe->coredump.lock, 0, 1))
        return;
    
    coredump = kzalloc(sizeof(*coredump));
    
    // 1. Timestamp
    coredump->timestamp = jiffies;
    
    // 2. Capture register state
    for_each_gt(gt, xe, id) {
        capture_gt_registers(gt, &coredump->hw);
    }
    
    // 3. Capture GuC logs
    xe_guc_log_capture(&xe->guc, &coredump->guc);
    
    // 4. Capture page table state
    capture_page_tables(xe, &coredump->hw);
    
    // 5. Capture job state
    capture_job_queue_state(xe, &coredump->jobs);
    
    // 6. Initiate devcoredump (writes to /dev/devcoredump)
    dev_coredumpm(&xe->pdev->dev,
                 NULL,
                 coredump,
                 &xe_devcoredump_ops,
                 GFP_KERNEL);
    
    dev_info(&xe->pdev->dev,
            "Coredump captured: %s\n", reason);
}
```

### Coredump Access

```bash
# On GPU hang/error, /dev/devcoredump appears:
ls -la /dev/devcoredump

# Read coredump
cat /dev/devcoredump > xe_coredump.bin

# Analyze with custom tools
xe_devcoredump_dump xe_coredump.bin

# Clear after reading
echo 'N' > /sys/class/devcoredump/devcoredump0/delete
```

## 6. Recovery Modes

### Auto Recovery

```c
// Default: Attempt recovery on error
int xe_device_wedge_mode = XE_WEDGE_ON_FAILURE;

// On error:
// 1. Try soft reset (per-engine)
// 2. Try GT reset
// 3. Try tile reset
// 4. Give up, mark wedged
```

### Never Wedge (Debugging)

```bash
# Module parameter for development
modprobe xe wedge_mode=0  # XE_WEDGE_NEVER

# Driver will always attempt recovery
# Infinite recovery loop possible in severe cases
```

### Always Wedge (Testing)

```bash
# Force wedge on any error (for testing)
modprobe xe wedge_mode=2  # XE_WEDGE_ALWAYS

# Any error → device marked wedged
# Good for testing error paths
```

## 7. Firmware Collaboration (CSME)

### Execution Submission & Reporting (ESR)

GuC communicates with CSME (Intel firmware) about errors:

```
Error Flow with Firmware:
│
├─ GPU error occurs
├─ GuC firmware detects it
├─ GuC reports to CSME via message
├─ CSME logs in security enclave
├─ CSME may request full device reset
└─ Driver enforces CSME request
```

### CSME Reset Request

```c
void handle_csme_reset_request(struct xe_device *xe)
{
    dev_warn(&xe->pdev->dev,
            "CSME requested device reset\n");
    
    // CSME has higher authority
    // Must comply with reset request
    
    // 1. Shutdown GuC cleanly
    xe_guc_stop(&xe->guc);
    
    // 2. Perform device reset
    xe_pci_reset(xe);
    
    // 3. Re-initialize
    xe_device_probe(xe);
}
```

## 8. Common Recovery Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Reset timeout | GT not responding | Check power state, force full device reset |
| Reset loop | Recurring error | Wedge device, capture coredump |
| Hung GT | Lockup in reset code | Watchdog timeout, system reset |
| Lost context | Memory corruption | Reinitialize from saved state |

## 9. Testing Error Recovery

### Fault Injection Interface

```bash
# Inject single-bit ECC error
echo "inject_ecc_correctable" > \
  /sys/kernel/debug/dri/0/hw_error

# Inject nonfatal error (triggers reset)
echo "inject_engine_timeout" > \
  /sys/kernel/debug/dri/0/hw_error

# Inject fatal error (wedges device)
echo "inject_fatal_error" > \
  /sys/kernel/debug/dri/0/hw_error

# Verify recovery
cat /sys/class/drm/card0/device/wedged
# 0 = not wedged, device operational
# 1 = wedged, device not operational
```

## Summary

GPU error recovery involves:
1. **Detection** - Hardware, GuC, or manual injection
2. **Classification** - Determine severity (correctable/nonfatal/fatal)
3. **Capture** - Coredump for post-mortem analysis
4. **Recovery** - Reset appropriate level (engine/GT/tile/device)
5. **Restart** - Resume job execution or wedge device

Critical considerations:
- Fast recovery for user-perceived latency
- Complete state capture for debugging
- Firmware coordination for security
- Graceful degradation to wedged state

---

**See Also**:
- [07-interrupt-handling.md](07-interrupt-handling.md) - Error interrupt handling
- [02-guc-integration.md](02-guc-integration.md) - GuC firmware
- [10-potential-issues.md](10-potential-issues.md) - Known issues
