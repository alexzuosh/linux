# XE GPU DRIVER: BASIC IMMUTABLE FACTS

## Fundamental Architectural Properties of All Xe GPUs

---

## HARDWARE ARCHITECTURE

### Device/Tile/GT Hierarchy

**FACT 1: Maximum 2 tiles per device**
- Source: `xe_device_types.h:73-75`: `#define XE_MAX_TILES_PER_DEVICE 2`
- Implication: Multi-GPU is implemented as tiles within one PCI device, not separate devices

**FACT 2: Each tile contains 1-2 GTs (Graphics Technology units)**
- Source: `xe_device_types.h:286-288`: `u8 max_gt_per_tile` (1 or 2)
- Implication: Primary GT (type=MAIN) handles graphics/compute. Optional media GT (type=MEDIA) on DG2+ platforms

**FACT 3: GT types are MAIN, MEDIA, or UNINITIALIZED**
- Source: `xe_gt_types.h:27-31`
- Implication: No other GT types exist. Distinct execution pipelines for each type

**FACT 4: Each tile has exactly one GGTT (Global Graphics Translation Table)**
- Source: `xe_device_types.h:194-195`: `struct xe_ggtt *ggtt;` (per-tile field)
- Implication: All GTs in a tile share physical memory translation via single GGTT

**FACT 5: 6 engine classes exist: RENDER, VIDEO_DECODE, VIDEO_ENHANCE, COPY, OTHER, COMPUTE**
- Source: `xe_hw_engine_types.h:13-22`
- Implication: Multiple instances of each class (RCS=1, BCS=8, VCS=8, VECS=4, CCS=4)

**FACT 6: Engine class has one IRQ/fence handler per GT**
- Source: `xe_gt_types.h:242-243`: `struct xe_hw_fence_irq fence_irq[XE_ENGINE_CLASS_MAX]`
- Implication: All instances of an engine class share one seqno counter and IRQ handler

---

## MEMORY MODEL

### Virtual Address Space

**FACT 7: VA space is either 48-bit or 57-bit, platform-dependent**
- Source: `xe_pci.c` platform descriptors + `xe_device_types.h:293`
- Examples: Xe-LPM/DG2 = 48-bit (256TB), Xe2-HPG = 57-bit (128PB)
- Implication: Newer platforms support vastly larger address spaces

**FACT 8: Maximum 2^20 unique VMs (1,048,575)**
- Source: `xe_device_types.h:77`: `#define XE_MAX_ASID (BIT(20))`
- Implication: ASIDs used for unified shared memory fault routing

**FACT 9: VMs assigned ASIDs cyclically from global pool**
- Source: `xe_device_types.h:432-435`: `u32 next_asid` with cyclic allocation
- Implication: ASID exhaustion handled via wraparound

### Page Tables

**FACT 10: Maximum 4 page table levels**
- Source: `xe_pt_types.h:25`: `#define XE_VM_MAX_LEVEL 4`
- Level 0 = leaf PTEs, Level 4 = root
- Implication: Level count = 3 for 48-bit VA, 4 for 57-bit VA

**FACT 11: Page sizes: 4K, 64K, 2M, 1GB**
- Source: `xe_vm_types.h:42-46`: VMA flags for PTE sizes
- Implication: 4K/64K at leaf level, larger pages for VRAM only

**FACT 12: TLB invalidation at 3 granularities: all/GGTT/PPGTT-range**
- Source: `xe_tlb_inval_types.h:24,34,47`: three ops functions
- Implication: PPGTT range-based invalidation ASID-aware for multi-process

**FACT 13: Cache levels: NONE, WT, WB, NONE_COMPRESSION**
- Source: `xe_pt_types.h:17-23`: `enum xe_cache_level`
- Implication: Coherency managed via cache mode + PAT. COMPRESSION for VRAM only

**FACT 14: PTE encoding platform-specific via function pointers**
- Source: `xe_pt_types.h:40-49`: `struct xe_pt_ops` callbacks
- Implication: Encoding includes cache level, page level, device memory flag

### Physical Memory

**FACT 15: DMA address width: 39-52 bits**
- Source: `xe_pci.c`: integrated=39-bit, DG2=46/52-bit, Xe2=52-bit
- Implication: Discrete GPUs address more physical memory

**FACT 16: VRAM requires 64K alignment on select platforms (XE_VRAM_FLAGS_NEED64K)**
- Source: `xe_device_types.h:71`: `#define XE_VRAM_FLAGS_NEED64K`
- Implication: Platform-specific minimum page size enforcement

**FACT 17: Two VRAM regions per tile: kernel_vram and user vram**
- Source: `xe_device_types.h:178-192`: per-tile mem struct
- Implication: Discrete GPUs have local VRAM; integrated access system RAM

---

## EXECUTION MODEL

### Execution Queues

**FACT 18: Each execution queue bound to exactly one GT**
- Source: `xe_exec_queue_types.h:45-46`: `struct xe_gt *gt` (single pointer)
- Implication: Queues cannot span multiple GTs; multi-GT work requires separate queues

**FACT 19: Queues support variable width (1 to N parallel contexts)**
- Source: `xe_exec_queue_types.h:71-72`: `u16 width; struct xe_lrc *lrc[] __counted_by(width)`
- Implication: Single queue manages parallel execution contexts

**FACT 20: Two submission backends: GuC (modern) and Execlist (legacy)**
- Source: `xe_exec_queue_types.h:106-118`: union with guc/execlist pointers
- Implication: GuC default; Execlist fallback for debug/legacy platforms

### Job Submission

**FACT 21: Jobs submitted via submission rings in Logical Ring Contexts (LRCs)**
- Source: `xe_lrc_types.h:18-47`: LRC struct with ring size/tail
- Implication: Ring buffer model: CPU writes tail, GPU reads head

**FACT 22: Hardware fences signal completion via 32-bit seqno**
- Source: `xe_hw_fence_types.h:50`: `u32 next_seqno` (explicit u32)
- Implication: 4B submission limit before wraparound; handled by fence framework

**FACT 23: One seqno per engine class, not per instance**
- Source: `xe_gt_types.h:242-243`: `fence_irq[XE_ENGINE_CLASS_MAX]` (6 total)
- Implication: All RCS share seqno; all BCS share seqno; etc.

### Preemption

**FACT 24: Preemption via explicit suspend/suspend_wait/resume**
- Source: `xe_exec_queue_types.h:227-243`: three callback functions
- Implication: Not automatic; CPU must request, wait, then resume

---

## SYNCHRONIZATION

**FACT 25: 32-bit seqnos support wraparound via signed comparison**
- Source: Common kernel pattern: `(s32)(a - b) >= 0`
- Implication: Wraparound transparent; linear seqno space despite 32-bit width

**FACT 26: Fences use DMA fence framework with IRQ-triggered signaling**
- Source: `xe_hw_fence_types.h:24-33`: irq_work and pending list
- Implication: Non-blocking fence completion via seqno memory write

---

## INTERRUPTS

**FACT 27: MSI-X supported; no LSI fallback**
- Source: `xe_device_types.h:384-390`: msix struct with nvec
- Implication: All Xe platforms use MSI-X; no legacy interrupt support

**FACT 28: Memory-based interrupts (MEMIRQ) on Xe12.5+ (GRAPHICS_VERx100 >= 1250)**
- Source: `xe_device.h:160-163`: `xe_device_has_memirq()` check
- Implication: Newer platforms can use memory-based signaling instead of MSI-X

---

## VIRTUAL ADDRESS SPACE

**FACT 29: Each VM gets full exclusive VA space (2^va_bits bytes)**
- Source: `xe_vm_doc.h:79-80` + `xe_vm_types.h:168-204`
- Implication: No kernel VA hole; entire space user-accessible

---

## CACHE COHERENCY

**FACT 30: Device atomics on system memory conditionally supported**
- Source: `xe_device_types.h:305-306`: `has_device_atomics_on_smem` flag
- Implication: Older platforms only support atomics on VRAM

**FACT 31: Atomic enable controlled per-page via PTE bit (conditional)**
- Source: `xe_device_types.h:303-304`: `has_atomic_enable_pte_bit` flag
- Implication: Some platforms have per-page atomic control; others are global

**FACT 32: Explicit cache flushes available (TD, L1, L2)**
- Source: `xe_device.h:182-183`: `xe_device_l2_flush()`, `xe_device_td_flush()`
- Implication: CPU can explicitly flush GPU caches when needed

---

## CAPABILITIES

**FACT 33: All device capabilities determined at probe time and immutable**
- Source: `xe_device_types.h:256-358`: intel_device_info struct
- Implication: Cannot change at runtime; gates entire feature sets

**FACT 34: Key capability flags: has_asid, has_usm, has_range_tlb_inval, has_flat_ccs, has_llc**
- Source: `xe_device_types.h:301-334`: all capability flags
- Implication: Each flag gates a fundamental architectural feature

---

## COHERENT ARCHITECTURAL MODEL

```
┌─ XE_DEVICE (1 PCI device)
   ├─ Tiles[0..1]              // 1-2 tiles max
   │  ├─ Primary GT (MAIN)
   │  │  ├─ Engines: RCS, BCS0-8, VCS0-7, VECS0-3, CCS0-3
   │  │  ├─ Fence IRQs: 6 (one per engine class)
   │  │  └─ Ring Buffers: one LRC per engine instance
   │  ├─ Media GT (MEDIA) [optional on DG2+]
   │  │  └─ Media engines (VCS, VECS)
   │  ├─ GGTT: single global translation table
   │  ├─ kernel_vram region
   │  └─ user_vram region
   └─ VMs[0..1M]               // Up to 2^20 VMs with ASID
      ├─ VA Space: 2^48 or 2^57 bytes
      ├─ PT Levels: 3 or 4
      └─ ASIDs: unique 20-bit ID per VM
```

---

## KEY INVARIANTS (True for ALL Xe GPUs)

1. **Topology**: Single device, 1-2 tiles, 1-2 GTs per tile
2. **Execution**: 6 engine classes, multiple instances per class
3. **Memory**: 48-bit or 57-bit VA space, 1-4 PT levels, 4 cache modes
4. **Fencing**: 32-bit seqno, class-level granularity, wraparound-safe
5. **Multi-Process**: Up to 1M VMs via 20-bit ASIDs
6. **Immutability**: All capabilities fixed at probe time
7. **Interrupts**: MSI-X only, class-based IRQ handling
8. **Atomicity**: Platform-dependent; can be per-page or global

---

## DERIVED FACTS (Deducible from above)

**Deduction A**: Ring buffer model enables lock-free submission
- From: FACT 21 (submission rings) + FACT 22 (32-bit seqno)
- Conclusion: CPU writes tail asynchronously; GPU reads head; no synchronization needed

**Deduction B**: TLB invalidation semantics vary by platform
- From: FACT 12 (3 invalidation granularities) + FACT 28 (MEMIRQ on Xe12.5+)
- Conclusion: Newer platforms can use memory-based TLB invalidation signaling

**Deduction C**: Page size alignment enforced at binding time
- From: FACT 11 (4 page sizes) + FACT 16 (64K on select platforms)
- Conclusion: Bind operations must align VA/PA to minimum page size

**Deduction D**: Older platforms limited to 256TB; newer to 128PB
- From: FACT 7 (48 or 57-bit VA) + memory mapping math
- Conclusion: Hardware capability dictates maximum addressable space

