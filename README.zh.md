# Alpindroid  

[English](README.md) | [简体中文](README.zh.md)  

这是一个专为 H618 硬件（WalnutPi ZeroW / 核桃派 ZeroW）打造的 Linux 系统，它结合了官方 Debian 12 的引导链（boot chain）与 Alpine Linux 3.23 的用户空间，旨在支持 Waydroid（Android 容器）。  

它修复了核桃派官方 Debian 12 镜像中存在的 GPU 和供电缺陷，并将 1.1 GB 的 Debian 根文件系统（rootfs）替换为约 90 MB 的 Alpine 根文件系统。  

**无需编译 AOSP。** 项目的目标是构建一个轻量级的 Alpine 宿主机，并在其上运行 Android。  

自 **v0.22** 版本起，该目标已达成：Android (LineageOS 20) 在 LXC 容器中作为主 UI 运行，通过与 Weston 宿主机相同的 Panfrost/Mesa 图形栈进行渲染，并且无需任何网络连接即可启动。  
Android 系统的完整功能将逐步实现。  

---
## 状态：v0.18（稳定基线）
| 子系统 | 状态 | 关键实现 |
|---|---|---|
| 引导链 (Boot chain) | 正常工作 | 厂商的 `boot.bin` / `boot.scr` 保持原样；p1 分区存放我们的 `Image`、DTB 和一个 overlay（设备树覆盖），p2 分区被完全替换 |
| CPU 频率调节 | 正常工作 | `ondemand` 策略，480 MHz – 1.344 GHz，AXP313A `dcdc2` → `vdd-cpu` |
| 电源管理 | 正常工作 | 通过设备树覆盖（DT overlay）将 PMIC 正确识别为 `axp313a` |
| HDMI | 正常工作 | 基于 `sun4i-drm` 的 KMS 输出；最高支持 1080p 模式，此处测试的屏幕为 1024x600@60.1 |
| **WiFi** | 正常工作 | 紫光展锐 UNISOC uwe5622 (`sprdwl_ng`)，支持自动连接 |
| 蓝牙 | 正常工作 | `sprdbt_tty` |
| USB | 正常工作 | 包含对 `OpenWare Multi-Touch-V5000` 触摸屏的支持 |
| **GPU** | 正常工作 | **Mali-G31 / Panfrost，支持 OpenGL ES 3.1** |
| BinderFS | 正常工作 | `binder` / `hwbinder` / `vndbinder`（Waydroid 的前置条件） |
| cgroup2 | 正常工作 | 自动挂载（Waydroid 的前置条件） |
| GPU 验证工具 | 内置 | `/usr/local/bin/gpu_render_test` 开箱即用 |
| Waydroid 容器 | 正常工作 | LXC 6 + `lineage-20.0` VANILLA/MAINLINE，启用 PSI、DMABUF_HEAPS，包含六项容器修复 |
| Android UI | 正常工作 | 在 Weston 上运行 `waydroid show-full-ui`；会话日志成功到达 `Android with user 0 is ready` |
| 离线启动 | 正常工作 | 移除默认路由后容器仍能正常启动（系统镜像均在本地） |

### 验证证据
```
GL_RENDERER   : Mali-G31 (Panfrost)
GL_VERSION    : OpenGL ES 3.1 Mesa 25.2.7
压力测试      : 10,000 帧 / 2 秒 / 0 错误
上下文切换测试 : 100 次创建-销毁循环 / 16 秒 / 0 失败
运行时电源管理 : 100/100 次真实的 挂起(suspend) -> 唤醒(resume) -> 渲染 循环测试通过
温度表现      : 持续负载下从 51°C 升至 68°C，无降频问题
```
---
## 版本里程碑
| 版本 | 里程碑 |
|---|---|
| v0.10 | 硬件基础：PMIC、CPU 调频 (cpufreq)、温控 (thermal)、HDMI、USB、WiFi |
| v0.18 | GPU 加速：Panfrost、GLES 3.1、压力测试 + 运行时电源管理 (runtime-PM) 测试 |
| v0.19 | Wayland：基于 DRM/card0 的 Weston、HDMI、通用 HID 触控、自适应显示接口 (head) 检测 |
| **v0.22** | **以 Waydroid 作为主 UI**：BinderFS、DMABUF heaps、LXC、PSI、Android SystemServer + SurfaceFlinger，容器运行稳定且支持离线启动 |

v0.20 和 v0.21 未作为独立标签发布——容器相关的开发工作直接合并为一个版本（v0.22）落地。

---
## 仓库目录结构
```
.
├── README.md                        本文件
├── CHANGELOG-Alpindroid.md          完整的技术记录（根本原因 + 验证证据）
├── CHANGES-vs-debian12.md           与官方 Debian 12 镜像的差异对比
├── IMAGES.md                        镜像校验和与烧录指南
│
├── kernel/
│   └── config/config-6.1.31-plus    编译所使用的确切 .config 配置文件
│
├── config/
│   ├── weston.ini                   Weston 14 配置文件（DRM + GL 后端，HDMI 输出）
│   └── udev/99-alpindroid-touch.rules  将通用触摸屏绑定到 HDMI-A-1
│
├── patches/
│   ├── kernel/
│   │   ├── alpindroid-kernel.patch  完整的内核补丁差异（8 个文件，17 KB）
│   │   ├── panfrost-resume-fix.patch
│   │   ├── fix_panfrost_pmrt.py     幂等地（可重复安全地）应用唤醒修复补丁
│   │   ├── panfrost_device.c        打好补丁的 panfrost 源码（参考副本）
│   │   ├── panfrost_devfreq.c       打好补丁的 panfrost 源码（参考副本）
│   │   └── ppu/                     PPU 驱动：主线副本 + 适配 6.1 内核的副本
│   └── dts/
│       ├── pmic-axp313a.dts         设备树覆盖（overlay）源码（8 个片段）
│       ├── pmic-axp313a.dtbo        编译后的设备树覆盖文件
│       ├── AXP313A-FINDINGS.md      PMIC 测量日志
│       └── gpu/                     GPU 补丁集与技术总结
│
└── scripts/
    ├── setup-kernel.sh              克隆上游源码 + 应用补丁（单条命令完成）
    ├── build-alpindroid.sh          一键式镜像构建脚本（包含 28 项自动检查）
    ├── build-image.sh               底层组装脚本（基础镜像 + rootfs -> 最终镜像）
    ├── replace-modules.sh           安全的内核模块替换脚本
    ├── install-gpu-stack.sh         通过 chroot + QEMU 注入 Mesa 图形栈
    ├── install-wayland-stack.sh     安装 Weston 14 + seatd + libinput + 触控绑定
    ├── wayland-adapt.sh             解析 Weston 显示输出端（head）以用于触摸屏枚举
    ├── install-terminal-polish.sh   彩色提示符、LS_COLORS、字体（含中日韩 CJK）、控制台字体
    ├── install-waydroid.sh          Android 容器 (LXC) + 所有容器端的修复
    ├── firstboot-grow.sh            首次启动时的分区扩容 growpart + resize2fs（local.d 钩子）
    ├── debug-waydroid.sh            抓取 logcat + dmesg + 管理器日志 + 状态时间线
    ├── analyse-waydroid.sh          汇总分析 debug-waydroid 抓取的数据
    ├── start-weston.sh              在 card0 上启动 Weston（开发模式，通过 seatd）
    ├── wayland-check.sh             Wayland/GPU/触控 验收检查（共 21 项）
    ├── alpine-run.sh                在 Alpine rootfs 内部运行命令（基于 QEMU/binfmt）
    ├── gpu_render_test.c            单帧 GPU 验收测试源码
    ├── gpu_render_test              同上测试，针对 aarch64 的预编译二进制文件
    ├── gpu_stress_test.c            GPU 压力测试源码（1万帧 + 100次上下文切换）
    ├── hw-check.sh                  硬件自检脚本
    ├── full-test.sh                 板载全功能测试 / 状态快照
    ├── hdmi-diag.sh                 HDMI / DRM / fbcon 显示路径诊断
    ├── pmic-diag.sh                 只读 PMIC 信息转储
    ├── pmic-regs.sh                 原始 PMIC 寄存器转储（使用 i2c-tools）
    ├── pmic-force-read.sh           强制读取 PMIC（使用 -f 参数，绕过已绑定的驱动）
    ├── serial-capture.ps1           串口日志抓取脚本 (Windows)
    └── ssh-run.ps1                  远程执行/上传/下载辅助脚本 (Windows)
```
---
## 快速开始
### 1. 准备内核源码树
```bash
sh scripts/setup-kernel.sh
# 将 https://github.com/walnutpi/linux.git (6.1.31 分支) 克隆到 ./build/linux-6.1.31
# 应用 patches/kernel/alpindroid-kernel.patch 补丁
# 将 kernel/config/config-6.1.31-plus 安装为 .config
# 验证 7 项关键修改
```
### 2. 编译内核
```bash
cd build/linux-6.1.31
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) Image modules dtbs
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=/tmp/newmods modules_install
```
在 8 核处理器上大约需要 25 分钟。

### 3. 构建 SD 卡镜像
```bash
sudo sh scripts/build-alpindroid.sh v0.19 /tmp/newmods/lib/modules/6.1.31+
```
输出文件：`output/Alpindroid-ZeroW-v0.19.img`。该脚本会运行 28 项自动检查（包括模块目录结构、关键 `.ko` 文件、DTB 节点、固件、`config.txt`、Mesa、cgroup2），任何一项失败都会中止构建。

### 4. 在开发板上验证
```sh
/usr/local/bin/gpu_render_test     # 预期输出 RESULT: PASS
```
---
## Wayland + 触控（v0.19，进行中）
Weston 14.0.2 直接运行在 v0.18 的开发板镜像上，**无需修改内核，也无需自行编译 Weston**——所有软件包均直接来自 Alpine 3.23 aarch64 官方仓库。

| 层级 | 实现方式 |
|---|---|
| 显示输出 (Output) | `sun4i-drm`, `/dev/dri/card0`（KMS，原子模式设置 atomic modeset）；在启动时自动检测显示接口（head）名称，分辨率模式设为 `preferred`（由 EDID 决定） |
| 渲染器 (Renderer) | 基于 `/dev/dri/renderD128` 的 GL (`gl-renderer.so`) —— Mesa 25.2.7, Mali-G31 (Panfrost) |
| 会话管理 (Session) | `seatd` + `libseat` (`LIBSEAT_BACKEND=seatd`)；无需 root 权限，无需执行 `chmod 777 /dev/dri` |
| 触控 (Touch) | 通用 USB HID 多点触控：使用内核自带的 `hid_multitouch`，MT-B 协议槽位，`INPUT_PROP_DIRECT`，10 点触控 → 交由 `libinput` 处理 |
| 输出绑定 | 在枚举触摸屏时，`wayland-adapt.sh` 会为 udev 规则解析板载显示连接器名称 (`WL_OUTPUT=<head>`) |

触摸屏**并非**由厂商专有驱动驱动。该屏幕枚举为一个标准的 USB HID 设备（`222a:0001`，`OpenWare Multi-Touch-V5000`），内核自带的 `hid_multitouch` 模块（在 `kernel/config/config-6.1.31-plus` 中配置了 `CONFIG_HID_MULTITOUCH=m`）会将其转换为带有 `INPUT_PROP_DIRECT` 属性的 MT-B 输入节点。因此，这套连接机制适用于任何 HID 多点触控屏——没有任何针对这块特定屏幕的硬编码。

```sh
# 在开发板上以 root 身份运行：安装软件包、加载 hid_multitouch、配置 seatd、udev 规则和 weston.ini
sh scripts/install-wayland-stack.sh alpine
# 以桌面用户身份运行：在 card0 上启用 DRM 后端，在 Panfrost 上启用 GL 渲染器
sh scripts/start-weston.sh -b
# 验收测试：检查 HDMI 输出、GL 渲染器（非 llvmpipe 软渲染）、触控绑定、dmabuf
sh scripts/wayland-check.sh --client --dmabuf
# 在 Weston 运行时抓取原始触控数据；请在 15 秒内触摸屏幕
sudo -E sh scripts/wayland-check.sh --touch=15
```

此层中的所有配置均未死板地绑定到单一屏幕上：

| 可能被硬编码的项 | 实际采取的做法 |
|---|---|
| 连接器名称 (`HDMI-A-1`) | `wayland-adapt.sh` 会读取 Weston `card0` 上已连接的显示接口（head）。输入规则会在开发板上动态解析它，而在等待 EDID 时将 `HDMI-A-1` 作为 ZeroW 的回退默认值。桌面服务在启动 Weston 前会等待 udev 就绪并检查绑定状态 |
| 屏幕分辨率 | `mode=preferred` —— 由显示器的 EDID 决定，因此 1024x600 和 1080p 的屏幕都能正常工作 |
| 触控坐标范围 | 屏幕上报的坐标范围是 0..1920 / 0..1080，libinput 会自动将其缩放到当前的实际输出分辨率上；未内置任何死板的校准矩阵 |
| 字体 / 颜色 | `install-terminal-polish.sh` 将等宽字体设为 `Source Code Pro`，无衬线字体设为 `Noto Sans`，中日韩（CJK）字体回退到文泉驿（WenQuanYi），并为 ash 和 bash 配置了 `LS_COLORS` 和彩色提示符 |

在 ZeroW 上的实测输出：
```
weston 14.0.2
libseat: session control granted            (seatd backend)
using /dev/dri/card0                        (sun4i-drm)
Using rendering device: /dev/dri/renderD128
GL renderer: Mali-G31 (Panfrost)            OpenGL ES 3.1 Mesa 25.2.7
Output 'HDMI-A-1' enabled                   1024x600@60.1, preferred
Touchscreen - OpenWare Multi-Touch-V5000 - event3   (MT-B, PROP=DIRECT)
wayland-check.sh                            PASS 19  FAIL 0
```
当前状态：**仅限开发模式**——Weston 需要通过 SSH/TTY 手动启动。目前尚未安装开机自启服务，这样即使合成器（compositor）崩溃也不会将你锁死在开发板之外。v0.18 基线保持冻结状态；以上所有改动均位于用户空间层。

> **触控已验证正常工作。** 该屏幕是一个标准的 USB HID 多点触控设备：`hid-multitouch` 将其绑定为 MT-B 节点（`INPUT_PROP_DIRECT`，10 点触控，Report ID `0x15`），libinput 将其 1920x1080 的坐标空间缩放到 1024x600 的显示输出上，并且我们的 udev 规则将其固定到 `HDMI-A-1`。实测数据：从 `PROP=2` 节点读取到 6768 字节 / 282 个事件，Weston 桌面上的鼠标指针能够紧跟手指移动。详细细节以及曾导致该问题看起来像硬件故障的测量误区，请参见 `CHANGELOG-Alpindroid.md` 的 §14.6/§14.7 章节。

---
## 问题排查与修改过程
共发现五个独立的缺陷，均在硬件上得到证实。完整的技术分析请见 `CHANGELOG-Alpindroid.md`。

| # | 缺陷 | 症状 | 修复方法 |
|---|---|---|---|
| 1 | **厂商移除了 PPU 驱动**（GPU 电源域控制器） | 执行 `modprobe panfrost` 时在 `panfrost_gpu_init` 处永久卡死 | 向后移植（Backport）`sun50i-h6-prcm-ppu.c`，在设备树中添加 `prcm_ppu` 节点及 `power-domains` 属性 |
| 2 | **`mali-supply` 被错误连接到 `dcdc3`**（实际是固定 1.3 V 的 `vdd-dram` 内存供电） | 供电轨错误 | 将其指向 `reg_dcdc1` (`vdd-gpu-sys`)，配置单一工作点（OPP）432 MHz @ 0.9 V |
| 3 | **`resume`（唤醒）函数缺失 `reset_control_deassert` + `clk_prepare_enable`** | 驱动探测（Probe）成功，但任何对 GPU 的使用都会导致系统死机，且控制台无任何输出 | 与主线内核的 `panfrost_device_runtime_resume` 逻辑对齐 |
| 4 | **`/lib/modules` 缺失内核版本号目录** | 所有内核模块均无法加载（WiFi/蓝牙/GPU 全部失效），但系统仍能启动 | 使用 `cp -a src/. dst/` 以保留 `6.1.31+` 这一层级目录 |
| 5 | **DT overlay（设备树覆盖）使用了硬编码的 phandle** | 重新编译 DTB 后 phandle 发生偏移，静默返回 `-EPROBE_DEFER` 错误 | 在所有地方均使用符号引用（如 `&reg_dcdc1`），绝不使用原始的 `0x..` 十六进制值 |

### 调试方法论
整个排查的突破口来自**串口控制台（serial console）的接入**。在此之前，三轮盲目的尝试（调整初始化顺序、禁用 DVFS 动态调频调压、固定 OPP）全部失败，因为当时唯一可见的症状只有“最后一条日志是 `bus_clock rate`”。
通过在每个初始化步骤前插入 `pr_info("Alpindroid-STEP: ...")`，最终将死机位置精准定位到了 `panfrost_gpu_init`——这直接排除了时钟、电压和 DVFS 的嫌疑，将问题指向了硬件电源域。

ZeroW 提供了两个 UART 串口：
- **UART0 = PH0/PH1** —— 官方调试串口，位于 FPC 排线座上（需要转接板）
- **UART2 = PI5(TX) / PI6(RX)** —— 位于 40-pin 排针上，需在 `config.txt` 中设置 `console_uart=uart2`

接线方式：`PI5 -> USB-TTL RX`，`PI6 -> USB-TTL TX`，`GND -> GND`，**VCC 悬空不接**（使用 3.3 V 逻辑电平）。

---
## 参考资料
| 来源 | 用途 |
|---|---|
| `https://github.com/walnutpi/linux` (6.1.31 分支, HEAD `30ff3fd5cf`) | 厂商内核，是我们打补丁的基础 |
| `https://github.com/walnutpi/walnutpi_wiki` | 官方文档（引脚定义、启动流程） |
| `https://github.com/walnutpi/walnutpi-build` | 厂商镜像构建工具（仅供参考，本项目未使用） |
| `torvalds/linux` master 主分支 | PPU 驱动与 panfrost 运行时电源管理（runtime-PM）逻辑的参考来源 |

## 未纳入本仓库版本控制的文件
| 路径 | 原因 |
|---|---|
| `output/*.img` | 每个镜像高达 2.7 GB；其校验和已记录在 `IMAGES.md` 中 |
| `images/`, `work/`, `build/` | 厂商基础镜像与内核源码树 |
| `ssh.txt` | **包含明文密码凭据** |
| `debian12/` | 厂商构建工具的副本，可从 GitHub 重新获取 |

## v0.22 镜像是如何组装的
Android 系统镜像（2.3 GB）**不包含**在基础镜像中——发布的构建镜像保持在 ~2.7 GB 左右，容器会在开发板上现场初始化展开，这样以后升级 Android 时就无需重新发布整个系统镜像文件。

| 阶段 | 执行的操作 |
|---|---|
| 构建 `[3b]` | `config.txt` 保留原有的 `extraargs` 参数，并新增 `psi=1`（Android 的 lmkd 需要 PSI；内核配置中特意保留了 `CONFIG_PSI_DEFAULT_DISABLED=y`） |
| 构建 `[5c]` | 安装 Weston / seatd / libinput / 触控配置 / 字体（与 v0.19 相同） |
| 构建 `[5d]` | 在 chroot 环境中运行 `install-waydroid.sh --no-images`：安装软件包、OpenRC 服务、六项容器修复、启动钩子，并在 `/usr/local/sbin` 下安装 `waydroid-setup` / `debug-waydroid` / `analyse-waydroid` |
| 首次启动 | `/etc/local.d/alpindroid-firstboot.start` 会运行一次 `growpart` + `resize2fs` 进行分区扩容（生成标记文件 `/etc/.alpindroid-grown`） |
| 首次启动 | 运行 `waydroid-setup` 下载 Android 镜像（约 1.1 GB）并初始化容器 |
| 每次会话 | 运行 `start-weston.sh -b`，然后执行 `dbus-run-session -- sh -c 'waydroid session start &'`, 最后执行 `waydroid show-full-ui` |

目前特意**未启用**开机自启：只有当 Android 桌面、Android 内部的触控以及通过 Panfrost 运行的 GLES 硬件加速在真机上全部验证通过后，v0.22 才会成为候选发布版本（Release Candidate）。

---
## 路线图 (Roadmap)
- **v0.19** —— Weston (DRM 后端) + Panfrost + HDMI + 触摸屏 —— 在硬件上已全链路跑通，待集成到镜像构建流程中
- **v0.22** —— 以 Waydroid 作为主 UI —— **进行中**：Android 已能启动至 `user 0 is ready` 状态，容器保持稳定运行，支持无网离线启动；Android 内部的触控以及基于 Panfrost 的 GLES 渲染仍有待验证

## 许可证 (License)
MIT —— 详见 [LICENSE](LICENSE) 文件。
