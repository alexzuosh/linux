# Power Management

## Overview

This document covers Intel Xe GPU driver power management including runtime PM, system suspend, and frequency scaling.

**Target Audience**: Power/thermal engineers, driver developers  
**Key Files**: `xe_pm.c`, `xe_gt_idle.c`, `xe_gt_freq.c`, `xe_guc_pc.c`

## 1. Power Management Layers

### System-Level PM
- S0ix (Modern Standby)
- S3 (Suspend to RAM)
- S4 (Suspend to Disk)

### Device-Level PM (Runtime PM)
- D3hot (Low power, fast resume)
- D3cold (Minimal power consumption)

### GT Idle Management
- RC6 (Render C6) - GPU idle states
- Package C-states

### Dynamic Frequency Scaling
- RPS (Render Performance State)
- Min/Max frequency control
- Workload-based scaling

## 2. Runtime PM Flow

```c
// Device entering low-power state
xe_pm_runtime_suspend():
    ├─ Wait for all engines idle
    ├─ Flush caches
    ├─ Stop GuC/HuC
    ├─ Enter D3 state
    └─ Device now low-power

// Device resuming
xe_pm_runtime_resume():
    ├─ Power on GPU
    ├─ Restore GuC/HuC
    ├─ Resume engines
    └─ Device ready for work
```

## 3. Frequency Scaling (RPS)

GuC firmware manages frequency based on:
- GPU load
- Thermal constraints
- User-set min/max limits

Configuration via:
```bash
# Set frequency limits
echo 400  > /sys/class/drm/card0/device/gt_freq_min
echo 1500 > /sys/class/drm/card0/device/gt_freq_max
```

## 4. Known Issues

See `10-potential-issues.md` for:
- PM race condition (FIXME in xe_pm.c)
- Lunarlake scheduling latency workaround

## 5. Debug & Monitoring

```bash
# Check current frequency
cat /sys/kernel/debug/dri/0/frequency

# Check GT idle state
cat /sys/kernel/debug/dri/0/gt_idle_state

# Enable PM tracing
echo "1" > /sys/kernel/debug/tracing/events/pm/enable
```

---

**See Also**: [00-architecture-overview.md](00-architecture-overview.md), [02-guc-integration.md](02-guc-integration.md)
