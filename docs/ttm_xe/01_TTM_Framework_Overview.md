# Part 1: TTM 框架基础 (TTM Framework Overview)

> **Source files**: `include/drm/ttm/ttm_bo.h`, `include/drm/ttm/ttm_device.h`,  
> `include/drm/ttm/ttm_resource.h`, `include/drm/ttm/ttm_placement.h`,  
> `include/drm/ttm/ttm_tt.h`, `drivers/gpu/drm/ttm/`

---

## 1.1 TTM 整体架构与设计目标

TTM (Translation Table Manager) 是 Linux DRM 子系统中专用于 GPU 显存管理的核心框架，最初由 Tungsten Graphics 开发，后合入主线内核。它解决了 GPU 驱动中最核心的问题：**如何在多种内存类型（显存、系统内存、Stolen 内存）之间自动搬移缓冲区对象（Buffer Objects），同时保持 GPU 的并发执行**。

### TTM 解决的核心问题

```
问题一: 内存压力
  GPU 显存容量有限 (4GB ~ 80GB)
  → 当显存满时，必须将不活跃 BO 驱逐(evict)到系统内存
  → BO 再次被使用时，需要迁移(migrate)回显存

问题二: CPU/GPU 可见性
  系统内存 CPU 可直接访问，GPU 需要通过 GTT (Graphics Translation Table) 映射
  显存 GPU 直接访问，CPU 只能通过 BAR 窗口访问

问题三: 并发安全
  多个 CPU 线程 / GPU 引擎同时操作 BO
  → 需要引用计数 + reservation 锁保护
```

### TTM vs GEM 的关系

```
┌─────────────────────────────────────────┐
│              用户空间 (libdrm)            │
│   DRM_IOCTL_XE_GEM_CREATE               │
│   DRM_IOCTL_XE_GEM_MMAP_OFFSET          │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│              GEM 层 (drm_gem)            │
│  drm_gem_object  ← 用户可见的 GEM object │
│  GEM handle → fd (prime) 导出/导入       │
└──────────────────┬──────────────────────┘
                   │  内嵌关系
┌──────────────────▼──────────────────────┐
│              TTM 层                      │
│  ttm_buffer_object → 内存管理核心        │
│  placement / eviction / migration        │
└──────────────────┬──────────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
  ┌──────────┐         ┌──────────────┐
  │ VRAM mgr │         │  TT/System   │
  │(drm_buddy)│        │  mgr (pages) │
  └──────────┘         └──────────────┘
```

GEM 只负责用户空间对象生命周期（handle、fd、refcount），**TTM 才是真正管理物理内存的层**。`drm_gem_object` 内嵌在 `ttm_buffer_object` 的 `base` 字段中。

---

## 1.2 ttm_device — 设备级别生命周期

`ttm_device` 是 TTM 在设备层面的上下文，每个 GPU 设备有一个实例。它持有所有内存类型管理器、LRU 列表，以及指向驱动回调函数的指针。

```c
// include/drm/ttm/ttm_device.h
struct ttm_device {
    struct list_head device_list;      // 全局 TTM 设备链表
    const struct ttm_device_funcs *funcs; // 驱动回调函数表 ← 最重要
    
    struct ttm_resource_manager sysman; // 系统内存管理器(内嵌)
    struct ttm_resource_manager *man_drv[TTM_NUM_MEM_TYPES]; // 所有内存类型管理器数组
    
    // LRU 列表：按 priority 分级，eviction 从低 priority 开始
    struct ttm_lru_bulk_move *lru_bulk_move;
    spinlock_t lru_lock;
    
    struct drm_vma_offset_manager *vma_manager; // mmap 地址空间管理
    struct list_head pinned;           // 所有 pinned BO 链表
};
```

### ttm_device_init 调用（Xe 中）

```c
// drivers/gpu/drm/xe/xe_bo.c
int xe_bo_init(struct xe_device *xe)
{
    int err;

    // 初始化 TTM 设备核心
    err = ttm_device_init(&xe->ttm,             // struct ttm_device
                          &xe_ttm_funcs,         // 驱动回调表(见 Part 3)
                          xe->drm.dev,           // 底层 device
                          xe->drm.anon_inode->i_mapping,
                          xe->drm.vma_offset_manager,
                          false,   // no_io_mem_pfn
                          false);  // use_dma32
    ...
    // 注册各内存类型管理器(见 Part 7)
    xe_ttm_sys_mgr_init(xe);
    for_each_mem_region(...)
        xe_ttm_vram_mgr_init(xe, mr);
    xe_ttm_stolen_mgr_init(xe);
}
```

### ttm_device_funcs — 驱动回调表

```c
// include/drm/ttm/ttm_device.h
struct ttm_device_funcs {
    // TT 页面管理
    struct ttm_tt *(*ttm_tt_create)(struct ttm_buffer_object *, u32 page_flags);
    int  (*ttm_tt_populate)(struct ttm_device *, struct ttm_tt *, ctx);
    void (*ttm_tt_unpopulate)(struct ttm_device *, struct ttm_tt *);
    void (*ttm_tt_destroy)(struct ttm_device *, struct ttm_tt *);

    // BO 迁移与驱逐
    void (*evict_flags)(struct ttm_buffer_object *, struct ttm_placement *);
    int  (*move)(struct ttm_buffer_object *, bool evict, ctx, new_mem, hop);
    bool (*eviction_valuable)(struct ttm_buffer_object *, const struct ttm_place *);

    // MMIO/CPU 访问
    int  (*io_mem_reserve)(struct ttm_device *, struct ttm_resource *);
    unsigned long (*io_mem_pfn)(struct ttm_buffer_object *, unsigned long page_offset);
    int  (*access_memory)(struct ttm_buffer_object *, uint64_t, void *, int, int);

    // 通知钩子
    void (*release_notify)(struct ttm_buffer_object *);
    void (*delete_mem_notify)(struct ttm_buffer_object *);
    void (*swap_notify)(struct ttm_buffer_object *);
};
```

---

## 1.3 ttm_buffer_object 结构剖析

`ttm_buffer_object` 是 TTM 的核心对象，代表一块 GPU 可使用的内存缓冲区。

```c
// include/drm/ttm/ttm_bo.h
struct ttm_buffer_object {
    // ─── GEM 基类 ───────────────────────────
    struct drm_gem_object base;     // .resv: dma_resv (ww_mutex 所在)
                                    // .gpuva.list: 关联的 GPU VMA 链表

    // ─── 初始化后不变 ────────────────────────
    struct ttm_device    *bdev;     // 所属设备
    enum ttm_bo_type      type;     // device / kernel / sg
    uint32_t page_alignment;        // 物理页对齐要求
    void (*destroy)(struct ttm_buffer_object *); // 自定义销毁函数

    // ─── 引用计数 ────────────────────────────
    struct kref kref;               // 归零时触发 destroy

    // ─── reservation lock 保护 ───────────────
    struct ttm_resource  *resource; // 当前物理内存位置描述
    struct ttm_tt        *ttm;      // TT 页面后备(系统内存时有效)
    bool deleted;                   // 是否已被标记删除
    struct ttm_lru_bulk_move *bulk_move; // LRU bulk 迁移组
    unsigned priority;              // LRU 优先级(越低越先被驱逐)
    unsigned pin_count;             // pin 计数(>0 则不可驱逐)

    // ─── 其他 ────────────────────────────────
    struct work_struct delayed_delete; // 延迟销毁工作项
    struct sg_table *sg;            // DMA-buf 外部页面(sg BO)
};
```

### enum ttm_bo_type 详解

| 类型 | 说明 | Xe 使用场景 |
|------|------|------------|
| `ttm_bo_type_device` | 用户空间可 mmap 的 BO | GEM_CREATE 用户 BO |
| `ttm_bo_type_kernel` | 仅内核使用，不可 mmap | 页表 BO、固件 BO、GGTT |
| `ttm_bo_type_sg` | 来自 DMA-buf 的外部 scatter-gather | dma_buf import |

---

## 1.4 ttm_resource 与 ttm_resource_manager

`ttm_resource` 描述 BO **当前的物理内存位置**，由 `ttm_resource_manager::alloc()` 分配：

```c
// include/drm/ttm/ttm_resource.h
struct ttm_resource {
    // ─── LRU 节点 ────────────────────────────
    struct ttm_lru_item lru;        // TTM LRU 链表节点

    // ─── 位置描述 ────────────────────────────
    uint32_t mem_type;              // XE_PL_SYSTEM / XE_PL_TT / XE_PL_VRAM0 ...
    uint32_t placement;             // TTM_PL_FLAG_* 标志(连续/topdown等)
    uint64_t start;                 // 在该内存类型中的起始页帧号(PFN)
                                    // 非连续则为 XE_BO_INVALID_OFFSET

    // ─── 大小 ────────────────────────────────
    size_t   size;                  // 字节大小

    struct ttm_buffer_object *bo;   // 反向指针
    struct ttm_resource_manager *man; // 所属管理器
};
```

`ttm_resource_manager` 是每种内存类型的管理器接口：

```c
// include/drm/ttm/ttm_resource.h
struct ttm_resource_manager {
    bool use_type;     // 是否启用此内存类型
    bool use_tt;       // 此类型是否使用 TT 页面后备(SYSTEM/TT=true, VRAM=false)
    uint64_t size;     // 管理的总内存大小(页数)
    
    const struct ttm_resource_manager_func *func; // alloc/free/intersects/compatible
    
    struct ttm_lru_bulk_move *move;  // bulk eviction 列表
    struct list_head lru[TTM_MAX_BO_PRIORITY]; // 分优先级 LRU 列表
};

struct ttm_resource_manager_func {
    int  (*alloc)(man, bo, place, **res);       // 分配内存
    void (*free)(man, *res);                     // 释放内存
    bool (*intersects)(man, *res, *place, size); // 驱逐候选筛选
    bool (*compatible)(man, *res, *place, size); // 判断是否需要迁移
    void (*debug)(man, *printer);                // debugfs 输出
};
```

### 内存类型管理器注册

```c
// TTM 内置宏：将 manager 注册到 bdev->man_drv[mem_type]
ttm_set_driver_manager(bdev, mem_type, manager);

// 获取指定 mem_type 的管理器
mgr = ttm_manager_type(bdev, mem_type);
```

---

## 1.5 ttm_placement 与 ttm_place

`ttm_placement` 是 BO 创建或迁移时指定的**候选内存位置列表**，TTM 按顺序尝试，第一个成功的即为结果：

```c
// include/drm/ttm/ttm_placement.h
struct ttm_placement {
    unsigned int  num_placement;        // 候选数量(Xe 最多 3 个)
    const struct ttm_place *placement;  // 候选数组指针
};

struct ttm_place {
    unsigned int fpfn;    // first valid page frame number (范围下界)
    unsigned long lpfn;   // last valid page frame number  (范围上界, 0=no limit)
    uint32_t mem_type;    // XE_PL_VRAM0 / XE_PL_TT / XE_PL_SYSTEM ...
    uint32_t flags;       // TTM_PL_FLAG_* 组合
};
```

### TTM_PL_FLAG_* 标志含义

| 标志 | 含义 |
|------|------|
| `TTM_PL_FLAG_DESIRED` | 首选位置（优先尝试） |
| `TTM_PL_FLAG_FALLBACK` | 备用位置（首选失败后尝试） |
| `TTM_PL_FLAG_TOPDOWN` | 从地址空间顶端分配（远离 BAR 窗口） |
| `TTM_PL_FLAG_CONTIGUOUS` | 要求物理连续（用于 vmap、DMA 等） |
| `TTM_PL_FLAG_TEMPORARY` | 临时位置（multihop 中转用） |

### TTM_PL_* 内存类型常量

```c
// include/drm/ttm/ttm_placement.h
#define TTM_PL_SYSTEM  0   // 虚拟系统内存（无 DMA 地址）
#define TTM_PL_TT      1   // GTT 映射的系统内存（有 DMA 地址）
#define TTM_PL_VRAM    2   // 显存（驱动自定义扩展基础）
#define TTM_PL_PRIV    3   // 驱动私有内存类型
// ...
#define TTM_NUM_MEM_TYPES 9
```

---

## 1.6 ttm_tt — TT 页面后备

`ttm_tt` 管理系统内存页面，当 BO 位于 `SYSTEM` 或 `TT` 时有效：

```c
// include/drm/ttm/ttm_tt.h
struct ttm_tt {
    struct ttm_device *bdev;
    uint32_t page_flags;     // TTM_TT_FLAG_* 
    uint64_t num_pages;      // 页面数量
    struct page **pages;     // 页面指针数组(populate 后填充)
    enum ttm_caching caching; // CPU 缓存模式: WB/WC/UC
    
    // DMA 相关
    struct sg_table *sg;     // DMA 散列表(外部 dmabuf)
};

// TTM_TT_FLAG_*
#define TTM_TT_FLAG_SWAPPED        BIT(0) // 被 swap 到磁盘
#define TTM_TT_FLAG_ZERO_ALLOC     BIT(1) // 分配时需要清零
#define TTM_TT_FLAG_EXTERNAL       BIT(2) // 外部管理的页面
#define TTM_TT_FLAG_EXTERNAL_MAPPABLE BIT(3) // 外部页面可映射

// 生命周期
ttm_tt_create()    → 驱动 ttm_tt_create 回调: 分配 ttm_tt 结构
ttm_tt_populate()  → 驱动回调: 分配实际物理页面
ttm_tt_unpopulate()→ 驱动回调: 释放物理页面(但保留结构)
ttm_tt_destroy()   → 驱动回调: 释放 ttm_tt 结构本身
```

### ttm_pool — 页面缓存池

TTM 内建页面池，避免频繁 `alloc_page` / `free_page`：

```c
// drivers/gpu/drm/ttm/ttm_pool.c
// Xe 的 xe_ttm_tt_populate 调用:
ttm_pool_alloc(&xe->ttm.pool, tt, ctx);
// 释放:
ttm_pool_free(&xe->ttm.pool, tt);
```

---

## 1.7 dma_resv 与 ww_mutex

每个 `ttm_buffer_object` 内嵌一个 `dma_resv`（通过 `drm_gem_object.resv`），它是 BO 并发访问的核心同步机制：

```
dma_resv
    └── ww_mutex lock      → 互斥保护（防死锁: wound-wait）
    └── fence list (RCU)   → 追踪 GPU 操作完成的 dma_fence
```

### ww_mutex 死锁预防原理

```
普通 mutex 场景（死锁）:
  线程A: lock(B1) → 等待 lock(B2)
  线程B: lock(B2) → 等待 lock(B1)   ← 死锁!

ww_mutex（wound-wait）:
  每个 lock 操作有 ww_acquire_ctx（时间戳）
  若检测到循环等待：较年轻的持有者被 wound（强制解锁并返回 -EDEADLK）
  受害者 cleanup 后慢路径重获锁
```

### 在 TTM 中的使用模式

```c
// 单个 BO 锁定（无死锁风险）
dma_resv_lock(&bo->ttm.base.resv, NULL);  // ctx=NULL

// 多个 BO 锁定（VM_BIND 路径，通过 drm_exec）
struct drm_exec exec;
drm_exec_init(&exec, DRM_EXEC_INTERRUPTIBLE_WAIT, 0);
drm_exec_until_all_locked(&exec) {
    drm_exec_prepare_obj(&exec, &bo->ttm.base, 1);
}
// ... 操作 BO ...
drm_exec_fini(&exec);
```

---

## 1.8 TTM LRU 与驱逐框架

TTM 维护每种内存类型的 LRU 列表，当内存不足时按 LRU 顺序驱逐：

```c
// 驱逐核心流程 (ttm_bo.c)
ttm_bo_evict_first(bdev, man, ctx)
    ├── 从 LRU 尾部取出 BO（最久未使用）
    ├── 调用 bo->bdev->funcs->evict_flags() → 获取驱逐目标 placement
    ├── ttm_bo_validate(bo, evict_placement, ctx) → 触发实际搬移
    │       └── 调用 bo->bdev->funcs->move()
    └── 更新 LRU

// BO 优先级（Xe 中）
#define XE_BO_PRIORITY_NORMAL   1
bo->ttm.priority = XE_BO_PRIORITY_NORMAL;
// 数字越小 → 越先被驱逐
```

### TTM 驱逐整体状态机

```mermaid
stateDiagram-v2
    [*] --> SYSTEM : bo_create (无数据)
    SYSTEM --> TT : populate + DMA map
    TT --> VRAM : GPU blit upload
    VRAM --> TT : evict (GPU blit download)
    TT --> SYSTEM : evict (unmap DMA)
    SYSTEM --> [*] : bo_destroy
    VRAM --> [*] : bo_destroy
    TT --> [*] : bo_destroy

    note right of VRAM : GPU 最快访问
    note right of TT : CPU+GPU 都能访问
    note right of SYSTEM : CPU 可访问 GPU 不能直接用
```

---

## 架构总图

```mermaid
graph TD
    subgraph TTM_Core["TTM Core (drivers/gpu/drm/ttm/)"]
        ttm_dev["ttm_device\n设备上下文\n含 man_drv[] 数组"]
        ttm_bo_core["ttm_bo.c\nvalidate/evict/move\nLRU 管理"]
        ttm_pool["ttm_pool.c\n物理页面缓存池"]
        ttm_resource_h["ttm_resource.h\nttm_resource_manager\nttm_resource"]
    end

    subgraph TTM_Headers["TTM Headers (include/drm/ttm/)"]
        ttm_bo_h["ttm_bo.h\nttm_buffer_object\nenum ttm_bo_type"]
        ttm_placement_h["ttm_placement.h\nttm_placement\nttm_place\nTTM_PL_*"]
        ttm_device_h["ttm_device.h\nttm_device\nttm_device_funcs"]
        ttm_tt_h["ttm_tt.h\nttm_tt\nTTM_TT_FLAG_*"]
    end

    subgraph Sync["同步机制"]
        dma_resv["dma_resv\n(per-BO resv 对象)"]
        ww_mutex["ww_mutex\n防死锁互斥锁"]
        dma_fence["dma_fence\nGPU 操作异步完成信号"]
    end

    ttm_dev --> ttm_resource_h
    ttm_dev --> ttm_bo_core
    ttm_bo_core --> ttm_pool
    ttm_bo_h --> dma_resv
    dma_resv --> ww_mutex
    dma_resv --> dma_fence
    ttm_bo_h --> ttm_device_h
```

---

## 关键常量速查

```c
// 内存类型 ID
TTM_PL_SYSTEM  = 0   XE_PL_SYSTEM = TTM_PL_SYSTEM
TTM_PL_TT      = 1   XE_PL_TT     = TTM_PL_TT
TTM_PL_VRAM    = 2   XE_PL_VRAM0  = TTM_PL_VRAM
                     XE_PL_VRAM1  = 3
                     XE_PL_STOLEN = 8 (TTM_NUM_MEM_TYPES-1)

// LRU 优先级范围
TTM_MAX_BO_PRIORITY = 4   (0..3, 数字小的先被驱逐)

// Xe 默认优先级
XE_BO_PRIORITY_NORMAL = 1
```

---

## 1.9 用户态到 XE TTM 的完整路径图解

本节用五张图从不同维度展示用户态操作如何流经 DRM / XE 驱动最终落地到 TTM 内存管理层。

---

### 图 1：整体分层架构（用户态 → TTM）

```mermaid
flowchart TD
    subgraph US["用户态 (User Space)"]
        A1["DRM_IOCTL_XE_GEM_CREATE\n创建 BO，指定 placement / cpu_caching"]
        A2["DRM_IOCTL_XE_GEM_MMAP_OFFSET\n获取 fake mmap offset"]
        A3["mmap(drm_fd, offset)\nCPU 虚拟地址映射 BO"]
        A4["DRM_IOCTL_XE_VM_BIND\n将 BO 映射进 GPU VA 空间"]
        A5["DRM_IOCTL_XE_EXEC\n提交 GPU 命令"]
    end

    subgraph DRM["DRM 核心层 (drm_core)"]
        B1["drm_gem_object_create()\nGEM 对象分配 + handle 注册"]
        B2["drm_vma_node_offset_addr()\n虚拟节点 offset 查询"]
        B3["ttm_bo_mmap_obj()\n设置 vm_ops (ttm_bo_vm_ops)"]
    end

    subgraph XE["XE 驱动层 (drivers/gpu/drm/xe/)"]
        C1["xe_bo_create()\nxe_bo 初始化 + placement 构建\nadd_vram() / try_add_system()"]
        C2["xe_ttm_tt_create()\nttm_tt 分配 + cpu_caching 选择"]
        C3["xe_ttm_io_mem_reserve()\nbus.addr / bus.caching 填充"]
        C4["xe_ttm_io_mem_pfn()\nxe_res_cursor 逐页 PFN 计算"]
        C5["xe_vm_bind()\nxe_pt_stage_bind()\nPPGTT PTE 写入"]
        C6["xe_ttm_vram_mgr_new()\ndrm_buddy 块分配\nvisible_avail 更新"]
    end

    subgraph TTM["TTM 核心层 (drivers/gpu/drm/ttm/)"]
        D1["ttm_bo_init()\nttm_buffer_object 初始化\nLRU 插入"]
        D2["ttm_bo_validate()\nplacement 检查\n触发 move / evict"]
        D3["ttm_pool_alloc()\n系统内存页面分配\nDMA map"]
        D4["ttm_bo_vm_fault()\nttm_bo_vm_fault_reserved()\n缺页处理"]
        D5["ttm_io_prot()\nttm_prot_from_caching()\npgprot_writecombine()"]
    end

    subgraph HW["硬件层"]
        E1["系统内存 (DRAM)\n通过 IOMMU → GPU SMEM"]
        E2["VRAM (LMEM / dGPU)\nBAR2 窗口 / GPU 直接访问"]
        E3["PPGTT PTE 指向 VRAM PA\nGPU CS / EU 读写"]
    end

    A1 --> B1 --> C1 --> D1
    C1 --> C2
    D1 --> D2
    D2 --> C6
    D2 --> D3
    C6 --> E2
    D3 --> E1
    A2 --> B2
    A3 --> B3 --> D4 --> D5 --> E2
    C3 --> D5
    C4 --> D4
    A4 --> C5 --> E3
    A5 --> E3
```

---

### 图 2：系统内存（SYSMEM）BO 创建完整序列

CPU 内存路径：用户态请求分配在系统内存的 BO。

```mermaid
sequenceDiagram
    participant APP as 用户态应用
    participant GEM as drm_gem / xe_bo.c
    participant TTM as ttm_bo.c
    participant TT  as ttm_pool.c / xe_ttm_tt
    participant IOMMU as IOMMU / DMA
    participant DRAM as 系统 DRAM

    APP->>GEM: ioctl(DRM_IOCTL_XE_GEM_CREATE)\n.placement=SYSMEM, .cpu_caching=WB
    GEM->>GEM: xe_bo_create()\n__xe_bo_placement_for_flags()\ntry_add_system() → place.mem_type=XE_PL_TT
    GEM->>TTM: ttm_bo_init(bo, &placement)
    TTM->>TT: xe_ttm_tt_create(bo)\nalloc ttm_tt\ncaching=ttm_cached(WB)
    Note over TTM,TT: ttm_tt 此时仅是描述符\n尚未分配物理页面
    TTM->>TTM: ttm_bo_validate(bo, placement)\n检查当前 resource 是否满足 placement
    TTM->>TT: ttm_tt_populate(bo->ttm)\n→ ttm_pool_alloc()
    TT->>DRAM: alloc_page(GFP_KERNEL) × N\n物理页面分配
    TT->>IOMMU: dma_map_sgtable(sg, DMA_BIDIRECTIONAL)\nIOMMU 建立 IOVA → PA 映射
    IOMMU-->>TT: sg 中填入 IOVA
    TTM-->>GEM: 返回 ttm_resource (mem_type=TT)
    GEM-->>APP: 返回 GEM handle (u32)
    Note over APP: bo 现在在系统内存\nCPU 可直接缺页访问\nGPU 通过 IOMMU 访问 IOVA
```

---

### 图 3：VRAM（GPU 显存）BO 创建完整序列

GPU 内存路径：用户态请求分配在 dGPU 显存的 BO。

```mermaid
sequenceDiagram
    participant APP as 用户态应用
    participant XE  as xe_bo.c / add_vram()
    participant TTM as ttm_bo.c / ttm_bo_validate
    participant VMGR as xe_ttm_vram_mgr.c
    participant BUDDY as drm_buddy.c
    participant VRAM as 物理 VRAM (LMEM)

    APP->>XE: ioctl(DRM_IOCTL_XE_GEM_CREATE)\n.placement=VRAM, .flags=NEEDS_VISIBLE_VRAM
    XE->>XE: add_vram(bo, &place)\nio_size < usable_size?\n  YES(small-BAR): lpfn=io_size>>PAGE_SHIFT\n  NO (rBAR): place 无约束
    XE->>TTM: ttm_bo_init(bo, &placement)
    TTM->>TTM: ttm_bo_validate(bo, &placement)\nman = xe_ttm_vram_mgr(XE_PL_VRAM0)
    TTM->>VMGR: man->func->alloc(man, bo, place, &res)\n→ xe_ttm_vram_mgr_new()
    VMGR->>VMGR: mutex_lock(&mgr->lock)\ncheck: lpfn <= visible_size && size > visible_avail?\n  YES: -ENOSPC (触发驱逐)\n  NO: 继续
    VMGR->>BUDDY: drm_buddy_alloc_blocks(mm,\nfpfn<<PAGE_SHIFT, lpfn<<PAGE_SHIFT,\nsize, min_page_size, flags)
    BUDDY->>VRAM: 从 drm_mm 空闲链表分配物理块
    BUDDY-->>VMGR: drm_buddy_block 链表
    VMGR->>VMGR: 计算 used_visible_size\nmgr->visible_avail -= used_visible_size
    VMGR-->>TTM: ttm_resource (mem_type=VRAM, start=PA>>PAGE_SHIFT)
    TTM-->>XE: bo->resource 更新为 VRAM
    XE-->>APP: GEM handle (u32)
    Note over APP: bo 现在在 VRAM\n通过 BAR2 CPU mmap 或 VM_BIND GPU 访问
```

---

### 图 4：CPU 通过 mmap/BAR2 访问 VRAM（缺页处理序列）

```mermaid
sequenceDiagram
    participant APP  as 用户态应用
    participant LIBC as glibc / 系统调用
    participant DRM  as drm_gem_ttm_helper
    participant TTM  as ttm_bo_vm.c
    participant XE   as xe_bo.c (xe_ttm_io_mem_pfn)
    participant PAT  as ttm_module.c (ttm_prot_from_caching)
    participant MMU  as CPU MMU / TLB
    participant BAR2 as PCIe BAR2 → 物理 VRAM

    APP->>LIBC: ioctl(DRM_IOCTL_XE_GEM_MMAP_OFFSET, {handle})
    LIBC-->>APP: mmo.offset (fake vma 偏移)
    APP->>LIBC: mmap(NULL, SIZE, PROT_READ|PROT_WRITE,\nMAP_SHARED, drm_fd, mmo.offset)
    LIBC->>DRM: drm_gem_mmap() → ttm_bo_mmap_obj()
    DRM->>TTM: vma->vm_ops = &ttm_bo_vm_ops\nvm_flags |= VM_PFNMAP | VM_IO
    Note over DRM,TTM: 此时仅设置 vm_ops\n未分配任何 PTE，WC 属性也未设置

    APP->>APP: *cpu_ptr = value  ← 触发缺页
    MMU->>TTM: 缺页中断 → ttm_bo_vm_fault()
    TTM->>TTM: ttm_bo_vm_reserve(bo)\ndma_resv_lock(bo->base.resv)
    TTM->>TTM: ttm_bo_vm_fault_reserved(vmf, vma->vm_page_prot)
    TTM->>TTM: ttm_mem_io_reserve(bdev, bo->resource)\n填充 bus.addr / bus.is_iomem
    TTM->>PAT: ttm_io_prot(bo, bo->resource, prot)\n→ caching = res->bus.caching\n   = ttm_write_combined
    PAT->>PAT: ttm_prot_from_caching(ttm_write_combined, prot)\n→ pgprot_writecombine(prot)
    PAT-->>TTM: WC pgprot_t
    TTM->>XE: ttm_bo_io_mem_pfn(bo, page_offset)\n→ xe_ttm_io_mem_pfn(bo, page_offset)\n→ xe_res_cursor 遍历 drm_buddy blocks\n→ pfn = (io_start + cursor.start) >> PAGE_SHIFT
    XE-->>TTM: pfn (BAR2 物理页帧号)
    TTM->>MMU: vmf_insert_pfn_prot(vma, addr, pfn, WC_prot)\n写入 CPU PTE: VA → BAR2 PA (WC)
    MMU-->>APP: 缺页处理完成，CPU PTE 已建立
    APP->>BAR2: CPU 写操作 → WC 合并 → PCIe TLP → 物理 VRAM
    Note over BAR2: 数据直接落入 VRAM 物理页\nGPU 后续通过 PPGTT 读同一物理页
```

---

### 图 5：GPU 通过 PPGTT VM_BIND 访问 VRAM（GPU VA → 物理 VRAM）

```mermaid
sequenceDiagram
    participant APP  as 用户态应用
    participant XEV  as xe_vm.c / xe_vm_bind
    participant PT   as xe_pt.c / xe_pt_stage_bind
    participant VMGR as xe_ttm_vram_mgr
    participant PPGTT as PPGTT (GPU 页表 BO in VRAM)
    participant CS   as GPU Command Streamer
    participant VRAM as 物理 VRAM

    APP->>XEV: ioctl(DRM_IOCTL_XE_VM_BIND)\n.op=MAP, .bo_handle=h, .addr=GPU_VA, .range=SIZE
    XEV->>XEV: xe_vm_bind(vm, vma, bo, bp)\n验证 GPU_VA 对齐 / 范围
    XEV->>VMGR: xe_bo_validate(bo)\n确保 bo 在 VRAM 中 (ttm_bo_validate)
    VMGR-->>XEV: bo->resource = VRAM ttm_resource\nstart = VRAM_PA >> PAGE_SHIFT

    XEV->>PT: xe_pt_update_ops_run()\n→ xe_pt_stage_bind(vm, vma, bo)
    PT->>PT: 遍历 GPU VA 范围 (4K / 2M / 1G 页)\n对每个 entry:
    PT->>PT: pte = vm->pt_ops->pte_encode_bo(bo, offset, pat_index, level)\n= VRAM_PA | PAT_WB | VALID | ...
    Note over PT: VRAM_PA = drm_buddy_block_offset(block)\n+ vram_region_gpu_offset(resource)\n此 PA 与 BAR2 窗口覆盖同一物理 VRAM 页
    PT->>PPGTT: xe_map_wr(xe, &pt_bo->vmap, pte_offset, u64, pte)\n将 PTE 写入 PPGTT 页表 BO (在 VRAM 中)
    PT-->>XEV: 页表更新完成

    XEV->>CS: xe_vm_tlb_invalidate(vm)\n发送 TLB 失效命令 (via GuC CT 或 MMIO)
    CS-->>XEV: TLB 已失效

    XEV-->>APP: VM_BIND 完成

    APP->>CS: ioctl(DRM_IOCTL_XE_EXEC)\n提交包含访问 GPU_VA 的命令 batch
    CS->>PPGTT: GPU MMU 查 PPGTT: GPU_VA → VRAM_PA
    PPGTT-->>CS: PTE: VRAM_PA + cache 属性
    CS->>VRAM: 读写物理 VRAM page (同 CPU BAR2 写入的同一物理页)
    Note over CS,VRAM: GPU RD/WR 直接访问 LMEM 控制器\n不经过 PCIe BAR2
```

---

### 图 6：CPU 内存 vs GPU 内存完整对比（双路径汇总）

```mermaid
flowchart LR
    subgraph USPACE["用户态"]
        GC["GEM_CREATE\n.placement"]
        MM["mmap\n(BAR2 CPU 访问)"]
        VB["VM_BIND\n(PPGTT GPU 访问)"]
        EX["EXEC\n(GPU 命令执行)"]
    end

    subgraph SYS_PATH["系统内存路径 (SYSMEM / TT)"]
        SP1["try_add_system()\nplace.mem_type = XE_PL_TT"]
        SP2["xe_ttm_tt_create()\ncaching = ttm_cached (WB)"]
        SP3["ttm_pool_alloc()\nalloc_page() × N"]
        SP4["dma_map_sgtable()\nIOMMU: IOVA → PA"]
        SP5["物理 DRAM 页面\nCPU 直接访问 (WB)"]
        SP6["GPU 通过 IOMMU C R/W"]
    end

    subgraph VRAM_PATH["VRAM 路径 (dGPU LMEM)"]
        VP1["add_vram()\nio_size < usable?\n→ lpfn / TOPDOWN"]
        VP2["xe_ttm_vram_mgr_new()\ndrm_buddy_alloc_blocks()"]
        VP3["物理 VRAM 块\ndrm_buddy_block"]
        VP4["BAR2 窗口\nio_start + block_offset\n→ PCIe TLP (WC PTE)"]
        VP5["PPGTT PTE\nVRAM_PA | PAT_WB\nGPU 直接读写 LMEM"]
    end

    GC -->|mem_type=TT| SP1 --> SP2 --> SP3 --> SP4 --> SP5
    SP4 --> SP6
    GC -->|mem_type=VRAM| VP1 --> VP2 --> VP3
    MM -->|mmap fault\nttm_io_prot=WC| VP4
    VP3 --> VP4
    VB -->|pte_encode_bo\nVRAM_PA| VP5
    VP3 --> VP5
    EX --> VP5

    style SYS_PATH fill:#e8f4e8,stroke:#4a8a4a
    style VRAM_PATH fill:#e8e8f8,stroke:#4a4a8a
```
