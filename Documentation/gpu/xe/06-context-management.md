# Context Management

## Overview

GPU contexts represent per-engine execution state including register values, memory mappings, and instruction stream pointers.

**Target Audience**: Context scheduling developers, HW engineers  
**Key Files**: `xe_lrc.c`, `xe_lrc_types.h`

## 1. Context Basics

### What is a Context?

A context holds the execution state for one hardware engine:
- Current instruction pointer
- Register values
- State save/restore area
- Timeline data

```c
struct xe_lrc {
    struct xe_bo *bo;               // State save area
    struct xe_ring *ring;           // Command ring
    u32 ring_head, ring_tail;       // Ring pointers
    u32 guc_id;                     // GuC's ID
    u64 timestamp;                  // Last context switch
};
```

## 2. Context Lifecycle

### Creation
```c
ctx = xe_lrc_create(vm, engine_class)
    ├─ Allocate context save area
    ├─ Initialize ring buffer
    └─ Register with GuC
```

### Usage
- Used by execution queues
- GuC switches between contexts based on priority
- Context state saved on preemption

### Destruction
- Flush remaining jobs
- Free context structure

## 3. Logical Ring Context (LRC)

### Ring Buffer
Commands submitted to GPU via ring buffer:

```
Ring Buffer (circular):
┌────────────────────┐
│ Batch Buffer Start │
├────────────────────┤
│ Memory Operations  │
├────────────────────┤
│ ...                │
├────────────────────┤
│ Fence Signal       │
├────────────────────┤
│ (wrap around)      │
└────────────────────┘
```

### Context Switching
GuC performs context switching:
1. Save current context state
2. Load new context state
3. Resume execution

## 4. Context Save/Restore

When context switched out:
```c
// GuC performs:
save_context_registers(ctx_id)
    ├─ Save PC (program counter)
    ├─ Save register state
    ├─ Save memory pointers
    └─ Mark context saved

load_context_registers(ctx_id)
    ├─ Load PC
    ├─ Restore registers
    └─ Resume execution
```

## 5. Debugging

```bash
# List all contexts
cat /sys/kernel/debug/dri/0/contexts

# Context information
cat /sys/kernel/debug/dri/0/context_info

# GuC context logs
cat /sys/kernel/debug/dri/0/guc_logs | grep context
```

---

**See Also**: [04-execution-scheduling.md](04-execution-scheduling.md)
