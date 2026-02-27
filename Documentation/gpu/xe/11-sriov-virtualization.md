# SR-IOV Virtualization Support

## Overview

This document details Single-Root I/O Virtualization (SR-IOV) support in the Intel Xe GPU driver, enabling multiple virtual machines to share a single physical GPU.

**Target Audience**: Virtualization engineers, cloud platform developers  
**Complexity**: VERY HIGH  
**Key Files** (20+ files):
- `xe_sriov.c/h` - Main SR-IOV coordination
- `xe_sriov_pf*.c` - Physical Function (host driver)
- `xe_sriov_vf*.c` - Virtual Function (VM driver)
- `xe_gt_sriov_pf*.c` - PF-side GT management
- `xe_pci_sriov.c` - PCI SR-IOV configuration

## 1. SR-IOV Architecture

### Physical & Virtual Functions

```
Physical GPU (PCI Device)
│
├─ Physical Function (PF) - Host Driver
│  │
│  ├─ Owns all hardware resources
│  ├─ Multiplexes between VFs
│  ├─ Enforces isolation
│  ├─ Manages power/frequency
│  └─ Handles GuC communication
│
└─ Virtual Functions (VF0, VF1, VF2, ...)
   │
   ├─ VF0 Driver → VM0
   ├─ VF1 Driver → VM1
   ├─ VF2 Driver → VM2
   └─ ...
   
   Each VF:
   ├─ Sees dedicated "GPU" in its VM
   ├─ Has isolated address space
   ├─ Cannot directly access hardware
   ├─ Communicates via GuC relay
   └─ Resource-limited allocation
```

### PCI Discovery

```c
// Physical Function (PF) detection
if (is_pci_device_id_matching(pdev, pf_device_ids)) {
    xe_sriov_pf_init(xe);  // PF-side initialization
    return 0;
}

// Virtual Function (VF) detection
if (is_vf(pdev)) {
    xe_sriov_vf_init(xe);  // VF-side initialization
    return 0;
}
```

## 2. Resource Allocation

### How Resources Divided

```
Physical GPU Resources
├─ Compute Units
│  ├─ Allocated to PF: 20%
│  ├─ Allocated to VF0: 30%
│  ├─ Allocated to VF1: 25%
│  └─ Reserved: 25%
│
├─ Memory (VRAM)
│  ├─ PF can allocate up to X GB
│  └─ Each VF allocated Y GB
│
├─ Execution Contexts
│  ├─ PF: up to 100 contexts
│  └─ Each VF: up to 50 contexts
│
└─ Power Budget
   ├─ PF: High priority
   ├─ VFs: Share remaining budget
   └─ GuC enforces limits
```

### CCS (Compute Context Sharing) Mode

```c
struct xe_sriov_vf {
    // Per-VF CCS configuration
    enum {
        XE_SRIOV_CCS_SHARED = 0,   // All VFs share CCS
        XE_SRIOV_CCS_DEDICATED = 1 // Dedicated CCS per VF
    } ccs_mode;
    
    u32 ccs_mask;  // Which CCS engines assigned to this VF
};
```

In shared mode:
- All VFs share available CCS units
- GuC schedules workload fairly
- Maximum flexibility

In dedicated mode:
- Each VF has guaranteed CCS resources
- More predictable, less flexible

## 3. Virtual Function Lifecycle

### Creating Virtual Functions

```bash
# On host, create VF from PF
# (Requires PF driver support)

# Enable SR-IOV on PCI device
echo N > /sys/bus/pci/devices/0000:00:02.0/sriov_numvfs

# Create 4 VFs
echo 4 > /sys/bus/pci/devices/0000:00:02.0/sriov_numvfs

# New devices appear:
# 0000:00:02.1  <- VF0
# 0000:00:02.2  <- VF1
# 0000:00:02.3  <- VF2
# 0000:00:02.4  <- VF3
```

### Assigning to VMs

```bash
# Assign VF to QEMU VM
qemu-system-x86_64 \
  -device vfio-pci,host=0000:00:02.1 \  # VF0
  -m 16G \
  guest.iso
```

### VF Driver Initialization

```c
// In VM, Xe driver initializes as VF
int xe_pci_probe(struct pci_dev *pdev, ...)
{
    // Detect VF
    if (pci_sriov_get_totalvfs(pdev) == 0 && 
        is_virtual_function(pdev)) {
        
        // Initialize as VF
        return xe_sriov_vf_init(xe);
    }
}

int xe_sriov_vf_init(struct xe_device *xe)
{
    // 1. Detect allocated resources
    u32 allocated_contexts = read_from_mmio(ALLOCATED_CTX_REG);
    u64 allocated_memory = read_from_mmio(ALLOCATED_MEM_REG);
    
    // 2. Initialize to work within allocation
    xe->info.max_exec_queues = allocated_contexts;
    
    // 3. Register with PF via GuC relay
    send_vf_ready_message();
    
    // VF now ready
    return 0;
}
```

## 4. Communication: GuC Relay Protocol

### VF → PF Message Flow

VFs cannot directly command hardware. All requests go through PF:

```
VF Application
  │
  v
VF Driver (DRM ioctl)
  │
  ├─ DRM_XE_EXEC (submit job)
  ├─ DRM_XE_VM_CREATE (create VM)
  └─ DRM_XE_VM_BIND (bind memory)
  │
  v
VF GuC Driver
  │
  ├─ Can't talk to hardware directly
  ├─ Can't access real registers
  └─ Can't allocate real contexts
  │
  v
GuC Relay (via CTB)
  │
  ├─ Send relay message to GuC
  ├─ "PF, please register a context for me"
  ├─ "PF, please allocate memory range"
  └─ "PF, please submit this job"
  │
  v
GuC on Hardware
  │
  ├─ Receives relay message
  ├─ Validates VF hasn't exceeded quota
  └─ Performs actual operation
  │
  v
GuC → VF Response (via relay)
  │
  ├─ "Context registered with ID 42"
  ├─ "Memory allocated at GGTT offset 0x1000"
  └─ "Job submitted, will signal fence when complete"
  │
  v
VF Continue Processing
```

### Relay Protocol Details

```c
struct xe_guc_relay_message {
    u32 header;              // Version, message type
    
    union {
        struct {
            // VF → PF: Register context
            u32 context_id;
            u32 engine_class;
            u32 priority;
        } register_context;
        
        struct {
            // VF → PF: Allocate memory
            u64 size;
            u32 placement;   // Prefer VRAM, system, etc.
        } allocate_memory;
        
        struct {
            // VF → PF: Submit job
            u32 context_id;
            u64 batch_addr;
            u32 batch_len;
        } submit_job;
    } payload;
};

int send_relay_message(struct xe_guc *guc,
                      struct xe_guc_relay_message *msg)
{
    // 1. Send via GuC command transport
    ret = xe_guc_ct_send(&guc->ct, msg);
    
    // 2. Wait for response (with timeout)
    response = wait_for_relay_response(timeout_ms);
    
    // 3. Check response status
    if (response->status != SUCCESS) {
        // Request denied (quota exceeded, etc.)
        return -response->error_code;
    }
    
    return 0;
}
```

## 5. Resource Quotas & Enforcement

### PF-Side Quota Tracking

```c
struct xe_sriov_pf {
    // Per-VF resource limits
    struct {
        u32 vf_id;
        
        // Resource budgets
        struct {
            u32 max_contexts;
            u64 max_vram;
            u32 max_exec_queues;
            u32 max_vms;
        } quota;
        
        // Current usage
        struct {
            u32 contexts_used;
            u64 vram_used;
            u32 exec_queues_used;
            u32 vms_used;
        } usage;
        
        // Service level
        enum {
            SERVICE_LEVEL_LOW = 0,
            SERVICE_LEVEL_NORMAL = 1,
            SERVICE_LEVEL_HIGH = 2,
        } service_level;
    } vf[num_vfs];
};
```

### Quota Enforcement

```c
int xe_sriov_pf_allocate_context(struct xe_sriov_pf *pf,
                                u32 vf_id)
{
    struct xe_sriov_vf *vf = &pf->vf[vf_id];
    
    // Check quota
    if (vf->usage.contexts_used >= vf->quota.max_contexts) {
        // VF exceeded quota, deny request
        return -ENOSPC;
    }
    
    // Allocate
    vf->usage.contexts_used++;
    
    return allocated_context_id;
}
```

## 6. Memory Management in SR-IOV

### Shared GGTT Space

```
Device GGTT (e.g., 256MB)
│
├─ PF Region (100MB)
│  └─ GuC structures, firmware, display
│
├─ VF0 Region (30MB)
│  └─ Exec contexts, scratch, shared structures
│
├─ VF1 Region (30MB)
│  └─ Exec contexts, scratch, shared structures
│
├─ VF2 Region (30MB)
│  └─ Exec contexts, scratch, shared structures
│
└─ Reserved (36MB)
```

Each VF gets isolated GGTT range:

```c
int xe_sriov_vf_init_ggtt(struct xe_sriov_vf *vf)
{
    struct xe_ggtt *ggtt = &vf->tile->ggtt;
    u64 vf_ggtt_size = GGTT_SIZE / (num_vfs + 1);
    u64 vf_ggtt_offset = (vf_id + 1) * vf_ggtt_size;
    
    // VF can only allocate within its range
    vf->ggtt_offset = vf_ggtt_offset;
    vf->ggtt_size = vf_ggtt_size;
    
    // Create drm_mm for allocation within range
    drm_mm_init(&vf->ggtt_mm, vf_ggtt_offset, vf_ggtt_size);
}
```

## 7. Live Migration

SR-IOV allows VFs to migrate between servers:

```
VM on Host A (running on GPU0)
  │
  ├─ Pause VM
  ├─ Stop GPU workload
  ├─ Capture VF state
  │  ├─ Active contexts
  │  ├─ Job queues
  │  ├─ Memory mappings
  │  └─ Register state
  │
  ├─ Transfer state to Host B
  │
  └─ Resume on Host B (GPU1)
     ├─ Restore VF state
     ├─ Re-initialize GPU contexts
     └─ Resume workloads
```

### Migration Code Path

```c
void xe_sriov_pf_prepare_migration(struct xe_sriov_pf *pf,
                                  u32 vf_id)
{
    // 1. Flush all pending work
    flush_vf_workloads(pf, vf_id);
    
    // 2. Snapshot GPU state
    char *snapshot = capture_vf_state(pf, vf_id);
    
    // 3. Detach VF
    detach_vf(pf, vf_id);
    
    // snapshot can now be transferred
}

void xe_sriov_vf_migrate_in(struct xe_device *xe,
                           char *snapshot)
{
    // On new host, restore from snapshot
    restore_vf_state(xe, snapshot);
    
    // Resume execution
    resume_vf_workloads(xe);
}
```

## 8. Monitoring & Debugging

### PF-Side Monitoring

```bash
# Check VF status
cat /sys/kernel/debug/dri/0/sriov_pf_vf_info

# View VF resource usage
cat /sys/kernel/debug/dri/0/sriov_pf_vf_usage

# Check quota violations
cat /sys/kernel/debug/dri/0/sriov_pf_quota_violations
```

### VF-Side Debugging

```bash
# Check allocated resources
cat /sys/kernel/debug/dri/0/sriov_vf_info

# View relay communication
cat /sys/kernel/debug/dri/0/guc_relay_stats

# Check message errors
cat /sys/kernel/debug/dri/0/relay_errors
```

## 9. Known SR-IOV Limitations

See [10-potential-issues.md](10-potential-issues.md) section 3.3:

- Complex implementation (20+ files, 1000+ LOC)
- Limited production deployment
- VF driver crashes can affect other VFs
- Migration not fully tested
- Resource quota enforcement evolving

## 10. Common SR-IOV Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| VF fails to boot | "No GPU found" | Check PF allocated resources, check relay |
| Quota exceeded | Memory allocation fails | Check VF quota limits, request more resources |
| Migration fails | VF state corrupted | Re-migrate from source, check snapshot integrity |
| GuC relay timeout | Relay message delayed >1s | Check PF GuC status, increase timeout |
| Resource leak | Quota gradually exhausted | Check for resource cleanup in error paths |

## Summary

SR-IOV enables:
1. **Multi-tenant GPU** - Multiple VMs on one GPU
2. **Fair sharing** - Resource quotas prevent one VM hogging GPU
3. **Live migration** - Move VMs between servers without stopping workload
4. **Hardware isolation** - VMs can't directly access hardware

Key mechanisms:
- GuC relay protocol for VF→PF communication
- Per-VF resource quotas and tracking
- Shared GGTT with isolated ranges per VF
- Migration via state snapshot/restore

SR-IOV is complex but powerful for cloud/data center scenarios.

---

**See Also**:
- [02-guc-integration.md](02-guc-integration.md) - GuC relay protocol
- [03-memory-management.md](03-memory-management.md) - Memory in VFs
- [10-potential-issues.md](10-potential-issues.md) - SR-IOV limitations
