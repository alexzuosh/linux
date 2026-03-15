# TTM 与 Intel Xe KMD 内存管理深度参考

这是一套涵盖 Linux 内核 TTM（Translation Table Manager）框架及其在 Intel Xe KMD（`drivers/gpu/drm/xe/`）中使用方式的深度技术文档，面向内核驱动开发者和 GPU 软件工程师。

---

## 文档结构

| 文件 | 主题 | 核心内容 |
|------|------|---------|
| [01_TTM_Framework_Overview.md](01_TTM_Framework_Overview.md) | TTM 框架基础 | TTM 架构、核心数据结构、LRU/驱逐框架、与 GEM 的关系 |
| [02_Xe_Core_Data_Structures.md](02_Xe_Core_Data_Structures.md) | Xe 核心数据结构 | `struct xe_bo` 完整注解、`XE_PL_*` / `XE_BO_FLAG_*` 全览、`struct xe_ttm_tt` |
| [03_xe_ttm_funcs_Callbacks.md](03_xe_ttm_funcs_Callbacks.md) | TTM 回调实现 | 13 个 `xe_ttm_funcs` 回调详解、CPU 缓存模式决策、回调执行时序 |
| [04_Placement_Strategy.md](04_Placement_Strategy.md) | Placement 策略 | `__xe_bo_placement_for_flags()`、静态 placement、BAR 约束、`xe_evict_flags()` |
| [05_BO_Move_Migration.md](05_BO_Move_Migration.md) | BO 迁移机制 | `xe_bo_move()` 六条路径、MULTIHOP 两跳迁移、`ttm_bo_move_accel_cleanup()`、CCS 处理 |
| [06_xe_migrate_Engine.md](06_xe_migrate_Engine.md) | xe_migrate 引擎 | `struct xe_migrate`、GPU BLT 命令（`XY_FAST_COPY_BLT` / `MEM_COPY_CMD`）、分块策略、fence 链 |
| [07_Memory_Managers.md](07_Memory_Managers.md) | 内存管理器实现 | `xe_ttm_vram_mgr`（drm_buddy）、`xe_ttm_sys_mgr`、`xe_ttm_stolen_mgr`（dGFX/iGFX 探测） |
| [08_Concurrency_Reservation.md](08_Concurrency_Reservation.md) | 并发与保留机制 | `dma_resv`、`ww_mutex`、`drm_exec` 多 BO 原子锁、VM 共享 resv、pin 机制 |
| [09_BO_Lifecycle.md](09_BO_Lifecycle.md) | BO 生命周期 | 创建链（`xe_bo_alloc` → `ttm_bo_init_reserved`）、`ttm_bo_validate()`、销毁清理、挂起恢复 |
| [10_HW_Interaction.md](10_HW_Interaction.md) | 硬件交互 | BAR 窗口约束、Flat CCS 元数据、GGTT 映射、CPU 缓存模式、PVC vs Xe BLT 命令 |
| [11_Debug_Observability.md](11_Debug_Observability.md) | 调试与可观测性 | debugfs 节点、dmesg 模式、IGT 测试、`intel_gpu_top`、内存问题诊断 |

---

## 快速导航

### 我想了解...

**TTM 基础概念** → [Part 1](01_TTM_Framework_Overview.md)  
**xe_bo 结构字段含义** → [Part 2](02_Xe_Core_Data_Structures.md)  
**BO 如何在 VRAM 和系统内存之间移动** → [Part 5](05_BO_Move_Migration.md)  
**GPU 如何执行内存复制** → [Part 6](06_xe_migrate_Engine.md)  
**VRAM/Stolen/TT 分配器如何工作** → [Part 7](07_Memory_Managers.md)  
**如何防止多线程访问 BO 时死锁** → [Part 8](08_Concurrency_Reservation.md)  
**调试 VRAM 分配失败** → [Part 11](11_Debug_Observability.md)

---

## 关键常量速查

```
内存类型（XE_PL_* / TTM_PL_*）:
  XE_PL_SYSTEM = 0    CPU DRAM，无 DMA 映射
  XE_PL_TT     = 1    CPU DRAM，有 DMA scatter-gather（TTM TT 页面）
  XE_PL_VRAM0  = 2    tile-0 VRAM（本地显存）
  XE_PL_VRAM1  = 3    tile-1 VRAM（多 tile dGPU）
  XE_PL_STOLEN = 8    Stolen 内存（BIOS 预留区域）
  TTM_NUM_MEM_TYPES = 9

重要 BO 标志（XE_BO_FLAG_*）:
  SYSTEM              → 指定加 TT/SYSTEM placement
  VRAM0 / VRAM1       → 指定 VRAM placement
  STOLEN              → 指定 Stolen placement
  PINNED              → 防止 TTM 驱逐（pin_count）
  NEEDS_CPU_ACCESS    → 限制在 BAR 可见区（small BAR 设备）
  GGTT0 ~ GGTT3       → 在指定 tile 建立 GGTT 映射

迁移常量:
  MAX_PREEMPTDISABLE_TRANSFER = SZ_8M  单次不可抢占最大传输量
  min_chunk_size = SZ_64K (typical)    CCS 对齐要求的最小分块

容量:
  XE_BO_MAX_PLACEMENTS = 3    每个 BO 最多 3 个备选 placement
  XE_BO_PRIORITY_NORMAL = 1   用于 xe_bo_create 的默认优先级
```

---

## GPU 平台覆盖范围

| 代号 | 架构 | verx100 | 关键特性 |
|------|------|---------|---------|
| DG2/Alchemist | Xe-HPG | 1200 | dGPU, GDDR6, XY_FAST_COPY_BLT, Flat CCS |
| PVC/Ponte Vecchio | Xe-HPC | 1260 | dGPU, HBM2e, MEM_COPY_CMD, 多 Tile |
| MTL/Meteor Lake | Xe-LPM+ | 1270 | iGPU, 系统 TT CCS, 直接 BAR2 Stolen 访问 |
| BMG/Battlemage | Xe2-HPG | 2001 | dGPU, Xe2 BLT variant, 新 MOCS |
| LNL/Lunar Lake | Xe2-LPM | 2004 | iGPU, 高效 Xe2, 系统 TT CCS |

---

## Source 目录结构

```
drivers/gpu/drm/xe/        ← Xe KMD 主目录
├── xe_bo.c                  BO 核心（placement, move, callbacks）
├── xe_bo.h / xe_bo_types.h  数据结构定义
├── xe_ttm_vram_mgr.c/h      VRAM 分配器（drm_buddy）
├── xe_ttm_sys_mgr.c/h       系统内存/TT 管理器
├── xe_ttm_stolen_mgr.c/h    Stolen 内存管理器
├── xe_migrate.c/h           GPU BLT 迁移引擎
├── xe_ggtt.c/h              GGTT 映射管理
└── xe_debugfs.c             调试文件系统

include/drm/ttm/           ← TTM 框架头文件
├── ttm_bo.h                 ttm_buffer_object 定义
├── ttm_resource.h           ttm_resource / manager 接口
├── ttm_placement.h          placement / place 结构
└── ttm_tt.h                 TTM TT 页面后备

include/drm/
└── drm_exec.h               多 BO 原子加锁框架
```

---

## 生成信息

- **内核版本**: Linux mainline (torvalds/linux master)
- **目标架构**: x86_64 (Intel GPU)
- **文档生成工具**: GitHub Copilot (Claude Sonnet 4.6)
- **覆盖平台**: DG2, PVC, MTL, BMG, LNL
