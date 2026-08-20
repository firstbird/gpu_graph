# Android 16 显示架构(Display Stack)全景解析

> 基于本仓库 `android16/` 源码(frameworks_base、frameworks_native、hardware_interfaces)整理。
> 本文定位为**全栈总览**:自上而下贯穿 App → Framework Java 层 → Native 服务层 → HAL → 硬件;
> SurfaceFlinger 内部深度细节不在本文重复展开,统一引用同目录下的既有文档
> (见[附录 B](#附录-b与既有文档的关系与阅读顺序))。

---

## 目录

- [一、概览](#一概览)
  - 1.1 显示栈的分层模型
  - 1.2 一帧的完整生命周期(主链路)
  - 1.3 Android 16 相对 Android 13 的架构主线变化
  - 1.4 关键进程与线程模型
- [二、Framework Java 层:DisplayManager 子系统](#二framework-java-层displaymanager-子系统)
- [三、App 到 SurfaceFlinger 的生产端链路](#三app-到-surfaceflinger-的生产端链路)
- [四、SurfaceFlinger 内部架构(总览+索引)](#四surfaceflinger-内部架构总览索引)
- [五、HAL 层:Composer 3](#五hal-层composer-3)
- [六、Gralloc 图形内存子系统](#六gralloc-图形内存子系统)
- [七、关键跨层专题](#七关键跨层专题)
- [八、调试与观测](#八调试与观测)
- [附录 A:核心类速查表](#附录-a核心类速查表)
- [附录 B:与既有文档的关系与阅读顺序](#附录-b与既有文档的关系与阅读顺序)
- [附录 C:术语表](#附录-c术语表)

---

## 一、概览

### 1.1 显示栈的分层模型

Android 的显示栈是一条生产者—合成者—消费者流水线,自上而下分为五层。
每一层只与相邻层交互,层间边界由明确的 IPC/HAL 接口定义:

![diag-01](assets/display-arch/diag-01.png)

各层职责概括:

| 层 | 承载进程 | 核心职责 | 代码位置 |
|---|---|---|---|
| App 绘制层 | 各 App 进程 | 生成一帧的内容(测量/布局/录制/GPU 渲染),把结果写入 GraphicBuffer | `frameworks_base/core/java/android/view/`、HWUI |
| 窗口/显示策略层 | system_server | 窗口层级管理(WMS)、显示设备的逻辑抽象与策略(DMS:亮度、刷新率投票、拓扑、电源) | `frameworks_base/services/core/java/com/android/server/display/` |
| 合成服务层 | surfaceflinger | 收集所有窗口的 buffer 与属性,决定合成方式,按 VSYNC 节奏产出最终帧 | `frameworks_native/services/surfaceflinger/` |
| 图形 HAL 层 | vendor 进程 | composer3 负责硬件合成与送显;allocator/mapper(Gralloc) 负责图形内存分配与访问 | `hardware_interfaces/graphics/` |
| 内核/硬件层 | kernel | DRM/KMS 驱动、Display Controller(DPU)、GPU | (不在本仓库范围) |

分层图中的两个要点:

1. **内容(buffer)与元数据(窗口属性)走的是两条路**。App 的像素数据经
   BLASTBufferQueue 直达 SurfaceFlinger,**不经过** system_server;而窗口的
   创建/层级/可见性等策略信息由 WMS 管理,WMS 再以 `SurfaceControl.Transaction`
   的形式下发给 SurfaceFlinger。system_server 不处理像素数据。
2. **SurfaceFlinger 是策略与机制的汇合点**。DMS 决定显示模式与亮度、WMS 决定
   窗口布局,逐帧执行并与硬件交互的只有 SurfaceFlinger 与 HAL。

### 1.2 一帧的完整生命周期(主链路)

下图是贯穿全文的主线:一帧从 App 触发绘制到显示在屏幕上的完整过程。
后续各章分别展开图中的某一段。

![diag-02](assets/display-arch/diag-02.png)

五个关键节点:

- **① Choreographer**:App 不随时绘制,而是等待 SF 分发的 VSYNC-app 信号,
  在 `doFrame` 中依次执行输入、动画、布局遍历,每帧只执行一次(详见 [3.4](#34-choreographer--displayeventreceiver--sf-vsync-回传通路))。
- **② 原子提交(BLAST)**:Android 16 中 buffer 与窗口属性(位置、透明度、裁剪…)
  打包进同一个 `Transaction` 一次性提交,避免内容与几何属性不同步导致的撕裂
  (详见 [3.2](#32-blast-架构))。
- **③ commit / ④ composite**:SF 主循环的两个阶段:commit 阶段处理事务、生成
  本帧的不可变快照;composite 阶段执行合成(详见 [四](#四surfaceflinger-内部架构总览索引))。
- **validate/present 两段式**:SF 先以全部图层期望 DEVICE 合成的状态调用 validate,
  HWC 返回其中必须回退 GPU 的图层,之后才 present 送显(详见 [五](#五hal-层composer-3))。
- **⑤ fence 闭环**:全链路不做 CPU 等待,GPU/显示硬件的完成时机用 fence 传递,
  buffer 的归还也由 release fence 驱动(详见 [3.5](#35-fence-机制))。

### 1.3 Android 16 相对 Android 13 的架构主线变化

> 详细的逐类逐函数对比见既有文档
> `Android13-vs-16-DisplayManager-SurfaceFlinger-HWC-变更汇总.md` 与
> `android13-16-update-all.md`,此处只汇总架构主线。

| 主线 | Android 13 | Android 16 | 代码证据 |
|---|---|---|---|
| SF 前端 | Layer 类自身持有绘制状态,遍历 Layer 树 | **数据驱动 FrontEnd**:`RequestedLayerState` → `LayerLifecycleManager` → `LayerSnapshotBuilder` 生成不可变快照,合成端只读快照 | `surfaceflinger/FrontEnd/` |
| 生产端 | BufferQueue 为主,BLAST 初步落地 | **BLAST 全面接管**,新增 `BufferReleaseChannel` 单向 socket 通道加速 buffer 归还 | `libs/gui/BLASTBufferQueue.cpp`、`BufferReleaseChannel.cpp` |
| 合成 HAL | HIDL composer 2.4 + AIDL composer3 V1 | **AIDL composer3 V4**:VRR/notifyExpectedPresent、`DisplayLuts`、`OverlayProperties`、批量图层生命周期命令 | `hardware_interfaces/graphics/composer/aidl/aidl_api/.../4` |
| Gralloc | HIDL IMapper 4.0 + AIDL IAllocator V1 | **AIDL IAllocator V2 + stable-c AIMapper v5**(mapper 从 Binder IPC 变为 vendor 库直接 dlopen 进程内调用) | `graphics/mapper/stable-c/`、`libs/ui/Gralloc5.cpp` |
| DisplayManagerService | 单体大类为主 | **高度模块化**:`brightness/`(策略模式)、`mode/`(投票制)、`feature/`(aconfig 开关)、`state/`、`layout/`、`plugin/` 子包 | `server/display/` 各子包 |
| 多屏 | LogicalDisplayMapper 为主 | 新增**显示拓扑**:`DisplayTopologyCoordinator`/`DisplayTopologyStore`(多屏相对位置持久化)、`ExternalDisplayPolicy` | `server/display/DisplayTopology*.java` |
| 刷新率 | DisplayModeDirector 初版投票 | 投票体系类型化(每种投票一个类:`SizeVote`、`SupportedModesVote`…),SF 侧 `RefreshRateSelector` 联动,支持 VRR/合成帧率与显示帧率解耦 | `server/display/mode/`、`surfaceflinger/Scheduler/RefreshRateSelector.*` |
| 功能开关 | build flag / sysprop | **aconfig flag** 全面接管(`surfaceflinger_flags_new.aconfig`、`display_flags.aconfig`) | 各模块 `.aconfig` 文件 |

### 1.4 关键进程与线程模型

显示链路横跨四类进程,每类进程内部又有明确的线程分工:

![diag-03](assets/display-arch/diag-03.png)

要点:

- **SF 主线程是唯一的帧驱动线程**:`Scheduler/MessageQueue` 驱动,所有 commit/composite
  都在主线程串行执行;耗时的 GPU 合成移交 RenderEngine 线程,重查询走 binder 线程池。
- **两个 EventThread**:`app` 与 `appSf` 两条 VSYNC 分发线程,允许给 App 的 VSYNC
  相位(phase offset)与 SF 自身错开,这是 App 渲染与 SF 合成形成流水线的基础。
- **App 双线程渲染**:UI Thread 只录制显示列表,GPU 工作在 RenderThread,
  与 BLASTBufferQueue 的 dequeue/queue 都发生在 RenderThread。
- **mapper 不再是独立进程**:Android 16 的 Gralloc mapper(stable-c)以 vendor
  动态库形式被**直接加载进调用方进程**(App/SF),lock/unlock 零 IPC(见[六](#六gralloc-图形内存子系统))。

---

## 二、Framework Java 层:DisplayManager 子系统

代码根目录:`android16/frameworks_base/services/core/java/com/android/server/display/`。
DisplayManagerService(下称 DMS)运行在 system_server 中,是显示器的
**策略中枢**:它不处理像素,负责管理系统中有哪些显示器、每个显示器的状态、
应使用的模式与亮度。

### 2.1 DMS 的职责与 Android 16 的模块化拆分

Android 13 中 `DisplayManagerService.java` 与 `DisplayPowerController.java` 两个
大类承担了绝大部分逻辑;Android 16 将其拆分为按职责划分的子包:

![diag-04](assets/display-arch/diag-04.png)

| 子包/类 | 职责 |
|---|---|
| `DisplayManagerService` | Binder 服务入口;持有全局锁 `mSyncRoot`;组织 Adapter/Repository/Mapper;向 App 提供 `DisplayManager` API |
| `mode/` | 刷新率与分辨率决策,投票制(见 2.6) |
| `brightness/` | 亮度决策链:策略选择器 + 各 Strategy + clamper(限幅器) |
| `layout/` + `DeviceStateToLayoutMap` | 设备形态(折叠/展开)→ 显示布局的映射,驱动内外屏切换 |
| `state/` | 屏幕状态(ON/OFF/DOZE/SUSPEND)迁移的执行 |
| `feature/DisplayManagerFlags` | 集中封装 `display_flags.aconfig` 的功能开关查询 |
| `DisplayPowerController`(DPC) | 每个 LogicalDisplay 一个实例,执行亮灭屏动画、亮度 ramp、Doze |

### 2.2 显示设备的接入:DisplayAdapter 家族

所有显示器(本机屏幕、HDMI 外接屏、无线投屏、App 创建的虚拟屏)都通过
**Adapter 模式**接入 DMS,统一抽象为 `DisplayDevice`:

![diag-05](assets/display-arch/diag-05.png)

其中物理屏由 `LocalDisplayAdapter` 接入:它通过 `DisplayControl`/`SurfaceControl` 的
JNI 接口获取 SF 上报的物理显示 token 与 `DisplayConfiguration` 列表,物理屏的
热插拔(hotplug)事件也从 SF 经此进入 Java 层。

### 2.3 三级映射:DisplayDevice → LogicalDisplay → DisplayGroup

DMS 对象模型的核心是一条三级映射链,描述物理硬件如何映射为 App 可见的 Display:

![diag-06](assets/display-arch/diag-06.png)

- **`LogicalDisplay`** 是 App 通过 `DisplayManager.getDisplay()` 看到的实体,持有
  `DisplayInfo`;它与 `DisplayDevice` 之间是**可动态改绑**的:折叠屏开合时,
  `LogicalDisplayMapper` 根据 `DeviceStateToLayoutMap`(读取 `display_layout_configuration.xml`)
  把 display 0 从内屏 device 切换到外屏 device,App 只收到一次配置变更,
  不感知底层物理屏的更换。
- **`DisplayGroup`** 把电源状态需要联动的逻辑屏编成一组(如默认组随电源键同亮同灭),
  `DisplayGroupAllocator`(16 新增)负责分配策略。

**关键流程:物理屏热插拔**

![diag-07](assets/display-arch/diag-07.png)

### 2.4 Android 16 新增:多屏拓扑(Display Topology)

`DisplayTopologyCoordinator` / `DisplayTopologyStore` / `DisplayTopologyXmlStore`
是 Android 16(面向桌面模式)引入的机制:当连接外接显示器时,记录**多块屏幕之间
的相对位置关系**(与桌面操作系统的显示器排列功能类似),并按显示器组合持久化到 XML,
下次连接同一台显示器时还原布局。`ExternalDisplayPolicy` 则集中管理外接屏的启停策略
(是否允许、温控限制,配合 `ExternalDisplayStatsService` 上报统计)。

拓扑信息最终影响 WMS 的窗口坐标换算与输入路由:DMS 维护模型,WMS 负责应用。

### 2.5 亮度与电源:DisplayPowerController + brightness/

每个 LogicalDisplay 配一个 `DisplayPowerController`(DPC)。Android 16 的亮度
决策已重构为**策略模式**:

![diag-08](assets/display-arch/diag-08.png)

- 每次决策产出 `DisplayBrightnessState`,附带 `BrightnessReason` 记录亮度值的来源策略,
  便于 dumpsys 排查。
- 高亮度模式由 `HighBrightnessModeController` 管理(强光环境下短时超过常规亮度上限,受温度/时长预算限制);
  `DisplayOffloadSessionImpl` 支持 AOD 场景把亮度控制下放给低功耗协处理器。
- 端到端链路(光感 → HAL 亮度)见 [7.5](#75-亮度调节全链路)。

### 2.6 刷新率决策:mode/ 投票机制

`mode/DisplayModeDirector` 采用**投票(Vote)制**收敛各方对刷新率/分辨率的需求,
Android 16 把每类投票类型化为独立类:

![diag-09](assets/display-arch/diag-09.png)

关键设计:DMD 输出的不是单个模式而是**一个允许范围**(primary/app request
两组 min/max 物理与渲染帧率),SF 的 `RefreshRateSelector` 在范围内结合各 Layer
的实际帧率需求(`LayerHistory`)做最终选择:策略在 Java 层、逐帧机制在 SF,
两级决策职责分离。`SyntheticModeManager` 进一步支持合成出渲染 60Hz、显示 120Hz
这类解耦模式(VRR 基础,见 [7.1](#71-可变刷新率vrr全链路))。

### 2.7 与相邻服务的交互边界

| 对端 | 方向 | 内容 |
|---|---|---|
| WindowManagerService | DMS→WMS | display 增删改事件;WMS 据此维护 DisplayContent |
| PowerManagerService | PMS→DMS | `requestPowerState`(亮灭屏/Doze);DMS 完成后回调确认 |
| SurfaceFlinger | DMS→SF | 模式范围(setDesiredDisplayModeSpecs)、亮度、电源模式(setPowerMode);SF→DMS:热插拔、模式切换完成事件 |
| InputManagerService | DMS→IMS | 显示 viewport(坐标变换),触摸坐标映射依赖它 |

---

## 三、App 到 SurfaceFlinger 的生产端链路

代码根目录:`android16/frameworks_native/libs/gui/`(libgui,生产端客户端库)。
本章说明 App 渲染完成的一帧如何传递到 SurfaceFlinger。

### 3.1 概念区分:Surface / SurfaceControl / SurfaceControlViewHost

三者的区别如下:

| 类 | 本质 | 创建者 | 作用 |
|---|---|---|---|
| `SurfaceControl` | SF 中一个 Layer 的**句柄** | WMS 创建(窗口),App 也可自建(子层) | 图层的控制句柄:位置/大小/Z 序/透明度等属性通过它配合 Transaction 设置 |
| `Surface` | buffer 的**生产端出口**(实现 `ANativeWindow`) | 从 SurfaceControl/BBQ 派生 | buffer 写入入口:Canvas、EGL、MediaCodec、Camera 通过 Surface 提交帧 |
| `SurfaceControlViewHost` | 跨进程嵌入 View 层级的封装 | 嵌入方 | 把一个进程的 View 树渲染到另一个进程的 SurfaceControl 上(如 WebView 独立渲染、车机分屏) |

关系:WMS 为每个窗口建 `SurfaceControl` → App 的 `ViewRootImpl` 在其上挂一个
`BLASTBufferQueue` → 从 BBQ 取出 `Surface` 交给 HWUI 渲染。

### 3.2 BLAST 架构

BLAST(**B**uffer **La**yer **S**urface **T**ransaction)是 Android 16 生产端的
标准形态:buffer 不再经由 SF 侧的 BufferQueue 被动消费,而是由 App 侧的
`BLASTBufferQueue` 主动打包进 `SurfaceComposerClient::Transaction` 提交。

核心代码:`libs/gui/BLASTBufferQueue.cpp`、`SurfaceComposerClient.cpp`、
`TransactionState.cpp`、`LayerState.cpp`。

![diag-10](assets/display-arch/diag-10.png)

与 SF 侧持有 BufferQueue 的旧模式相比,主要变化有三点:

1. **队列位于 App 进程内**。BBQ 内部仍复用 `BufferQueueProducer/Consumer`
   代码,但生产者与消费者都在 App 进程里,SF 看到的只是 Transaction 里的一个
   buffer 引用;SF 端不再有每个 Layer 一条的 buffer 队列线程模型。
2. **buffer 与几何原子化**。窗口 resize 时,新尺寸的 buffer 与新的窗口 crop/position
   在同一个 Transaction 中生效,避免旧架构下几何属性与 buffer 分别到达导致的闪黑/拉伸。
3. **归还路径提速(16 新增)**:`BufferReleaseChannel.cpp` 用一条单向 socket 把
   release fence 从 SF 推回 App,替代此前的 Binder 回调
   (`ITransactionCompletedListener`)批量往返,降低高帧率下的归还延迟。

`Transaction` 是一个**可合并、可暂存、可跨进程传递**的
状态包(`TransactionState`),`merge()` 支持把 WMS 的动画属性与 App 的 buffer
合并成一次提交;`setDesiredPresentTime`/`setFrameTimelineInfo` 则把帧的时间语义
(希望哪个 VSYNC 上屏)一并携带,供 SF Scheduler 与 FrameTimeline 使用。

### 3.3 传统 BufferQueue

`libs/gui/BufferQueue*.cpp` 在 Android 16 中仍然是 buffer 传递的通用基础设施,
使用场景包括:相机预览(`Surface` 连接 CameraService)、`MediaCodec` 编解码输入/输出、
`SurfaceTexture/GLConsumer`(把外部帧作为 GL 纹理)、`ImageReader`
(`BufferItemConsumer`)、以及 BLASTBufferQueue 内部(3.2)。变化仅在于:通往
SurfaceFlinger 的最后一跳由 BLAST 承担,SF 进程内不再持有每个 Layer 的 BufferQueue。

**组成**(`libs/gui/BufferQueue.cpp` 创建三者并绑定):

| 类 | 文件 | 职责 |
|---|---|---|
| `BufferQueueCore` | `BufferQueueCore.cpp` | 状态载体:`mSlots[NUM_BUFFER_SLOTS]`(每槽位一个 `BufferSlot`,含 GraphicBuffer、fence、状态)、`mQueue`(已入队待消费的 `BufferItem` 列表)、`mFreeSlots`/`mFreeBuffers` 空闲集合、互斥锁与条件变量 |
| `BufferQueueProducer` | `BufferQueueProducer.cpp` | 实现 `IGraphicBufferProducer`:`dequeueBuffer` / `requestBuffer` / `queueBuffer` / `cancelBuffer` / `connect` / `setMaxDequeuedBufferCount` 等,供生产端调用(可跨进程,Binder) |
| `BufferQueueConsumer` | `BufferQueueConsumer.cpp` | 实现 `IGraphicBufferConsumer`:`acquireBuffer` / `releaseBuffer` / `setMaxAcquiredBufferCount` / `consumerConnect` 等,供消费端调用 |
| `Surface` | `Surface.cpp` | 生产端封装,实现 `ANativeWindow`,把 `dequeueBuffer/queueBuffer` 等窗口调用转发给 `IGraphicBufferProducer` |
| `ConsumerBase` 及子类 | `ConsumerBase.cpp`、`GLConsumer.cpp`、`BufferItemConsumer.cpp`、`CpuConsumer.cpp` | 消费端封装,注册 `FrameAvailableListener`,封装 acquire/release |

**槽位状态机**:每个 `BufferSlot` 在 `FREE → DEQUEUED → QUEUED → ACQUIRED → FREE`
之间迁移(`BufferSlot.h` 中 `BufferState`),`dequeueBuffer` 触发 FREE→DEQUEUED,
`queueBuffer` 触发 DEQUEUED→QUEUED,`acquireBuffer` 触发 QUEUED→ACQUIRED,
`releaseBuffer` 触发 ACQUIRED→FREE。`cancelBuffer` 使 DEQUEUED 直接回 FREE。

**一次完整的生产—消费循环:**

![diag-28](assets/display-arch/diag-28.png)

流程说明:

1. 生产者经 `Surface::dequeueBuffer` 调用 `BufferQueueProducer::dequeueBuffer`。
   若无空闲槽位则阻塞等待(受 `maxDequeuedBufferCount` 与 `maxAcquiredBufferCount`
   约束);选中槽位若尚无 GraphicBuffer 或规格(尺寸/格式/usage)不匹配,则在此
   分配新 buffer(分配链路见 6.2)。返回值携带上一轮消费者给出的 release fence,
   生产者写入前必须等待该 fence。
2. 首次获得某槽位或该槽位重新分配后,`Surface` 调用 `requestBuffer` 取回
   GraphicBuffer 句柄(跨进程时在此传输 handle 并 import)。
3. 生产者填充内容后调用 `queueBuffer`,`QueueBufferInput` 携带 acquire fence、
   时间戳、crop、transform、dataspace 等元数据;槽位转为 QUEUED,`BufferItem` 进入
   `mQueue`,消费者通过 `onFrameAvailable` 收到通知。
4. 消费者调用 `acquireBuffer` 取出 `BufferItem`(槽位转为 ACQUIRED),按 acquire
   fence 等待内容就绪后消费;完成后 `releaseBuffer` 附带 release fence,槽位回到
   FREE,并回调 `onBufferReleased` 唤醒阻塞中的生产者。

**同步与异步模式**:默认同步模式下 `mQueue` 中的帧不会被丢弃;异步模式
(`setAsyncMode`)下新入队的帧可以替换尚未被 acquire 的旧帧,用于对延迟敏感、
可丢帧的场景。

**BLASTBufferQueue 与 BufferQueue 的关系**:BBQ 在 App 进程内创建一对
`BufferQueueProducer/Consumer`,自身充当消费者:`onFrameAvailable` 中 acquire
buffer,再把 buffer 与 acquire fence 写入 `Transaction` 提交给 SF;SF 归还
release fence 后,BBQ 调用 `releaseBuffer` 使槽位回到 FREE。

### 3.4 Choreographer → DisplayEventReceiver → SF VSYNC 回传通路

App 的绘制节奏由 SF 反向驱动,这是显示栈中方向自下而上的常驻信号流:

![diag-11](assets/display-arch/diag-11.png)

- App 侧 `Choreographer`(Java,`core/java/android/view/Choreographer.java`;
  native 版 `libs/gui/Choreographer.cpp` 供 NDK `AChoreographer` 使用)按需
  `scheduleVsync()`:没有注册回调时不分发信号,静止画面时整条链路处于空闲状态。
- `VsyncEventData` 携带的不只是时间戳,还有若干条 **frame timeline**
  (期望的 deadline / present time 候选),App 可据此得知当前帧错过 deadline 时
  的下一条候选时间线,这是 Frame Pacing(如游戏 Swappy)的基础。
- EventThread 有 `app` 与 `appSf` 两个实例,相位不同(`VsyncConfiguration`),
  实现 App 与 SF 的流水线错峰。

### 3.5 Fence 机制

**Fence 是什么**:Fence 是一个同步对象,不是函数。在内核中它是 sync_file
(封装 dma_fence),以文件描述符形式存在;在用户态由 `libs/ui/include/ui/Fence.h`
的 `android::Fence` 类封装(持有 `mFenceFd`),通过 `sp<Fence>` 在各模块间传递。
`Fence` 类的主要方法:

| 方法 | 作用 |
|---|---|
| `Fence(int fd)` | 用一个 sync_file fd 构造 |
| `wait(timeout)` / `waitForever(name)` | 阻塞等待 fence 达成(signal) |
| `getStatus()` | 非阻塞查询:Signaled / Unsignaled / Invalid |
| `getSignalTime()` | 读取 fence 达成的时间戳(present fence 用它取上屏时刻) |
| `merge(name, f1, f2)` | 合并两个 fence 为一个(两者都达成时才达成) |
| `dup()` / `get()` | 复制 / 读取底层 fd,用于跨进程传递(Binder 传 fd)或交给 HAL |
| `flatten` / `unflatten` | Parcel 序列化,`BufferItem`、`layer_state_t` 等携带 fence 时使用 |

acquire / release / present 是同一种 `Fence` 对象在不同环节的**三种用途**,区别在于
创建方、承载的数据结构、等待方各不相同。三者不是三个函数,而是三个 `sp<Fence>`
类型的字段/返回值:

| 用途 | 创建方 | 载体(代码位置) | 等待/消费方 | 语义 |
|---|---|---|---|---|
| acquire fence | 生产者的 GPU/ISP/解码器驱动,在提交渲染命令时产生 | 生产端:`IGraphicBufferProducer::QueueBufferInput::fence`(`libs/gui/include/gui/IGraphicBufferProducer.h`);BLAST:`layer_state_t::bufferData->acquireFence`(`libs/gui/include/gui/LayerState.h:134`);HAL:`LayerCommand.buffer.fence` | SF commit 阶段检查 `acquireFence->getStatus()` 决定是否 latch(`SurfaceFlinger.cpp` 约 5219 行);DEVICE 合成时把 fd 交给 composer3,由显示硬件等待;CLIENT 合成时由 RenderEngine 等待 | 内容写入完成的时刻 |
| release fence | 消费者:HWC 在 present 后按图层返回;GPU 合成时由 RenderEngine 返回 | HAL 返回:`presentAndGetReleaseFences` / `getLayerReleaseFence`(`services/surfaceflinger/DisplayHardware/HWComposer.h:174,194`);SF → App:`BLASTBufferQueue::releaseBufferCallback(id, releaseFence, ...)`(`libs/gui/include/gui/BLASTBufferQueue.h:111`),经 `BufferReleaseChannel` 传输;BufferQueue 内:`IGraphicBufferConsumer::releaseBuffer(slot, frameNumber, fence)`;返回给生产者:`dequeueBuffer(int* slot, sp<Fence>* fence, ...)` 的出参 | 生产者在下一次写该 buffer 前等待(GPU 驱动把它作为渲染前置依赖;CPU 路径在 `lockAsync` 传入由 mapper 等待) | 消费端不再读取该 buffer 的时刻 |
| present fence | composer3 HAL / 显示驱动,在 presentDisplay 时产生 | HAL 返回:`HWComposer::getPresentFence(HalDisplayId)`(`HWComposer.h:190`);CompositionEngine:`Output::FrameFences::presentFence`(`CompositionEngine/include/compositionengine/Output.h:133`);SF:`onCompositionPresented` 中封装为 `FenceTime` | `Scheduler::addPresentFence(PhysicalDisplayId, shared_ptr<FenceTime>)`(`Scheduler/Scheduler.h:261`)→ `VSyncPredictor` 校准 VSYNC 模型;`FrameTimeline` 用 `getSignalTime()` 记录实际上屏时刻;`TimeStats` 统计 | 该帧实际显示到屏幕的时刻 |

三种 fence 在一帧中的流转:

![diag-12](assets/display-arch/diag-12.png)

设计要点:传递 buffer 所有权时总是同时传递一个 fence,任何环节都不需要 CPU 阻塞
等待 GPU 或显示硬件完成;等待被推迟到真正读写内存的一方(GPU 驱动、显示硬件、
或 CPU lock)。SF 侧只对 acquire fence 做非阻塞的状态查询,以决定本帧是否采用该
buffer。

---

## 四、SurfaceFlinger 内部架构(总览+索引)

代码根目录:`android16/frameworks_native/services/surfaceflinger/`。
本章只做**架构级**说明;每个子模块的类图、成员级数据结构与关键代码,
请跳转到 [surfaceflinger16_claude.md](surfaceflinger16_claude.md)(下称《SF 详解》)
对应章节,类图的全局精炼版见
[sf16_class_all_claude.md](sf16_class_all_claude.md)。

### 4.1 顶层模块划分

![diag-13](assets/display-arch/diag-13.png)

| 模块 | 目录 | 职责 |
|---|---|---|
| SurfaceFlinger 主类 | `SurfaceFlinger.cpp/.h` | Binder 服务入口;实现 `ICompositor`,组织各子模块 |
| FrontEnd | `FrontEnd/` | 事务 → 层级 → 每帧不可变 `LayerSnapshot` |
| Scheduler | `Scheduler/` | VSYNC 建模与分发、帧节奏、刷新率选择 |
| CompositionEngine | `CompositionEngine/` | 合成策略:每个 Output 上各层的几何/色彩处理,以及 DEVICE 或 CLIENT 的决定 |
| RenderEngine | `libs/renderengine/` | GPU 合成执行器(Skia),截图路径亦使用 |
| DisplayHardware | `DisplayHardware/` | composer3 HAL 的 C++ 封装(HWComposer/AidlComposerHal) |
| Display | `Display/`、`DisplayDevice.*` | 每个显示器的模式/色彩/电源状态 |
| 观测 | `Tracing/ TimeStats/ FrameTracer/ Jank/` | layer trace、transaction trace、帧统计 |

各模块在《SF 详解》中的对应章节:主类 §4.1、FrontEnd §4.2、Scheduler §4.3、
CompositionEngine 与 RenderEngine §4.4、DisplayHardware §4.5、Display §4.6、观测 §4.7。

### 4.2 主循环:ICompositor 的 commit / composite

SF 主类实现 `ICompositor` 接口,`MessageQueue` 每个调度周期回调一次,
这是 SF 的主循环:

![diag-14](assets/display-arch/diag-14.png)

要点:

- **commit 与 composite 分离**:commit 只做数据整理,产出不可变快照;
  composite 只读快照执行合成。两阶段间没有共享可变状态,
  为跳帧(commit 后不执行 composite)和并行化提供了基础。
- **latch 条件**:一个事务要在本帧生效,必须满足其 buffer 的 acquire fence
  已(或即将)达成、`desiredPresentTime` 已到等条件,否则推迟到下一帧
  (`TransactionHandler` 的 ready filter);延迟一帧的现象通常源于此处。

### 4.3 FrontEnd:数据驱动的图层前端

Android 16 已完成去 Layer 树遍历的改造(`FrontEnd/readme.md` 有说明):

![diag-15](assets/display-arch/diag-15.png)

`Layer.cpp` 仍然存在,但仅作兼容用途;绘制状态的唯一来源(source of truth)
是 `RequestedLayerState`,合成端消费的是 `LayerSnapshot`。
详见《SF 详解》§4.2(含关键数据结构与代码摘录)。

### 4.4 Scheduler:VSYNC 与帧节奏

![diag-16](assets/display-arch/diag-16.png)

- SF 不依赖每个硬件 VSYNC 中断:`VSyncPredictor` 用最近样本拟合出周期与相位,
  之后由**软件预测**触发,present fence 时间戳持续校准模型,漂移超过阈值时才重新开启硬件采样。
- `LayerHistory/LayerInfo` 逐层统计提交节奏(视频 24fps、滚动 120fps…),
  投票给 `RefreshRateSelector`,与 DMS 的允许范围(见 2.6)共同决定当前模式;
  `FrameRateOverrideMappings` 支持对个别 App 下发不同于显示刷新率的 VSYNC(帧率覆盖)。
- `FrameTimeline` 为每帧建立预期与实际的时间线,jank 归因(App 超时/SF 超时/调度偏差)
  输出到 Perfetto。
详见《SF 详解》§4.3。

### 4.5 CompositionEngine:合成策略

每个显示器对应一个 `Output`(实现为 `Display`),每个可见 Layer 在该 Output 上
有一个 `OutputLayer`。composite 的核心决策是**每层使用硬件叠加还是 GPU 合成**,
流程如下:

![diag-17](assets/display-arch/diag-17.png)

触发 CLIENT 回退的典型原因:层数超过硬件 overlay 通道数、复杂变换/圆角/模糊、
色彩空间转换硬件不支持、HDR tonemap 需要 GPU shader 等。
详见《SF 详解》§4.4;RenderEngine 的 Skia 后端(GL/Vulkan 双实现,
`libs/renderengine/skia/`)亦在该节展开。

### 4.6 DisplayHardware 与 Display

- `DisplayHardware/HWComposer` 是 SF 侧对 composer3 HAL 的总封装:维护
  displayId ↔ hwcDisplayId 映射、能力缓存、每显示器的 `HWC2::Display/Layer`
  代理对象;实际 IPC 走 `AidlComposerHal.cpp`。
- `DisplayDevice` + `Display/` 目录管理每个显示器的运行时状态:当前模式、
  期望模式切换(`DisplayModeController`)、色彩模式、电源模式;
  模式切换的完整流程(desired → HWC setActiveConfigWithConstraints →
  等待 refresh → 完成回调)见《SF 详解》§4.6.3。
- 观测设施(`Tracing/` 的 LayerTracing/TransactionTracing、`TimeStats`、
  `FrameTracer`、`Jank/`)见《SF 详解》§4.7 与[第八章](#八调试与观测)。

---

## 五、HAL 层:Composer 3

代码根目录:`android16/hardware_interfaces/graphics/composer/`。
composer HAL 是 SF 与显示硬件之间的接口:SF 描述期望的最终画面(各图层的
buffer 与属性),vendor 实现决定其中哪些图层由显示硬件合成、哪些需要 SF 用 GPU 合成。

### 5.1 从 HIDL composer 2.x 到 AIDL composer3

仓库中两代并存:`composer/2.1 ~ 2.4`(HIDL,`IComposer.hal`,维护态)与
`composer/aidl`(`android.hardware.graphics.composer3`,现役)。
`aidl_api/android.hardware.graphics.composer3/` 下冻结版本为 **1/2/3/4**,
Android 16 对应 **V4**。演进主线:

| 版本 | 代表能力 |
|---|---|
| V1 (A13) | AIDL 化;亮度进合成命令(`DisplayBrightness`、`LayerBrightness`,HDR/SDR 混合调光);`ClientTargetPropertyWithBrightness`、`DimmingStage` |
| V2 (A14) | `OverlayProperties`(overlay 支持的格式/dataspace 组合);`getDisplayConfigurations`(带 vrrConfig 的 `DisplayConfiguration` 取代旧 attribute 查询) |
| V3 (A15) | **VRR 正式化**:`IComposerCallback.onRefreshRateChangedDebug`、`notifyExpectedPresent`、合成帧率与显示刷新率解耦 |
| V4 (A16) | `DisplayLuts`/`Luts`(把 3D LUT 色彩变换下沉到显示硬件)、`LutProperties`、picture profile(`DisplayHint`/画质处理器配额)等 |

### 5.2 命令模型:一次 IPC 提交一帧的全部命令

composer3 沿用并强化了 2.x 的**批量命令队列**设计:SF 不逐属性调用 HAL,
而是把一帧内对某显示器及其所有图层的设置打包为一个 `DisplayCommand`
(内含 `LayerCommand[]`),经共享内存 FMQ 风格的 `executeCommands` 一次提交:

![diag-18](assets/display-arch/diag-18.png)

V3 起新增 `LayerLifecycleBatchCommandType`:图层的创建/销毁也并入批量命令,
省去逐层的 createLayer/destroyLayer IPC。

### 5.3 validate / present 两阶段握手

SF 与 HWC 之间的核心交互流程如下:

![diag-19](assets/display-arch/diag-19.png)

- **DEVICE 合成**在功耗与延迟上优于 CLIENT:DPU 直接在扫描输出时叠加多个 plane,
  buffer 无拷贝上屏;CLIENT 回退则多一次 GPU 全屏渲染。
- `ClientTargetProperty` 允许 HWC 指定 ClientTarget 的最优格式/dataspace;
  带亮度的 `ClientTargetPropertyWithBrightness` 用于 HDR/SDR 混合场景
  (GPU 合成时需要对 SDR 内容做压暗处理)。

### 5.4 能力协商与显示配置

SF 启动及热插拔时通过以下接口查询硬件能力:

| 接口/类型 | 内容 |
|---|---|
| `getCapabilities` / `Capability` | 全局能力:跳过 validate、边合成边亮度等 |
| `getDisplayCapabilities` / `DisplayCapability` | 单显示器:DOZE、亮度控制、防烧屏建议 |
| `getDisplayConfigurations` → `DisplayConfiguration` | 每个模式的分辨率/刷新率/DPI/组(group)+ `vrrConfig`(最小帧间隔等) |
| `getDisplayIdentificationData` | EDID,产生稳定的物理显示 ID |
| `getOverlaySupport` → `OverlayProperties` | overlay 支持的像素格式×dataspace 组合 |
| `getHdrCapabilities` / `getPerFrameMetadataKeys` | HDR 类型与逐帧元数据 |
| `getDisplayedContentSample` | 屏幕内容直方图采样(自适应亮度用) |

### 5.5 回调通道:IComposerCallback

HAL → SF 的反向通知,是多条上层链路的源头:

![diag-20](assets/display-arch/diag-20.png)

---

## 六、Gralloc 图形内存子系统

代码位置:HAL 定义在 `android16/hardware_interfaces/graphics/allocator/`(分配)与
`graphics/mapper/`(映射);framework 侧封装在
`android16/frameworks_native/libs/ui/`(`GraphicBuffer*`、`Gralloc2~5.cpp`)。
Gralloc 解决所有图形模块共同依赖的问题:**一块能被 CPU/GPU/DPU/编解码器
共同访问的图形内存,如何分配、如何跨进程传递、如何正确读写**。

### 6.1 总体架构:allocate 与 map 分离

![diag-21](assets/display-arch/diag-21.png)

关键分工:

- **分配(allocator)开销大、频率低**:经 Binder IPC 到 vendor HAL 进程,由它
  向内核 dmabuf heap 申请物理内存,返回 `native_handle`(内含 dmabuf fd 与
  vendor 私有元数据)。
- **映射(mapper)开销小、频率高**:import/lock/unlock/元数据读写可能发生在每一帧,
  Android 16 用 **stable-c 接口**(`mapper/stable-c/include/.../IMapper.h`,
  `AIMapper_loadIMapper()` 入口)实现为**进程内直接调用**:`Gralloc5.cpp` 先向
  IAllocator 查询 `getIMapperLibrarySuffix()`,再 dlopen
  `mapper.<suffix>.so`,此后所有 mapper 调用都是普通函数调用,没有 IPC。
  相比 Gralloc4(HIDL IMapper 4.0,每次 lock 都是一次 HwBinder 往返),
  这是 mapper 路径上的主要性能改进。

`libs/ui` 里 `Gralloc2/3/4/5.cpp` 并存,`GraphicBufferAllocator/Mapper` 启动时
按设备实际 HAL 版本探测选用;Android 16 新平台为 Gralloc5
(= IAllocator AIDL V2 + AIMapper v5)。

**Gralloc 接口一览**

(1) 分配 HAL:`IAllocator`(AIDL V2,`hardware_interfaces/graphics/allocator/aidl/android/hardware/graphics/allocator/IAllocator.aidl`)

| 接口 | 作用 |
|---|---|
| `allocate(byte[] descriptor, int count)` | V1 接口:按 IMapper 4.0 编码的 descriptor 分配 count 块 buffer(兼容旧 mapper) |
| `allocate2(BufferDescriptorInfo descriptor, int count)` | V2 接口:按结构化的 `BufferDescriptorInfo{name, width, height, layerCount, format, usage, reservedSize, additionalOptions}` 分配,返回 `AllocationResult{stride, buffers[]}`(每个为 native_handle) |
| `isSupported(BufferDescriptorInfo)` | 查询该规格是否可分配,不实际分配(用于格式/用途探测) |
| `getIMapperLibrarySuffix()` | 返回 mapper 库后缀,framework 据此 dlopen `mapper.<suffix>.so`(V2 新增,是 stable-c mapper 的发现机制) |

(2) 映射 HAL:`AIMapperV5` 函数表(stable-c,`hardware_interfaces/graphics/mapper/stable-c/include/android/hardware/graphics/mapper/IMapper.h`,入口 `AIMapper_loadIMapper()`)

| 函数 | 作用 |
|---|---|
| `importBuffer(rawHandle, &outBuffer)` | 把跨进程收到的原始 native_handle 导入本进程,得到可用的 `buffer_handle_t`(建立映射、增加引用) |
| `freeBuffer(buffer)` | 释放导入的 handle(解映射、减引用) |
| `getTransportSize(buffer, &numFds, &numInts)` | 查询跨进程传输 handle 时需要携带的 fd/int 数量 |
| `lock(buffer, cpuUsage, region, acquireFence, &outData)` | 等待 acquireFence,把 buffer 映射为 CPU 可访问地址,必要时做 cache invalidate;返回虚拟地址 |
| `unlock(buffer, &releaseFence)` | 结束 CPU 访问,做 cache flush,返回 release fence |
| `flushLockedBuffer(buffer)` | 在持有 lock 期间把 CPU 写入刷到内存(不解锁) |
| `rereadLockedBuffer(buffer)` | 在持有 lock 期间使 CPU cache 失效以读取设备的新写入 |
| `getMetadata` / `getStandardMetadata` | 读取 vendor 或标准元数据(见 6.4) |
| `setMetadata` / `setStandardMetadata` | 写入元数据(如生产者动态改写 DATASPACE、HDR 元数据) |
| `listSupportedMetadataTypes` | 枚举实现支持的元数据类型 |
| `dumpBuffer` / `dumpAllBuffers` | 调试:导出单个/全部 buffer 的元数据 |
| `getReservedRegion(buffer, &addr, &size)` | 取得分配时 `reservedSize` 预留的、CPU 可访问的附加区域(供 vendor/框架存放私有数据) |

(3) framework 封装(`frameworks_native/libs/ui/`)

| 类/函数 | 文件 | 作用 |
|---|---|---|
| `GraphicBufferAllocator::allocate(w,h,format,layerCount,usage,&handle,&stride,requestorName)` | `GraphicBufferAllocator.cpp` | 进程单例;调用 `Gralloc5Allocator::allocate` → `IAllocator::allocate2`,并登记到进程内的分配记录(供 dumpsys 内存统计) |
| `GraphicBufferAllocator::free(handle)` | 同上 | 释放并从记录移除 |
| `GraphicBufferMapper::importBuffer / freeBuffer` | `GraphicBufferMapper.cpp` | 进程单例;转发到 `Gralloc5Mapper` → `AIMapperV5.importBuffer/freeBuffer` |
| `GraphicBufferMapper::lock / lockYCbCr / lockAsync / unlock / unlockAsync` | 同上 | CPU 访问入口;`*Async` 版本以 fence fd 形式传入/返回同步对象 |
| `GraphicBufferMapper::getXxx(handle, &out)`(getWidth/getUsage/getPlaneLayouts/getDataspace/…) | 同上 | 标准元数据读取的类型安全封装 |
| `GraphicBuffer` | `GraphicBuffer.cpp` | 一块 buffer 的对象表示(`ANativeWindowBuffer` 子类):构造时经 Allocator 分配、经 Mapper import;`lock/lockAsync/unlock` 转发到 Mapper;支持 Parcel 序列化(跨进程携带 handle) |
| `Gralloc5Allocator` / `Gralloc5Mapper` | `Gralloc5.cpp` | Gralloc5 版本适配层:前者持有 `IAllocator` Binder 代理,后者持有 dlopen 得到的 `AIMapper*` 函数表 |

### 6.2 关键流程:buffer 的分配与跨进程传递

**(1) 从 App 到 GraphicBufferAllocator 的调用链**

分配不是由 App 直接发起的,而是在 `dequeueBuffer` 过程中按需触发:

![diag-29](assets/display-arch/diag-29.png)

说明:

- ①~④:生产者(HWUI RenderThread、相机、解码器等)通过 `ANativeWindow`
  接口调用 `Surface::dequeueBuffer`,进入 `IGraphicBufferProducer::dequeueBuffer`。
  BLAST 场景下该 producer 是 App 进程内的 `BufferQueueProducer`(3.3),
  跨进程 BufferQueue 场景下则是 Binder 代理。
- ④~⑤:`BufferQueueProducer::dequeueBuffer` 从空闲集合选出槽位;仅当该槽位
  尚无 GraphicBuffer,或其尺寸/格式/usage 与本次请求不一致(`needsReallocation`)时,
  才以 `sp<GraphicBuffer>::make(...)` 新建 buffer。这意味着稳定运行时(槽位已满
  且规格不变)`dequeueBuffer` 不触发任何分配。
- ⑤~⑨:`GraphicBuffer` 构造函数调用 `initWithSize`,进入进程单例
  `GraphicBufferAllocator::allocate`,由 `Gralloc5Allocator` 组装
  `BufferDescriptorInfo` 并经 AIDL Binder 调用 vendor 进程的
  `IAllocator::allocate2`,后者向内核 dmabuf heap 申请物理内存并返回 native_handle
  与 stride。
- ⑩:分配返回的是原始 handle,`GraphicBuffer` 随即调用
  `GraphicBufferMapper::importBuffer`(进程内直调 `AIMapperV5.importBuffer`)
  得到本进程可用的 `buffer_handle_t`。
- ⑪:槽位记录该 GraphicBuffer,`dequeueBuffer` 返回 `BUFFER_NEEDS_REALLOCATION`
  标志,`Surface` 据此调用 `requestBuffer` 取回句柄。

**(2) 分配完成后 buffer 的跨进程传递**

![diag-22](assets/display-arch/diag-22.png)

配套要点:

- **usage 位是分配的规格约定**(`BufferUsage`:CPU_READ/WRITE、GPU_TEXTURE/RENDER_TARGET、
  COMPOSER_OVERLAY、VIDEO_ENCODER、PROTECTED…):它同时决定选哪个 heap、
  cache 策略、是否允许压缩格式(如 UBWC/AFBC)。申请时声明全部用途,才能保证
  同一块 buffer 在相机→编码器→GPU→DPU 之间无拷贝流转。
- **format 可以是 IMPLEMENTATION_DEFINED / FLEX**:真实内存排布由 vendor 决定,
  调用方通过 mapper 的 metadata 接口(而不是假设 stride)读取
  `PlaneLayouts`/`PixelFormatFourCC` 等标准元数据(`StandardMetadataType`,
  定义于 `graphics/common`)。
- **跨进程传递的是 handle 不是内存**:`native_handle` 里的 dmabuf fd 经 Binder
  dup 后在对端 `importBuffer`,物理内存全程只有一份。

### 6.3 CPU 访问:lock / unlock

以下场景需要 CPU 直接读写 buffer 内存:软件绘制(`Canvas` 非硬件加速路径、
`SurfaceHolder.lockCanvas`)、截图/录屏的像素回读、`ImageReader` 读取相机
YUV 帧、`AHardwareBuffer_lock` NDK 调用。lock/unlock 的语义如下:

![diag-23](assets/display-arch/diag-23.png)

**软件绘制的完整调用链(以 `SurfaceHolder.lockCanvas` 为例)**

![diag-30](assets/display-arch/diag-30.png)

说明:

- ①~③:Java 层 `Surface.lockCanvas`(`frameworks_base/core/java/android/view/Surface.java`)
  经 JNI `nativeLockCanvas`(`core/jni/android_view_Surface.cpp`)调用
  `ANativeWindow_lock`(`frameworks_native/libs/nativewindow/ANativeWindow.cpp`)。
- ④:`Surface::lock`(`libs/gui/Surface.cpp`)内部先执行一次
  `Surface::dequeueBuffer` 取得后备 buffer 及其 release fence;若窗口尺寸变化
  或前一帧内容需要保留,还会在此把前一帧未脏区域拷贝到新 buffer。
- ⑤~⑦:`GraphicBuffer::lockAsync` → `GraphicBufferMapper::lockAsync`
  → `Gralloc5Mapper::lock`,usage 为 `GRALLOC_USAGE_SW_WRITE_OFTEN`,
  fence fd 为 dequeue 返回的 release fence。
- ⑧:vendor 的 `AIMapperV5.lock` 在调用方进程内执行:等待传入 fence
  (确保显示端已不再读取该内存)、建立 CPU 映射、按需 cache invalidate,
  返回虚拟地址。Java 层随后把 `Canvas` 绑定到该地址,由 Skia 软件光栅化。
- ⑨~⑪:`unlockCanvasAndPost` 反向执行 `unlockAsync`(cache flush、返回
  release fence),然后 `Surface::queueBuffer` 把 buffer 提交,进入 3.2 的
  BLAST 链路。此路径下 acquire fence 通常为 `Fence::NO_FENCE`(CPU 写入在
  unlock 时已完成)。

mapper 在 lock/unlock 中负责 cache 一致性维护与私有格式(压缩、tiling)的
解算,这些操作与具体 SoC 相关,因此 mapper 必须由 vendor 实现。

### 6.4 元数据(Metadata)机制

Gralloc4 起引入、AIMapper v5 延续的标准化元数据接口
(`getMetadata/setMetadata` + `StandardMetadataType`),使 buffer 的属性可以
由 buffer 自身携带并被任何进程查询:

| 元数据 | 用途 |
|---|---|
| WIDTH/HEIGHT/PIXEL_FORMAT_REQUESTED/USAGE | 分配参数回查 |
| PLANE_LAYOUTS | 真实内存排布(offset/stride/子采样),FLEX 格式解读的标准途径 |
| DATASPACE | 色彩空间(可由生产者动态改写,SF/HWC 读取) |
| SMPTE2086 / CTA861_3 / SMPTE2094_40 | HDR 静态/动态元数据,随 buffer 走到 HWC |
| COMPRESSION / INTERLACED / CHROMA_SITING | vendor 压缩与视频属性 |
| BUFFER_ID / NAME / ALLOCATION_SIZE | 调试与内存记账(dumpsys、perfetto 内存曲线) |

这套机制取代了旧的 vendor 私有 `perform()` 调用,是 HDR 视频、相机 P010、
UBWC 压缩帧能在标准框架内流转的前提。

### 6.5 Gralloc 在一帧显示链路中的参与点

本节把第六章的内容对应到 1.2 的主链路,说明一块 buffer 从分配到释放的整个
生命周期中,Gralloc 在哪些环节、哪个进程被调用,以及为什么整条路径不需要
像素拷贝。

![diag-31](assets/display-arch/diag-31.png)

| 环节 | 所在进程 | Gralloc 操作 | 说明 |
|---|---|---|---|
| ① 分配 | App | `IAllocator::allocate2`(IPC)+ `AIMapper importBuffer` | 仅在 `dequeueBuffer` 首次取到空槽位或规格变化时发生(6.2);稳定运行时不再分配 |
| ② 生产 | App | GPU 路径:不调用 mapper,GPU 驱动直接以 handle 建立纹理/渲染目标;CPU 路径:`lock`/`unlock`(6.3) | 内容写入 dmabuf 物理页 |
| ③ 跨进程传递 | App → SF | Binder 传输 `GraphicBuffer`(handle 内的 dmabuf fd 被 dup);SF 进程 `AIMapper importBuffer` | 传递的是句柄,物理内存不复制;SF 侧的 `ClientCache` 缓存已 import 的 buffer,避免每帧重复 import |
| ④ 合成 | SF | DEVICE 合成:把 handle 写入 composer3 `LayerCommand.buffer`;CLIENT 合成:RenderEngine 以 handle 建立 GPU 纹理 | 两条路径都不经过 CPU 拷贝 |
| ⑤ 送显 | vendor HAL / DPU | 无 mapper 调用;DPU 直接扫描该 dmabuf 物理页 | DEVICE 合成时,App GPU 写入的内存就是 DPU 读取的内存 |
| ⑥ 释放 | 各进程 | `AIMapper freeBuffer` | 各进程各自解映射;内核 dmabuf 引用计数归零后释放物理页 |

结论:在 DEVICE 合成路径下,从 App GPU 写入到 DPU 扫描输出,像素数据始终位于
同一块 dmabuf 物理内存中,期间只有句柄传递与映射建立,没有像素拷贝;
Gralloc 的分配只在 buffer 生命周期开始时发生一次,逐帧开销仅为 import
(有缓存)与元数据读取。

---

## 七、关键跨层专题

每个专题给出一条端到端链路图,把前六章的模块按实际调用关系连接起来。

### 7.1 可变刷新率(VRR)全链路

Android 15/16 的 VRR 把显示刷新率与合成/渲染帧率解耦:面板保持高刷新率,
内容可以按低帧率渲染,由硬件决定何时刷新面板。

![diag-24](assets/display-arch/diag-24.png)

效果:视频 24fps 播放时,App 只按 24Hz 接收 VSYNC、SF 只按 24Hz 合成,
面板由硬件维持在合适刷新率,CPU/GPU/DPU 三级同时降低功耗,且切换过程
没有传统模式切换的黑帧/闪烁。

### 7.2 HDR 与色彩管理

![diag-25](assets/display-arch/diag-25.png)

Java 层入口:`Window.setDesiredHdrHeadroom`、`Display.getHdrSdrRatio`
(数据源是 DMS 亮度链路与 SF 的 `HdrLayerInfoReporter` 互通)。
Android 16 的 `ActivePictureTracker`(SF)配合 composer3 V4 的 picture profile,
把画质增强处理器资源的按层分配也纳入了这条链路。

### 7.3 多显示器与折叠屏

三种场景走同一套 2.3 的对象模型,差异在映射策略:

| 场景 | 链路要点 |
|---|---|
| 折叠屏开合 | 传感器 → DeviceStateManager → `LogicalDisplayMapper` 按 `DeviceStateToLayoutMap` 重绑 display 0 的 DisplayDevice → WMS 配置变更 → SF 侧仅是另一个物理 display 的 on/off |
| HDMI/DP 外接 | HWC `onHotplugEvent` → SF → DMS `LocalDisplayAdapter`(2.3 时序图)→ `ExternalDisplayPolicy` 决定启用与镜像/扩展 → 16 的 `DisplayTopologyCoordinator` 恢复排列 |
| 虚拟屏(录屏/投屏) | App/系统经 `VirtualDisplayAdapter` 创建 → SF 里对应一个以 GPU 为输出的 Output(无 HWC),合成结果写进调用方给的 Surface |

**镜像 vs 扩展**在 SF 侧的实现区别:镜像 = 两个 Output 引用同一棵 layer 子树
(FrontEnd 的 LayerHierarchy 支持 mirror 关系);扩展 = WMS 把不同窗口分配到
不同 layerStack,SF 按 layerStack 过滤每个 Output 的可见层。

### 7.4 屏幕截图与录屏路径

截图是一次不经过 HWC、完全由 GPU 执行的合成:

![diag-26](assets/display-arch/diag-26.png)

录屏/投屏不使用这条按需路径,而是 7.3 的虚拟屏:每帧常规合成,输出连续。
安全窗口(FLAG_SECURE)在两条路径中都会被过滤或涂黑。

### 7.5 亮度调节全链路

![diag-27](assets/display-arch/diag-27.png)

架构要点:亮度最终**作为合成命令的一部分**下发(而非独立的 sysfs 写入),
从而实现亮度变化与画面内容同帧生效。这对 HDR/SDR 混合调光(7.2)是必需的,
因为 SDR 层的压暗系数必须与背光变化严格同步。

---

## 八、调试与观测

### 8.1 dumpsys 速查

| 命令 | 输出内容 |
|---|---|
| `dumpsys SurfaceFlinger` | 图层树快照、每层合成类型(DEVICE/CLIENT)、显示器状态、HWC 能力、VSYNC 状态 |
| `dumpsys SurfaceFlinger --latency <layer>` | 逐帧时间戳(desiredPresent/actualPresent/frameReady) |
| `dumpsys display` | DMS 全量:LogicalDisplay/DisplayDevice 映射、DPC 状态、`BrightnessReason`、DisplayModeDirector 各投票现值 |
| `dumpsys gfxinfo <pkg>` | App 侧 HWUI 帧耗时直方图与 jank 统计 |
| `service call SurfaceFlinger 1008` 等 | 强制 GPU 合成等调试开关(平台调试用) |

定位问题层次的第一步:在 `dumpsys SurfaceFlinger` 中查看目标层的
**composition type** 与 **buffer 计数**,在 `dumpsys display` 中查看
**BrightnessReason/投票列表**;两者分别反映机制层与策略层的当前状态。

### 8.2 Perfetto / Winscope / 追踪设施

| 工具/设施 | 数据源(代码) | 用途 |
|---|---|---|
| Perfetto `surfaceflinger.frametimeline` | `Scheduler/FrameTimeline` | 每帧预期与实际时间线,jank 类型归因(AppDeadlineMissed/SurfaceFlingerCpuDeadlineMissed…) |
| Perfetto `android.gpu.memory` + gralloc 记账 | mapper metadata(ALLOCATION_SIZE) | 图形内存归属 |
| Winscope layer trace | `Tracing/LayerTracing`(围绕 LayerSnapshot) | 逐帧回放图层树/可见区域/变换 |
| Winscope transaction trace | `Tracing/TransactionTracing` | 记录每个 layer 属性变更的来源与时刻(BLAST 架构下定位属性问题的主要手段) |
| `FrameTracer`/`TimeStats` | 同名目录 | 长期帧统计上报(statsd) |

### 8.3 常见问题定位路径

| 症状 | 排查链路 |
|---|---|
| 掉帧/卡顿 | FrameTimeline(哪一方错过 deadline)→ App 侧 gfxinfo → SF 主线程 trace(commit/composite 耗时)→ 是否 CLIENT 回退导致 GPU 峰值 |
| 合成回退 GPU(功耗高) | dumpsys 查看哪些层为 CLIENT → validate 返回的 ChangedCompositionTypes → 对照 5.4 的 OverlayProperties(格式/dataspace 不支持?层数超限?) |
| 黑屏/闪屏 | 分层定位:App 有无 queueBuffer(BBQ 日志)→ SF 有无 latch(transaction trace)→ HWC present 有无错误(CommandError)→ 电源状态(dumpsys display 的 DPC 状态机) |
| 撕裂/错位 | 检查是否绕过了 Transaction 原子性(直接 setGeometry 与 buffer 分离提交);present fence 时间是否异常 |
| 模式切换失败/闪烁 | DMD 投票现值 → SF `DisplayModeController` 状态 → HWC `onVsyncPeriodTimingChanged` 是否回调 |

---

## 附录 A:核心类速查表

| 类 | 路径(android16/ 下) | 职责 |
|---|---|---|
| `DisplayManagerService` | `frameworks_base/services/core/java/com/android/server/display/` | 显示策略中枢 Binder 服务 |
| `LocalDisplayAdapter` | 同上 | 物理屏接入,SF 热插拔事件入口 |
| `LogicalDisplayMapper` | 同上 | DisplayDevice→LogicalDisplay 动态映射(折叠屏核心) |
| `DisplayPowerController` | 同上 | 每个显示器的电源/亮度执行 |
| `DisplayModeDirector` | `.../display/mode/` | 刷新率投票收敛 |
| `DisplayBrightnessController` | `.../display/brightness/` | 亮度策略选择 |
| `DisplayTopologyCoordinator` | `.../display/` | 多屏相对位置拓扑(16 新增) |
| `Choreographer` | `frameworks_base/core/java/android/view/` | App 帧节奏调度 |
| `BLASTBufferQueue` | `frameworks_native/libs/gui/` | App 进程内 buffer 队列 + Transaction 提交 |
| `SurfaceComposerClient::Transaction` | `libs/gui/SurfaceComposerClient.cpp` | 原子状态提交单元 |
| `BufferReleaseChannel` | `libs/gui/` | buffer 归还快速通道(16 新增) |
| `SurfaceFlinger` | `services/surfaceflinger/` | 合成服务主类(ICompositor) |
| `LayerLifecycleManager` / `LayerSnapshotBuilder` | `.../surfaceflinger/FrontEnd/` | 前端状态→每帧快照 |
| `Scheduler` / `VsyncSchedule` / `VSyncPredictor` | `.../surfaceflinger/Scheduler/` | VSYNC 建模与帧调度 |
| `RefreshRateSelector` / `LayerHistory` | 同上 | 刷新率最终选择与逐层帧率统计 |
| `FrameTimeline` | 同上 | 帧时间线与 jank 归因 |
| `CompositionEngine` / `Output` / `OutputLayer` | `.../surfaceflinger/CompositionEngine/` | 合成策略执行 |
| `HWComposer` / `AidlComposerHal` | `.../surfaceflinger/DisplayHardware/` | composer3 HAL 封装 |
| `RenderEngine`(Skia) | `frameworks_native/libs/renderengine/` | GPU 合成/截图执行器 |
| `IComposer/IComposerClient/IComposerCallback` | `hardware_interfaces/graphics/composer/aidl/` | 合成 HAL 接口(V4) |
| `IAllocator`(AIDL V2) | `hardware_interfaces/graphics/allocator/` | 图形内存分配 HAL |
| `AIMapper`(stable-c v5) | `hardware_interfaces/graphics/mapper/stable-c/` | 图形内存映射(进程内直调) |
| `GraphicBuffer` / `GraphicBufferAllocator` / `GraphicBufferMapper` / `Gralloc5` | `frameworks_native/libs/ui/` | framework 侧 Gralloc 封装 |

## 附录 B:与既有文档的关系与阅读顺序

本文是同目录文档群的**总入口**,建议阅读顺序:

1. **本文**:建立全栈框架与各层边界。
2. [surfaceflinger16_claude.md](surfaceflinger16_claude.md):SF 内部逐模块深读
   (类图/数据结构/流程/关键代码五段式),本文第四章的所有 §引用指向它。
3. [sf16_class_all_claude.md](sf16_class_all_claude.md) + 配图
   `sf16-class-claude.jpg`:SF 89 个核心类的一张总图与 16 条关键关系。
4. [surfaceflinger16.md](surfaceflinger16.md)、[android16-sf-分析.md](android16-sf-分析.md):另一视角的 SF 综述与
   SurfaceFlinger.h 成员级解析,可作交叉验证。
5. [displaymanager-doc.md](displaymanager-doc.md) + `displaymanager16-class.png`:DMS 类图细节。
6. 需要版本对比时再读:`Android13-vs-16-DisplayManager-SurfaceFlinger-HWC-变更汇总.md`、
   `android13-16-update-all.md`、`android13_16_hwc_compare.md`、
   `Android13-vs-16-显示栈与HWC全链路源码对比.md`。

## 附录 C:术语表

| 术语 | 含义 |
|---|---|
| VSYNC | 垂直同步信号;SF 据此建模出 VSYNC-app/VSYNC-sf 两个相位不同的软件信号 |
| BLAST | Buffer Layer Surface Transaction,buffer 随事务原子提交的生产端架构 |
| latch | SF 在 commit 阶段采用某个 buffer/状态进入本帧 |
| Layer / LayerSnapshot | SF 中的合成单元 / 其每帧不可变快照 |
| layerStack | 逻辑显示器的图层命名空间,Output 按它过滤可见层 |
| DEVICE / CLIENT 合成 | 显示硬件 overlay 直接叠加 / GPU 先合成为整屏 ClientTarget |
| ClientTarget | CLIENT 合成的输出目标 buffer,作为一个特殊 plane 交给 HWC |
| overlay / plane | 显示控制器可在扫描输出时直接叠加的硬件图层通道 |
| fence(acquire/release/present) | 跨设备异步同步原语,见 3.5 |
| dmabuf / native_handle | 内核跨设备共享内存机制 / 携带其 fd 与元数据的跨进程句柄 |
| dataspace | 色彩空间+传递函数+范围的组合描述 |
| tonemap | HDR→显示能力范围的亮度/色彩映射 |
| VRR / LFC | 可变刷新率 / 低帧率补偿(硬件自动重复帧) |
| DPU | Display Processing Unit,即显示控制器 |
| aconfig | Android 配置化功能开关系统(`*.aconfig` 文件) |
| FMQ | Fast Message Queue,HAL 命令批量传输所用共享内存队列 |
