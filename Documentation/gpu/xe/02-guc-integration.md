# GuC (GPU Microcontroller) Integration

## Overview

The GPU Microcontroller (GuC) is a specialized firmware component that runs autonomously on Intel GPUs. It handles critical functions including job scheduling, power management, and cache operations, offloading these tasks from the CPU driver.

**Target Audience**: GPU driver developers, firmware engineers  
**Key Files**:
- `xe_guc*.c` (20+ files) - GuC implementation
- `xe_uc_fw.c` - Firmware loading
- `xe_guc_submit.c` - Job submission
- `xe_guc_ct.c` - Command transport

## 1. GuC Architecture & Role

### Historical Context

Early Intel GPU drivers (i915) submitted jobs directly to hardware without firmware assistance. Modern Intel GPUs employ GuC for:
- **Job scheduling**: GuC decides which job runs on which engine
- **Power management**: Dynamic frequency/voltage scaling
- **Virtualization**: Multiplexing resources among VMs
- **Security**: Enforcing isolation between jobs
- **Telemetry**: Collecting performance data

### GuC Firmware Components

```
GuC Firmware Binary (xe*.bin from /lib/firmware/xe/)
├─ GuC Code + Data Sections
├─ Initialization Data
├─ Instruction Sets
└─ Platform-Specific Parameters

Runs on: GPU Microcontroller (separate processor)
Memory: WOPCM (Where Out-of-band Processing is located)
Communication: Host to GuC / GuC to Host via CTB (Command Transport Buffer)
```

### Execution Model

```
User Submits Job
     |
     v
DRM Ioctl (DRM_XE_EXEC)
     |
     v
Driver Creates Job Structure (xe_sched_job)
     |
     v
GPU Scheduler Deems Job Ready
     |
     v
Driver: Submit to GuC (xe_guc_submit)
     |
     v
GuC Firmware:
  ├─ Schedule on appropriate engine
  ├─ Set up execution context
  └─ Notify hardware engine
     |
     v
Hardware Engine Executes Batch Buffer
     |
     v
Hardware Signals Completion (Interrupt)
     |
     v
Driver Processes Completion
```

## 2. GuC Memory Layout

### WOPCM (Where Out-of-band Processing is located)

WOPCM is a reserved GPU memory region for GuC:

```
GPU Memory (GFX Virtual Address Space)
│
├─ 0x00000000 ─────────────────────
│  GuC Firmware Code/Data (Downloaded from file)
│  Size varies by platform (256KB - 1MB+)
│
├─ GuC WOPCM Offset ────────────────
│  GuC Runtime Data
│  (Maintained by GuC firmware)
│
├─ WOPCM Top ──────────────────────
│  Available for other uses
│
└─ Driver-visible GGTT Area ────────
   GuC Shared Structures (mapped via GGTT)
   ├─ GuC Descriptor (ADS - Address Data Structures)
   ├─ GuC Log Buffer
   ├─ Scheduling Table (GPCCS)
   ├─ Command Transport Buffers (CTB)
   ├─ Doorbell Registers
   └─ Execution Queue State
```

**Key Concepts**:
- **WOPCM Size**: Region reserved for GuC (xe_wopcm_size())
- **ADS**: Shared memory block with device descriptor data
- **GGTT-accessible**: GuC structures above WOPCM that driver can access

### GuC Shared Memory Structures (ADS)

```c
struct xe_guc_ads {
    // Device descriptor
    struct guc_gt_system_info gt_system_info;
    
    // Memory layout info
    struct guc_addr_range wopcm;
    
    // Engine/execution context info
    struct guc_engine_usage_record engine_usage[num_engines];
    
    // Frequency table (for RPS)
    struct guc_gt_pwr_config pwr_config;
    
    // Logging info
    struct guc_log_buffer_state log_state;
    
    // Virtual scheduling info
    struct guc_virtual_engine_config veng_config;
    
    // And more platform-specific structures...
};
```

**Built during probe**: `xe_guc_ads_create()`  
**Updated at runtime**: Frequency changes, execution queue updates

## 3. GuC Firmware Lifecycle

### Phase 1: Firmware Discovery

**File**: `xe_uc_fw.c`

```c
int xe_uc_fw_init(struct xe_uc_fw * fw, enum xe_uc_fw_type type)
{
    // 1. Determine firmware filename based on platform
    //    e.g., "xe/xe2_guc_62.0.3.bin" for MTL GuC
    
    // 2. Request firmware from kernel fw subsystem
    ret = request_firmware(&fw->fw, fw_name, dev);
    
    // 3. Parse firmware header
    // - Check version
    // - Extract code/data sizes
    // - Validate signature
    
    // 4. Allocate memory for firmware
    fw->obj = xe_bo_create_from_data(xe, fw_data, size);
    
    return ret;
}
```

### Phase 2: GuC Initialization (Probe)

**File**: `xe_guc.c`

```c
int xe_guc_init(struct xe_guc *guc, struct xe_gt *gt)
{
    struct xe_device *xe = gt->xe;
    int ret;
    
    // 1. Get GuC base address in WOPCM
    guc->fw.type = XE_UC_FW_TYPE_GUC;
    ret = xe_uc_fw_init(&guc->fw, XE_UC_FW_TYPE_GUC);
    if (ret)
        return ret;
    
    // 2. Allocate GuC BO (buffer object) for firmware
    guc->fw.obj = xe_bo_create_from_data(xe, fw_data, fw_size);
    
    // 3. Create GuC shared memory structures
    ret = xe_guc_ads_create(guc);
    if (ret)
        return ret;
    
    // 4. Initialize Command Transport (bidirectional messaging)
    ret = xe_guc_ct_init(&guc->ct, guc);
    if (ret)
        return ret;
    
    // 5. Initialize doorbell manager
    ret = xe_guc_db_mgr_init(&guc->dbm, gt);
    if (ret)
        return ret;
    
    // 6. Initialize GuC logs
    ret = xe_guc_log_init(&guc->log, gt);
    if (ret)
        return ret;
    
    // 7. Create relay structures for SR-IOV
    if (IS_SRIOV_PF(xe)) {
        ret = xe_guc_relay_init(&guc->relay);
        if (ret)
            return ret;
    }
    
    // GuC still not running - firmware not loaded yet
    
    return 0;
}
```

### Phase 3: GuC Firmware Loading & Boot

**File**: `xe_guc.c`

```c
int xe_guc_post_load_init(struct xe_guc *guc, struct xe_gt *gt)
{
    struct xe_device *xe = gt->xe;
    int ret;
    
    // 1. Set GuC parameters before boot
    // - WOPCM addresses
    // - Shared memory locations
    // - Config flags
    xe_guc_params_init(guc);
    
    // 2. Write GuC firmware to GPU memory (WOPCM)
    ret = xe_uc_fw_upload(&guc->fw, gt);
    if (ret)
        return ret;
    
    // 3. Configure GuC
    ret = xe_guc_fw_init(guc);
    if (ret)
        return ret;
    
    // 4. Boot GuC microcontroller
    ret = xe_guc_force_enable(guc);  // Wake it up
    if (ret)
        return ret;
    
    ret = xe_guc_boot(guc);  // Start execution
    if (ret)
        return ret;
    
    // 5. Wait for GuC to indicate ready
    ret = xe_guc_wait_handshake(guc);
    if (ret)
        return ret;
    // At this point, GuC is running and responsive
    
    // 6. Send initialization commands
    ret = xe_guc_init_submission(guc);
    if (ret)
        return ret;
    
    // 7. Setup frequency ranges (RPS)
    ret = xe_guc_pc_init(guc);
    if (ret)
        return ret;
    
    // 8. Enable GuC features
    ret = xe_guc_enable_features(guc);
    if (ret)
        return ret;
    
    // GuC now fully operational
    
    return 0;
}
```

### Phase 4: Post-Boot Configuration

Once GuC is running, the driver sends configuration commands:

```c
int xe_guc_init_submission(struct xe_guc *guc)
{
    struct xe_gt *gt = guc_to_gt(guc);
    int ret;
    
    // 1. Register all available hardware engines with GuC
    for (int i = 0; i < num_engines; i++) {
        ret = guc_submit_engine_register(guc, &engines[i]);
        if (ret)
            return ret;
    }
    
    // 2. Setup priority levels
    ret = guc_setup_priority_levels(guc);
    if (ret)
        return ret;
    
    // 3. Enable GuC scheduler
    ret = guc_enable_scheduler(guc);
    if (ret)
        return ret;
    
    // 4. Setup preemption (context switching)
    ret = guc_setup_preemption(guc);
    if (ret)
        return ret;
    
    return 0;
}
```

## 4. Command Transport Layer (CTB)

The CTB provides bidirectional messaging between driver and GuC firmware.

### CTB Structure

```c
struct xe_guc_ct {
    // Host-to-GuC (H2G) channel
    struct xe_guc_ct_buffer h2g;
    
    // GuC-to-Host (G2H) channel
    struct xe_guc_ct_buffer g2h;
    
    // Locks and synchronization
    struct mutex lock;
    
    // Command timeout handling
    struct delayed_work g2h_timeout_work;
    
    // Channel state
    bool enabled;
    u32 version;
};

struct xe_guc_ct_buffer {
    struct xe_bo *bo;           // Buffer object
    struct guc_ct_buffer_desc *desc;  // Descriptor in GGTT
    
    u32 write_index;            // Write position
    u32 read_index;             // Read position (maintained by other side)
    
    u32 *cmds;                  // Command array
    u32 size;                   // Buffer size in u32s
};
```

### Message Format

```
Command Message Layout:
┌──────────────────────────────────────┐
│  Word 0: Data0                       │ ─┐
│         [31:28] HEADER_LEN           │  │
│         [27:24] HEADER_VERSION       │  ├─ Header (1+ words)
│         [23:16] MESSAGE_ORIGINATOR   │  │
│         [15:0]  MESSAGE_TYPE         │ ─┘
├──────────────────────────────────────┤
│  Word 1+: Additional Data (optional) │ ─┐
│          (payload varies by command) │  ├─ Payload (0+ words)
│                                      │ ─┘
└──────────────────────────────────────┘

Total message size: Flexible (1-N words)
```

### Command Types

```
GUC_ACTION_* commands (from xe_guc_fwif.h):
├─ GUC_ACTION_REQUEST_RESOURCE - Allocate doorbell
├─ GUC_ACTION_REGISTER_CONTEXT - Register exec queue with GuC
├─ GUC_ACTION_DEREGISTER_CONTEXT - Unregister exec queue
├─ GUC_ACTION_FLUSH_LOG - Request GuC to flush logs
├─ GUC_ACTION_PREEMPT_RATE - Set preemption rate
├─ GUC_ACTION_KLV_GENERIC_SET - Set key-value pair config
├─ GUC_ACTION_TLB_INVAL - Request TLB invalidation
└─ ... (20+ more actions)
```

### Message Flow Example: Register Context

```
Driver wants to submit jobs to a new execution queue:

1. Create execution queue context locally
2. Send GUC_ACTION_REGISTER_CONTEXT command
   Payload:
   ├─ Execution Queue ID
   ├─ Engine class
   ├─ Context descriptor GGTT address
   └─ Priority level

3. Wait for response (via interrupt or polling)

4. GuC response:
   ├─ Assigned doorbell ID
   ├─ Status (success/error)
   └─ Additional info

5. Driver now can submit to doorbell
```

**Related Files**:
- `xe_guc_ct.c` - CTB implementation
- `xe_guc_ct_types.h` - CTB structures
- `xe_guc_fwif.h` - GuC firmware interface

## 5. Job Submission via GuC

### Submission Flow

```
User DRM_XE_EXEC ioctl
    │
    v
xe_exec() [IOCTL handler]
    │
    v
Create Job (xe_sched_job_create)
    │
    v
GPU Scheduler deems ready
    │
    v
Call Job->ops->run() → xe_guc_submit()
    │
    v
xe_guc_submit():
    ├─ Ensure context registered with GuC
    ├─ Fill in batch buffer address
    ├─ Set batch length
    ├─ Set priority
    └─ Write DOORBELL register [GT offset + doorbell_id]
    │
    v
GuC firmware receives doorbell interrupt
    │
    v
GuC scheduler decides when to execute
    │
    v
GuC instructs hardware engine
    │
    v
Hardware engine fetches batch buffer
    │
    v
GPU executes batch buffer
    │
    v
Hardware completion interrupt
    │
    v
Driver completes job
```

### Submission Code

**File**: `xe_guc_submit.c`

```c
int xe_guc_submit(struct xe_guc *guc, struct xe_sched_job *job)
{
    struct xe_exec_queue *q = job->q;
    struct xe_lrc *lrc = job->lrc;
    int ret;
    
    // 1. Prepare batch buffer info
    u64 batch_addr = xe_bo_ggtt_addr(job->batch_bo);
    u32 batch_len = job->batch_len;
    u32 engine_class = q->engine_class;
    u32 priority = q->priority;
    
    // 2. Ensure context registered with GuC
    if (!lrc->guc_registered) {
        ret = xe_guc_context_register(guc, lrc, engine_class);
        if (ret)
            return ret;
        lrc->guc_registered = true;
    }
    
    // 3. Update context descriptor with batch info
    lrc->batch_addr = batch_addr;
    lrc->batch_len = batch_len;
    wmb();  // Ensure visible to GuC
    
    // 4. Get doorbell for this exec queue
    u32 db_id = lrc->guc_db_id;
    u32 db_offset = gt_to_guc(gt)->doorbell_offset + 
                    (db_id * sizeof(u32));
    
    // 5. Update doorbell value (triggers GuC)
    xe_mmio_write32(gt, db_offset, GUC_DOORBELL_ENABLED);
    
    // 6. GuC firmware receives this and starts processing
    
    return 0;
}
```

## 6. Power Management via GuC

### Render Performance State (RPS)

GuC manages frequency scaling through RPS (Render Performance State):

```c
struct xe_guc_pc {
    u8 cur_freq;      // Current frequency index
    u8 min_freq;      // Minimum frequency (user-settable)
    u8 max_freq;      // Maximum frequency (user-settable)
    u8 req_freq;      // Requested frequency (internal)
};
```

**Frequency Scaling Process**:

```
1. Driver sets frequency range (e.g., min=400MHz, max=1.5GHz)
   └─ Send GUC_ACTION_KLV_GENERIC_SET to GuC
      └─ GuC_FREQUENCY_MIN = 400MHz index
      └─ GuC_FREQUENCY_MAX = 1500MHz index

2. GuC monitors GPU load

3. Load increases
   └─ GuC increases frequency
   └─ More FLOPS available
   └─ Workload completes faster

4. Load decreases
   └─ GuC decreases frequency
   └─ Less power consumption
   └─ Temperature drops

5. Thermal throttling (if temps high)
   └─ GuC limits maximum frequency
```

### Related Files
- `xe_guc_pc.c` - Power conservation (RPS)
- `xe_gt_freq.c` - Frequency management API

## 7. GuC Logging & Debugging

### GuC Log Buffer

GuC firmware outputs debug messages to a circular log buffer:

```
GuC Log Buffer Layout:
┌─────────────────┐
│ Buffer Header   │  (Metadata)
├─────────────────┤
│ Flush Work      │
│ Items           │  (Circular buffer for messages)
│ ...             │
├─────────────────┤
│ Scratch Pages   │
└─────────────────┘

Size: Platform-dependent (256KB - 1MB)
Format: Binary with timestamp, level, tags
Access: Read via xe_guc_log_dump()
```

### Log Collection

```c
int xe_guc_log_dump(struct xe_guc *guc, struct seq_file *m)
{
    // 1. Copy log buffer from GuC memory
    char *log = kmalloc(log_size);
    memcpy(log, guc->log.obj->map.vaddr, log_size);
    
    // 2. Parse log entries
    for (each entry in log) {
        u32 timestamp = entry->timestamp;
        u8 level = entry->level;  // DEBUG, INFO, WARN, ERROR
        char *message = entry->message;
        
        seq_printf(m, "[%u] %s: %s\n", timestamp, level, message);
    }
    
    kfree(log);
    return 0;
}
```

**Access Methods**:
- Kernel logging (dmesg): `xe_guc_log_level_set()`
- Debugfs: `/sys/kernel/debug/dri/*/guc_logs`
- Device coredump: On GPU hang, full log captured

## 8. Error Handling & Recovery

### GuC Error Types

```c
enum xe_guc_error {
    XE_GUC_ERROR_NONE = 0,
    XE_GUC_ERROR_EXCEEDED_LIP = 1,      // Lost interrupt pkt
    XE_GUC_ERROR_MEMORY_ACCESS_FAULT = 2,
    XE_GUC_ERROR_PREEMPT_TIMEOUT = 3,
    XE_GUC_ERROR_RESPONSE_TIMEOUT = 4,
    // ... more error types
};
```

### Recovery Process

On GuC error:

```
1. Interrupt signals GuC error
2. xe_irq_handler() called
3. Identifies GuC error type
4. Initiates GT reset:
   └─ Abort pending jobs
   └─ Reset GuC
   └─ Reload firmware
   └─ Restart execution

5. If repeated failures → wedged device
   └─ Device marked as broken
   └─ Reject new submissions
```

## 9. SR-IOV Support

In virtualized configurations (SR-IOV), multiple VMs share a single physical GPU:

```
Physical Function (PF) Driver        Virtual Function (VF) Drivers
      │                                    │        │        │
      │                                    │        │        │
      v                                    v        v        v
   Physical GPU with GuC firmware
      │
      ├─ Primary GuC (Manages hardware)
      │
      └─ Relay (Communicates with VFs)
             │
             ├─ G2H Proxy (GuC to Host)
             └─ H2G Proxy (Host to GuC)
```

GuC Relay Features:
- Routes VF commands through PF to GuC
- Enforces isolation between VMs
- Manages shared resources (frequency, power)

**Related Files**:
- `xe_guc_relay.c` - GuC relay for SR-IOV
- `xe_guc_relay_types.h` - Relay structures

## 10. Key GuC-Related Files Reference

| File | Purpose |
|------|---------|
| `xe_guc.c/h` | Main GuC logic |
| `xe_guc_types.h` | GuC data structures |
| `xe_guc_ct.c/h` | Command transport |
| `xe_guc_ads.c/h` | Address data structures |
| `xe_guc_submit.c/h` | Job submission |
| `xe_guc_pc.c/h` | Power conservation (RPS) |
| `xe_guc_log.c/h` | Logging support |
| `xe_guc_relay.c/h` | SR-IOV relay |
| `xe_guc_db_mgr.c/h` | Doorbell management |
| `xe_guc_tlb_inval.c/h` | TLB invalidation |
| `xe_guc_capture.c/h` | Error state capture |
| `xe_guc_fwif.h` | GuC firmware interface |
| `xe_uc_fw.c/h` | Firmware loading |

## 11. Monitoring GuC Health

### Debugfs Interface

```bash
# Check GuC status
cat /sys/kernel/debug/dri/0/guc_info

# View GuC logs
cat /sys/kernel/debug/dri/0/guc_logs

# Check communication health
cat /sys/kernel/debug/dri/0/guc_ct_stats

# Set GuC log level
echo "verbose" > /sys/kernel/debug/dri/0/guc_log_level
```

### Sysfs Monitoring

```bash
# Check frequencies
cat /sys/class/drm/card0/device/gt_freq

# Check GT idle (RC6)
cat /sys/class/drm/card0/device/gt_idle

# Check power state
cat /sys/class/drm/card0/device/power_state
```

## Summary

The GuC is a sophisticated firmware component that:
1. **Offloads scheduling** - Decides job execution order
2. **Manages power** - Dynamic frequency/voltage scaling
3. **Provides virtualization** - Multiplexing resources
4. **Enables security** - Enforcing job isolation
5. **Collects telemetry** - Performance monitoring

The driver communicates with GuC via:
- **CTB (Command Transport Buffer)** for commands
- **Shared Memory** for configuration
- **Doorbell Registers** for job submission
- **Interrupts** for events/errors

Understanding GuC is essential for:
- Debugging job submission issues
- Analyzing performance problems
- Implementing new features
- Handling error recovery

---

**See Also**:
- [00-architecture-overview.md](00-architecture-overview.md) - GuC's role in architecture
- [04-execution-scheduling.md](04-execution-scheduling.md) - Job submission details
- [05-power-management.md](05-power-management.md) - Power management details
