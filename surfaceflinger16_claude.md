# SurfaceFlinger 全面文档(基于 Android 16 源码)

> 源码路径:`frameworks/native/services/surfaceflinger/`(Android 16)
> 编写依据:逐文件阅读 `SurfaceFlinger.cpp/h`、`FrontEnd/`、`Scheduler/`、`CompositionEngine/`、`DisplayHardware/`、`Display/`、`Tracing/`、`PowerAdvisor/`、`TimeStats/`、`Jank/`、`FrameTracer/`、`Effects/`、`Utils/` 等模块的实际代码与 `readme.md`,所有类名/方法名/文件路径均与 A16 源码一致。

## 目录

- [一、基础知识介绍](#一基础知识介绍)
  - [1.1 SurfaceFlinger 是什么 / 在 Android 中的定位](#11-surfaceflinger-是什么--在-android-中的定位)
  - [1.2 进程模型与启动](#12-进程模型与启动)
  - [1.3 与上层模块的交互](#13-与上层模块的交互)
  - [1.4 与下层模块的交互](#14-与下层模块的交互)
  - [1.5 整体数据流与控制流](#15-整体数据流与控制流)
- [二、设计模式与模块组成](#二设计模式与模块组成)
  - [2.1 总体设计哲学](#21-总体设计哲学)
  - [2.2 所用主要设计模式](#22-所用主要设计模式)
  - [2.3 顶层模块清单与职责](#23-顶层模块清单与职责)
  - [2.4 模块间的依赖关系](#24-模块间的依赖关系)
- [三、整体类图与主时序图](#三整体类图与主时序图)
  - [3.1 顶层类图](#31-顶层类图)
  - [3.2 启动时序](#32-启动时序)
  - [3.3 一帧的主时序](#33-一帧的主时序)
- [四、各子模块详细分析](#四各子模块详细分析)
  - [4.1 SurfaceFlinger 主类](#41-surfaceflinger-主类)
  - [4.2 FrontEnd](#42-frontend)
  - [4.3 Scheduler](#43-scheduler)
  - [4.4 CompositionEngine](#44-compositionengine)
  - [4.5 DisplayHardware(HWComposer)](#45-displayhardwarehwcomposer)
  - [4.6 Display](#46-display)
  - [4.7 Tracing](#47-tracing)
  - [4.8 TimeStats / Jank / FrameTracer](#48-timestats--jank--frametracer)
  - [4.9 PowerAdvisor](#49-poweradvisor)
  - [4.10 Effects / Reporters / Overlays](#410-effects--reporters--overlays)
  - [4.11 Utils / Factory / 工厂注入](#411-utils--factory--工厂注入)

---

## 一、基础知识介绍

### 1.1 SurfaceFlinger 是什么 / 在 Android 中的定位

**SurfaceFlinger(下文简称 SF)是 Android 系统级的合成器(System Compositor)**。它是一个独立的 native 系统进程(`/system/bin/surfaceflinger`),职责是把所有应用/系统进程提交的若干"图层(Layer)及其缓冲区(buffer)"按 z 序、几何与色彩属性合成为最终屏幕画面,并经显示控制器扫出。

![SurfaceFlinger 在 Android 16 显示栈中的位置与交互](sf-assets/01-sf-position.svg)

把 SF 放在 Android 显示栈的位置坐标系里看:

| 上方 | App 进程(UI 线程 + RenderThread)、SystemUI、`system_server` 中的 WindowManagerService / DisplayManagerService、InputFlinger |
| --- | --- |
| **SF 自身** | **独立进程,经 Binder 暴露 `ISurfaceComposer` 等接口,内部驱动 HWC 完成合成** |
| 下方 | Composer HAL(AIDL `composer3`)、Gralloc(`IAllocator` / `IMapper`)、Power HAL(ADPF)、GPU 用户态驱动 + 内核 DRM/KMS |

SF 的关键事实有四条,后续章节都围绕它们展开:

1. **它是 Binder 服务端**。其类 `SurfaceFlinger` 继承 `BnSurfaceComposer`,把"创建层、提交事务、查询显示信息、截屏"等能力暴露给所有客户端。
2. **它是 HWC 的回调订阅者**。其类 `SurfaceFlinger` 同时继承 `HWC2::ComposerCallback`,接收来自硬件的 hotplug/vsync/refresh 通知。
3. **它的"主线程"由 Scheduler 反向驱动**。SF 实现了 `scheduler::ICompositor` 接口,Scheduler 在每个 vsync 到达时回调 `configure() / commit() / composite() / sample()` 四步,SF 不像传统服务那样自己 spin 一个 loop。
4. **它通过 Factory 注入所有可替换组件**。`SurfaceFlingerFactory::Factory` 接口让 `HWComposer`、`DisplayDevice`、`Layer`、`LayerFE`、`CompositionEngine`、`FrameTimeline`、`VsyncConfiguration`、`VsyncController`、`FrameTracer` 等都可被替换为测试/mock 实现。

### 1.2 进程模型与启动

SF 由 `init` 通过 `surfaceflinger.rc` 拉起:

```rc
service surfaceflinger /system/bin/surfaceflinger
    class core animation
    user system
    group graphics drmrpc readproc
    capabilities SYS_NICE
    onrestart restart --only-if-running zygote
    task_profiles HighPerformance
```

要点:

- **`class core animation`**:`init` 阶段(early-init → init → late-init)归类,与 zygote 同属 core 类。
- **`user system / group graphics drmrpc readproc`**:运行身份;`graphics` 组允许访问 GPU 设备节点,`drmrpc` 用于 DRM 通信。
- **`capabilities SYS_NICE`**:可设置 SCHED_FIFO 实时优先级。
- **`onrestart restart --only-if-running zygote`**:SF 重启会带着 zygote 重启,从而所有应用同时重启——因为 App 持有的 `SurfaceControl` 句柄会失效。
- **`task_profiles HighPerformance`**:加入高性能 CPU 调度档。

进程入口 `main_surfaceflinger.cpp`:

```cpp
int main() {
    signal(SIGPIPE, SIG_IGN);

    hardware::configureRpcThreadpool(1, false);
    startGraphicsAllocatorService();                  // (条件) passthrough Gralloc HIDL
    ProcessState::self()->setThreadPoolMaxThreadCount(4);   // Binder 线程池 = 4
    SurfaceFlinger::setSchedAttr(true, __func__);    // uclamp.min 设置
    // ... 创建 sp<SurfaceFlinger>,init(), 注册到 ServiceManager,run()
}
```

SF 进程内有几类长寿命线程:

| 线程 | 作用 |
| --- | --- |
| **主线程**(`SurfaceFlinger` 自身) | 跑 `MessageQueue`,接收 vsync 触发的 invalidate,执行 `commit/composite/sample`。被设置为 SCHED_FIFO + uclamp.min。 |
| **Binder 线程池**(4 个) | 处理 ISurfaceComposer / Client / Producer 等入站 IPC 调用。 |
| **EventThread**(App / SF 各一) | 由 `Scheduler::createEventThread(Cycle)` 创建,负责把 vsync 派发给 App 的 `DisplayEventReceiver` 与 SF 主线程。 |
| **RenderEngine 工作线程** | `librenderengine` 内部线程,执行 GPU 离屏渲染(client 合成)。 |
| **HwcAsyncWorker** | 把 HWC validate/present 放到工作线程,与主线程并行。 |
| **BackgroundExecutor** | 非关键路径杂活,如延迟释放、统计上报。 |
| **RegionSamplingThread** | 屏幕区域颜色采样(给 SystemUI 提供动态色)。 |

### 1.3 与上层模块的交互

SF 对上层暴露三条核心 Binder 接口:`ISurfaceComposer`(全局服务能力)、`ISurfaceComposerClient`(每客户端会话)、`IGraphicBufferProducer`(向 SF 提交 buffer 的生产者端,通常透传给 BufferQueue)。

**App / SystemUI 通常通过 libgui 客户端库使用 SF**,主要场景:

1. **创建图层与表面**
   - `SurfaceComposerClient::createSurface()` → `ISurfaceComposerClient::createSurface()` → SF 在内部创建 `Layer` + `LayerFE` + `RequestedLayerState`,通过 handle(强引用 binder)交还给 App。
   - 现代应用普遍使用 `BLASTBufferQueue` 模式:App 自己跑 BufferQueue 的生产/消费两端,把生产出的 buffer 通过事务的 `setBuffer()` 提交给 SF。

2. **提交事务(Transaction)**
   - App 侧 `SurfaceControl.Transaction` 收集若干"对若干 SurfaceControl 的更改"(`setPosition/setLayer/setBuffer/setAlpha/...`),`apply()` 时打包成 `TransactionState` 调 `ISurfaceComposer::setTransactionState(state, applyToken)`。
   - 同一 `applyToken` 内事务严格有序;不同 token 间独立,可并发,默认每个 BufferProducer 一个独立 token。
   - 跨进程定序通过 barrier 实现:进程 A 入队带 WAIT token `t`,进程 B 入队带 SIGNAL token `t`,A 会等 B 的 SIGNAL 才真正"就绪"。

3. **订阅 vsync / hotplug**
   - `createDisplayEventConnection(VsyncSource, EventRegistrationFlags, layerHandle, eventDispatch)` 返回 `IDisplayEventConnection`,App 通过 `BitTube` 收 vsync/hotplug/modeChange/frameRateOverride。
   - Android Choreographer 即基于此通道。

4. **截屏与画面捕获**
   - `captureDisplay(DisplayCaptureArgs, IScreenCaptureListener)`、`captureLayers(LayerCaptureArgs, IScreenCaptureListener)`:SF 上构造一个临时 Output(`ScreenCaptureOutput`),把目标内容合成到指定 buffer,异步通知监听者。

5. **窗口信息(给 input/可访问性)**
   - `WindowInfosListener` 接口:输入子系统订阅 SF 派发的 WindowInfo(窗口对应的 SurfaceControl 边界、是否可触摸等),用于触摸命中测试。SF 内由 `WindowInfosListenerInvoker` 负责广播。

**WindowManagerService(WMS,在 `system_server`)** 是 SF 最重要的上层客户端:

- WMS 持有每个窗口对应的 `SurfaceControl`,管理它们的层级/几何/裁剪;在窗口动画、转场过渡时使用 SF 的事务原子提交多窗口的几何变更。
- WMS 也持有"显示"维度的 `SurfaceControl`(显示根),决定 `LayerStack`、旋转、display projection。
- WMS 不直接创建合成策略,真正的合成策略在 SF 内由 `Scheduler` + `RefreshRateSelector` 决定,WMS 只通过 `setDesiredDisplayModeSpecs()` 等接口表达"我希望 60–120Hz 可切"等偏好。

**DisplayManagerService(DMS)** 通过 `getStaticDisplayInfo / getDynamicDisplayInfo / setDisplayBrightness / setHdrConversionMode / setAutoLowLatencyMode / setGameContentType` 等接口在 SF 上控制显示模式、亮度、HDR。

### 1.4 与下层模块的交互

下层主要是三类 HAL 与 GPU:

1. **Composer HAL**(`android.hardware.graphics.composer3` AIDL)
   - SF 在 `DisplayHardware/HWComposer` 中包装为 `HWComposer` 接口,具体实现 `AidlComposerHal`(主路径)与 `HidlComposerHal`(向后兼容)。
   - SF → HAL 关键调用:`createLayer / setLayer* / validateDisplay / presentDisplay / setClientTarget / setPowerMode / setDisplayBrightness`。
   - HAL → SF 回调(`HWC2::ComposerCallback`):`onVsync(timestamp)`、`onHotplug(connected)`、`onRefresh()`、`onVsyncPeriodTimingChanged()`、`onSeamlessPossible()`、`onHdcpLevelsChanged()`。

2. **Gralloc**(`IAllocator` + `IMapper`)
   - SF 自己很少直接分配 buffer(只在虚拟显示、截屏、client 合成中分配 framebuffer);绝大多数 buffer 由 App 端通过 Gralloc 分配后,经 BufferQueue/BLAST 透传到 SF。SF 通过 `GraphicBuffer` 抽象访问,底层走 Gralloc5 stable-C mapper(A16)或回退 Gralloc4 HIDL。

3. **Power HAL**(ADPF / Performance Hint)
   - SF 通过 `PowerAdvisor/` 子模块向 Power HAL 上报"目标 work duration"与"实际 work duration",HAL 据此调节 CPU/GPU 频率。

4. **GPU 用户态驱动 / DRM**
   - 实际像素的栅格化(client 合成)经 `librenderengine` 在 Skia/Vulkan(或保留的 SkiaGL)路径完成,最终由 GPU 驱动落到 DMA-BUF;扫出由内核 DRM/KMS 完成,SF 通过 HWC 与之间接交互。

### 1.5 整体数据流与控制流

**数据流(buffer flow):**

```
App.Surface → 队列(BufferQueue/BLAST) → 当 vsync 抵达,SF 通过 acquireBuffer 拿到 buffer
   → CompositionEngine 把每个 LayerSnapshot 的 buffer 通过 OutputLayer.writeStateToHWC()
   → HWComposer.setLayerBuffer (per HWC layer)
   → HWComposer.validateDisplay → 返回 device/client 分配
   → 对 client 合成层,RenderEngine 在 GPU 离屏渲染到 framebuffer
   → HWComposer.setClientTarget(framebuffer) + presentDisplay()
   → 显示控制器扫出 → 像素到面板
```

**控制流(transaction & timing):**

```
App.Transaction.apply() → SF.setTransactionState() → TransactionHandler.queueTransaction() → scheduleCommit()
                                                                                    ↑
HWC.onVsync ─► VsyncSchedule.addPresentFence/predict ─► VSyncDispatch.callback(SF) ─┘
                                                          │
                                                          ▼
                              SF.ICompositor::configure() → commit() → composite() → sample()
```

每帧"事务何时被消费"由 vsync 节拍决定:事务到了 SF 仍可能等下一帧才被 flush;Scheduler 根据 work duration 计算最优唤醒时刻,通过 `VSyncDispatch` 在 vsync 前若干毫秒唤醒 SF 主线程。

---

## 二、设计模式与模块组成

### 2.1 总体设计哲学

读 A16 的 SF 源码,有四个明显的设计意图:

1. **职责单一、组件可替换**:每个子目录是一个有边界的模块,`SurfaceFlingerFactory` 让所有重型组件可通过依赖注入替换,服务于测试与产品差异化。
2. **数据流不可变 + 后台并行**:典型如 FrontEnd 的 `LayerSnapshot`、Tracing 的 `TransactionRingBuffer` 与 `LocklessStack/Queue`,生产者一次写入、消费者只读复制——为多线程并行让路。
3. **时序由 vsync 反向驱动**:SF 不维护自己的"渲染循环",Scheduler 借助 `ICompositor` 接口在合适的时刻反向调主线程,把"渲染节拍"与"业务逻辑"清晰分离。
4. **HAL 边界清晰**:`HWComposer` 是接口,`AidlComposerHal` / `HidlComposerHal` 是两套实现,A16 主推 AIDL `composer3`,HIDL 仅做兼容回退。

### 2.2 所用主要设计模式

![SurfaceFlinger 内部模块组成与依赖关系](sf-assets/02-sf-modules.svg)

| 模式 | 在 SF 中的体现 |
| --- | --- |
| **Factory(工厂)** | `surfaceflinger::Factory` 抽象工厂 + `SurfaceFlingerDefaultFactory` 默认实现,统一创建 HWComposer / DisplayDevice / Layer / LayerFE / CompositionEngine / FrameTimeline / VsyncConfiguration / VsyncController / GraphicBuffer / FrameTracer。 |
| **Observer / Callback** | `HWC2::ComposerCallback`(HWC→SF)、`scheduler::ICompositor`(Scheduler→SF)、`scheduler::ISchedulerCallback`(Scheduler→SF 的另一族回调)、`compositionengine::ICEPowerCallback`、`ILifecycleListener`(FrontEnd 层生命周期监听)、`TransactionFilter`(事务就绪过滤)、`WindowInfosListener`。SF 类同时实现多个,聚合多种"被回调"角色。 |
| **Pipeline + Snapshot** | FrontEnd 五阶段流水线 + 不可变 `LayerSnapshot`(详见 §4.2)。 |
| **Strategy(策略)** | `RefreshRateSelector` 的 `Policy` + 各类 `Vote` 评分(详见 §4.3);`CompositionEngine/planner` 的 `Flattener` 选择是否合并/缓存层组。 |
| **State Machine** | `VsyncSchedule` 的 hw vsync 启用/挂起/禁用三态;`DisplayModeController` 的模式切换状态(申请 → 协商 → 应用)。 |
| **Bridge / Adapter** | `Layer` 是兼容期"门面",内部把"旧客户端语义"桥接到新的 `LayerFE` + `RequestedLayerState` 数据;`AidlComposerHal` / `HidlComposerHal` 适配两代 HAL。 |
| **Producer-Consumer 无锁队列** | `LocklessQueue.h`、`TransactionRingBuffer.h`、`Tracing/LocklessStack.h`:把跨线程发布事件转成无锁 SPSC/MPSC。 |
| **Command(命令)** | `QueuedTransactionState` 把"对若干 layer 的若干操作"封装为可缓存、可过滤、可延迟执行的对象,等待 `applyToken` 与 buffer ready。 |

### 2.3 顶层模块清单与职责

下表是 A16 `services/surfaceflinger/` 下所有子目录与重要顶层文件的职责一览。

| 顶层 / 子目录 | 文件数(粗略) | 职责一句话总结 |
| --- | --- | --- |
| **`SurfaceFlinger.{cpp,h}`** + `main_surfaceflinger.cpp` | 12526 行(.cpp+.h)| 主类,Binder 服务端 + ICompositor 实现 + 多回调聚合 |
| **`FrontEnd/`** | 21 | 事务到层快照的五阶段流水线(`TransactionHandler` / `LayerLifecycleManager` / `LayerHierarchy` / `LayerSnapshotBuilder` / `LayerSnapshot` / 缓存层级 `Caching/`) |
| **`Scheduler/`** | 39 | VSYNC 调度、刷新率选择、帧时间线、消息队列、事件线程(`Scheduler` / `VsyncSchedule` / `RefreshRateSelector` / `EventThread` / `FrameTimeline` / `LayerHistory` / `VsyncModulator` / `SmallAreaDetection*`) |
| **`CompositionEngine/`** | (含 `include` / `src` / `mock`) | 设备无关的合成引擎,`Output` / `OutputLayer` / `RenderSurface` / `DisplayColorProfile` / `planner/` |
| **`DisplayHardware/`** | 14 + `VirtualDisplay/` | HWC 抽象与 AIDL/HIDL 双实现,`FramebufferSurface`,虚拟显示线程 |
| **`Display/`** | 10 | 物理/虚拟显示的快照、`DisplayModeController`(模式切换)、`DisplayIdentification`(EDID) |
| **`Tracing/`** | 13 | Perfetto 数据源(`LayerDataSource` / `TransactionDataSource`)、本地环形缓冲、proto 解析 |
| **`PowerAdvisor/`** | 10 | ADPF Performance Hint Session,`SessionManager` / `SessionLayerMap` / `Workload` |
| **`TimeStats/`** | 4 | 帧时序聚合上报(atom proto) |
| **`Jank/`** | 2 | `JankTracker` 卡顿归因 |
| **`FrameTracer/`** | 3 | Perfetto trace 包(帧粒度 stage) |
| **`Effects/`** | 2 | `Daltonizer`(色盲滤镜矩阵) |
| **`Utils/`** | 4 | `Dumper` / `FenceUtils` / `OnceFuture` / `OverlayUtils` |
| **`common/`** | 3 | 跨模块小工具 |
| **`layerproto/`** | 1 | 层 proto schema |
| **`sysprop/`** | 2 | 由 `*.sysprop` 文件生成的 boot-time 属性读取器 |
| **顶层独立文件** | | `DisplayDevice` / `Layer` / `LayerFE` / `LayerVector` / `Client` / `ClientCache` / `TransactionCallbackInvoker` / `WindowInfosListenerInvoker` / `RegionSamplingThread` / `RefreshRateOverlay` / `HdrSdrRatioOverlay` / `FpsReporter` / `HdrLayerInfoReporter` / `TunnelModeEnabledReporter` / `ActivePictureTracker` / `BackgroundExecutor` / `FrameTracker` |
| **构建/测试** | | `Android.bp`、`fuzzer/`、`tests/`(39 文件) |

### 2.4 模块间的依赖关系

观察 §2.2 总图,核心依赖可总结为四条线:

1. **入站**:客户端经 Binder → `SurfaceFlinger` 主类。`SurfaceFlinger` 既是协议实现,也是各子系统的"中央枢纽"。
2. **控制流(反向)**:`HWC HAL → SurfaceFlinger(ComposerCallback) → Scheduler/VsyncSchedule → SurfaceFlinger(ICompositor) → FrontEnd → CompositionEngine`。SF 既"收"vsync,又被 Scheduler"驱动"——这种反向是关键。
3. **合成流**:`FrontEnd.LayerSnapshot → CompositionEngine.Output/OutputLayer → HWComposer → HAL`。仅在 device 合成失败时,`Output` 调 `RenderEngine` 走 client 合成回退。
4. **旁路**:`Tracing` / `TimeStats` / `JankTracker` / `FrameTracer` / `PowerAdvisor` 订阅 SF 的关键事件,只读消费,不在合成关键路径上加锁。

---

## 三、整体类图与主时序图

### 3.1 顶层类图

![SurfaceFlinger 顶层类图](sf-assets/04-sf-class.svg)

上图把 SF 主类与它实现的所有接口、聚合的所有子系统画在一起。要点:

- `SurfaceFlinger` 同时实现 7 个接口:`BnSurfaceComposer`、`PriorityDumper`、`IBinder::DeathRecipient`、`HWC2::ComposerCallback`、`scheduler::ICompositor`、`scheduler::ISchedulerCallback`、`compositionengine::ICEPowerCallback`。这是"中央调度者"模式:所有外部回调线索都汇聚到这一个对象上,实现的方法分散在 `SurfaceFlinger.cpp` 的不同 section。
- `surfaceflinger::Factory` 接口仅由 `SurfaceFlingerDefaultFactory` 实现(测试可注入 mock 工厂),它生成所有重型组件。
- `DisplayDevice` 是"显示"的 SF 侧门面,内部聚合一个 `compositionengine::Display`(后者是 `Output` 的具体子类)——这条聚合链把"我是哪块显示"与"我如何被合成"分开。
- `Layer` 是兼容 wrapper(保留 A13 公共 API 形态),实质数据已下放到 `FrontEnd::RequestedLayerState` + `LayerFE` + `LayerSnapshot`,`LayerFE` 才是 `CompositionEngine` 直接读取的"层前端"。

### 3.2 启动时序

`main_surfaceflinger.cpp` 的关键步骤:

```
init.rc 拉起 surfaceflinger 进程
   │
   ▼
main()  ── signal(SIGPIPE, SIG_IGN)
   ├─► hardware::configureRpcThreadpool(1, false)           # HIDL 兼容
   ├─► startGraphicsAllocatorService()                       # 条件 passthrough Gralloc
   ├─► ProcessState::self()->setThreadPoolMaxThreadCount(4)  # Binder 线程池 = 4
   ├─► SurfaceFlinger::setSchedAttr(true, __func__)          # uclamp.min
   │
   ├─► flinger = new SurfaceFlinger(factory)                  # 工厂创建
   │      └── 内部建 mScheduler / mCompositionEngine 等成员的占位
   │
   ├─► flinger->init()                                        # 真正完成初始化
   │      ├─ 读 sysprop / FlagManager
   │      ├─ mScheduler 建 VsyncSchedule/RefreshRateSelector 占位
   │      ├─ mCompositionEngine = factory.createCompositionEngine()
   │      ├─ mHwComposer (经工厂) → setCallback(this) (SF 同时是 ComposerCallback)
   │      ├─ 拉起 RenderEngine、Daltonizer、TimeStats、FrameTracer
   │      └─ initializeDisplays() — 注册当前已连接的物理显示
   │
   ├─► defaultServiceManager()->addService("SurfaceFlinger", flinger)  # 注册到 SM
   ├─► startDisplayService()                                  # 旧 IDisplayService(可选)
   ├─► flinger->run()                                         # 进入主消息循环
   │      └─ 主线程开始消费 MessageQueue,被 vsync 触发 invalidate
   │
   └─► 之后:
       HWC.onHotplug(primary) → SF.onComposerHalHotplug → setup primary display
       SF.bootFinished() 被 system_server 通过 Binder 触发
```

### 3.3 一帧的主时序

![SurfaceFlinger 一帧的关键时序](sf-assets/03-sf-frame-seq.svg)

时序图分四个阶段:

1. **入站事务**:App 经 `setTransactionState()` 把 `TransactionState` 投递给 SF,`TransactionHandler.queueTransaction()` 收下,`scheduleCommit()` 触发下一次 vsync 唤醒。
2. **VSYNC 抵达**:HWC `onVsync` 回到 SF → 转给 `Scheduler` → `VsyncSchedule` 拟合时间戳 → `VSyncDispatch` 在合适时刻把 `MessageQueue.invalidate` 调度到 SF 主线程。
3. **ICompositor 流水线**:`configure() → commit() → composite() → sample()`,这是真正"做事"的四步。
4. **回调/旁路**:`TransactionCallbackInvoker` 把"onCommitted / onPresented"等回调发回 App,`WindowInfosListenerInvoker` 同步窗口信息给输入子系统。

> ICompositor 接口(`Scheduler/include/scheduler/interface/ICompositor.h`)
>
> ```cpp
> struct ICompositor {
>     virtual void configure() = 0;
>     virtual bool commit(PhysicalDisplayId pacesetterId, const scheduler::FrameTargets&) = 0;
>     virtual CompositeResultsPerDisplay composite(PhysicalDisplayId pacesetterId,
>                                                  const scheduler::FrameTargeters&) = 0;
>     virtual void sendNotifyExpectedPresentHint(PhysicalDisplayId) = 0;
>     virtual void sample() = 0;
> };
> ```

`commit()` 返回 `bool` 表示"是否有 dirty / 需要 composite";若无,SF 跳过 `composite()` 直接退出,节省 GPU/HWC 调用。

---
## 四、各子模块详细分析

> 每个子模块按"职责定位 → 类图(若有独立 SVG)→ 关键数据结构 → 流程 → 实现细节 → 关键代码"展开。

### 4.1 SurfaceFlinger 主类

**文件**:`SurfaceFlinger.{cpp,h}`(`.cpp` 10637 行,`.h` 1889 行)
**职责**:Binder 服务端(`BnSurfaceComposer`)+ 全部回调实现 + 全部子系统聚合者 + ICompositor 的"被驱动"主线程。

#### 4.1.1 类图

参见前面 §3.1 的 [顶层类图](sf-assets/04-sf-class.svg)。这里再强调三个聚合关系:

- `mScheduler : std::shared_ptr<scheduler::Scheduler>`——SF 持有 Scheduler,但 Scheduler 又通过 `ICompositor` 反向调 SF,形成"中央枢纽"。
- `mCompositionEngine : std::unique_ptr<compositionengine::CompositionEngine>`——SF 拥有合成引擎。
- `mDisplayModeController`、`mLayerLifecycleManager`、`mLayerHierarchyBuilder`、`mLayerSnapshotBuilder`、`mTransactionHandler`——FrontEnd 的几个组件直接以成员形式持有,因为它们是轻量的"状态机"对象。

#### 4.1.2 关键数据结构(节选成员)

| 成员 | 作用 |
| --- | --- |
| `mStateLock`(`Mutex`) | 保护"显示集合 / 全局 SF 状态"的大锁。许多读取处用 `FTL_FAKE_GUARD(mStateLock, ...)` 标注。 |
| `mPhysicalDisplays`、`mDisplays` | 物理与逻辑显示集合 |
| `mDisplayModeController` | 处理模式切换的状态机(详见 §4.6) |
| `mTransactionHandler` | FrontEnd 第一阶段对象(详见 §4.2) |
| `mLayerLifecycleManager` | FrontEnd 第二阶段对象 |
| `mLayerHierarchyBuilder` | FrontEnd 第三阶段对象 |
| `mLayerSnapshotBuilder` | FrontEnd 第四阶段对象 |
| `mDrawingState`、`mCurrentState` | "当前生效"与"待生效"的全局状态(色彩矩阵等);事务先入 `mCurrentState`,commit 时 swap |
| `mVisibleRegionsDirty`、`mGeometryDirty` | 触发本帧"重算可见区域/几何"的脏标记 |
| `mPowerAdvisor`、`mTimeStats`、`mFrameTracer`、`mFrameTimeline` | 旁路模块 |
| `mBufferIdsToUncache` | 本帧需要从 HWC buffer cache 中淘汰的 buffer id |
| `mForceColorMode`、`mDisplayColorSetting` | 强制色彩模式与全局色彩设置 |
| `mFactory : surfaceflinger::Factory&` | 注入工厂引用,用于创建所有重型组件 |

#### 4.1.3 流程:`ICompositor` 四步

**configure()**(`SurfaceFlinger.cpp:2568`):应用所有"待生效的显示模式变更",仅在 `mDisplayModeController.isModeSetPending()` 时才真正下发到 HWC。

**commit()**(`SurfaceFlinger.cpp:2845`)——返回 `bool` 表示"有 dirty 否",流程:

1. 取 pacesetter 显示的 `FrameTarget`,记录 `vsyncId`,若 `didMissFrame()` 增加 missed counter。
2. 校验 pacesetter 是否还存在(显示热插拔后)。
3. 检查"有 mode-set 待生效且 fence 未 signal" → 返回 `false`,等下一帧再试。
4. 在 `mStateLock` 下:`finalizeDisplayModeChange()` 完成模式切换。
5. 推进 FrontEnd 流水线(参见 §4.2),把就绪事务合并到层状态,产出本帧 `LayerSnapshot` 列表。
6. 如果没有任何 dirty(无新 buffer、无几何变化、无颜色变化)→ `return false`,跳过 composite。

**composite()**(`SurfaceFlinger.cpp:3118`)——返回 `CompositeResultsPerDisplay`,流程:

1. 构造 `compositionengine::CompositionRefreshArgs`(把本帧的 outputs、color setting、几何脏标志、`bufferIdsToUncache`、`frameInterval`、`powerCallback = this` 等填进去)。
2. `setForcedClientCompositionLayerStacks(refreshArgs)`——某些 layerStack(如截屏目标)被强制走 client 合成。
3. 调用 `mCompositionEngine->present(refreshArgs)` —— 这里把控制权交给 §4.4 详述的合成流水线。
4. 收集每个 Output 的 `CompositeResult`(fence、stats),回填到 `frameTargeters`。

**sample()**(`SurfaceFlinger.cpp:8936`)——给 `RegionSamplingThread`、`FrameTracer`、`TimeStats` 等旁路模块一个"本帧已完成"的钩子。

#### 4.1.4 实现细节

1. **线程上下文断言**:大量方法用 `REQUIRES(kMainThreadContext)` / `EXCLUDES(mStateLock)` 标注 Clang Thread Safety Analysis 标签,确保仅在主线程上执行。`ThreadContext.h` 定义了 `kMainThreadContext` 这个虚拟锁。
2. **FTL_FAKE_GUARD**:在确知"逻辑上持锁"但编译器无法推断的位置,用 `FTL_FAKE_GUARD(lock, expr)` 来"骗"过 TSA。
3. **mDrawingState / mCurrentState 双状态**:命令式事务进入 `mCurrentState`,commit 内做 swap → `mDrawingState`,这样 composite 阶段读取的是"凝固的"状态。
4. **多次 `mScheduler->scheduleFrame()`**:当本帧因为 fence 未就绪而"放弃"时,显式排队下一帧,避免漏帧。
5. **`SFTRACE_NAME / SFTRACE_ASYNC_FOR_TRACK_BEGIN`**:贯穿合成关键路径的 trace 桩,数据流到 Perfetto 的 `gfx` track 与 `WorkloadTracer`。

#### 4.1.5 关键代码

`commit()` 入口:

```cpp
bool SurfaceFlinger::commit(PhysicalDisplayId pacesetterId,
                            const scheduler::FrameTargets& frameTargets) EXCLUDES(mStateLock) {
    const auto& pacesetterFrameTarget = *frameTargets.get(pacesetterId)->get();
    const VsyncId vsyncId = pacesetterFrameTarget.vsyncId();
    SFTRACE_NAME(ftl::Concat(__func__, ' ', ftl::to_underlying(vsyncId)).c_str());

    if (pacesetterFrameTarget.didMissFrame()) {
        mTimeStats->incrementMissedFrames();
    }
    // ... 校验 pacesetter / 模式切换 fence ...
    // ... 推进 FrontEnd 流水线 ...
    return /* anyDirty */;
}
```

`composite()` 装配 refreshArgs:

```cpp
CompositeResultsPerDisplay SurfaceFlinger::composite(
        PhysicalDisplayId pacesetterId, const scheduler::FrameTargeters& frameTargeters) {
    compositionengine::CompositionRefreshArgs refreshArgs;
    refreshArgs.powerCallback = this;
    refreshArgs.outputs.reserve(mDisplays.size());
    // ...
    refreshArgs.bufferIdsToUncache = std::move(mBufferIdsToUncache);
    refreshArgs.outputColorSetting = mDisplayColorSetting;
    refreshArgs.forceOutputColorMode = mForceColorMode;
    refreshArgs.updatingGeometryThisFrame = mGeometryDirty.exchange(false) || /*...*/;
    refreshArgs.frameInterval = mScheduler->getNextFrameInterval(
            pacesetterId, pacesetterTarget.expectedPresentTime());
    // ...
    return mCompositionEngine->present(refreshArgs);  // 走 §4.4
}
```

---

### 4.2 FrontEnd

**目录**:`FrontEnd/`(21 个文件 + `Caching/` 子目录,含自带的 `readme.md`)
**职责**:把"客户端事务"高效、可并行地转化为"每帧的不可变 `LayerSnapshot` 列表",供 `CompositionEngine` 和 WindowInfos 监听者消费。

#### 4.2.1 类图与时序

![FrontEnd 五阶段流水线](sf-assets/05-frontend.svg)

主要类:

| 类 | 文件 | 角色 |
| --- | --- | --- |
| `TransactionHandler` | `TransactionHandler.{cpp,h}` | 排队、过滤、释放事务;支持 barrier / stalled |
| `QueuedTransactionState` | `QueuedTransactionState.h` | 包装 `TransactionState` + applyToken + 元信息 |
| `LayerLifecycleManager` | `LayerLifecycleManager.{cpp,h}` | 用事务合并到 `RequestedLayerState`,生成 change flags |
| `RequestedLayerState` | `RequestedLayerState.{cpp,h}` | 服务端"客户端请求状态"(可由初始 `LayerCreationArgs` + 事务序列还原) |
| `LayerHierarchy` / `LayerHierarchyBuilder` | `LayerHierarchy.{cpp,h}` | z 序/相对父/镜像层级图 |
| `LayerSnapshot` | `LayerSnapshot.{cpp,h}` | 不可变快照,继承 `compositionengine::LayerFECompositionState` |
| `LayerSnapshotBuilder` | `LayerSnapshotBuilder.{cpp,h}` | 把 `LayerHierarchy` + `lifecycleManager` 扁平化为 z 序快照列表 |
| `LayerCreationArgs` / `LayerHandle` | `LayerCreationArgs.{cpp,h}` / `LayerHandle.{cpp,h}` | 创建参数与生命周期句柄 |
| `LayerLog`、`SwapErase`、`Update`、`DisplayInfo` | 辅助 | 日志、容器、显示视图、Update 命令包 |
| `Caching/MergeableHierarchy*` | `Caching/` | 子层级缓存(用于减少快照重建工作) |

#### 4.2.2 关键数据结构

**`RequestedLayerState`**(摘录):

```cpp
class RequestedLayerState : public layer_state_t {
public:
    RequestedLayerState(const LayerCreationArgs&);
    void merge(const ResolvedComposerState&);           // 合并一次事务对该层的更新
    bool isSimpleBufferUpdate(const layer_state_t&) const;
    // 数据成员:继承自 layer_state_t 的全部"客户端请求状态"
    // + name / id / parentId / mirrorOf / pendingBuffers / ...
};
```

**`LayerSnapshot`**(关键点):

```cpp
struct LayerSnapshot : public compositionengine::LayerFECompositionState {
    bool invalidTransform = false;
    bool isHiddenByPolicyFromParent = false;
    bool isHiddenByPolicyFromRelativeParent = false;
    ftl::Flags<RequestedLayerState::Changes> changes;   // 本帧相对上帧的变更
    uint64_t clientChanges = 0;
    uint32_t uniqueSequence;                            // 唯一序号(便于 diffing/缓存)
    int32_t sequence;
    // 继承自 LayerFECompositionState 的几何/颜色/buffer/HDR 元数据
};
```

继承 `LayerFECompositionState` 这一点非常关键:**Snapshot 直接是合成状态**,`CompositionEngine` 不需要任何转换。

**`LayerHierarchy`**:用图(graph)表示;镜像层(`mirrorOf`)用同一个节点的多父表示——避免克隆。

#### 4.2.3 流程

按 `FrontEnd/readme.md` 的描述(SF 内置文档),分五阶段:

1. **TransactionHandler:排队与过滤** — `queueTransaction()` 把事务塞进 per-applyToken 队列,主线程在 `commit()` 起点遍历 `mTransactionFilters` 找"已就绪"事务。就绪条件:buffer fence signal、barrier 满足、applyToken 之前没有未应用的事务。
2. **LayerLifecycleManager:层生命周期 & 状态合并** — 把就绪事务 `applyTransactions()` 到对应 `RequestedLayerState`,生成 `Changes` 标志(`GEOMETRY / BUFFER / VISIBILITY / METADATA / Z / ...`)。`onHandlesDestroyed()` 处理 layer handle 释放(`Layer` 的强引用消亡后异步通知)。
3. **LayerHierarchyBuilder:层级树更新** — 由 `RequestedLayerState[]` 构造或更新 `LayerHierarchy`;解析 parent / relative;`fixRelativeZLoop()` 处理 relativeZ 形成环的退化情况;镜像层用"同节点多父"表示。
4. **LayerSnapshotBuilder:快照构建** — 依据 `Args{ hierarchy, lifecycle, displays, globalShadowSettings, ... }` 与 `Changes`,要么走"快路径"(仅更新 buffer 字段),要么走"全量"(几何/相对父变了)。结果是按 z 序展开的 `vector<LayerSnapshot>`。
5. **回调与消费** — 快照通过 `std::move` 给到 `CompositionEngine`(避免拷贝);WindowInfos / 输入 / 可访问性监听者克隆消费;`ILifecycleListener` 接收"层被创建/销毁"通知。

#### 4.2.4 实现细节

1. **不可变性 + change flags**:`LayerSnapshot` 一旦构建即不再 mutate,变更通过"下一帧再生一份新 snapshot"表达。`changes` flags 让消费者(尤其是缓存 planner)能够短路:几何不变只更新 buffer。
2. **路径化索引(`PathToSnapshot`)**:用层在层级中的"路径"作为索引,跨帧稳定,即便层在 vector 中位置变化也能找到上一帧对应快照。
3. **镜像层零拷贝**:层级图同一节点可被多个父引用——Snapshot 也只会出现一份,不再像 A13 那样克隆出 `MirrorLayer`。
4. **barrier 与 stalled transactions**:当事务带 WAIT token 等待时,会在 `mStalledTransactions` 中"挂起",并设置 TTL(默认从 `mTransactionBarrierTtl`)防止永久挂死。
5. **批量回调**:`LifecycleListener` 接口允许多个旁路监听者(输入、tracing、metric)在每帧成批接收变更。

#### 4.2.5 关键代码

**`TransactionHandler.queueTransaction()` 提交事务到队列**:

```cpp
void TransactionHandler::queueTransaction(QueuedTransactionState&& state) {
    mLocklessTransactionQueue.push(std::move(state));
    mPendingTransactionCount.fetch_add(1);
}
```

**`LayerLifecycleManager.applyTransactions()` 推进层状态**:

```cpp
void LayerLifecycleManager::applyTransactions(
        const std::vector<QueuedTransactionState>& transactions,
        bool ignoreUnreachableLayers) {
    for (auto& t : transactions) {
        for (auto& resolved : t.states) {
            RequestedLayerState* layer = getLayerFromId(resolved.layerId);
            if (!layer) continue;
            layer->merge(resolved);                  // 合并字段 + 累积 Changes
            mGlobalChanges |= layer->changes;
        }
        // 处理 transactionListeners、bufferData 等
    }
}
```

**`LayerSnapshotBuilder.update()` 入口**(摘要):

```cpp
struct LayerSnapshotBuilder::Args {
    const LayerHierarchy& root;
    const LayerLifecycleManager& layerLifecycleManager;
    bool forceUpdate = false;
    // displays, globalShadowSettings, supportedLayerGenericMetadata, ...
};

void LayerSnapshotBuilder::update(const Args& args) {
    if (!args.forceUpdate && args.layerLifecycleManager.getGlobalChanges().get() == 0) return;
    // 走快/慢路径
    updateSnapshots(args);
}
```

`LayerSnapshot` 继承 `LayerFECompositionState`,流入合成引擎时即"现成的合成状态",不再需要中间转换。

---

### 4.3 Scheduler

**目录**:`Scheduler/`(39 个文件)
**职责**:管理 vsync(硬件 vsync 拟合、回调派发)、刷新率选择(RefreshRateSelector)、帧时间线(FrameTimeline)、把控制权反向交回 SF(ICompositor)。

#### 4.3.1 类图与时序

![Scheduler 类图与 VSYNC 时序](sf-assets/06-scheduler.svg)

主要类:

| 类 | 文件 | 角色 |
| --- | --- | --- |
| `Scheduler` | `Scheduler.{cpp,h}` | 顶层协调,管理多显示、policy、EventThread |
| `VsyncSchedule` | `VsyncSchedule.{cpp,h}` | 单显示的 vsync 节奏对象 |
| `VSyncTracker` / `VSyncPredictor` | `VSyncTracker.h` / `VSyncPredictor.{cpp,h}` | 拟合 vsync 周期与下一帧时间 |
| `VSyncDispatch` / `VSyncDispatchTimerQueue` | `VSyncDispatch.h` / `VSyncDispatchTimerQueue.{cpp,h}` | 定时器队列,在 deadline 前唤醒注册回调 |
| `VsyncController` / `VSyncReactor` | `VsyncController.h` / `VSyncReactor.{cpp,h}` | 把"present fence 时间"灌进 tracker,启用/禁用 hw vsync |
| `VsyncModulator` | `VsyncModulator.{cpp,h}` | 动画/触摸时段对相位/duration 动态调制 |
| `VsyncConfiguration` | `VsyncConfiguration.{cpp,h}` | 从设备配置加载默认 phase offsets / work duration |
| `RefreshRateSelector` | `RefreshRateSelector.{cpp,h}` | 刷新率决策(详见 4.3.4) |
| `RefreshRateStats` | `RefreshRateStats.h` | 各刷新率档位的累计时长统计 |
| `EventThread` | `EventThread.{cpp,h}` | 把 vsync 派发给 App 与 SF 主线程的工作线程 |
| `MessageQueue` | `MessageQueue.{cpp,h}` | SF 主线程的消息队列(`invalidate`) |
| `LayerHistory` / `LayerInfo` | `LayerHistory.{cpp,h}` / `LayerInfo.{cpp,h}` | 记录每层最近帧率作为投票输入 |
| `FrameTimeline` | `FrameTimeline.{cpp,h}` | 帧 token / 实际 vs 预测帧时间,供 ATrace、TimeStats |
| `FrameRateOverrideMappings` | `FrameRateOverrideMappings.{cpp,h}` | App 级"应该跑多少 fps"的覆盖映射 |
| `SmallAreaDetectionAllowMappings` | `SmallAreaDetection*.{cpp,h}` | 局部小面积更新的 VRR 白名单 |
| `OneShotTimer` | `OneShotTimer.{cpp,h}` | 单次定时器(kernel idle timer) |
| `ISchedulerCallback` | `ISchedulerCallback.h` | Scheduler → SF 的回调接口 |
| `ICompositor` | `include/scheduler/interface/ICompositor.h` | SF → 被 Scheduler 反向驱动的接口 |

#### 4.3.2 关键数据结构

**`Scheduler::Display`**(per-physical-display):

```cpp
struct Display {
    PhysicalDisplayId id;
    RefreshRateSelectorPtr selectorPtr;
    VsyncSchedulePtr vsyncSchedulePtr;
    // pacesetter 与 displays 集合在 Scheduler 中维护
};
```

`mDisplays` 是 `Scheduler` 的核心集合;`mPacesetterDisplayId` 决定"哪一块显示是节拍源"(通常是主屏)。

**`RefreshRateSelector::Policy`**:

```cpp
struct Policy {
    DisplayModeId defaultMode;
    FpsRange primaryRanges;     // 例如 [60, 120] —— 用户/系统期望
    FpsRange appRequestRanges;  // App 可达到的范围
    bool idleScreenConfigValid;
    // ...
};
```

**`LayerRequirement`**:

```cpp
struct LayerRequirement {
    std::string name;
    Fps desiredRefreshRate;
    LayerVoteType vote;   // Default/Min/Max/Heuristic/ExplicitDefault/ExplicitExact/...
    float weight;
    Seamlessness seamlessness;
    FrameRateCategory frameRateCategory;
};
```

**`GlobalSignals`**:`touch`、`idle`、`powerOnImminent`、`heuristicIdle` 等运行时上下文。

**`RankedFrameRates`**:`getRankedFrameRates()` 的返回值,内含 `frameRateMode`(最佳)与备选 `ScoredFrameRate[]`、`consideredSignals`、`pacesetterDisplayModeId`。

#### 4.3.3 流程

**VSYNC 输入侧**:

1. HAL → `HWC2::ComposerCallback::onVsync(displayId, timestamp, period)` 在 SF 进程内被调用。
2. SF 在主线程上(via `Scheduler::dispatchHotplug/onVsync`)把 timestamp 灌入对应显示的 `VsyncSchedule.addPresentFence/addVsyncTimestamp`,内部 `VSyncPredictor` 滑动拟合周期与抖动。
3. `VsyncController` 决定是否需要继续保持 hw vsync 开启(若长时间无更新自动关闭)。

**VSYNC 派发侧**:

1. App 端 EventThread 注册回调到 `VSyncDispatch`,描述"期望 present 时间前 N ms 唤醒我";SF 主线程也有一份对应注册(work duration 不同)。
2. `VSyncDispatch` 在 deadline 前唤醒:App → `Choreographer.doFrame()`;SF → `MessageQueue.invalidate()`。
3. SF 主线程被唤醒后,调用 `Scheduler::commit/composite` 即 ICompositor 四步(实际由 `Scheduler` 包裹 `ICompositor` 调用)。

**刷新率决策**:

1. `LayerHistory` 周期性把"每个活跃层"的最近帧率写入 `LayerInfo`。
2. Scheduler 定时(或显示模式变更触发)调 `RefreshRateSelector.getRankedFrameRates(LayerRequirements[], GlobalSignals, ...)`。
3. 选出 `bestMode` 后:若与当前不同 → 经 `requestDisplayModes()` 上抛给 SF → `DisplayModeController` 协调 HWC 切换。
4. 若仅渲染率变了(同物理模式不同 render rate / ARR)→ `setRenderRate()` 直接生效。

#### 4.3.4 实现细节

1. **Pacesetter**:多显示并存时,SF 的"节拍源"是 `pacesetterDisplayId`。其他显示有自己的 vsync,但只有 pacesetter 的 vsync 触发 `commit/composite`,其余显示在同一帧里以"该显示的 expected present time"被各自合成。
2. **App vs SF 两套 phase offset**:App 早唤醒,准备绘制;SF 略晚唤醒,等 App buffer 到达后开始合成。`VsyncConfiguration` 从 sysprop 加载这套时序。
3. **VsyncModulator**:动画/输入开始/结束的瞬间,用预设档位(`Default / Early / EarlyApp / EarlyGl`)动态调整,缩短端到端延迟。
4. **ARR(自适应刷新率)与合成模式**:`RefreshRateSelector` 与 `SyntheticModeManager`(在 `services/display` 端)的合成模式配合,SF 内 `FrameRateMode` 区分"物理模式"与"渲染率"。
5. **小面积 VRR**:`SmallAreaDetectionAllowMappings` 允许指定 App+region 走低帧率(秒针/闪烁光标),减少功耗。

#### 4.3.5 关键代码

`ICompositor` 接口(`Scheduler/include/scheduler/interface/ICompositor.h`):

```cpp
struct ICompositor {
    virtual void configure() = 0;
    virtual bool commit(PhysicalDisplayId pacesetterId,
                        const scheduler::FrameTargets&) = 0;
    virtual CompositeResultsPerDisplay composite(PhysicalDisplayId pacesetterId,
                                                 const scheduler::FrameTargeters&) = 0;
    virtual void sendNotifyExpectedPresentHint(PhysicalDisplayId) = 0;
    virtual void sample() = 0;
};
```

`RefreshRateSelector::getRankedFrameRates()` 签名:

```cpp
RankedFrameRates getRankedFrameRates(const std::vector<LayerRequirement>& layers,
                                     GlobalSignals signals,
                                     Fps pacesetterFps = Fps()) const EXCLUDES(mLock);
```

`Scheduler` 的"动态相位调制"模板:

```cpp
template <typename Handler, typename... Args>
void modulateVsync(std::optional<PhysicalDisplayId> id, Handler handler, Args&&... args) {
    std::scoped_lock lock(mDisplayLock);
    // 调 mVsyncModulator->handler(args...) 拿到新 VsyncConfig
    // 然后 setDuration(...) 应用
}
```

`Scheduler::onDisplayModeChanged()` 把"模式已经在 HWC 切完"事件回灌到 `VsyncSchedule`,通知它"vsync 周期变了,clear 预测器":

```cpp
bool Scheduler::onDisplayModeChanged(PhysicalDisplayId id, const FrameRateMode& mode,
                                     bool clearContentRequirements) EXCLUDES(mPolicyLock);
```

---

### 4.4 CompositionEngine

**目录**:`CompositionEngine/include/compositionengine/` + `CompositionEngine/src/`
**职责**:与"具体输出设备无关"的合成引擎。把 FrontEnd 喂来的 `LayerSnapshot[]` 经过 prepare/validate/finishFrame 三段流水线落到 HWC + RenderEngine。

#### 4.4.1 类图与时序

![CompositionEngine 类图与时序](sf-assets/07-composition-engine.svg)

主要类:

| 类 / 接口 | 文件 | 角色 |
| --- | --- | --- |
| `CompositionEngine`(接口/impl) | `include/.../CompositionEngine.h` / `src/CompositionEngine.cpp` | 顶层引擎,持有 outputs 与 `HwcAsyncWorker` |
| `Output`(接口/impl) | `include/.../Output.h` / `impl/Output.h` / `src/Output.cpp` | 一个合成目标(物理/虚拟/截屏);每个 Output 自带状态机 |
| `Display`(继承 `Output`) | `include/.../Display.h` / `impl/Display.h` / `src/Display.cpp` | "面向显示"的 Output,持有 `DisplayColorProfile` 与 HWC 关联 |
| `OutputLayer` | `include/.../OutputLayer.h` / `impl/OutputLayer.h` / `src/OutputLayer.cpp` | 一个 layer 在某个 Output 上的"合成项",含 `mHwcLayerId` |
| `OutputCompositionState` / `OutputLayerCompositionState` | impl 头 | Output / OutputLayer 的合成期状态 |
| `LayerFE`(接口) | `include/.../LayerFE.h` | Output 看到的"层前端",由 SF 端 `surfaceflinger::LayerFE` 实现 |
| `LayerFECompositionState` | `include/.../LayerFECompositionState.h` | 层前端提供给 Output 的合成状态;`LayerSnapshot` 直接继承它 |
| `RenderSurface` | `include/.../RenderSurface.h` / `impl/RenderSurface.h` | Native Window 抽象;client 合成用它换 framebuffer |
| `DisplayColorProfile` | `include/.../DisplayColorProfile.h` / `impl/.../*` | 该显示支持哪些 ColorMode / RenderIntent / HDR |
| `HwcAsyncWorker` | `impl/HwcAsyncWorker.h` | 把 validate/present 推到工作线程 |
| `HwcBufferCache` | `impl/HwcBufferCache.h` | 每 HWC 层一份的 buffer slot 缓存,setBuffer 时只传变化的 slot |
| `ClientCompositionRequestCache` | `impl/ClientCompositionRequestCache.h` | 缓存 client 合成请求,跨帧避免重算 |
| `GpuCompositionResult` | `impl/GpuCompositionResult.h` | GPU 合成结果(buffer + acquireFence) |
| `planner/` 子目录 | `impl/planner/{Flattener,Predictor,CachedSet,LayerState,TexturePool}.h` | GPU 合成缓存(把若干层"压平"成可复用纹理) |
| `CompositionRefreshArgs` | `include/.../CompositionRefreshArgs.h` | 一次 present 的输入参数包 |

#### 4.4.2 关键数据结构

**`CompositionRefreshArgs`** —— `composite()` 阶段传入的所有信息:

```cpp
struct CompositionRefreshArgs {
    Outputs outputs;
    Layers layers;                    // 全部 LayerFE
    Layers layersWithQueuedFrames;
    OutputColorSetting outputColorSetting;
    ForceOutputColorMode forceOutputColorMode;
    bool updatingOutputGeometryThisFrame;
    bool updatingGeometryThisFrame;
    bool blursAreExpensive;
    ftl::Optional<std::chrono::nanoseconds> frameInterval;
    mat4 colorTransformMatrix;
    Region damageRegion;
    std::vector<uint64_t> bufferIdsToUncache;
    ICEPowerCallback* powerCallback = nullptr;
    // ...
};
```

**`OutputCompositionState`** —— Output 自身合成状态(orientation / projection / display brightness / color profile / dirty region 等)。

**`OutputLayerCompositionState`** —— 每"层在该输出上的合成项"的状态(displayFrame、sourceCrop、bufferSlot、acquireFence、blendMode、hwc layer id 等)。

#### 4.4.3 流程

`CompositionEngine::present(refreshArgs)` 的关键步骤:

1. **preComposition**:`mPowerAdvisor->setExpectedPresentTime(...)`,通知 ADPF;各 Layer 的 `onPreComposition()` 钩子。
2. **rebuildLayerStacks**:把全局 `layers` 按 `layerStack` 分配给各 Output 的 `OutputLayer[]`(物理显示一份、虚拟显示各一份、截屏一份)。
3. **updateColorProfile**:为每个 Output 选 ColorMode/RenderIntent。
4. **updateCompositionState**:每个 Output 调用其每个 `OutputLayer.updateCompositionState(args)`,从 `LayerFE.prepareCompositionState()` 拿来层状态,生成 `OutputLayerCompositionState`。
5. **planner(可选)**:`Flattener.flattenLayers()` 决定是否复用上帧 `CachedSet`;若新构,本帧 client 合成产物会被缓存。
6. **writeCompositionState**:把每层意图通过 `setBuffer / setDataspace / setHdrMetadata / setBlendMode / setDisplayFrame / setSourceCrop` 写到对应 HWC 层。
7. **prepareFrame**:`HWComposer.getDeviceCompositionChanges(displayId, mustValidate, ...)` 触发 HWC `validateDisplay`,得到"哪些层 device 接管 / 哪些层必须 client"的分配,以及 `skipValidate` 提示。
8. **finishFrame**:
   - **client 合成**:把所有标为 client 的层(或 planner cached set)通过 `RenderEngine.drawLayers()` 离屏渲染到该 Output 的 framebuffer,产出 `acquireFence`。
   - **setClientTarget(fbBuffer, acquireFence)**:把 framebuffer 交给 HWC。
   - **presentAndGetReleaseFences()**:让 HAL 真正去呈现,返回 `presentFence` 与 per-layer `releaseFence`。
9. **postFramePerDisplay**:通知 `TimeStats / FrameTracer / FrameTimeline`,触发 fence callbacks。

#### 4.4.4 实现细节

1. **`HwcBufferCache`**:HWC 层每帧有一组"buffer slot",同一个 buffer 第二次出现可只传 slot id 不传文件描述符,显著减少 setBuffer 开销。
2. **`HwcAsyncWorker`**:`validateDisplay / presentDisplay` 可以异步,让 SF 主线程在等 HWC 时去做其他事(例如下一帧 commit 的准备)。
3. **planner.Flattener**:观察"哪些相邻 z 序的层近期不变",把它们离屏渲染成一张 `CachedSet`,后续若不变就用一张大纹理替代多次 client 合成,功耗收益明显。
4. **Output 类型多态**:同一份 `Output` 接口被 `impl::Output`(虚拟显示/截屏)与 `impl::Display`(物理显示)分别实现;`Display` 多了 `setDisplayBrightness / setDisplayColorMode / setActiveModeFromBackend` 等显示特性。
5. **ICEPowerCallback**:`SurfaceFlinger` 同时实现该接口,合成引擎里 `powerCallback->notifyCpuLoadUp()` 可触发 ADPF 加速。
6. **planner 与 Graphite**:A16 RenderEngine 支持 Skia Graphite 后端,planner 的 `CachedSet` 产物对接 Graphite 的纹理共享更友好。

#### 4.4.5 关键代码

`Output` 接口的核心入口(`include/compositionengine/Output.h`):

```cpp
virtual void setProjection(ui::Rotation orientation, const Rect& layerStackSpaceRect,
                           const Rect& orientedDisplaySpaceRect) = 0;
virtual void setLayerFilter(LayerFilter) = 0;
virtual void setColorProfile(const ColorProfile&) = 0;
virtual void setNextBrightness(float brightness) = 0;
virtual void setDisplayBrightness(float sdrWhitePointNits, float displayBrightnessNits) = 0;
virtual void setLayerCachingEnabled(bool) = 0;
virtual OutputCompositionState& editState() = 0;
virtual Region getDirtyRegion() const = 0;
```

`OutputLayer.writeStateToHWC()`(摘要):

```cpp
void writeStateToHWC(bool includeGeometry, bool skipLayer, uint32_t z, bool zIsOverridden) {
    if (includeGeometry) {
        hwcLayer->setDisplayFrame(...); hwcLayer->setSourceCrop(...);
        hwcLayer->setTransform(...);    hwcLayer->setVisibleRegion(...);
        hwcLayer->setBlendMode(...);    hwcLayer->setZOrder(z);
    }
    hwcLayer->setDataspace(...);
    hwcLayer->setColorTransform(...);   hwcLayer->setHdrMetadata(...);
    hwcLayer->setBuffer(slot, buffer, acquireFence);    // 借助 HwcBufferCache
}
```

`HWComposer.getDeviceCompositionChanges()`:

```cpp
virtual status_t getDeviceCompositionChanges(
        HalDisplayId, bool frameUsesClientComposition,
        std::optional<std::chrono::steady_clock::time_point> earliestPresentTime,
        nsecs_t expectedPresentTime, Fps frameInterval,
        DeviceRequestedChanges* outChanges) = 0;
```

---

### 4.5 DisplayHardware(HWComposer)

**目录**:`DisplayHardware/`(14 + `VirtualDisplay/`)
**职责**:对 HWC HAL 的抽象。提供"分配/销毁层、配置层属性、validate、present、电源/亮度/色彩控制"等接口,屏蔽 AIDL/HIDL 差异。

#### 4.5.1 类图

```
┌─────────────────────────────────────────────────────────────────┐
│  HWComposer (interface)                                          │
│   + setCallback(ComposerCallback&)                               │
│   + allocate(Physical|Virtual)Display()                          │
│   + createLayer(displayId) : shared_ptr<HWC2::Layer>             │
│   + getDeviceCompositionChanges(...)                             │
│   + setClientTarget() / presentAndGetReleaseFences()             │
│   + setPowerMode / setColorTransform / setDisplayBrightness      │
│   + getPresentFence / getLayerReleaseFence / getHdrCapabilities  │
└─────────────────────────────────────────────────────────────────┘
        ▲                                                ▲
        │ implements                                     │ implements
┌─────────────────────────┐                ┌────────────────────────────┐
│  HWComposer (impl)      │                │  (tests::MockHWComposer)   │
│   uses Composer (Hal*)  │                └────────────────────────────┘
└──────────┬──────────────┘
           │ Hal Composer interface
   ┌───────┴────────┐                ┌────────────────────────┐
   │ AidlComposerHal │  ◄── 主路径 ──►│ HidlComposerHal        │
   │ composer3 AIDL  │                │ HIDL 2.1–2.4 兼容回退 │
   └─────────────────┘                └────────────────────────┘

HWC2/ 抽象层:
  HWC2::Device  (per-process)      hotplug 注册,管 mDisplays
   ├─ HWC2::Display(per display)   配置/模式列表/属性查询
   │   └─ HWC2::Layer (per layer)  setBuffer/setColor/setComposition/setDataspace/...
   │
   ComposerCallback (= SurfaceFlinger):
   onComposerHalHotplug / onComposerHalVsync / onComposerHalRefresh /
   onComposerHalVsyncPeriodTimingChanged / onRefreshRateChangedDebug / ...

辅助:
  FramebufferSurface         物理显示的 framebuffer Native Window
  DisplayMode (HWC 内部表示) 不同于 ui::DisplayMode
  Hal.h                      hal::* 类型 typedef
  VirtualDisplay/
    VirtualDisplaySurface          虚拟显示对应的 Surface(包了 BufferQueue)
    LegacyVirtualDisplaySurface    旧路径兼容
    VirtualDisplayThread(Manager)  独立线程驱动虚拟显示
    SinkSurfaceHelper              把 Output 接到 sink BufferQueue
    VirtualDisplayBufferSlotTracker
```

#### 4.5.2 关键数据结构

**`HWC2::Layer`**(`DisplayHardware/HWC2.h`,接口与实现):

```cpp
class Layer {
    virtual hal::Error setBuffer(uint32_t slot,
                                 const sp<GraphicBuffer>& buffer,
                                 const sp<Fence>& acquireFence) = 0;
    virtual hal::Error setBufferSlotsToClear(const std::vector<uint32_t>& slotsToClear, ...) = 0;
    virtual hal::Error setColor(hal::Color) = 0;
    virtual hal::Error setCompositionType(hal::Composition) = 0;
    virtual hal::Error setColorTransform(const mat4& matrix) = 0;
    // setDisplayFrame / setSourceCrop / setBlendMode / setHdrMetadata / setLayerLuts ...
};
```

**`HWComposer::DeviceRequestedChanges`**:`getDeviceCompositionChanges()` 的输出。包含每层的 `Composition`(DEVICE/CLIENT/SOLID_COLOR/CURSOR/SIDEBAND/…)、`displayRequests`、`clientTargetProperty`、`skipValidate`。

**`ComposerCallback`** —— SF 实现的回调接口:`onComposerHalHotplug / onComposerHalVsync / onComposerHalVsyncPeriodTimingChanged / onComposerHalRefresh / onRefreshRateChangedDebug / onSeamlessPossible / ...`。

#### 4.5.3 流程

**初始化**:`SurfaceFlinger.init()` → `factory.createHWComposer(serviceName)` →(默认实现选 AIDL composer3,失败回退 HIDL)→ `hwc->setCallback(this)` 将 SF 作为回调接收方。

**显示注册(hotplug)**:HAL `onHotplug(connected)` → `HWC2::Device::onHotplug` 创建 `HWC2::Display` → SF `onComposerHalHotplug` → 在 `mPhysicalDisplays` 注册 → 通过 `Scheduler::registerDisplay()` 让调度器掌握其 vsync 节奏 → `DisplayModeController::registerDisplay()` 接入模式管理。

**一帧 IO**:见 §4.4 Composition 流程。要点:

- `setLayerCompositionType` 在 validateDisplay 前由 SF 写入(基于"是否能被 device 合成"的预判);validate 返回后再按 HAL 的最终分配调整。
- `presentDisplay` 返回 `presentFence`,SF 把它递给 `VsyncSchedule.addPresentFence()` 用于 vsync 预测。
- `releaseFences` 通过 `getLayerReleaseFence(displayId, hwcLayer)` 取回,SF 把它派发给对应的 LayerFE 用于"layer 内 buffer 已被读完"的回调。

#### 4.5.4 实现细节

1. **AIDL composer3 主路径**:A16 仍保留 HIDL 兼容回退,但默认走 `AidlComposerHal`。AIDL `IComposerClient::executeCommands(displayId, payload)` 把一帧的所有"per-layer setter"打包成一个 binary 命令缓冲,**单次跨进程调用**完成,极大降低 IPC 开销。
2. **HWC1 没有 layer 概念**(久远历史),HWC2 中"层"是一等公民;A16 完全基于 HWC2 + composer3。
3. **virtual display**:`VirtualDisplayThreadManager` 把每个虚拟显示放到一个独立线程,因为虚拟显示往往输出到 BufferQueue sink(如录屏、Cast),其消费节奏由 sink 决定,与物理显示解耦。
4. **`skipValidate`**:HAL 可标记"这帧与上帧分配一致,无需 validate",节省 IPC。
5. **per-frame metadata / per-frame metadata blob / HDR**:`setPerFrameMetadata`、`setPerFrameMetadataBlob` 把 HDR 的 SMPTE 2086 / CTA-861.3 / HDR10+ / Dolby Vision 元数据传给 HAL。

#### 4.5.5 关键代码

`HWComposer` 接口节选(`DisplayHardware/HWComposer.h`):

```cpp
virtual bool getDisplayIdentificationData(hal::HWDisplayId, uint8_t* outPort,
                                          DisplayIdentificationData* outData) const = 0;
virtual status_t getDeviceCompositionChanges(
        HalDisplayId, bool frameUsesClientComposition,
        std::optional<std::chrono::steady_clock::time_point> earliestPresentTime,
        nsecs_t expectedPresentTime, Fps frameInterval,
        DeviceRequestedChanges* outChanges) = 0;
virtual status_t setClientTarget(HalDisplayId, uint32_t slot,
                                 const sp<Fence>& acquireFence,
                                 const sp<GraphicBuffer>&, ui::Dataspace) = 0;
virtual status_t presentAndGetReleaseFences(
        HalDisplayId, std::chrono::steady_clock::time_point earliestPresentTime) = 0;
virtual status_t executeCommands(HalDisplayId) = 0;
virtual sp<Fence> getPresentFence(HalDisplayId) const = 0;
virtual ftl::Future<status_t> setDisplayBrightness(/*...*/) = 0;
```

---

### 4.6 Display

**目录**:`Display/`(10 个文件)
**职责**:与"显示"相关的数据与控制——快照(`DisplaySnapshot`)、识别(`DisplayIdentification`)、模式切换状态机(`DisplayModeController`)、物理/虚拟显示视图。

#### 4.6.1 类图

```
┌──────────────────────┐        ┌──────────────────────────────┐
│ DisplaySnapshot      │◄──────│ PhysicalDisplay               │
│  - id / connectionType│       │  - displayId / port           │
│  - modes / supportedHdr      │  - SnapshotRef                │
│  - colorModes / RenderIntents│ │                              │
│  + translateModeId()         │ └──────────────────────────────┘
└──────────────────────┘
           ▲
           │ snapshotRef
┌──────────────────────────────────────────────────────────────┐
│ DisplayModeController                                         │
│  - mComposerPtr : HWComposer*                                 │
│  - mDisplays : map<PhysicalDisplayId, Display>                │
│      Display { snapshotRef, selectorPtr, desiredModeOpt,      │
│                isModeSetPending, isKernelIdleTimerEnabled }   │
│  + registerDisplay / unregisterDisplay                        │
│  + setDesiredMode / clearDesiredMode / isModeSetPending       │
│  + finalizeModeChange(vsyncRate, renderFps)                   │
│  + setActiveMode / updateKernelIdleTimer / startHdcpNegotiation│
└──────────────────────────────────────────────────────────────┘

DisplayIdentification: 解析 EDID -> 推导 stable DisplayId
DisplayModeRequest:    "我希望切到这个 mode" 的请求对象
VirtualDisplaySnapshot:虚拟显示的快照
```

#### 4.6.2 关键数据结构

`DisplaySnapshot` 是只读对象:

```cpp
class DisplaySnapshot {
public:
    PhysicalDisplayId displayId() const;
    ui::DisplayConnectionType connectionType() const;
    DisplayModes modes() const;
    const std::vector<ui::HdrCapabilities>& supportedHdr() const;
    std::optional<DisplayModeId> translateModeId(hal::HWConfigId) const;
    void dump(utils::Dumper&) const;
};
```

`DisplayModeController::Display`(内部 struct,每物理显示一份):

```cpp
struct Display {
    DisplaySnapshotRef snapshotRef;
    scheduler::RefreshRateSelectorPtr selectorPtr;
    std::optional<DisplayModeRequest> desiredModeOpt GUARDED_BY(kMainThreadContext);
    bool isModeSetPending GUARDED_BY(kMainThreadContext) = false;
    bool isKernelIdleTimerEnabled GUARDED_BY(kMainThreadContext) = false;
    void setSecure(bool secure);
};
```

#### 4.6.3 流程:模式切换

```
WMS/DMS/Scheduler 发起 -> SF.setDesiredDisplayModeSpecs() 或 RefreshRateSelector 决策
    │
    ▼
DisplayModeController.setDesiredMode(displayId, DisplayModeRequest)
    │  desiredModeOpt = request
    │  isModeSetPending = true
    │
    ▼ 下一个 SF commit():
SF.configure() -> 检查 isModeSetPending(displayId)
    │
    ▼ 如果允许立刻切:
HWComposer.setActiveConfigWithConstraints() / executeCommands → HAL 接受
    │
    ▼ HAL 完成后 onVsyncPeriodTimingChanged()
SF.onComposerHalVsyncPeriodTimingChanged → DisplayModeController.finalizeModeChange(vsyncRate, renderFps)
    │  把新 mode 写回 snapshotRef / Selector,标记 isModeSetPending = false
    │
    ▼
Scheduler.onDisplayModeChanged() → VsyncSchedule.onDisplayModeChanged(force=true) → clear 预测器
```

中间任何一帧 commit 发现 mode-set 待生效但 fence 未 signal,会 `return false` 等下一帧重试(参见 §4.1.5 commit 代码片段)。

#### 4.6.4 实现细节

1. **`DisplaySnapshotRef`**:`std::shared_ptr<const DisplaySnapshot>`-like 引用,只读 + 引用计数,可在多个线程读取(Scheduler/RefreshRateSelector/Display 都用)。
2. **`DisplayIdentification`**:解析 EDID → SHA1 → 推导稳定的 `PhysicalDisplayId`(64 位),跨重启稳定,用于把 HWC HAL 的 `HWDisplayId` 与平台无关的稳定 ID 关联。
3. **`startHdcpNegotiation()`**:在受保护内容(DRM)需要 HDCP 时主动触发。
4. **`updateKernelIdleTimer`**:与 RefreshRateSelector 的 kernel idle timer controller 协调,决定何时启用"内核空闲计时器"让面板进入低刷新率。

#### 4.6.5 关键代码

`DisplayModeController.setDesiredMode()`(行为摘要):

```cpp
void setDesiredMode(PhysicalDisplayId id, DisplayModeRequest&& request) {
    auto& d = mDisplays.at(id);
    d.desiredModeOpt = std::move(request);
    d.isModeSetPending = true;
    // 触发下一帧 commit 进入 configure 阶段处理
}

void finalizeModeChange(PhysicalDisplayId id, DisplayModeId modeId,
                        Fps vsyncRate, Fps renderFps) {
    auto& d = mDisplays.at(id);
    d.snapshotRef.applyMode(modeId);
    d.selectorPtr->setActiveMode(modeId, renderFps);
    d.isModeSetPending = false;
    mActiveModeListener(id, modeId, vsyncRate, renderFps);
}
```

---

### 4.7 Tracing

**目录**:`Tracing/`(13 个文件,含 `ContributingToPerfetto.md`)
**职责**:把 SF 的事务/层状态/合成事件落到 Perfetto trace,以及本地环形缓冲(用于 `dumpsys SurfaceFlinger --tracing`)。

#### 4.7.1 类图

```
Perfetto 数据源(独立注册到 perfetto producer):
   LayerDataSource         — 每帧序列化 LayerSnapshot 集合
   TransactionDataSource   — 序列化每个进入 FrontEnd 的 Transaction
   FrameTracer::FrameTracerDataSource (在 FrameTracer/ 模块)

本地环形缓冲:
   LayerTracing            — 控制 layer trace 开关、buffer 大小
   TransactionTracing      — 控制 transaction trace 开关
   TransactionRingBuffer   — 写入侧无锁 ring buffer
   LocklessStack           — SPSC 无锁栈,用于事件直送 trace 线程

协议层:
   TransactionProtoParser  — Transaction <-> proto 双向序列化
   layerproto/             — proto schema
```

#### 4.7.2 关键数据结构

`LayerDataSource` 是 `perfetto::DataSource<LayerDataSource>` 的子类。Perfetto SDK 在每个 `Trace()` 调用点会广播到所有 enabled 的 `DataSource`,SF 在 `commit()` 末尾调用其 `Trace()` 把当前帧的 layer 列表序列化。

`TransactionProtoParser` 提供 `toProto(const TransactionState&)` 与 `fromProto(...)`,在录制/回放/调试时双向使用。

#### 4.7.3 流程

**录制路径**(每帧):

```
SF.commit() 末尾:
  if (layerTracingEnabled) LayerDataSource::Trace([&] { 写 snapshots 到 proto });
  if (txTracingEnabled)    TransactionDataSource::Trace([&] { 写本帧 applied tx });
```

**dumpsys 离线导出**:`dumpsys SurfaceFlinger --layer-trace start/stop/dump` → `LayerTracing` 控制本地 ring buffer 大小与导出。

#### 4.7.4 实现细节

1. **零拷贝 trace**:用 `LocklessStack`/`TransactionRingBuffer` 把"生产 trace 包"与"被 perfetto 消费"解耦,主合成线程几乎无锁开销。
2. **Perfetto Data Source 与 atrace 互补**:atrace 适合"线程时间线"上点状事件,Perfetto data source 适合"结构化对象"。
3. **可贡献模式**(`ContributingToPerfetto.md`):SF trace 包遵循特定 schema,Perfetto UI(`https://ui.perfetto.dev`)对其有图形化展示(图层条目、buffer queue、SurfaceFlinger track 等)。

#### 4.7.5 关键代码

`LayerDataSource` 是 perfetto data source,典型注册形式:

```cpp
class LayerDataSource : public perfetto::DataSource<LayerDataSource> {
public:
    void OnSetup(const SetupArgs&) override;
    void OnStart(const StartArgs&) override;
    void OnStop(const StopArgs&) override;
    static void Initialize();  // 注册到 perfetto producer
};
```

SF 在主循环里:

```cpp
LayerDataSource::Trace([&](LayerDataSource::TraceContext ctx) {
    auto packet = ctx.NewTracePacket();
    auto* layers = packet->set_surfaceflinger_layers_snapshot();
    serializeSnapshotsTo(*layers);
});
```

---

### 4.8 TimeStats / Jank / FrameTracer

这三个模块都是"帧时序的旁路观察者",但侧重点不同。

#### 4.8.1 TimeStats(`TimeStats/`,4 文件)

**职责**:聚合"显示一帧用了多久 / 漏帧次数 / refresh-rate switch 次数 / 各层延迟"等指标,以 stats atom proto 形式上报给 statsd。

**关键接口**(`TimeStats.h`):

```cpp
virtual void incrementTotalFrames();
virtual void incrementMissedFrames();
virtual void incrementRefreshRateSwitches();
virtual void recordFrameDuration(nsecs_t start, nsecs_t end);
virtual void recordRenderEngineDuration(nsecs_t start, nsecs_t end);
virtual void setPostTime(int32_t layerId, uint64_t frame, const std::string& name,
                         uid_t ownerUid, nsecs_t postTime, GameMode);
virtual void setLatchTime(int32_t layerId, uint64_t frame, nsecs_t latchTime);
virtual void setAcquireFence(int32_t layerId, uint64_t frame, const sp<FenceTime>&);
virtual void setPresentTime(int32_t layerId, uint64_t frame, nsecs_t presentTime, ...);
virtual bool onPullAtom(int atomId, std::vector<uint8_t>* pulledData);
```

**流程**:SF 在事务消费 / latch / present 的关键时刻打点 → TimeStats 内部聚合 → statsd 拉取(`onPullAtom`)。

#### 4.8.2 JankTracker(`Jank/`,2 文件)

**职责**:跟踪每层的卡顿事件,提供 listener 接口供 App / FrameworkTrace 订阅。

**关键接口**(`JankTracker.h`,全静态单例):

```cpp
static JankTracker& getInstance();
static void addJankListener(int32_t layerId, sp<IBinder> listener);
static void removeJankListener(int32_t layerId, sp<IBinder> listener, int64_t afterVsync);
static void flushJankData(int32_t layerId);
static void onJankData(int32_t layerId, gui::JankData data);
```

**实现细节**:`JankTracker` 是单例;`onJankData` 在 `FrameTimeline` 计算出实际 vs 预测帧时间偏差超阈值时被调用。`gui::JankData` 通过 `IJankListener` AIDL 派发回 App(新的 `JankData.aidl`)。

#### 4.8.3 FrameTracer(`FrameTracer/`,3 文件)

**职责**:把"每层每帧的关键时间点(post/queue/latch/present)"作为 trace 事件写入 Perfetto。

**关键 API**:

```cpp
void traceNewLayer(int32_t layerId, const std::string& name);
void traceTimestamp(int32_t layerId, uint64_t bufferID, uint64_t frameNumber,
                    nsecs_t timestamp, FrameEvent::BufferEventType type,
                    nsecs_t duration = 0);
void traceFence(int32_t layerId, uint64_t bufferID, uint64_t frameNumber,
                const std::shared_ptr<FenceTime>& fence, ...);
```

是一个独立的 Perfetto `DataSource<FrameTracerDataSource>`。配合 Perfetto UI 可以看到每个 SurfaceControl 的 buffer 在时间轴上的"post → queue → latch → present"全过程,与卡顿分析配套使用。

---

### 4.9 PowerAdvisor

**目录**:`PowerAdvisor/`(10 文件,含 `aidl/` 子目录)
**职责**:把 SF 的工作负载(CPU/GPU 实际耗时、目标 work duration)通过 ADPF(Android Dynamic Performance Framework)上报给 Power HAL,使 SoC 频率/电压更准确地配合显示节奏。

#### 4.9.1 类图与数据流

```
┌──────────────────────────────────────────────────────────────┐
│  PowerAdvisor (interface)                                     │
│   + init / onBootFinished                                     │
│   + updateTargetWorkDuration(targetDuration)                  │
│   + reportActualWorkDuration()                                │
│   + setExpensiveRenderingExpected(displayId, bool)            │
│   + startPowerHintSession(threadIds)                          │
│   + setGpuStartTime / setGpuFenceTime                         │
│   + setHwcValidateTiming / setHwcPresentTiming                │
│   + setSfPresentTiming / setExpectedPresentTime               │
│   + setRequiresRenderEngine(displayId, bool)                  │
│   + notifyCpuLoadUp() / notifyDisplayUpdateImminentAndCpuReset│
└──────────────────────────────────────────────────────────────┘
                       ▲
                       │
              ┌────────────────────────┐
              │  PowerAdvisor (impl)   │
              │   - mSessionManager    │  ◄── 管 PerformanceHintSession
              │   - mWorkload          │  ◄── 当前帧的 workload 数据
              └────────┬───────────────┘
                       │
              ┌────────────────────────┐
              │  SessionManager / SessionLayerMap │
              │   管理 per-uid 的 hint session    │
              └─────────────────────────────────┘
                       │
              ┌────────────────────────┐
              │  Power HAL (AIDL)      │
              │  IPower / IPowerHintSession│
              └────────────────────────┘
```

#### 4.9.2 关键数据结构

`Workload`(`PowerAdvisor/Workload.h`):一帧 SF 工作的时间统计聚合体,内含 hwc validate/present 起止、SF present 起止、GPU fence 时间、是否需要 RenderEngine 等。

`SessionManager` / `SessionLayerMap`:维护"每个 uid → PerformanceHintSession"的映射,允许 ADPF 按 App 聚合 hint。

#### 4.9.3 流程

**每帧**(摘自 `SurfaceFlinger.cpp` composite 路径):

1. composite 开始:`powerAdvisor->setExpectedPresentTime(...)`、`setSfPresentTiming(...)` 标记目标。
2. HWC validate 前后:`setHwcValidateTiming(start, end)`。
3. HWC present 前后:`setHwcPresentTiming(start, end)`。
4. 若有 client 合成:`setGpuStartTime(...)` 与 `setGpuFenceTime(...)`。
5. composite 结束:`reportActualWorkDuration()` → 把 `Workload` 总时长上报给 Power HAL。
6. 如本帧 expensive(大量 GPU 合成):`setExpensiveRenderingExpected(displayId, true)` 提示 HAL。

**目标 duration 更新**:`updateTargetWorkDuration()` 在刷新率/work duration 变化时被调用,告诉 HAL "下一帧期望 work duration"。

#### 4.9.4 实现细节

1. **multi-display hint**:`setDisplays(displayIds)` 让 HAL 知道当前活跃显示集合;ADPF 可对多屏并发场景调整。
2. **`notifyCpuLoadUp`**:`SurfaceFlinger` 作为 `ICEPowerCallback` 实现把这个钩子转给 PowerAdvisor,在大动作发生前(如 mode set / hotplug)预先加速 CPU。
3. **`startPowerHintSession(threadIds)`**:把 SF 主线程、RenderEngine 线程、HwcAsyncWorker 等关键线程 id 注册成一个 hint session。

---

### 4.10 Effects / Reporters / Overlays

这是一组"小而独立"的辅助模块,集中放在 SF 顶层与 `Effects/`、`layerproto/` 等位置。

| 文件 / 目录 | 职责 |
| --- | --- |
| `Effects/Daltonizer.{cpp,h}` | 色盲滤镜:`setType(ColorBlindnessType)` + `setLevel(0..1)` → 输出 `mat4` 颜色变换矩阵,经 `SurfaceFlinger::setDaltonizerType()` 应用到全局色彩变换 |
| `RefreshRateOverlay.{cpp,h}` | 屏幕角落叠加"当前刷新率"调试信息 |
| `HdrSdrRatioOverlay.{cpp,h}` | 叠加 HDR/SDR 亮度比的调试信息 |
| `RegionSamplingThread.{cpp,h}` | SystemUI 调它在屏幕指定区域采样平均色,用于动态色/状态栏适配。独立线程跑 GPU readback |
| `FpsReporter.{cpp,h}` | 把"每层近段 fps"汇报给监听者 |
| `HdrLayerInfoReporter.{cpp,h}` | 通知 SystemUI 是否有 HDR 层在前台(用于亮度策略) |
| `TunnelModeEnabledReporter.{cpp,h}` | 视频 tunnel mode(直通)启用状态变化通知 |
| `ActivePictureTracker.{cpp,h}` | "活跃画面"识别(TV 画质处理) |
| `WindowInfosListenerInvoker.{cpp,h}` | 把窗口信息广播给输入 / 可访问性监听器 |
| `TransactionCallbackInvoker.{cpp,h}` | 把"事务已提交 / 已呈现 / WindowInfos 已更新"等事件回调发给 App |
| `BackgroundExecutor.{cpp,h}` | 非关键工作线程(定时器、缓存清理) |
| `FrameTracker.{cpp,h}` | 每层维护近期几帧的时间数据(present/desired/acquire),`dumpsys` 输出"frame stats" |
| `ClientCache.{cpp,h}` | 缓存 client 提交的 buffer/handle 数据,加速 `setBuffer` |
| `LayerVector.{cpp,h}` | 兼容 wrapper:旧 API 暴露的"layer 容器" |

`Daltonizer` 示例:

```cpp
class Daltonizer {
public:
    void setType(ColorBlindnessType type);     // Protanomaly / Deuteranomaly / Tritanomaly
    void setMode(ColorBlindnessMode mode);     // Simulation / Correction
    void setLevel(int32_t level);              // 强度
    const mat4& operator()();                  // 计算并返回当前颜色变换矩阵
private:
    bool mDirty = true;
    mat4 mColorTransform;
    float mLevel = 0.7f;
};
```

`SurfaceFlinger` 把 `Daltonizer()` 产物乘到全局色彩变换矩阵中,经 `Output.setColorTransform` 下发到 HWC(若 HAL 支持)或 RenderEngine(client 合成路径)。

---

### 4.11 Utils / Factory / 工厂注入

#### 4.11.1 `Utils/`

| 文件 | 作用 |
| --- | --- |
| `Dumper.h` | 把 `dumpsys` 输出按 section/缩进格式化 |
| `FenceUtils.h` | Fence 合并、等待、超时辅助 |
| `OnceFuture.h` | 单次 promise/future 同步原语 |
| `OverlayUtils.h` | Overlay 绘制辅助(配合 `RefreshRateOverlay` / `HdrSdrRatioOverlay`) |

#### 4.11.2 `SurfaceFlingerFactory`

**接口**(`SurfaceFlingerFactory.h`):

```cpp
class Factory {
public:
    virtual std::unique_ptr<HWComposer> createHWComposer(const std::string& serviceName) = 0;
    virtual std::unique_ptr<scheduler::VsyncConfiguration> createVsyncConfiguration(Fps) = 0;
    virtual sp<DisplayDevice> createDisplayDevice(DisplayDeviceCreationArgs&) = 0;
    virtual sp<GraphicBuffer> createGraphicBuffer(uint32_t w, uint32_t h, PixelFormat, uint32_t layerCount, uint64_t usage, std::string name) = 0;
    virtual std::unique_ptr<compositionengine::CompositionEngine> createCompositionEngine() = 0;
    virtual sp<Layer> createLayer(const LayerCreationArgs& args) = 0;
    virtual sp<LayerFE> createLayerFE(const std::string& layerName, const Layer* owner) = 0;
    virtual std::unique_ptr<FrameTracer> createFrameTracer() = 0;
    virtual std::unique_ptr<scheduler::FrameTimeline> createFrameTimeline(...) = 0;
};
```

**默认实现**(`SurfaceFlingerDefaultFactory.cpp`):各 `create*` 直接 `new` 真实实现;`fuzzer/`、`tests/mock/` 通过自定义工厂注入 mock。

`SurfaceFlinger` 构造时持有 `Factory&` 引用,从此**所有重型组件**都经工厂创建——这是 A16 把整个 SF 变得"可单测"的关键支点。

#### 4.11.3 关键注入点

- `init()` 阶段:`mCompositionEngine = mFactory.createCompositionEngine();` `mHwComposer = mFactory.createHWComposer(name);`。
- 每显示注册:`mFactory.createDisplayDevice(args)`,`mFactory.createVsyncConfiguration(refreshRate)`。
- 每层创建:`mFactory.createLayer(args)`、`mFactory.createLayerFE(name, owner)`。
- 调度初始化:`mScheduler->createFrameTimeline = mFactory.createFrameTimeline(...);`。

至此,SurfaceFlinger 服务的全部模块与协作机制已系统阐述。文档每条结论均可在 `services/surfaceflinger/` 中的对应文件中验证。




