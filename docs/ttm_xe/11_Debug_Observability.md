# Part 11: 调试与可观测性

> **Source files**:  
> - `drivers/gpu/drm/xe/xe_bo.c`  
> - `drivers/gpu/drm/xe/xe_ttm_vram_mgr.c`  
> - `drivers/gpu/drm/xe/xe_ttm_sys_mgr.c`  
> - `drivers/gpu/drm/xe/xe_debugfs.c`  
> - `drivers/gpu/drm/xe/xe_bo_evict.c`

---

## 11.1 内存类型名称映射

`xe_mem_type_to_name[]` 数组将 TTM 内存类型常量转换为可读字符串：

```c
// xe_bo.c:44
const char * const xe_mem_type_to_name[TTM_NUM_MEM_TYPES] = {
    [XE_PL_SYSTEM]  = "system",    // [0] CPU DRAM, 无 DMA 映射
    [XE_PL_TT]      = "tt",        // [1] CPU DRAM + DMA 映射
    [XE_PL_VRAM0]   = "vram0",     // [2] tile 0 VRAM（本地显存）
    [XE_PL_VRAM1]   = "vram1",     // [3] tile 1 VRAM（多 tile dGPU）
    [4]             = NULL,
    [5]             = NULL,
    [6]             = NULL,
    [7]             = NULL,
    [XE_PL_STOLEN]  = "stolen",    // [8] Stolen 系统内存/VRAM 分区
};
// TTM_NUM_MEM_TYPES = 9
```

用途：
- `dmesg / drm_dbg` 打印 BO 当前位置
- `debugfs` 输出内存分布信息
- 内存统计和监控工具

---

## 11.2 debugfs 节点

### 11.2.1 TTM 通用节点

TTM 框架自动注册：

```
/sys/kernel/debug/dri/<N>/
    ttm_resource_manager     ← 所有 ttm_resource_manager 统计
        内容示例:
        "XE_PL_TT: size = 32768 MB, used = 1024 MB"
        "XE_PL_VRAM0: size = 8192 MB, used = 4096 MB"
```

```bash
# 查看所有内存类型使用量
cat /sys/kernel/debug/dri/0/ttm_resource_manager
```

### 11.2.2 `xe_vram_mm` — VRAM Buddy 分配器状态

```c
// xe_ttm_vram_mgr.c: xe_ttm_vram_mgr_debug()
static void xe_ttm_vram_mgr_debug(struct ttm_resource_manager *man,
                                   struct drm_printer *printer)
{
    struct xe_ttm_vram_mgr *mgr = to_xe_ttm_vram_mgr(man);

    mutex_lock(&mgr->lock);
    drm_printf(printer, "default_page_size: %llu KB\n",
               mgr->default_page_size >> 10);
    drm_printf(printer, "visible_size: %llu MB\n",
               mgr->visible_size >> 20);
    drm_printf(printer, "visible_avail: %llu MB\n",
               mgr->visible_avail >> 20);
    drm_printf(printer, "visible_used: %llu MB\n",
               (mgr->visible_size - mgr->visible_avail) >> 20);

    // drm_buddy print: 显示 buddy 树状态（空闲/已用 block）
    drm_buddy_print(&mgr->mm, printer);
    mutex_unlock(&mgr->lock);
}
```

```bash
# VRAM buddy 分配器详细状态
cat /sys/kernel/debug/dri/0/vram0_mm

# 输出示例:
# default_page_size: 64 KB
# visible_size: 256 MB
# visible_avail: 128 MB
# visible_used: 128 MB
# Buddy system:
#   total: 8192 MB
#   free blocks: [2048MB, 1024MB, 512MB, ...]
#   used blocks: [...]
```

### 11.2.3 `xe_gem_objects` — GEM 对象列表

```bash
# 列出所有活动 GEM 对象（BO）
cat /sys/kernel/debug/dri/0/gem_names

# 或通过 xe 专属节点（若存在）
cat /sys/kernel/debug/dri/0/xe_gem_objects
```

输出典型格式：
```
handle=3 size=1048576 mem_type=vram0 flags=0x41 pin_count=1 cpu_caching=WB
handle=4 size=4096    mem_type=tt    flags=0x02 pin_count=0 cpu_caching=WC
```

---

## 11.3 dmesg 调试输出

### 11.3.1 BO 移动追踪

当 `drm.debug=0x02`（DRM_UT_DRIVER）时，可以看到 BO 迁移日志：

```bash
# 启用 DRM 驱动调试
echo 0x02 > /sys/module/drm/parameters/debug

# 观察 dmesg
dmesg | grep "xe.*bo.*move\|xe.*migrate\|xe.*evict"
```

典型输出：
```
[  42.123] xe 0000:00:02.0: [drm:xe_bo_move] BO 0x... system -> tt (ttm)
[  42.124] xe 0000:00:02.0: [drm:xe_bo_move] BO 0x... tt -> vram0 (gpu blit)
[  42.234] xe 0000:00:02.0: [drm:xe_bo_evict] evicting 0x... from vram0 -> tt
```

### 11.3.2 内存压力/驱逐追踪

```bash
# 监控驱逐活动（需要 CONFIG_DRM_XE_DEBUG=y）
dmesg | grep -i "evict\|enospc\|ttm"

# 相关内核配置
CONFIG_DRM_XE_DEBUG=y      # 启用 Xe 调试断言
CONFIG_DRM_TTM_DEBUG=y     # 启用 TTM 核心调试
```

### 11.3.3 GuC 内存相关错误

```bash
# GuC 固件加载失败（可能与 Stolen 大小有关）
dmesg | grep "GuC\|HuC\|stolen\|wopcm"

# 典型 Stolen 问题日志:
# xe 0000:00:02.0: GuC: stolen size 0, disabled
# xe 0000:00:02.0: Failed to probe stolen memory region
```

---

## 11.4 `intel_gpu_top` — GPU 内存监控

```bash
# 实时查看 GPU 内存使用（需要 intel-gpu-tools）
intel_gpu_top

# 输出包括:
# - VRAM 使用量（总量/已用）
# - GTT 使用量
# - GPU 引擎利用率

# 也可查看 sysfs 节点
cat /sys/class/drm/card0/device/drm/card0/device/mem_info_vram_total
cat /sys/class/drm/card0/device/drm/card0/device/mem_info_vram_used
cat /sys/class/drm/card0/device/drm/card0/device/mem_info_gtt_total
cat /sys/class/drm/card0/device/drm/card0/device/mem_info_gtt_used
```

---

## 11.5 `perf` 内存带宽监控

```bash
# GPU uncore 内存带宽事件（需要 CONFIG_PERF_EVENTS）
# Intel GPU 暴露 PMU（Performance Monitoring Unit）

# 查看可用 PMU 事件
perf list | grep xe

# 监控 VRAM 读写带宽（示例）
perf stat -e xe/vram-bandwidth-read/,xe/vram-bandwidth-write/ \
    -I 1000 sleep 10
```

---

## 11.6 IGT 测试用例

以下 IGT 测试直接覆盖 TTM/xe 内存管理：

### 11.6.1 基础 BO 测试

```bash
# BO 创建/销毁
igt_runner igt@xe_create_destroy

# BO 大小/标志边界测试
igt_runner igt@xe_bo_flags

# 各内存区域（vram/system/stolen）的基本分配
igt_runner igt@xe_mmap
```

### 11.6.2 内存压力与驱逐

```bash
# 填满 VRAM，强制触发驱逐路径
igt_runner igt@xe_evict

# 多进程竞争分配
igt_runner igt@xe_evict_many_threads

# pin 住 BO 后驱逐（验证 pin_count 保护）
igt_runner igt@xe_evict_pinned
```

### 11.6.3 VM Bind 与 TTM 联动

```bash
# VRAM BO 绑定到 GPU 虚地址后迁移
igt_runner igt@xe_vm_bind

# BO 在 TT 和 VRAM 之间迁移
igt_runner igt@xe_migrate

# DMA fence 等待机制
igt_runner igt@gem_exec_fence
```

### 11.6.4 系统挂起恢复

```bash
# S3 挂起/恢复（BO 内容持久性验证）
igt_runner igt@xe_pm_s3

# D3cold 设备电源循环
igt_runner igt@xe_pm_d3cold
```

---

## 11.7 KASAN / KMEMLEAK 配合调试

```bash
# 使用 KASAN 检测 UAF（Use After Free）和越界访问
# 在内核配置开启：
# CONFIG_KASAN=y
# CONFIG_KASAN_GENERIC=y

# 运行可疑测试
igt_runner igt@xe_evict
dmesg | grep "KASAN\|BUG: KASAN"

# 使用 KMEMLEAK 检测内存泄漏
echo "scan" > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak | grep xe_bo
```

---

## 11.8 常见问题诊断

### 问题 1：ENOSPC 分配失败

```
现象: "drm_buddy_alloc_blocks: No space left"
原因: VRAM 满 / BAR 可见区满（small BAR 设备）

诊断:
  cat /sys/kernel/debug/dri/0/ttm_resource_manager
  cat /sys/kernel/debug/dri/0/vram0_mm

检查:
  1. visible_avail 是否充足
  2. 是否有 NEEDS_CPU_ACCESS + 大 BO 导致 BAR 耗尽
```

### 问题 2：迁移超时

```
现象: GPU hang 在 BCS 迁移期间
原因: BLT 命令超时 / GuC 调度问题

诊断:
  dmesg | grep "GPU HANG\|xe.*timeout\|guc.*hang"

检查:
  1. MAX_PREEMPTDISABLE_TRANSFER 是否过大（> 1ms 规则）
  2. min_chunk_size 对齐是否正确
  3. xe migrate VM 页表是否正确映射
```

### 问题 3：Stolen 内存不可用

```
现象: "Failed to init stolen memory manager"
原因: BIOS 未分配 Stolen / GMS 字段为 0

诊断:
  dmesg | grep -i stolen
  # 读取 GGC 寄存器值:
  sudo intel_reg read 0x50 --graphics-stall 2>/dev/null || \
  sudo cat /sys/kernel/debug/dri/0/gt0/registers | grep -i ggc

检查:
  1. 确认 BIOS 中 iGPU 已开启并分配 Stolen 内存
  2. WOPCM 大小是否与 Stolen 剩余大小兼容
```

---

## 11.9 内存调试快速参考表

| 目标 | 工具/命令 | 关键输出 |
|------|---------|---------|
| VRAM 使用量 | `cat /sys/kernel/debug/dri/0/ttm_resource_manager` | 各类型 used/total |
| VRAM 碎片 | `cat /sys/kernel/debug/dri/0/vram0_mm` | buddy 树空闲块列表 |
| 活动 BO 列表 | `cat /sys/kernel/debug/dri/0/gem_names` | handle/size/location |
| 迁移活动 | `dmesg \| grep xe_bo_move` | 各 BO 迁移日志 |
| 驱辞活动 | `dmesg \| grep evict` | 驱逐源/目的类型 |
| BAR 可见区 | `cat /sys/kernel/debug/dri/0/vram0_mm` | visible_avail |
| GPU 内存实时 | `intel_gpu_top` | VRAM/GTT 实时图表 |
| BO 泄漏 | `kmemleak \| grep xe_bo` | 未释放的 xe_bo 对象 |
