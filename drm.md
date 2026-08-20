# Linux DRM / KMS 框架说明

> 本文面向 `drm_hwcomposer`：说明 **DRM 是什么**、**KMS 与 DRM 的关系**、**内核 DRM 驱动组成**，以及 **HWC ↔ libdrm ↔ Kernel** 的重要接口。  
> 内核侧分析基于树内源码：`linux-5.4/`（`drivers/gpu/drm/`、`include/drm/`、`include/uapi/drm/`）。  
> 文中「DRM」均指 **Direct Rendering Manager**，不是版权 DRM。

---

## 1. DRM 框架是什么

**DRM（Direct Rendering Manager）** 是 Linux 内核提供的 **GPU / 显示设备用户态接口框架**。

它解决三类问题：

| 问题 | DRM 提供的能力 |
|------|----------------|
| 谁能操作显示设备 | 设备节点、Master/Auth、权限 |
| 像素数据在哪、怎么共享 | GEM buffer、Prime/DMA-BUF、Framebuffer |
| 怎么把图画到屏上 | **KMS**（模式设置、CRTC、Plane、Atomic Commit） |

用户态（本项目的 HWC、以及 Mesa 等）通过打开：

```text
/dev/dri/card0   （或 card1、card% …）
```

再用 **ioctl**（通常经 **libdrm** 封装）与内核通信，而不是直接访问硬件寄存器。

### 1.1 在 Android 图形栈中的位置

```text
App / SurfaceFlinger
        │  AIDL Composer3
        ▼
drm_hwcomposer (HWC)
        │  C API（xf86drm*.h）
        ▼
libdrm
        │  ioctl(DRM_IOCTL_*)
        ▼
Kernel DRM Core + Vendor DRM Driver + KMS
        │
        ▼
Display Controller / Panel / HDMI …
```

---

## 2. DRM 与 KMS 是什么关系

### 2.1 一句话

- **DRM** = 整套「渲染/显示设备」内核子系统的总称（含 buffer、权限、ioctl 框架）。
- **KMS（Kernel Mode Setting）** = DRM 里专门负责 **显示模式设置与扫描输出** 的那一块。

关系：**KMS ⊂ DRM**。日常说「走 DRM/KMS」时，往往特指显示路径；说「DRM」有时也泛指整卡（含 GPU 渲染，但 HWC 几乎只用显示侧）。

### 2.2 对照表

| 概念 | 范围 | 典型能力 | HWC 是否重度使用 |
|------|------|----------|------------------|
| **DRM Core** | 框架层 | 设备节点、ioctl 分发、GEM、Prime、同步对象、Master | 是（open、Prime、GEM_CLOSE、cap） |
| **KMS** | DRM 的显示子系统 | Connector/Encoder/CRTC/Plane、Mode、Property、Atomic、VBlank、DPMS | **是（核心）** |
| **GPU 渲染路径** | 常挂在同一 DRM 设备上 | 命令提交、着色器等（厂商 ioctl / Mesa） | 否（由 GPU HAL / GLES / Vulkan 负责） |

### 2.3 历史背景（帮助理解为何叫 “Mode Setting”）

早期显示模式常在用户态（Xorg）里直接配，不稳定。  
**KMS** 把「分辨率 / 刷新率 / 哪个 FB 上屏」下沉到内核，保证 VT 切换、多进程、热插拔更安全。  
现代路径进一步演进为 **Atomic Modesetting**：一次提交整帧显示状态，可先 `TEST_ONLY` 验证。

```text
┌─────────────────────────────────────────────┐
│                 DRM 子系统                   │
│  ┌─────────────┐    ┌─────────────────────┐ │
│  │ GEM / Prime │    │        KMS          │ │
│  │ Buffer 管理 │    │ Connector/CRTC/Plane│ │
│  │ FB 对象     │───►│ Atomic / VBlank     │ │
│  └─────────────┘    └─────────────────────┘ │
│         ▲                                   │
│         │  同一 /dev/dri/card*              │
└─────────┴───────────────────────────────────┘
```

---

## 3. 内核中 DRM 驱动的组成

一个完整的「DRM 驱动」通常由多层拼成，而不是单文件。

### 3.1 分层结构

```text
┌──────────────────────────────────────────────────────┐
│  Vendor Display / DPU 驱动（平台相关）                 │
│  例：msm、rockchip、mediatek、exynos、i915、amdgpu…   │
│  实现：connector 探测、mode、plane 能力、atomic_check │
│        atomic_commit、vblank、IRQ                     │
└──────────────────────────▲───────────────────────────┘
                           │ 注册 / 回调
┌──────────────────────────┴───────────────────────────┐
│  DRM KMS Helper + Atomic Helpers                     │
│  drm_atomic*.c、drm_crtc_helper、drm_plane_helper…    │
│  提供：状态复制、校验骨架、commit 编排、非阻塞路径     │
└──────────────────────────▲───────────────────────────┘
                           │
┌──────────────────────────┴───────────────────────────┐
│  DRM Core                                            │
│  drm_drv / drm_ioctl / drm_file / drm_gem / …        │
│  设备节点、ioctl 表、Master、Mode 对象、Property、Prime│
└──────────────────────────▲───────────────────────────┘
                           │ ioctl
┌──────────────────────────┴───────────────────────────┐
│  用户态：libdrm / HWC / Mesa                         │
└──────────────────────────────────────────────────────┘
```

### 3.2 内核侧主要模块（逻辑组成）

| 模块 | 职责 |
|------|------|
| **DRM Core** | `/dev/dri/*` 创建；`drm_ioctl` 分发；refcount；minor（card/render/control） |
| **GEM（Graphics Execution Manager）** | 缓冲对象句柄（handle）、映射、关闭；与 dma-buf 对接 |
| **Prime** | GEM ↔ DMA-BUF 互导（`DRM_IOCTL_PRIME_FD_TO_HANDLE` / `HANDLE_TO_FD`） |
| **KMS 对象模型** | `drm_connector` / `drm_encoder` / `drm_crtc` / `drm_plane` / `drm_framebuffer` |
| **Property 系统** | 对象上的命名属性（ACTIVE、MODE_ID、FB_ID、SRC_*/CRTC_*、zpos…） |
| **Atomic 状态机** | `drm_atomic_state`；check → commit；ALLOW_MODESET / NONBLOCK / TEST_ONLY |
| **VBlank / Event** | 垂直消隐中断；`DRM_IOCTL_WAIT_VBLANK`；page-flip / atomic 完成事件 |
| **Dumb Buffer** | 简单扫描缓冲创建（modeset 测试、无 GPU 场景）；HWC 偶用于 modeset 辅助 |
| **厂商 DPU/显示驱动** | 真正编程硬件：时序发生器、overlay、scaler、HDR、writeback 等 |

### 3.3 设备节点角色

| 节点 | 典型用途 |
|------|----------|
| `/dev/dri/cardN` | **Modeset + 完整 DRM**（HWC 必须，且通常需 Master） |
| `/dev/dri/renderD128` 等 | 仅渲染（无 modeset），给 GPU 进程用 |
| control 节点 | 历史遗留，现代很少用 |

`drm_hwcomposer` 打开的是 **card** 节点，并调用 `drmSetMaster`。

### 3.4 KMS 对象在内核中的关系

```text
          ┌────────────┐
          │ drm_plane  │  (Primary / Overlay / Cursor)
          │  FB_ID     │
          └─────┬──────┘
                │ 绑定到
          ┌─────▼──────┐         ┌──────────────┐
          │  drm_crtc  │◄────────│ drm_encoder  │
          │ ACTIVE     │         └──────▲───────┘
          │ MODE_ID    │                │
          └────────────┘         ┌──────┴───────┐
                                 │drm_connector │
                                 │ DPMS / EDID  │
                                 │ CRTC_ID      │
                                 └──────────────┘
```

本项目中一条逻辑显示 = `DrmDisplayPipeline`：  
`Connector + Encoder + Crtc + PrimaryPlane (+ Overlays) + DrmAtomicStateManager`。

### 3.5 Connector / Encoder / CRTC / Plane 分别是什么

这四个是 **KMS 显示管线**里的标准对象，合在一起描述「像素从哪来、经谁、以什么时序、从哪个接口出屏」。它们是 **显示控制器（DPU）在软件里的抽象**，不是四块独立显卡。

#### 一句话对照

| 对象 | 是什么 | 类比 |
|------|--------|------|
| **Plane** | 硬件叠加层，挂着一块可扫描的 Framebuffer | 一张「图层」 |
| **CRTC** | 显示时序控制器，把 Plane 合成结果按 mode 扫出去 | 「扫描引擎 / 时序发生器」 |
| **Encoder** | 把 CRTC 的像素流转成某种物理信号格式 | 「信号转换器」 |
| **Connector** | 对外物理或逻辑接口（屏、HDMI、DP、writeback…） | 「插头 / 面板口」 |

#### 管线关系

```text
Plane(s) ──合成/叠加──► CRTC ──► Encoder ──► Connector ──► 屏幕 / 接口
   ▲                    │
   │                    └─ MODE（分辨率、刷新率、时序）
   └─ 每层一个 FB（framebuffer）
```

| 问题 | 由谁回答 |
|------|----------|
| 像素内容从哪来？ | **Plane** + Framebuffer |
| 按什么时序扫？ | **CRTC** + Mode |
| 信号怎么编码？ | **Encoder** |
| 从哪个口出去？ | **Connector** |

#### Plane（平面 / 叠加层）

- 一块硬件层，绑定 **`FB_ID`**，还有 `SRC_*`（源裁剪）、`CRTC_*`（屏上位置/大小）、`zpos`、旋转、alpha 等。
- 常见类型：
  - **Primary**：主层，几乎每路显示至少一个
  - **Overlay**：额外硬件层，可少走 GPU 合成
  - **Cursor**：光标小层
- HWC 把 SF 的 Layer 尽量映射到 Plane（`DrmKmsPlan`）；硬件层不够或能力不够的走 Client（GPU）。
- 内核结构：`struct drm_plane` / `drm_plane_state`（`include/drm/drm_plane.h`）；本仓库封装：`drm/DrmPlane.*`。

#### CRTC（Cathode Ray Tube Controller，历史命名）

- 负责 **显示模式（MODE）** 与扫描输出：分辨率、刷新率、`ACTIVE` 开关。
- 多个 Plane 叠到同一个 CRTC；CRTC 再把结果送给 Encoder。
- Atomic 常见属性：`ACTIVE`、`MODE_ID`、`OUT_FENCE_PTR`、CTM 等。
- 可理解为：**这一路「什么时候扫哪一帧」的控制器**。
- 内核：`struct drm_crtc` / `drm_crtc_state`；本仓库：`drm/DrmCrtc.*`。

#### Encoder（编码器）

- 把 CRTC 出来的像素流，编成接口需要的格式（HDMI TMDS、DP、DSI 等）。
- 常有 `possible_crtcs`：能接到哪些 CRTC。
- 对 HWC/用户态往往较「薄」；下游还可能挂 **Bridge / Panel**（PHY、面板驱动），对 SF 基本透明。
- 内核：`struct drm_encoder`；本仓库：`drm/DrmEncoder.*`。

#### Connector（连接器）

- 表示 **接了什么输出**：内置 DSI 屏、HDMI、DP、eDP，或 **Writeback**（虚拟屏写回）。
- 带 **连接状态**、**EDID**、可用 **modes** 列表、`DPMS`、色彩/HDR 等属性。
- Atomic 里用 `CRTC_ID` 绑到某个 CRTC。
- **热插拔**主要体现在 Connector 状态变化 → HWC `BindDisplay` / `onHotplug`。
- 内核：`struct drm_connector` / `drm_connector_state`；本仓库：`drm/DrmConnector.*`。

#### 和「显卡 / card」的关系

一块 SoC 显示 IP（或 PC 上一块 GPU 的显示引擎）里，通常有 **若干 Plane、若干 CRTC、若干 Encoder/Connector**。  
用户态通过 `/dev/dri/cardN` 看到并配置这些对象；**一个 card 节点上可以有多路完整管线**（多屏）。

---

## 4. linux-5.4 源码地图

树内路径前缀：`linux-5.4/`。

| 路径 | 角色 |
|------|------|
| `drivers/gpu/drm/` | DRM 核心 `.c` + 各厂商驱动子目录 |
| `include/drm/` | 内核内部头（结构体 / helper API） |
| `include/uapi/drm/` | UAPI：`drm.h` / `drm_mode.h` / `drm_fourcc.h` |

### 4.1 核心实现文件（`drivers/gpu/drm/`）

| 文件 | 职责 |
|------|------|
| `drm_drv.c` | 设备生命周期：`drm_dev_init` / `drm_dev_alloc` / `drm_dev_register` |
| `drm_ioctl.c` | 核心 ioctl 表 `drm_ioctls[]`、`drm_ioctl_permit`、`drm_ioctl` 分发 |
| `drm_file.c` | open/close、`drm_file`、事件队列 |
| `drm_atomic.c` | atomic 状态机：`drm_atomic_check_only` / `commit` / `nonblocking_commit` |
| `drm_atomic_uapi.c` | UAPI 入口：`drm_mode_atomic_ioctl`、属性 set、fence 准备 |
| `drm_atomic_helper.c` | 通用 commit 流水线（prepare → swap → commit_tail） |
| `drm_atomic_state_helper.c` | 状态 duplicate / reset / destroy 默认实现 |
| `drm_crtc.c` / `drm_plane.c` / `drm_encoder.c` / `drm_connector.c` | KMS 对象生命周期与遗留 ioctl |
| `drm_framebuffer.c` | ADDFB / ADDFB2 / RMFB |
| `drm_mode_config.c` | `drm_mode_config_init`、标准属性创建 |
| `drm_mode_object.c` | 对象 IDR、属性 attach/find |
| `drm_property.c` | 属性类型、blob CREATE/DESTROY/GET |
| `drm_gem.c` | GEM 对象、handle、mmap |
| `drm_prime.c` | dma-buf import/export、fd ↔ handle |
| `drm_dumb_buffers.c` | CREATE / MAP / DESTROY_DUMB |
| `drm_vblank.c` | VBlank 计数、等待、事件投递 |
| `drm_auth.c` | magic auth、SET/DROP_MASTER |
| `drm_lease.c` | DRM leasing（多客户端资源租约） |
| `drm_syncobj.c` | Timeline / binary syncobj |
| `drm_bridge.c` / `drm_panel.c` / `drm_writeback.c` | Bridge 链、Panel、Writeback connector |
| `drm_blend.c` | zpos / alpha / blend_mode |
| `drm_color_mgmt.c` | DEGAMMA / CTM / GAMMA |
| `drm_edid.c` | EDID 解析 |
| `drm_probe_helper.c` | Connector detect / 热插拔轮询 |
| `drm_modeset_lock.c` | `drm_modeset_acquire_ctx`、死锁 backoff |
| `drm_gem_framebuffer_helper.c` 等 | GEM→FB、CMA/shmem 等常见后端 |

### 4.2 关键头文件

| Header | 内容 |
|--------|------|
| `include/drm/drm_device.h` | `struct drm_device` |
| `include/drm/drm_drv.h` | `struct drm_driver`、`drm_dev_*` API |
| `include/drm/drm_file.h` | `drm_file` / `drm_minor` |
| `include/drm/drm_mode_config.h` | `drm_mode_config`、`drm_mode_config_funcs` |
| `include/drm/drm_crtc.h` / `drm_plane.h` / `drm_encoder.h` / `drm_connector.h` | 显示管线对象与 `*_state` |
| `include/drm/drm_framebuffer.h` | `drm_framebuffer` |
| `include/drm/drm_atomic.h` | `drm_atomic_state` |
| `include/drm/drm_property.h` / `drm_mode_object.h` | Property / Mode object |
| `include/drm/drm_gem.h` / `drm_prime.h` | GEM / Prime |
| `include/drm/drm_ioctl.h` | `DRM_AUTH` / `MASTER` / `RENDER_ALLOW` 等标志 |
| `include/drm/drm_modeset_helper_vtables.h` | `*_helper_funcs` 回调表 |
| `include/uapi/drm/drm.h` | 核心 ioctl 编号 |
| `include/uapi/drm/drm_mode.h` | KMS 结构体、atomic flags |

### 4.3 常见厂商驱动目录（Android 相关）

| 目录 | 平台 |
|------|------|
| `msm/` | Qualcomm Adreno + MDP/DPU |
| `mediatek/` | MediaTek DDP / OVL |
| `rockchip/` | Rockchip VOP |
| `exynos/` | Samsung |
| `tegra/` | NVIDIA Tegra |
| `i915/` / `amd/` | PC / 部分板卡 |
| `bridge/` / `panel/` | 跨厂商 HDMI/DP/DSI bridge、面板驱动 |

厂商驱动负责：把硬件能力注册成 CRTC/Plane/Connector，并实现 `atomic_check` / `atomic_update` / IRQ→vblank。对 HWC 而言，看到的仍是统一 UAPI。

---

## 5. 核心数据结构（linux-5.4）

### 5.1 `drm_device` / `drm_driver`

- **`drm_device`**（`drm_device.h`）：设备根对象。含 `driver`、`primary`/`render` minor、`master`、嵌入的 `mode_config`、vblank/event、GEM idr 等。
- **`drm_driver`**（`drm_drv.h`）：驱动描述。feature 标志（`DRIVER_MODESET` / `ATOMIC` / `GEM` / `PRIME` / `RENDER`…）、生命周期回调、GEM/PRIME hooks、私有 `ioctls[]`、dumb ops。  
  5.4 中 `load`/`unload` 已弃用，要求在 `drm_dev_register()` **之前**完成对象与 mode_config 初始化。

### 5.2 `drm_minor` / `drm_file`

- **`drm_minor`**：字符设备节点（`DRM_MINOR_PRIMARY` → `cardN`，`DRM_MINOR_RENDER` → `renderD*`）。
- **`drm_file`**：每个 open FD 的私有状态：`authenticated` / `is_master`、`atomic` / `universal_planes`（由 `SET_CLIENT_CAP` 打开）、GEM handle 表（`object_idr`）、该 FD 创建的 FB 列表、事件队列。

### 5.3 `drm_mode_config`

全局 KMS 仓库（`drm_mode_config.h`）：

- 列表：`crtc_list` / `plane_list` / `encoder_list` / `connector_list` / `fb_list`
- `object_idr`：统一对象 ID
- `funcs`：`fb_create`、`atomic_check`、`atomic_commit`（厂商必填）
- `helper_private`：可挂 `atomic_commit_tail`
- 标准属性指针：`prop_active`、`prop_mode_id`、`prop_fb_id`、`prop_src_*`、`prop_crtc_*`、`prop_in_fence_fd`、`prop_out_fence_ptr` 等

### 5.4 显示管线对象与状态

| 结构 | Header | 要点 |
|------|--------|------|
| `drm_crtc` / `drm_crtc_state` | `drm_crtc.h` | CRTC；真正可提交状态在 `*_state`（`enable`/`active`、`mode`/`mode_blob`、color LUT、`out_fence`） |
| `drm_plane` / `drm_plane_state` | `drm_plane.h` | PRIMARY/OVERLAY/CURSOR；`crtc`/`fb`/`fence`、`src_*`(16.16)、`crtc_*`、`zpos`/`alpha`/`rotation` |
| `drm_encoder` | `drm_encoder.h` | `possible_crtcs`；常接 Bridge 链 |
| `drm_connector` / `drm_connector_state` | `drm_connector.h` | 热插拔、EDID、`crtc`、writeback 等 |
| `drm_framebuffer` | `drm_framebuffer.h` | format、pitches/offsets/modifier；backing 通常为 GEM |
| `drm_atomic_state` | `drm_atomic.h` | 一次 commit 的全部对象状态；`allow_modeset`、`acquire_ctx` |
| `drm_mode_object` / `drm_property` | 对应头文件 | 所有 KMS 对象基类 + 属性元数据 |
| `drm_gem_object` | `drm_gem.h` | `size`、mmap、`dma_buf`/`import_attach`、`resv` |

Atomic 的核心思想：**硬件当前状态不变，先在 `drm_atomic_state` 里拼出目标态，check 通过后再 swap + 编程硬件。**

---

## 6. 驱动注册与初始化流程

典型现代驱动顺序（linux-5.4）：

```text
1. drm_dev_alloc(driver, parent)     // 或嵌入 drm_device + drm_dev_init
     └─ drm_dev_init()
          ├─ 分配 primary（+ 可选 render）minor
          ├─ DRIVER_GEM → drm_gem_init()
          └─ drm_dev_set_unique()

2. drm_mode_config_init(dev)
     └─ drm_mode_create_standard_properties()
          （SRC_*/CRTC_*/FB_ID/IN_FENCE_FD/OUT_FENCE_PTR/CRTC_ID/ACTIVE/MODE_ID…）

3. 创建 KMS 对象（常见顺序）：
     drm_universal_plane_init()      // 挂 plane 标准属性
     drm_crtc_init_with_planes()     // 挂 ACTIVE、MODE_ID
     drm_encoder_init()
     drm_connector_init()            // atomic 时挂 CRTC_ID
     drm_connector_attach_encoder()
     可选：drm_bridge_attach / drm_panel_attach / writeback
     可选：drm_plane_create_zpos_property() 等

4. 填充 mode_config：
     min/max_width/height
     funcs = { .fb_create, .atomic_check, .atomic_commit }
     可选 helper_private.atomic_commit_tail

5. drm_vblank_init(dev, num_crtcs)   // 需要 VBlank 事件时

6. drm_dev_register(dev, flags)
     ├─ 注册 render/primary 字符设备 → /dev/dri/*
     └─ DRIVER_MODESET → drm_modeset_register_all()
```

关键实现：

- `drm_drv.c`：`drm_dev_init`、`drm_dev_alloc`、`drm_dev_register`
- `drm_mode_config.c`：`drm_mode_config_init`、`drm_mode_create_standard_properties`

卸载反向：`drm_dev_unregister` → `drm_mode_config_cleanup` → `drm_dev_put`。

---

## 7. ioctl 分发与权限

### 7.1 传输路径

```text
userspace: ioctl(fd, DRM_IOCTL_*, arg)
    → drm_ioctl()                         // drm_ioctl.c
         ├─ nr = DRM_IOCTL_NR(cmd)
         ├─ 查 drm_ioctls[nr]
         │   或 driver->ioctls[nr - DRM_COMMAND_BASE]（厂商私有）
         ├─ copy_from_user
         └─ drm_ioctl_permit(flags) → func()
```

UAPI 编号定义在 `include/uapi/drm/drm.h`，例如：

| ioctl | 编号（宏） |
|-------|------------|
| `DRM_IOCTL_SET_CLIENT_CAP` | `0x0d` |
| `DRM_IOCTL_PRIME_FD_TO_HANDLE` | `0x2e` |
| `DRM_IOCTL_WAIT_VBLANK` | `0x3a` |
| `DRM_IOCTL_MODE_ADDFB2` | `0xB8` |
| `DRM_IOCTL_MODE_ATOMIC` | `0xBC` |

### 7.2 权限标志（`drm_ioctl_permit`）

| Flag | 含义 |
|------|------|
| `DRM_AUTH` | 需 authenticated；render 客户端豁免 |
| `DRM_MASTER` | 需当前 master（`drm_is_current_master`） |
| `DRM_ROOT_ONLY` | 需 `CAP_SYS_ADMIN` |
| `DRM_UNLOCKED` | 不拿 `drm_global_mutex` |
| `DRM_RENDER_ALLOW` | 允许 render 节点调用 |

与 HWC 强相关的表项（`drm_ioctl.c`）：

| ioctl | 处理函数 | 权限 |
|-------|----------|------|
| `SET_CLIENT_CAP` | `drm_setclientcap` | 0 |
| `SET_MASTER` | `drm_setmaster_ioctl` | `DRM_ROOT_ONLY` |
| `PRIME_FD_TO_HANDLE` | `drm_prime_fd_to_handle_ioctl` | `DRM_AUTH\|RENDER_ALLOW` |
| `MODE_ADDFB2` | `drm_mode_addfb2_ioctl` | 0 |
| **`MODE_ATOMIC`** | **`drm_mode_atomic_ioctl`** | **`DRM_MASTER`** |

未 `SET_CLIENT_CAP(ATOMIC)` 时，`file_priv->atomic == false`，atomic ioctl 直接 `-EINVAL`。  
驱动还需声明 `DRIVER_ATOMIC`，否则 `-EOPNOTSUPP`。

---

## 8. 三层接口总览

```text
┌─────────────────────────────────────────────────────────┐
│  HWC（drm_hwcomposer C++ 类）                            │
│  DrmDevice / DrmPlane / DrmAtomicStateManager / …       │
└──────────────────────────┬──────────────────────────────┘
                           │  libdrm C API
                           │  （xf86drm.h / xf86drmMode.h）
┌──────────────────────────▼──────────────────────────────┐
│  libdrm                                                  │
│  把 C 函数翻译成 DRM_IOCTL_* + 结构体打包/拆包            │
└──────────────────────────┬──────────────────────────────┘
                           │  ioctl(fd, DRM_IOCTL_*, arg)
┌──────────────────────────▼──────────────────────────────┐
│  Kernel DRM Driver（Core + Helper + Vendor）             │
└─────────────────────────────────────────────────────────┘
```

下面两节分别细化 **HWC↔libdrm** 与 **libdrm↔Kernel**；再往后是内核 Atomic / Buffer 路径详解。

---

## 9. HWC ↔ libdrm：重要接口

头文件：`xf86drm.h`、`xf86drmMode.h`（本仓库 `drm/*.cpp` 直接调用）。

### 9.1 设备打开与能力

| libdrm API | HWC 调用位置 | 作用 |
|------------|--------------|------|
| `open("/dev/dri/card…")` | `DrmDevice::Init` | 获取 DRM fd（未用 `drmOpen`，直接 open） |
| `drmSetClientCap(DRM_CLIENT_CAP_UNIVERSAL_PLANES)` | 同上 | 暴露所有 plane（含 overlay/cursor） |
| `drmSetClientCap(DRM_CLIENT_CAP_ATOMIC)` | 同上 | **启用 Atomic Modeset**（本 HWC 硬性要求） |
| `drmSetClientCap(DRM_CLIENT_CAP_WRITEBACK_CONNECTORS)` | 同上 | 虚拟屏 writeback（可选） |
| `drmGetCap(DRM_CAP_ADDFB2_MODIFIERS)` | 同上 | 是否支持带 modifier 的 FB |
| `drmGetCap(DRM_CAP_CURSOR_WIDTH/HEIGHT)` | 同上 | Cursor 尺寸限制 |
| `drmSetMaster` / `drmIsMaster` | 同上 | 成为 modeset master |
| `drmGetVersion` | `DrmDevice::GetName` | 驱动名（Backend 选择等） |

### 9.2 资源枚举（KMS 拓扑发现）

| libdrm API | HWC 封装 | 作用 |
|------------|----------|------|
| `drmModeGetResources` | `MakeDrmModeResUnique` | 列出 CRTC / Encoder / Connector、分辨率范围 |
| `drmModeGetConnector` | `DrmConnector::CreateInstance` | 连接状态、modes、encoder 可能集、属性 |
| `drmModeGetEncoder` | `DrmEncoder` | Encoder↔CRTC 可能映射 |
| `drmModeGetCrtc` | `DrmCrtc` | CRTC 基本信息 |
| `drmModeGetPlaneResources` / `drmModeGetPlane` | `DrmPlane` | Plane 列表、可绑定 CRTC、支持 format |
| `drmModeObjectGetProperties` / `drmModeGetProperty` | `DrmDevice::GetProperty` | 读对象属性元数据与当前值 |
| `drmModeGetPropertyBlob` | EDID / blob 读取 | EDID、HDR 等 blob 数据 |
| `drmModeFree*` | `DrmUnique.h` 的 deleter | 释放 libdrm 堆对象 |

### 9.3 Buffer：DMA-BUF → GEM → Framebuffer

Android GraphicBuffer 经 gralloc 得到 **prime fd**，HWC 导入为 DRM FB：

| libdrm API | HWC 位置 | 作用 |
|------------|----------|------|
| `drmPrimeFDToHandle` | `DrmFbImporter` | DMA-BUF fd → GEM handle |
| `drmModeAddFB2` | `DrmFbIdHandle::CreateInstance` | 无 modifier 时创建 FB |
| `drmModeAddFB2WithModifiers` | 同上 | 有 modifier（AFBC 等）时创建 FB |
| `drmModeRmFB` | `~DrmFbIdHandle` | 销毁 FB |
| `drmIoctl(DRM_IOCTL_GEM_CLOSE)` | 同上 | 关闭 GEM handle（不关 DMA-BUF 本身） |
| `drmPrimeHandleToFD` | `CreateBufferForModeset` | GEM → fd（dumb buffer 场景） |

数据流：

```text
GraphicBuffer (prime_fds)
    → drmPrimeFDToHandle → gem_handles[]
    → drmModeAddFB2[WithModifiers] → fb_id
    → Plane 属性 FB_ID = fb_id（Atomic 提交）
```

### 9.4 Property Blob（Mode / CTM / HDR）

| libdrm / ioctl | HWC 位置 | 作用 |
|----------------|----------|------|
| `DRM_IOCTL_MODE_CREATEPROPBLOB` | `DrmDevice::RegisterUserPropertyBlob` | 创建 mode/CTM/HDR 等 blob |
| `DRM_IOCTL_MODE_DESTROYPROPBLOB` | blob unique_ptr deleter | 销毁 blob |
| `DrmMode::CreateModeBlob` | modeset | 把 `drmModeModeInfo` 做成 MODE_ID blob |

### 9.5 Atomic Commit（每帧核心）

| libdrm API | HWC 位置 | 作用 |
|------------|----------|------|
| `drmModeAtomicAlloc` / `Free` | `MakeDrmModeAtomicReqUnique` | 分配 atomic 请求集 |
| `drmModeAtomicAddProperty` | `DrmProperty::AtomicSet` | 往请求里加「对象+属性+值」 |
| `drmModeAtomicCommit` | `DrmAtomicStateManager::CommitFrame` | **提交 / 测试** 整帧状态 |

常用 flags（`include/uapi/drm/drm_mode.h`）：

| Flag | 值 | 含义 | HWC 用法 |
|------|-----|------|----------|
| `DRM_MODE_ATOMIC_ALLOW_MODESET` | `0x0400` | 允许改 mode / 连接关系 | 几乎总是带上 |
| `DRM_MODE_ATOMIC_TEST_ONLY` | `0x0100` | 只校验不生效 | Validate 路径（`test_only`） |
| `DRM_MODE_ATOMIC_NONBLOCK` | `0x0200` | 非阻塞提交 | 正常 present |

Plane 上典型写入的属性（经 `DrmPlane::AtomicSetState`）：

- `CRTC_ID`、`FB_ID`
- `CRTC_X/Y/W/H`、`SRC_X/Y/W/H`
- `zpos`、`rotation`、`alpha`、`pixel blend mode`
- `IN_FENCE_FD`（acquire）
- `COLOR_ENCODING` / `COLOR_RANGE`（YUV）

CRTC / Connector 上典型属性：

- CRTC：`ACTIVE`、`MODE_ID`、`OUT_FENCE_PTR`、`CTM`
- Connector：`CRTC_ID`、`DPMS`、Colorspace / HDR / writeback FB 等

### 9.6 电源、VBlank、Dumb（辅助）

| libdrm API | HWC 位置 | 作用 |
|------------|----------|------|
| `drmModeConnectorSetProperty`（DPMS） | `ActivateDisplayUsingDPMS` | 部分路径用传统 DPMS |
| `drmWaitVBlank` | `VSyncWorker` | 等待垂直消隐，生成 vsync 时间戳 |
| `DRM_IOCTL_MODE_CREATE_DUMB` / `MAP_DUMB` / `DESTROY_DUMB` | `CreateBufferForModeset` | 创建简单扫描缓冲（modeset 辅助） |

### 9.7 按场景归类（HWC 视角）

| HWC 场景 | 主要 libdrm 调用 |
|----------|------------------|
| 启动扫卡 | open、SetClientCap、GetCap、SetMaster、GetResources/Connector/Plane… |
| Hotplug 后绑 Pipeline | GetConnector、属性、组 `DrmDisplayPipeline` |
| Validate | AtomicAlloc → AddProperty → AtomicCommit(**TEST_ONLY**) |
| Present | 同上 → AtomicCommit（可 NONBLOCK）→ OUT_FENCE |
| Buffer 上场 | PrimeFDToHandle → AddFB2 → Plane FB_ID |
| VSync 回调 | WaitVBlank |
| 虚拟屏 | Writeback connector 属性 + Atomic |
| 销毁 | RmFB、GEM_CLOSE、DestroyPropBlob |

---

## 10. libdrm ↔ Kernel：重要接口

libdrm **几乎不做策略**，主要是：

1. 把参数填进内核 UAPI 结构体；  
2. 调用 `ioctl(fd, DRM_IOCTL_xxx, &arg)`；  
3. 把返回的指针/句柄包装成 `drmMode*` 对象。

### 10.1 传输机制

```text
用户态:  drmModeAtomicCommit(fd, req, flags, ...)
            │
            ▼
libdrm:  组装 drm_mode_atomic { ... }
            │
            ▼
ioctl(fd, DRM_IOCTL_MODE_ATOMIC, &drm_mode_atomic)
            │
            ▼
内核:   drm_ioctl → drm_mode_atomic_ioctl → 厂商 atomic_check/commit
```

设备 fd 来自 `open("/dev/dri/cardN")`，之后所有 libdrm 调用都带这个 fd。

### 10.2 与本 HWC 相关的内核 UAPI（按功能）

#### A. 客户端能力 / 查询

| 用户可见（libdrm） | 内核 ioctl / 常量 | 内核侧含义 |
|--------------------|-------------------|------------|
| `drmSetClientCap` | `DRM_IOCTL_SET_CLIENT_CAP` | 声明支持 universal planes / atomic / writeback |
| `drmGetCap` | `DRM_IOCTL_GET_CAP` | 查询驱动能力（modifier、cursor 尺寸等） |
| `drmSetMaster` | `DRM_IOCTL_SET_MASTER` | 抢占 modeset 控制权 |
| `drmGetVersion` | `DRM_IOCTL_VERSION` | 驱动名称与版本 |

#### B. 资源与对象

| libdrm | 内核 ioctl | 说明 |
|--------|------------|------|
| `drmModeGetResources` | `DRM_IOCTL_MODE_GETRESOURCES` | CRTC/Encoder/Connector id 列表 |
| `drmModeGetConnector` | `DRM_IOCTL_MODE_GETCONNECTOR` | 连接器详情与 mode 列表 |
| `drmModeGetEncoder` | `DRM_IOCTL_MODE_GETENCODER` | Encoder |
| `drmModeGetCrtc` | `DRM_IOCTL_MODE_GETCRTC` | CRTC |
| `drmModeGetPlaneResources` | `DRM_IOCTL_MODE_GETPLANERESOURCES` | Plane id 列表 |
| `drmModeGetPlane` | `DRM_IOCTL_MODE_GETPLANE` | 单个 Plane |
| `drmModeObjectGetProperties` | `DRM_IOCTL_MODE_OBJ_GETPROPERTIES` | 对象上的 property id 列表 |
| `drmModeGetProperty` | `DRM_IOCTL_MODE_GETPROPERTY` | property 元数据（枚举/range/blob） |
| `drmModeGetPropertyBlob` | `DRM_IOCTL_MODE_GETPROPBLOB` | 读取 blob 内容（如 EDID） |

#### C. Framebuffer / GEM / Prime

| libdrm | 内核 ioctl | 说明 |
|--------|------------|------|
| `drmPrimeFDToHandle` | `DRM_IOCTL_PRIME_FD_TO_HANDLE` | dma-buf → GEM handle |
| `drmPrimeHandleToFD` | `DRM_IOCTL_PRIME_HANDLE_TO_FD` | GEM → dma-buf |
| `drmModeAddFB2` | `DRM_IOCTL_MODE_ADDFB2` | 用 handles/pitch/offset 注册 FB |
| `drmModeAddFB2WithModifiers` | 同上（带 modifier 标志） | 带格式修饰符的 FB |
| `drmModeRmFB` | `DRM_IOCTL_MODE_RMFB` | 删除 FB |
| `drmIoctl(…, DRM_IOCTL_GEM_CLOSE)` | `DRM_IOCTL_GEM_CLOSE` | 释放本进程对该 GEM 的引用 |
| CREATE/MAP/DESTROY_DUMB | `DRM_IOCTL_MODE_CREATE_DUMB` 等 | 简单缓冲 |

#### D. Atomic / 传统 Modeset / DPMS

| libdrm | 内核 ioctl | 说明 |
|--------|------------|------|
| `drmModeAtomicCommit` | **`DRM_IOCTL_MODE_ATOMIC`** | Atomic 提交（本 HWC 主路径） |
| `drmModeAtomicAddProperty` | （用户态组包，无单独 ioctl） | 填充 atomic 请求中的 property 数组 |
| CREATE/DESTROY PROPBLOB | `DRM_IOCTL_MODE_CREATEPROPBLOB` / `DESTROYPROPBLOB` | Mode/CTM 等 blob |
| `drmModeConnectorSetProperty` | `DRM_IOCTL_MODE_SETPROPERTY` | 传统单属性设置（如 DPMS） |
| （旧路径，本 HWC 基本不用） | `DRM_IOCTL_MODE_SETCRTC` / `PAGE_FLIP` | Legacy modeset / 翻页 |

#### E. 同步与 VBlank

| libdrm | 内核 ioctl | 说明 |
|--------|------------|------|
| `drmWaitVBlank` | `DRM_IOCTL_WAIT_VBLANK` | 阻塞等到指定 CRTC 的 vblank |
| Atomic `OUT_FENCE_PTR` | 随 ATOMIC 提交返回 sync_file fd | Present fence 给 SF |

> 现代推荐：用 **in-fence / out-fence**（sync_file）而不是只靠 `WAIT_VBLANK` 做帧同步；本 HWC 两者都用（fence 做 present，WaitVBlank 做 vsync 时间戳）。

---

## 11. Atomic 内核路径详解（linux-5.4）

### 11.1 UAPI 入口

`DRM_IOCTL_MODE_ATOMIC` → `drm_mode_atomic_ioctl()`（`drm_atomic_uapi.c`）。

前置检查：

1. 驱动具备 `DRIVER_ATOMIC`
2. `file_priv->atomic` 已通过 `SET_CLIENT_CAP` 打开
3. flags 合法；`TEST_ONLY` 与 `PAGE_FLIP_EVENT` **不能同时**
4. 调用者是 **Master**（ioctl 表上的 `DRM_MASTER`）

### 11.2 调用顺序

```text
drm_mode_atomic_ioctl
  ├─ drm_atomic_state_alloc(dev)
  ├─ drm_modeset_acquire_init(INTERRUPTIBLE)
  ├─ state->allow_modeset = !!(flags & ALLOW_MODESET)
  ├─ [foreach obj × props]
  │     drm_mode_object_find()
  │     drm_atomic_set_property(state, file, obj, prop, value)
  ├─ prepare_signaling()          // PAGE_FLIP_EVENT + OUT_FENCE_PTR
  ├─ branch:
  │     TEST_ONLY  → drm_atomic_check_only(state)
  │     NONBLOCK   → drm_atomic_nonblocking_commit(state)
  │     else       → drm_atomic_commit(state)
  ├─ complete_signaling()
  ├─ -EDEADLK → clear + drm_modeset_backoff → retry
  └─ drm_atomic_state_put / drop_locks / acquire_fini
```

### 11.3 Core check / commit（`drm_atomic.c`）

**`drm_atomic_check_only`**：

1. 核心校验：`drm_atomic_plane_check` / `crtc_check` / `connector_check`
2. 驱动钩子：`mode_config.funcs->atomic_check(dev, state)`  
   （多数驱动填 `drm_atomic_helper_check` 或包一层）
3. 若 `!allow_modeset` 且任一 CRTC `drm_atomic_crtc_needs_modeset()` → `-EINVAL`

**`drm_atomic_commit` / `drm_atomic_nonblocking_commit`**：  
先 `check_only`，再调用 `funcs->atomic_commit(dev, state, nonblock)`。

这也是 HWC `Validate` 用 `TEST_ONLY` 的内核依据：只跑 check，不编程硬件。

### 11.4 Helper commit 流水线（`drm_atomic_helper.c`）

厂商通常挂：

```c
.atomic_check  = drm_atomic_helper_check,   // 或 wrapper
.atomic_commit = drm_atomic_helper_commit,
```

**`drm_atomic_helper_commit`** 要点：

```text
setup_commit(nonblock)
prepare_planes()                 // plane_helper.prepare_fb
[!nonblock] wait_for_fences(pre)
swap_state()                     // 软件状态切换点（此后难回滚）
nonblock ? queue_work(commit_work) : commit_tail()
```

**`drm_atomic_helper_commit_tail`** 默认硬件编程顺序：

```text
commit_modeset_disables
commit_planes
commit_modeset_enables
fake_vblank
commit_hw_done                   // 可在此前后信号 out-fence / event
wait_for_vblanks
cleanup_planes
```

需要「先开 CRTC 再配 plane」的驱动可用 `drm_atomic_helper_commit_tail_rpm`（先 enables 再 planes）。

### 11.5 厂商常挂的 helper hooks

| 挂载点 | 典型函数 | 用途 |
|--------|----------|------|
| `mode_config_funcs.atomic_check` | `drm_atomic_helper_check` (+ 私有校验) | 全局校验 |
| `mode_config_funcs.atomic_commit` | `drm_atomic_helper_commit` | 提交调度 |
| `mode_config_helper_funcs.atomic_commit_tail` | 自定义或 `*_commit_tail` | HW 编程顺序 |
| `drm_plane_helper_funcs` | `prepare_fb` / `cleanup_fb` / `atomic_check` / `atomic_update` / `atomic_disable` | Plane 级 |
| `drm_crtc_helper_funcs` | `atomic_check` / `atomic_flush` / `atomic_enable` / `atomic_disable` | CRTC 级 |
| `drm_encoder_helper_funcs` | `atomic_check` / `atomic_enable` / `atomic_disable` | Encoder |
| `drm_connector_helper_funcs` | `atomic_check` / `atomic_best_encoder` | Connector 路由 |

vtable：`include/drm/drm_modeset_helper_vtables.h`。

---

## 12. Property / GEM / Prime / FB 内核路径

### 12.1 Property 系统

1. `drm_mode_config_init` → `drm_mode_create_standard_properties()` 创建全局属性对象  
2. 对象 init 时 `drm_object_attach_property(&obj->base, prop, init_val)`  
3. Atomic：ioctl 按 `(obj_id, prop_id, value)` → `drm_atomic_set_property` → 写入对应 `*_state`  
4. Blob：`DRM_IOCTL_MODE_CREATEPROPBLOB` → MODE_ID / LUT / damage clips 等

**Plane 标准属性**（`drm_universal_plane_init` 自动挂）：  
`type`、`FB_ID`、`IN_FENCE_FD`、`CRTC_ID`、`CRTC_X/Y/W/H`、`SRC_X/Y/W/H`、`FB_DAMAGE_CLIPS`  

可选驱动创建：`zpos`（`drm_plane_create_zpos_property`）、`alpha`、`rotation`、`pixel blend mode`。

**CRTC**：`ACTIVE`、`MODE_ID`、（经 signaling）`OUT_FENCE_PTR`、色彩管理 blob。  

**Connector**：`CRTC_ID`（atomic）、`EDID`、`DPMS`、`link-status` 等。

### 12.2 HWC 典型 Buffer 导入路径

```text
dma-buf fd（gralloc）
  → DRM_IOCTL_PRIME_FD_TO_HANDLE
       drm_prime_fd_to_handle_ioctl
         → driver->prime_fd_to_handle
           通常 drm_gem_prime_fd_to_handle()
             → drm_gem_prime_import*()
             → drm_gem_handle_create()     // 得到 handle
  → DRM_IOCTL_MODE_ADDFB2（drm_mode_fb_cmd2: handles/pitches/modifier…）
       drm_mode_addfb2
         → drm_internal_framebuffer_create
           → mode_config.funcs->fb_create   // 常 drm_gem_fb_create*
  → Atomic: plane FB_ID = fb_id + SRC_*/CRTC_* + 可选 IN_FENCE_FD
```

关键文件：

- `drm_prime.c`：`drm_gem_prime_fd_to_handle`、`drm_gem_prime_import*`
- `drm_framebuffer.c`：`drm_mode_addfb2`
- `drm_gem.c`：`drm_gem_handle_create`、`drm_gem_object_lookup`
- `drm_gem_framebuffer_helper.c`：常见 FB 创建 helper

导出反向：`PRIME_HANDLE_TO_FD` → `drm_gem_prime_handle_to_fd` → `drm_gem_prime_export`。

---

## 13. VBlank、Master、Bridge / Panel / Writeback

### 13.1 VBlank / Events（`drm_vblank.c`）

- Init：`drm_vblank_init(dev, num_crtcs)`
- IRQ：驱动调用 `drm_crtc_handle_vblank(crtc)` → 更新计数 → 投递 pending events
- 引用：`drm_crtc_vblank_get/put`；modeset 时 `drm_crtc_vblank_off/on`
- Userspace：`DRM_IOCTL_WAIT_VBLANK` → `drm_wait_vblank_ioctl`
- Atomic flip 完成：`prepare_signaling` 挂 `drm_pending_event`，HW done / vblank 后 `drm_send_event*`
- 另有 `DRM_IOCTL_CRTC_GET_SEQUENCE` / `QUEUE_SEQUENCE`

### 13.2 Master / Auth（`drm_auth.c`）

- Primary 节点首个 opener 可通过 `drm_master_open` 成为 master
- `GET_MAGIC` / `AUTH_MAGIC`：legacy 认证（render 节点不需要）
- `SET_MASTER` / `DROP_MASTER`（`ROOT_ONLY`）：切换 master
- `drm_is_current_master(file)`：atomic / SETCRTC 等 MASTER ioctl 门槛
- Android 上 SF/HWC 通常持有 `card0` master；lease（`drm_lease.c`）可拆分资源给其他客户端

### 13.3 Bridge / Panel / Writeback

| 子系统 | 文件 | 角色 |
|--------|------|------|
| Bridge | `drm_bridge.c` | Encoder 后的链式转换器（HDMI PHY、DSI host…）；modeset disable/enable 链由 helper 调用 |
| Panel | `drm_panel.c` | 固定面板：`prepare`/`enable`/`get_modes` |
| Writeback | `drm_writeback.c` | 合成结果写回 FB 的虚拟 connector；配合虚拟屏 / oneshot |

对 HWC：通常只看到最终 connector/CRTC/plane；bridge/panel 对 userspace **透明**。Writeback connector 则会被 HWC 显式枚举使用。

---

## 14. 端到端：一帧如何穿过三层

以 **Present** 为例：

```text
SF executeCommands(present)
  → HwcDisplay::PresentStagedComposition
  → CreateComposition
  → DrmKmsPlan（Layer ↔ Plane）
  → DrmAtomicStateManager::ExecuteAtomicCommit
        │
        ├─ drmModeAtomicAlloc
        ├─ 对各 Plane: drmModeAtomicAddProperty(FB_ID, SRC_*, CRTC_*, …)
        ├─ CRTC: MODE_ID / ACTIVE / OUT_FENCE_PTR
        └─ drmModeAtomicCommit(fd, …, ALLOW_MODESET|NONBLOCK)
              │
              ▼ libdrm
           DRM_IOCTL_MODE_ATOMIC
              │
              ▼ Kernel（linux-5.4）
           drm_mode_atomic_ioctl
             → drm_atomic_check_only
                  （core check + driver atomic_check）
             → drm_atomic_nonblocking_commit
                  → drm_atomic_helper_commit
                       prepare_planes → swap_state
                       → queue_work → commit_tail
                            disables → planes → enables
                            → hw_done / out-fence → vblank → cleanup
              │
              ▼
           out_fence fd 回到 HWC → SF（PresentFence）
```

Validate 路径相同，仅多 `DRM_MODE_ATOMIC_TEST_ONLY`，硬件不切换。

---

## 15. linux-5.4 × Android HWC 注意点

1. **走 Atomic，不走 SETCRTC/PAGE_FLIP**：5.4 将 UAPI 放在 `drm_atomic_uapi.c`，与 core/helper 分离；本 HWC 主路径是 `DRM_IOCTL_MODE_ATOMIC`。
2. **必须 `SET_CLIENT_CAP`**：至少 `ATOMIC` + `UNIVERSAL_PLANES`；需要虚拟屏时再开 writeback。
3. **MASTER 独占**：`MODE_ATOMIC` 要求 master；SF/HWC 与测试工具争用 `card0` 会失败。
4. **Fence**：用 `IN_FENCE_FD` / `OUT_FENCE_PTR`；`WAIT_VBLANK` 主要用于 vsync 时间戳。
5. **`ALLOW_MODESET`**：改 mode / connector 路由必须带；纯 plane update 若不带且触发 `needs_modeset` → `-EINVAL`。本 HWC 目前几乎总是带上。
6. **`TEST_ONLY`**：Validate 专用；不可与 `PAGE_FLIP_EVENT` 同用。
7. **Modifier / ADDFB2**：Android buffer 常为 tiled/compressed；驱动 `fb_create` 必须认 modifier。
8. **zpos**：非全局自动属性，依赖驱动 `drm_plane_create_zpos_*`；helper 的 `normalize_zpos` 影响 stacking。
9. **Render 节点**：`renderD*` 不能 modeset；PRIME/GEM 可用。
10. **`-EDEADLK`**：内核 ioctl 层已 backoff 重试；用户态仍可能见到短暂失败，需理解 modeset 锁。

---

## 16. 与本仓库 / 内核代码的索引

### 16.1 drm_hwcomposer

| 主题 | 文件 |
|------|------|
| 打开设备 / Cap / 枚举 | `drm/DrmDevice.cpp` |
| libdrm 对象 RAII | `drm/DrmUnique.h` |
| Connector / CRTC / Plane | `drm/DrmConnector.cpp` 等 |
| FB 导入 | `drm/DrmFbImporter.cpp` |
| Atomic 提交 | `drm/DrmAtomicStateManager.cpp` |
| Plane 属性填充 | `drm/DrmPlane.cpp` |
| VBlank | `drm/VSyncWorker.cpp` |
| Pipeline 组装 | `drm/DrmDisplayPipeline.cpp` |

### 16.2 linux-5.4 内核

| 主题 | 文件 |
|------|------|
| 设备注册 | `drivers/gpu/drm/drm_drv.c` |
| ioctl 表 / 权限 | `drivers/gpu/drm/drm_ioctl.c` |
| Atomic UAPI | `drivers/gpu/drm/drm_atomic_uapi.c` |
| Atomic core | `drivers/gpu/drm/drm_atomic.c` |
| Atomic helper | `drivers/gpu/drm/drm_atomic_helper.c` |
| 标准属性 | `drivers/gpu/drm/drm_mode_config.c` |
| GEM / Prime / FB | `drm_gem.c` / `drm_prime.c` / `drm_framebuffer.c` |
| VBlank / Master | `drm_vblank.c` / `drm_auth.c` |
| UAPI | `include/uapi/drm/drm.h`、`drm_mode.h` |

---

## 17. 术语速查

| 术语 | 含义 |
|------|------|
| **DRM** | Linux 直接渲染管理框架（总称） |
| **KMS** | DRM 中的内核模式设置 / 显示子系统 |
| **Atomic** | 一次提交完整显示状态的 KMS 模式 |
| **GEM** | DRM 缓冲句柄抽象 |
| **Prime / DMA-BUF** | 跨驱动、跨进程共享 buffer 的 fd |
| **FB (Framebuffer)** | KMS 可扫描的缓冲描述（format+handles+pitch…） |
| **Plane** | 硬件叠加层（详见 §3.5） |
| **CRTC** | 显示时序控制器（详见 §3.5） |
| **Encoder** | CRTC 与 Connector 之间的信号转换（详见 §3.5） |
| **Connector** | 物理或虚拟显示接口（详见 §3.5） |
| **Bridge / Panel** | Encoder 下游硬件链 / 固定面板（对 HWC 常透明） |
| **Master** | 拥有 modeset 权限的客户端（通常 HWC/合成器） |
| **libdrm** | 用户态封装 DRM ioctl 的 C 库 |
| **mode_config** | 设备上全部 KMS 对象与标准属性的仓库 |
| **atomic_state** | 一次 check/commit 的完整目标状态快照 |

---

## 18. 小结

1. **DRM** 是内核图形/显示框架；**KMS 是其中负责「怎么上屏」的子模块**（KMS ⊂ DRM）。  
2. 内核驱动 = **DRM Core + KMS/Atomic Helper + 厂商 DPU 驱动**（linux-5.4 源码见第 4 节地图）。  
3. 厂商通过 `drm_dev_alloc` → 创建 CRTC/Plane/Connector → `drm_dev_register` 暴露 `/dev/dri/card*`。  
4. **HWC** 通过 **libdrm** 调用（Enumerate、Prime、AddFB、AtomicCommit、WaitVBlank…）。  
5. **libdrm** 把这些调用变成 **`DRM_IOCTL_*`**；本 HWC 的主路径是 **`DRM_IOCTL_MODE_ATOMIC`**（需 Master + CLIENT_CAP_ATOMIC）。  
6. 内核 Atomic：`drm_mode_atomic_ioctl` → `check_only` → `helper_commit`（prepare → swap → commit_tail 编程硬件）。  
7. Android 的 HWC 角色，本质是把 SF 的 Layer 列表翻译成 **KMS Plane + Atomic 状态**，交给内核扫到屏幕上。
