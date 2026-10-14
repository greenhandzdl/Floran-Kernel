# Floran Kernel Docker 支持 — 完整排查报告

**设备**: OnePlus Ace 3 Pro (OP5D06L1)
**ROM**: crDroid 16 (Android 15)
**内核**: GKI 6.1 (Floran-Kernel, ReSukiSU + SUSFS)
**日期**: 2026-09-25 ~ 2026-10-13

---

## 一、问题全景

| # | 问题 | 现象 | 最终状态 |
|---|------|------|----------|
| 1 | 内核不支持 Docker | `check-config.sh` 大量 missing | ✅ 已修复 |
| 2 | Docker 配置导致 bootloop | 刷入后卡第一屏 | ✅ 已修复 |
| 3 | 开机时间 375 秒 | boot_completed 需 ~375s | ✅ 已修复 (→40s) |
| 4 | 电池检测失败 | present=0, level=0 | ✅ 已修复 |
| 5 | Loop 挂载 I/O error | 50G 稀疏镜像挂载失败 | ✅ 已修复 |

---

## 二、Phase 1 — 内核 Docker 支持 (Day 1-3)

### 2.1 初始状态

用户运行 `check-config.sh` 发现大量 Docker 所需内核配置缺失：
- cgroup 控制器: `CGROUP_DEVICE`, `CGROUP_PIDS`, `CGROUP_PERF` 等 missing
- 网络: `NF_TABLES`, `IP_VS`, `BRIDGE_NETFILTER` 等 missing
- 存储: `BTRFS_FS` missing
- 已有: `NAMESPACES`, `MEMCG`, `OVERLAY_FS`, `EXT4_FS` (GKI 原生)

### 2.2 二分法定位 bootloop 元凶

首次加入全部 Docker/VM 配置后 **bootloop**。通过单变量控制法逐层缩小范围：

| 轮次 | 变量 | 结果 | 结论 |
|------|------|------|------|
| A | Baseline (ReSukiSU+SusFS, 无 Docker) | ✅ 开机 | 基线正常 |
| B | Docker 全开 | ❌ 卡 | 有问题 |
| C | Netfilter + CCM (无 Docker 核心) | ✅ 开机 | 网络无辜 |
| D | Docker 核心 (无网络) | ❌ 卡 | **元凶在 Docker 核心组** |
| E | Docker cgroup 簇 (9 项) | ❌ 卡 | cgroup 嫌疑 |
| F | Docker btrfs | ✅ 开机 | btrfs 无辜 |
| H | cgroup 核心 5 项 | ❌ 卡 | 两簇各有致命项 |
| G | cgroup 可选 4 项 | ❌ 卡 | 确认 |
| K1 | USER_NS + CFS_BANDWIDTH | ❌ 卡 | 需要更精简 |
| K2 | 仅 USER_NS | ❌ 卡 | |
| K3 | 仅 CFS_BANDWIDTH | ❌ 卡 | |

### 2.3 最终可用配置

经过 cgroup v2 分析，踢掉 v1 遗留项后确定最小安全集：

**保留的 Docker 配置**:
- `docker.config` 安全底座 (NAMESPACES, SECCOMP, VETH, BRIDGE, OVERLAY 等 — 多为 GKI 原生)
- `NF_TABLES` 相关 (iptables 替代)

**删除的配置**:
- `CGROUP_DEVICE` (v1 遗留, v2 走 BPF)
- `CGROUP_PIDS` (非必需)
- `CGROUP_PERF`, `CGROUP_HUGETLB`, `NET_CLS_CGROUP` (冷僻 v1 控制器)
- `BTRFS_FS` (用户不需要)
- `fix_cgroup.patch` 不再默认应用

**结果**: Docker 以 `--network=host` + `vfs` 存储驱动正常运行。

---

## 三、Phase 2 — 开机 375 秒排查 (Day 4-6)

### 3.1 发现慢启动

安装 repotitan 的 dockerd 模块后，开机时间从正常飙升到 **375 秒**。

### 3.2 排查过程

1. **排除脚本阻塞**: service.d 脚本都是后台执行，init 不等它们
2. **排除编译/落盘**: `/data/misc`, `/data/system` 等 mtime 零变化
3. **排除 fix_cgroup.patch**: 刷无该补丁的版本仍然 375s
4. **早期 dmesg 快照**: 在 t≈380s 拍第一张 dmesg，发现 **174 万条 SELinux AVC**

### 3.3 根因锁定

```
[398.835][T1] selinux: Could not set context for /data/adb/docker/lib/docker/image/vfs/...
[398.876][T1] selinux: Could not stat /data/adb/docker/run/docker.sock: Permission denied
```

```
t+380.3s [T1] init: Command 'restorecon --recursive --skip-ce /data'
              action=post-fs-data (/system/etc/init/hw/init.rc:1074)
              took 375256ms and succeeded
```

**根因**: AOSP init 的 `restorecon -R /data` 递归重标注，遍历 docker VFS 存储驱动的 **174 万个文件**（41GB），每个文件尝试 `setfilecon` → 被 SELinux 策略拒绝 → 每条打 2 条 printk。**375 秒 = 遍历 174 万文件的纯 I/O 开销**。

### 3.4 量化验证

| 实验 | 结果 |
|------|------|
| `find /data/adb/docker` 条目数 | 1,746,208 个 |
| 冷缓存 `restorecon -R -n /data/adb/docker` | 383 秒 (≈375s) |
| 修好标签后重启 | 仍 417s (遍历成本不变) |
| `docker system df` | 10 镜像, 6.89G 内容, 5.51G 可回收 |

### 3.5 解决方案: Loop Ext4 镜像

将 docker data-root 整体迁入 loop 挂载的 ext4 镜像文件。init 的 relabel 只看到"一个空目录 + 1 个镜像文件"，375 秒归零。

```bash
# 创建 50G 镜像
dd if=/dev/zero of=/data/adb/docker-data.img bs=1M count=51200
mkfs.ext4 /data/adb/docker-data.img

# 挂载 (关键: SELinux context 避免 restorecon)
DEV=$(losetup -f)
losetup $DEV /data/adb/docker-data.img
mount -t ext4 -o "context=\"u:object_r:adb_data_file:s0\"" \
    $DEV /data/adb/docker/lib/docker
```

**结果**: 开机 375s → **40s**。

---

## 四、Phase 3 — 电池检测修复 (Day 6)

### 4.1 现象

开机后 `dumpsys battery` 显示 `present: false, level: 0`。

### 4.2 根因

开机 375 秒时，充电驱动（OPLUS_CHG / MMS_GAUGE）在 3000 次重试内耗尽预算，注册失败。开机提速到 40s 后，充电栈在预算内完成注册。

**结果**: `present=1, level=100`。电池问题和慢启动是**同一个病根**。

---

## 五、Phase 4 — Loop 挂载 I/O Error (Day 7-10)

### 5.1 现象

重启后 docker-fix 模块的 action 按钮显示"挂载失败，需重启"。内核日志：
```
I/O error, dev loop42, sector 2
EXT4-fs (loop42): unable to read superblock
```

### 5.2 排查过程

| 实验 | 结果 | 结论 |
|------|------|------|
| 直接读镜像文件 (dd) | 正常, ext4 magic 53ef 可读 | 文件本身完好 |
| 小镜像 (10MB) loop 挂载 | 成功 | loop 驱动本身正常 |
| 50G 稀疏镜像 loop 挂载 | I/O error | 稀疏文件问题 |
| 清理 stale loop 设备后重试 | 仍失败 | 不是 loop 残留 |
| `cp` 转非稀疏后当前开机挂载 | 仍失败 | 当前开机周期 loop 状态损坏 |
| **转非稀疏后重启挂载** | **成功** | **根因确认** |

### 5.3 根因

**F2FS 文件系统的垃圾回收 (GC) 机制与 loop 驱动对稀疏文件的读取不兼容**。

- 稀疏文件的"空洞"(holes) 在 F2FS 上的物理块分布导致 loop 驱动无法正确读取
- 直接读取文件正常（F2FS 自己处理空洞），但 loop 驱动通过块设备接口读取时失败
- 小文件不受影响（空洞少，GC 干扰小）

### 5.4 解决方案

将稀疏镜像转换为非稀疏格式：

```bash
# 检测稀疏文件
blocks=$(stat -c %b $IMG); blocksize=$(stat -c %B $IMG)
actual=$((blocks * blocksize)); apparent=$(stat -c %s $IMG)
# actual < apparent → 稀疏文件

# 转换 (填充空洞)
cp $IMG ${IMG}.nosparse && mv ${IMG}.nosparse $IMG
```

**验证**: 转换后 50G actual = 50G apparent，重启后自动挂载成功。

---

## 六、Phase 5 — 模块封装与交付 (Day 10-13)

### 6.1 docker-fix 模块

| 文件 | 用途 |
|------|------|
| `action.sh` | KernelSU 按钮一键修复: 清理 stale loop → 挂载 → 启动 dockerd → 验证 |
| `service.sh` | boot_completed 后自动挂载 + 启动 dockerd (10 轮重试) |
| `post-fs-data.sh` | 开机早期快速挂载尝试 (2 轮) |
| `module.prop` | author=greenhandzdl, version=v1.0 |

### 6.2 Loop 设备编号坑

Android cmdline 带 `loop.max_part=7` 时，每个 loop 设备占 8 个次设备号：
- loopN 的真编号 = `N × 8` (loop42 = minor 336)
- **绝不能**用 `mknod /dev/block/loopN b 7 N` 造节点
- 正确做法: 让 `losetup -f` 自己建节点

### 6.3 GitHub 仓库

- **docker-fix 模块**: https://github.com/greenhandzdl/magisk-docker-fix
  - 作者: greenhandzdl
  - 前置要求: [mgksu/dockerd](https://github.com/mgksu/dockerd) 模块
  - 包含稀疏文件转换说明

- **Floran Kernel**: https://github.com/greenhandzdl/Floran-Kernel
  - 包含 module CRC bypass patch (修复蓝牙模块加载)
  - .gitignore 排除诊断文件

---

## 七、Phase 6 — 重启后空镜像与 dockerd 断连 (Day 14)

### 7.1 现象

- 第一次重启: `docker images` 显示 0 镜像，但镜像数据完好（imagedb 10 条、vfs 41G/100 层）
- 第二次重启（修了脚本后）: 更严重——CLI 直接 `Cannot connect to the Docker daemon`，而 dockerd 进程活着

### 7.2 根因链（三条并发叠加）

1. **ksud 执行 service.d / post-fs-data.d 下所有文件，不按 .sh 后缀过滤**。
   前期调试遗留、仅改名成 `.bak` 的旧挂载脚本（`dockerd_service.sh.bak`、
   `96-docker-imgmount.sh.bak`）开机照样执行，与 docker-fix 的 service.sh
   并发 losetup + mount 同一镜像 → **嵌套双重挂载**（loop42 + loop43 同挂
   `/data/adb/docker/lib/docker`）。两份开机日志同一秒（04:17:49）落盘即铁证。
2. **mgksu 模块的 dockerd.inotify 竞态**: 它监听 `disable` 文件——删除触发
   start、创建触发 stop。脚本里 `rm -f disable` 会并发拉起第二个 dockerd，
   且 `dockerd.service` 的 start 无 running 检查，直接 `nohup dockerd` →
   双实例抢 socket / pidfile。
3. **toybox pkill 只接受一个 pattern**: `pkill -9 dockerd containerd` 静默失败
   （多给的参数使整条命令无效），旧 dockerd 存活；随后脚本 `rm -f docker.sock`
   删掉的却是**活 daemon 正在监听的 socket** → CLI 永久断连。

> 早期误判记录: 曾归因于 "Android 无 mountpoint 命令导致已挂载检查失效"。
> 实测证伪——该设备 toybox 有 mountpoint 且工作正常，真凶是上面的并发竞态。

### 7.3 修复

- 设备端: 三个 `.bak` 遗留脚本 `mv` 出自动执行目录，隔离到 `/data/adb/docker/legacy/`
  （改后缀不算禁用，必须移出目录）
- docker-fix 脚本重写（v1.1 → v1.2）:
  - `mount_layers()` 用 `grep -c " $ROOT " /proc/mounts` 数挂载层数，>1 层时拆掉重建单一干净挂载
  - `kill_docker_all()`: 先 `pgrep -f dockerd.inotify` 杀监视器 → 按 `pgrep -x` 输出的 PID 逐个 `kill -9` → 轮询确认全部死亡
  - 只有进程死绝后才允许删 `docker.sock` / pidfile
  - 全流程保证只有一次 `start.sh` 调用，验证时显式 `export DOCKER_HOST`

### 7.4 重启实测（v1.2, 2026-10-14）

```
=== service 开始 04:28:41 ===
当前挂载层数: 0 → 挂载成功 (尝试 2)
dockerd/containerd 已停止 (等待 0 轮) → 启动 dockerd
30s 后镜像数: 10, 挂载层数: 1
```

- 挂载仅 1 层、loop 仅 1 个（loop42）
- dockerd / containerd 各 1 实例，CLI 连接正常
- 10 镜像 / 3 容器，nekro-nxt Up (healthy)

---

## 八、最终验证结果

| 指标 | 修复前 | 修复后 |
|------|--------|--------|
| 开机时间 | 375s | 40s → 133s (含 boot_completed) |
| 电池 | present=0, level=0 | present=1, level=100 |
| Docker 镜像 | 0 (挂载失败) | 10 镜像 / 3 容器 |
| nekro-nxt 容器 | 不可用 | healthy |
| Loop 挂载 | I/O error | 自动挂载成功 |
| VFS 层数 | N/A | 100 |
| 开机自动化 | 双重挂载/断连 | v1.2 单层挂载 + 单实例，重启验证通过 |

---

## 九、经验教训

1. **Android init 的 restorecon 是性能杀手**: 大量文件的目录树会被递归重标注，每次开机重复付代价。解决方案: 将数据放入 loop 镜像或使用正确的 SELinux context。

2. **F2FS + 稀疏文件 + loop 驱动不兼容**: 在 F2FS 分区上创建 loop ext4 镜像时，**不要用稀疏格式**。用 `dd if=/dev/zero` 而非 `dd seek=`。

3. **cgroup v2 下的 v1 遗留配置**: Android 使用 cgroup v2 unified hierarchy，`CGROUP_DEVICE` 等 v1 控制器不仅无用，还可能触发 vendor 模块的 struct 偏移问题。

4. **二分法需要单变量**: 多变量同时改变会导致错误归因。每一轮只改一个变量簇。

5. **Loop 设备编号**: `loop.max_part=7` 时次设备号计算是 `N×8`，手动 mknod 会造出错误节点。

6. **ksud 不筛后缀**: `/data/adb/service.d`、`post-fs-data.d` 下所有文件都会被执行，禁用旧脚本必须**移出目录**而不是改名 `.bak`。

7. **toybox pkill 只接受一个 pattern**: 多 pattern 写法整条静默失败。杀进程要用 `pgrep -x` 拿 PID 逐个 `kill -9`，并轮询确认死亡后才清理 socket/pidfile。

8. **inotify 守护会在你操作文件时抢先动手**: 脚本删 `disable` 文件会触发模块自身的 inotifyd 自动 start dockerd。做启停操作前先杀掉监视器，保证自己是唯一控制者。

9. **错误归因要留证据链复查**: mountpoint 缺失的初判被后续实测证伪——同一秒两份开机日志才暴露了真正的并发竞态。修复后要用最小可观测指标（挂载层数、loop 数、进程数）验证。

---

## 十、文件索引

| 路径 | 说明 |
|------|------|
| `patch/disable_module_crc_check.patch` | 模块 CRC 校验绕过 |
| `patch/fix_cgroup.patch` | cgroup v2 兼容 (仅 Droidspaces 时使用) |
| `log/` | 诊断脚本和日志 (gitignore) |
| `/data/adb/docker-data.img` | 50G 非稀疏 ext4 Docker data-root |
| `/data/adb/modules/docker-fix/` | docker-fix KernelSU 模块 (v1.2) |
| `/data/adb/docker/legacy/` | 隔离的旧开机脚本 (.bak)，防止 ksud 执行 |

---

*报告生成: 2026-10-13，更新: 2026-10-14 (Phase 6) | 作者: greenhandzdl*
