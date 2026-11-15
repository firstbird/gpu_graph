# Mali GPU 架构与 kbase 驱动原理 · PPT 说明文档

Nov 8, 2026 · @zhenli

本文档配合《Mali GPU 架构与 kbase 驱动原理》27 页 PPT 使用：每页一段说明讲这页想表达什么，再用一张表解释页上出现的关键术语。

函数名、数值都对照过 `midgard/` 源码（DDK r48p0，Android 13 分支）。读源码时可配合 `GPU_DRIVER_ARCHITECTURE.md` 一起看。

## 开场

### 第 1 页 · 封面

交代主题与范围：ARM Mali GPU 的内核驱动 kbase。右侧三层图是全片主线：用户态驱动 → 内核驱动 → GPU 硬件。

| 术语 | 说明 |
| --- | --- |
| Mali | ARM 的移动 GPU 系列，历经 Midgard、Bifrost、Valhall、第 5 代等架构 |
| kbase | Mali Kernel Base，Mali 的 Linux 内核驱动，编译产物为 mali\_kbase.ko |
| DDK | Driver Development Kit，ARM 发布的驱动包，包含用户态与内核态两部分；r48p0 是版本号 |
| libmali | DDK 的用户态部分，实现 GLES / Vulkan / OpenCL |
| MMIO / IRQ | 见第 5 页 |

### 第 2 页 · 目录

全片分 5 部分，第四部分（关键流程）是重点。颜色约定全篇统一：蓝色 JM、橙色 CSF、绿色内存与 MMU、紫色电源、红色中断与错误。

## 第一部分 · 定位与全景

### 第 3 页 · GPU 在 Android 图形栈中的位置

一帧画面从 App 到屏幕要经过 App → GPU → SurfaceFlinger → HWC → DRM/DPU。GPU 出现两次：App 渲染，以及 HWC 处理不了的图层由 SF 用 GPU 合成。两者通过 Gralloc 分配的同一块 buffer 零拷贝传递。

| 术语 | 说明 |
| --- | --- |
| SurfaceFlinger（SF） | Android 的合成服务进程，每帧把各应用的图层合成一张画面 |
| HWC（Composer3） | 硬件合成 HAL，用显示控制器的 overlay 能力直接叠加图层，省 GPU |
| CLIENT 图层 | HWC 处理不了、退回给 SF 用 GPU 合成的图层 |
| RenderEngine | SF 内部的 GPU 合成器，基于 Skia（GL / Vulkan） |
| DRM/KMS、DPU | Linux 显示子系统；DPU 是显示控制器硬件，负责扫描输出到屏幕 |
| Gralloc / GraphicBuffer | 图形内存分配器与它分配的 buffer，底层是 dmabuf |
| dmabuf | Linux 跨设备共享内存的机制，GPU、DPU、编解码器可访问同一块物理内存 |
| fence | 同步对象：acquire fence 表示写完了，release fence 表示读完了 |

### 第 4 页 · 用户态与内核态的分工

shader 编译、命令生成都在用户态 libmali。kbase 只做四件事：管地址空间、送活、管电管频、兜底观测。两者只通过 /dev/mali0 的几个系统调用交互。

| 术语 | 说明 |
| --- | --- |
| /dev/mali0 | kbase 注册的字符设备，用户态的唯一入口 |
| ioctl | 向驱动发命令的系统调用，如 MEM\_ALLOC、JOB\_SUBMIT |
| mmap | 把 GPU 内存映射到进程地址空间，让 CPU 能读写 |
| poll / read | 等待并取回作业完成事件 |
| job chain | 用户态在 GPU 内存里写好的一串任务描述符，GPU 按链执行 |
| platform driver | Linux 通过设备树匹配的驱动类型；kbase 按 compatible（如 arm,mali-bifrost）匹配 |

### 第 5 页 · 驱动视角下的 GPU 硬件组成

驱动眼里的 GPU 就是几类资源：3 块寄存器、3 个 Job Slot、核位图、16 个 AS 槽、3 根中断线。Shader Core 内部结构是通用知识，驱动不直接操作。

| 术语 | 说明 |
| --- | --- |
| MMIO | Memory-Mapped I/O，内存映射寄存器。GPU 的控制寄存器被映射到一段物理地址，驱动 ioremap 后像读写内存一样读写它们（kbase\_reg\_read / kbase\_reg\_write） |
| 寄存器块 | GPU\_CONTROL（0x0000，设备信息与上下电）、JOB\_CONTROL（0x1000，Job Slot）、MMU（0x2000，地址空间） |
| IRQ | 中断。GPU 有 JOB、MMU、GPU 三根中断线，分别报作业完成、缺页 / 总线错误、设备级事件 |
| GPU\_ID | 只读寄存器，标识 GPU 型号与版本；驱动据此选寄存器布局和勘误表 |
| \*\_PRESENT | SHADER / TILER / L2 / AS / JS\_PRESENT 等位图，表示硬件有哪些单元 |
| \*\_PWRON | 写 1 给对应单元上电的寄存器，如 SHADER\_PWRON、L2\_PWRON |
| Job Manager / Job Slot | JM 架构的作业调度硬件；3 个槽 JS0 / JS1 / JS2 各自执行 job chain |
| Shader Core | 执行着色器的计算核，数量由 SHADER\_PRESENT 决定 |
| Tiler | 分块渲染中把图元归类到各 tile 的硬件单元 |
| L2 Cache | 所有核共享的二级缓存 |
| GPU MMU / AS | GPU 自己的地址翻译单元；AS（Address Space）是它的翻译槽，最多 16 个 |

## 第二部分 · 核心原理

### 第 6 页 · 分块渲染（TBDR）原理

（通用知识补充）Mali 先把屏幕切成 16×16 像素的 tile，几何阶段记录每块覆盖哪些图元，片元阶段再逐块在片上完成着色、整块写回一次。好处是大幅减少 DDR 带宽和功耗。

| 术语 | 说明 |
| --- | --- |
| TBDR | Tile-Based Deferred Rendering，分块延迟渲染 |
| IMR | Immediate Mode Rendering，立即渲染，每次绘制直接读写帧缓冲，桌面 GPU 常见 |
| tile | 屏幕上的一小块，Mali 为 16×16 像素 |
| Binning | 把图元归类到它覆盖的 tile |
| Tiler heap | 存放图元表的 DDR 内存，随帧增长 |
| Tile 缓冲 | Shader Core 内的片上内存，存当前 tile 的颜色和深度 |
| GROW\_ON\_GPF / JIT | 内存缺页时自动增长 / 驱动池化的临时内存，用于 tiler heap 这类大小不定的内存 |

### 第 7 页 · 两代作业架构：JM 与 CSF

JM 下内核亲自把每个 atom 送进 Job Slot；CSF 下内核只调度队列组，命令流由 GPU 内的 MCU 固件执行。两者编译期二选一，我们平台是 JM。

| 术语 | 说明 |
| --- | --- |
| JM | Job Manager，Midgard / Bifrost / 早期 Valhall 的作业提交方式 |
| CSF | Command Stream Frontend，后期 Valhall 及之后的方式 |
| MALI\_USE\_CSF | 编译宏，决定编入 JM 还是 CSF 后端 |
| atom | JM 的提交单位：一条 job chain + 最多 2 个依赖 |
| queue group / CSG slot | CSF 的队列组，和它要占用的硬件槽（最多 31 个） |
| MCU 固件 | CSF GPU 里的小处理器及其固件，负责拉取并执行命令流 |
| doorbell（门铃） | 用户态写一个特殊地址来通知 GPU 有新命令 |
| sync object | CSF 下存在 GPU 内存里的同步变量，用户态轮询它判断完成 |

### 第 8 页 · JM 执行模型

atom 之间有依赖，内核按 core\_req 把它们分到 3 个 Job Slot。每个槽硬件上只有“正在跑”和“下一个”两个位置，所以驱动的镜像队列深度固定为 2。

| 术语 | 说明 |
| --- | --- |
| core\_req | atom 的需求位，表明类型：FS 片元、V 顶点、T Tiler、ONLY\_COMPUTE 等 |
| ORDER / DATA 依赖 | ORDER 只管先后；DATA 要求前驱成功，否则级联失败 |
| soft job | 由 CPU 执行的 atom，如等 fence、发 fence、JIT 分配 |
| kbase\_js\_get\_slot | 选槽函数：片元进 JS0，第 2 核组的纯计算进 JS2，其余进 JS1 |
| HEAD / HEAD\_NEXT | Job Slot 里正在执行的位置 / 已装填等待接续的位置 |
| slot\_rb | 驱动对每个槽这两个位置的软件镜像，SLOT\_RB\_SIZE = 2 |

## 第三部分 · 驱动架构与主要模块

### 第 9 页 · kbase 五层模型

从上到下是接口层、生命周期层、公共策略 / 作业策略（并列）、硬件后端。另有三种机制贯穿各层：一把大锁、复位框架、打点。

| 术语 | 说明 |
| --- | --- |
| ① core\_linux | mali\_kbase\_core\_linux.c，实现 open / ioctl / mmap 等文件操作 |
| ③ mem | mali\_kbase\_mem\*.c，GPU 内存管理：VA zone、region、页池、mmap、dma-buf 导入、JIT |
| ③ mmu | mmu/ 目录，GPU 页表的插入、拆除、更新，以及 page fault 补页 |
| ③ pm | mali\_kbase\_pm.c 与 pm\_backend / pm\_policy，电源前端与策略：active\_count、电源策略、系统休眠 |
| ③ hwcnt | 硬件性能计数器的软件栈，让多个消费者共用一套计数器 |
| ③ tl | timeline，给 Streamline / Perfetto 的事件流打点 |
| ③ ipa | 功耗模型，按计数器和温度估算 GPU 功耗，供温控调频 |
| ④ jd / js / jm | Job Dispatcher（解依赖）/ Job Scheduler（调度）/ Job Manager 前端（踢到后端） |
| ⑤ backend/gpu | 直接读写硬件的后端目录，下面四项都在这里 |
| ⑤ jm\_hw | mali\_kbase\_jm\_hw.c，写 Job Slot 寄存器（kbase\_job\_hw\_submit）、处理 JOB 中断（kbase\_job\_done）、软停 / 硬停、JM 复位 |
| ⑤ jm\_rb | mali\_kbase\_jm\_rb.c，管理 slot\_rb：入队出队、推进 gpu\_rb\_state、完成回收 |
| ⑤ pm\_driver | mali\_kbase\_pm\_driver.c，L2 / Shader 上下电状态机，写 \*\_PWRON / PWROFF；GPU 软复位入口 kbase\_pm\_init\_hw |
| ⑤ irq\_linux | mali\_kbase\_irq\_linux.c，注册 JOB / MMU / GPU 三路中断及其 handler |
| ⑤ mmu\_hw\_direct | mmu/mali\_kbase\_mmu\_hw\_direct.c，写 AS 寄存器：配置 TRANSTAB 等，发 LOCK / UNLOCK / FLUSH 命令并等完成 |
| ⑤ hw\_access regmap | 寄存器访问抽象：按 GPU 版本选好的寄存器偏移表，统一由 kbase\_reg\_read / write 访问 |
| hwaccess\_lock | 保护“硬件正在做什么”的自旋锁，可在中断里持有 |
| ktrace / tlstream | 内核环形 trace / 给性能工具的时间线事件流 |

### 第 10 页 · 两条业务路径

路径 A（如 MEM\_ALLOC）只准备资源，不经过作业策略层；路径 B（JOB\_SUBMIT）送活，会回头调用公共策略要电、拿 AS。关键：AS 与 Job Slot 是两套硬件，提交时 JS\_CONFIG\_NEXT 低位写入 as\_nr 把二者绑定。

| 术语 | 说明 |
| --- | --- |
| as\_nr | kctx 当前占用的 AS 槽编号，未绑定时为 INVALID |
| TLB / flush | GPU 缓存地址翻译结果的小表；改了页表要 flush 让它失效 |
| JS\_CONFIG\_NEXT | Job Slot 的配置寄存器，含 AS 号与 cache 策略 |

#### 补充 · JS\_CONFIG\_NEXT 与 Job Slot 双缓冲（概念补充，位定义未对照源码）

`JS_CONFIG_NEXT` 是 JM 架构（Midgard、Bifrost、早期 Valhall）下 Job Slot 寄存器块里「下一个 job」的配置寄存器。每个 Job Slot 的寄存器是双缓冲的：无后缀的一组（`JS_HEAD`、`JS_AFFINITY`、`JS_CONFIG`、`JS_STATUS`）描述正在执行的 job，带 `_NEXT` 的一组描述排队等待的 job。`kbase_job_hw_submit` 先填好 NEXT 集合，最后写 `JS_COMMAND_NEXT = START` 触发；当前 job 结束后，硬件把 NEXT 集合移入当前集合并开始执行，job 之间没有空泡。

| 寄存器 | 作用 |
| --- | --- |
| `JS_HEAD_NEXT` | 下一个 job chain 的头地址 |
| `JS_AFFINITY_NEXT` | 下一个 job 可用的 shader core 掩码 |
| `JS_CONFIG_NEXT` | 地址空间（AS）编号，job 开始前、结束后的缓存操作，flush reduction，线程优先级 |
| `JS_FLUSH_ID_NEXT` | 下一个 job 的 flush ID，配合 flush reduction |
| `JS_COMMAND_NEXT` | 写 START，真正触发提交 |

类似的配置寄存器：GPU 全局的 `JM_CONFIG`、`SHADER_CONFIG`、`TILER_CONFIG`、`L2_MMU_CONFIG`（kbase 常用它们打硬件 workaround）；每个 AS 一组的 `AS_TRANSTAB`、`AS_MEMATTR`、`AS_TRANSCFG`、`AS_COMMAND`（页表基址、内存属性、页表格式、MMU 命令）；CSF 架构没有 Job Slot 和 `_NEXT`，配置放在固件与驱动共享的 CSG / CS 接口里（如 `CS_CONFIG`、`CSG_CONFIG`）。

### 第 11 页 · 源码地图

midgard/ 共 367 个文件、16.8 万行；csf/ 最大但 JM 构建不编入。读 JM 主线主要看根目录的 jd / js / jm 和 backend/gpu。

| 术语 | 说明 |
| --- | --- |
| backend/gpu | 直接读写硬件的代码：slot\_rb、写寄存器、电源状态机、中断 |
| hwcnt / tl | 硬件计数器 / timeline 打点 |
| hw\_access | 寄存器访问抽象与各版本寄存器表 |

### 第 12 页 · 核心对象与它们的关系

每块 GPU 一个 kbase\_device；每次 open 一个 kbase\_file；SET\_FLAGS 后一个 kbase\_context。kctx 的页表通过 as\_nr 动态绑到硬件 AS。

| 术语 | 说明 |
| --- | --- |
| kbase\_device（kbdev） | 一块物理 GPU 的全部状态 |
| kbase\_file（kfile） | 一次 open 得到的对象，带 setup 状态机与事件队列 |
| kbase\_context（kctx） | 一个进程的 GPU 世界：页表、VA、页池、atom |
| kbase\_va\_region | 一段 GPU 虚地址及其物理页 |
| SET\_FLAGS | 创建 kctx 的 ioctl |

### 第 13 页 · 内存管理

GPU VA、CPU VA、物理页三者分开管理。物理页先从 kctx 私有池拿，不够从全机共享池拿，再不够向内核要。

| 术语 | 说明 |
| --- | --- |
| VA / PA | 虚拟地址 / 物理地址 |
| vma | Linux 进程地址空间里的一段映射区域 |
| zone | GPU VA 的分区：SAME\_VA、CUSTOM\_VA、EXEC\_VA 等 |
| SAME\_VA | GPU VA 与 CPU VA 取同一数值，64 位进程默认 |
| mem\_pools | 空闲物理页缓存，4KB 与 2MB 各一套 |
| buddy | Linux 内核的物理页分配器 |
| shrinker | 内存紧张时内核回调，让驱动归还缓存页 |
| dma-buf 导入（UMM） | 用别的设备已分配的内存，如 Gralloc buffer |

#### 一块 GPU 内存涉及的对象：完整交互流程

这里说的是 libmali 自己通过 MEM\_ALLOC 申请的 native 内存，典型用途有：顶点 / 索引缓冲、uniform 缓冲、App 上传的纹理、离屏渲染目标、shader 二进制（走 EXEC\_VA zone）、job chain 描述符。kbase 并不知道一块内存是纹理还是顶点，它只看分配 flags（读写权限、可否执行、cache 属性、可否增长），所以这些用途走的是同一条路。tiler heap、scratch 这类临时内存多走 JIT，对象关系相同，只是分配入口不同。

不走这条路的是窗口 / 帧缓冲：GraphicBuffer 由 Gralloc 分配（dma-buf），libmali 用 MEM\_IMPORT 导入。这时仍会建 region、写 GPU 页表，但物理页来自导出方，不从 kbase 页池拿，也不计入 used\_pages（下面第 2 步和第 7 步的取页 / 还页不适用）。

一块 GPU 内存从分配到释放，会依次碰到四个对象：CPU 侧的 vma、GPU VA 侧的 kbase\_va\_region（下称 region）、物理页集合 kbase\_mem\_phy\_alloc（下称 alloc）、GPU 页表 kctx->mmu。它们之间的指向关系是：vma->vm\_private\_data 指向 region；region 的 start\_pfn / nr\_pages 就是它的 GPU VA，gpu\_alloc / cpu\_alloc 指向 alloc；alloc.pages\[\] 里是物理页，来自页池；页表把 region 的 GPU VA 映射到这些物理页。

1. **分配（ioctl MEM\_ALLOC）。** kbase\_mem\_alloc 按 flags 选 zone（SAME\_VA / CUSTOM\_VA / EXEC\_VA），新建一个 region，填好 nr\_pages（= va\_pages）和权限、cache 属性；kbase\_reg\_prepare\_native 建 alloc（普通情况 cpu\_alloc 与 gpu\_alloc 是同一个，Infinite Cache + CPU\_CACHED 时各建一个）；kbase\_alloc\_phy\_pages 按 commit\_pages 从页池拿物理页填进 alloc.pages\[\]，并计入 used\_pages。
2. **取物理页（页池）。** 先从 kctx->mem\_pools 拿，不够沿 next\_pool 从 kbdev->mem\_pools 拿，再不够经 mgm\_alloc\_page 向内核 buddy 要。
3. **定 GPU VA 并写 GPU 页表（kbase\_gpu\_mmap）。**
   1. 非 SAME\_VA：在 ioctl 里直接调用，在 zone 的空闲区间里占一段地址写进 region.start\_pfn，把 region 插入 zone 红黑树，用 kbase\_mmu\_insert\_pages 写 L3 / L2 页表项；ioctl 返回真实 GPU VA。
   2. SAME\_VA（64 位进程默认）：ioctl 先把 region 挂到 pending\_regions\[cookie\]，返回 cookie；用户 mmap(cookie) 时，kbasep\_reg\_mmap 取回 region，以 vma 起点作 GPU VA 调 kbase\_gpu\_mmap，这样 GPU VA = CPU VA。
   3. 写完页表先 kbase\_mmu\_sync\_pgd 刷 CPU cache；若这个 kctx 正占着 AS，再发 AS\_COMMAND FLUSH 让 GPU TLB 作废旧翻译，没占着就跳过。
4. **CPU 映射与访问（mmap → kbase\_cpu\_mmap）。** kbase\_context\_mmap 按 vm\_pgoff 找到 region，kbase\_cpu\_mmap 把 vma 挂上 kbase\_vm\_ops、vm\_private\_data 指向 region，但不预先填 CPU 页表；CPU 第一次读写时进 kbase\_cpu\_vm\_fault，用 alloc.pages\[\] 里的物理页按需插 PFN。
5. **GPU 访问。** 用户把 GPU VA 写进 job chain 并提交；送活时 kbase\_ctx\_sched\_retain\_ctx 给 kctx 占 AS，必要时 kbase\_mmu\_update 把页表根写进 AS\_TRANSTAB；GPU 按这棵页表 L0→L3 翻译得到物理地址。
6. **缺页增长（可选）。** commit\_pages < va\_pages 且 region 带 PF\_GROW 时，GPU 访问还没有物理页的地址会触发 MMU 中断；pf\_wq 里 page\_fault\_worker 按 extension 从页池补页、写入页表，发 UNLOCK 让硬件重放那次访问。不可增长的 region 缺页则 report\_fault\_and\_kill。
7. **释放。** SAME\_VA 只能 munmap（MEM\_FREE 会拒绝），非 SAME\_VA 用 MEM\_FREE。kbase\_cpu\_vm\_close → kbase\_va\_region\_alloc\_put 递减 region 引用，归零时拆 GPU 页表（kbase\_mmu\_teardown\_pages，并 flush）、把 region 从 zone 树上摘下、释放物理页并从 used\_pages 扣减。物理页先回 kctx 池，满了 spill 到 kbdev 池，再满经 mgm\_free\_page 还给内核。

一句话：region 是中心，左边连 CPU 的 vma，右边连物理页，向下连 GPU 页表；GPU 映射一般在分配或 mmap 时就建好，CPU 映射则等到第一次访问才填。

#### 补充 · memory group 与页池的二维结构（概念补充，未对照源码）

上面第 2 步里的「kctx 池、kbdev 池」，其实各有一个按 memory group 和页大小组织的池阵列。memory group 是 Memory Group Manager（MGM）机制定义的物理页分区，用编号（`mem_group_id`）标识；kbase 只按编号分池，页从哪里来、带什么属性由平台的 MGM 模块决定（对应上面的 `mgm_alloc_page` / `mgm_free_page`）。group 与页大小是两个独立的维度：每个 group 内部再各有 4KB（small）和 2MB（large）两套池。

|  | small（4KB） | large（2MB） |
| --- | --- | --- |
| group 0 | pool | pool |
| group 1 | pool | pool |
| …… | …… | …… |

kctx 池的 `next_pool` 指向 kbdev 里同 group、同页大小的池，不会跨 group，也不会跨页大小；必须同 group，是因为页的来源和属性由 group 决定。申请和释放沿同一条链走，没有反向，区别只在往下一级的触发条件：

|  | 先到 | 往下一级的条件 | 最终兜底 |
| --- | --- | --- | --- |
| 申请 | kctx 池 | kctx 池空，改从 kbdev 池取 | kbdev 池也没有，经 `mgm_alloc_page` 向内核要 |
| 释放 | kctx 池 | kctx 池满，溢出到 kbdev 池 | kbdev 池也满，经 `mgm_free_page` 还给内核 |

group 编号本身只是索引：group 0 是默认组，总数由编译期宏决定，用户态通过分配 flags 里的 group id 字段指定；其余编号各代表什么，由 SoC 厂商的 MGM 实现决定，常见用途有给不同 group 的 GPU 页表项设置不同属性（如 PBHA 位）、页来自不同内存区域、按 group 计账。没有 MGM 的平台基本只用 group 0。

#### 补充 · CPU 页表，以及哪些 GPU 内存需要 CPU 映射（概念补充，未对照源码）

**CPU 页表是什么**：MMU 使用的多级映射表，把虚拟页号映射到物理页号，并带权限、缓存属性等位；ARM64 上常见 4 级（PGD、PUD、PMD、PTE），根页表由 TTBR 寄存器指向。CPU 访问一个 VA 时，MMU 先查 TLB；未命中就逐级走页表（page walk），走到有效 PTE 就得到 PA 并回填 TLB；PTE 无效（还没填）则触发缺页异常交给内核。这正是上面第 4 步「不预先填 CPU 页表」的含义：`mmap` 只建 vma，PTE 为空，首次访问缺页，`kbase_cpu_vm_fault` 才把物理页插进页表，好处是只访问到的页才占页表项。

**什么时候需要 CPU 映射**：只要 CPU 需要读写这块内存的内容。集成 GPU 用的就是系统 DRAM，所以映射的是 GPU 内存背后的那些物理页。

| 场景 | 典型内容 | 注意点 |
| --- | --- | --- |
| CPU 写、GPU 读 | job chain 描述符、shader 二进制、顶点和 uniform 缓冲、纹理上传 | 非一致平台写完要做缓存同步 |
| GPU 写、CPU 读 | compute 输出 buffer、query 结果、硬件计数器、同步对象状态 | 必须等 GPU 完成再读；注意缓存刷新与失效 |
| CPU 与 GPU 通信结构 | CSF 的命令流 ring buffer、doorbell 页、fence 与 semaphore 的状态字 | 内存序与原子性 |

**不是所有 GPU 内存都有 CPU 映射**：CPU 映射是可选的，由分配方通过 flags 决定。

| 类别 | 例子 | CPU 映射情况 |
| --- | --- | --- |
| CPU 无访问需求 | 离屏渲染目标、深度和模板缓冲、GPU 之间传递的临时数据 | 分配时不带 CPU 权限，不建 CPU 映射 |
| 带 CPU 权限但未访问 | 已 `mmap` 的 buffer | CPU 页表按需填，未访问的页没有页表项 |
| 导入的共享内存 | Gralloc 分配的窗口和帧缓冲（MEM\_IMPORT） | CPU 能否映射由导出方（gralloc usage 标志）决定 |
| 仅内核访问 | GPU 页表页、内核管理的元数据 | 内核用自己的映射，用户进程没有 vma |
| CPU 禁止访问 | protected memory（DRM 受保护 surface） | 硬件禁止 CPU 读，没有普通 CPU 映射 |

渲染目标不是「CPU 不能访问」：kbase 没有这条规则，CPU 映射由分配方按需求决定。默认常常不映射，是因为 CPU 通常不碰它，而且渲染目标常用 GPU 专有的 tiled 排布，开了 AFBC 的更是压缩块格式，CPU 直接读到的不是线性像素。截图、`glReadPixels` 这类回读，驱动通常先让 GPU 转成线性格式，拷到一块 CPU 可映射的 buffer 再读。

#### 补充 · native 内存、start\_pfn、cpu\_alloc / gpu\_alloc、SAME\_VA（概念补充，未对照源码）

**native 内存**：kbase 自己分配并拥有物理页的内存。用户态用 MEM\_ALLOC 申请，物理页从 kctx 页池拿（kbase\_reg\_prepare\_native，alloc 类型 NATIVE）。与它相对的是 IMPORT（UMM 即 dma-buf、USER\_BUF，只建映射，页不归 kbase）、ALIAS（把已有内存拼成一个 VA）。区别在谁管物理页：native 的页计入 used\_pages，释放时还回页池；import 类释放时不还页。

**start\_pfn 是 GPU 虚拟页号，不是物理页号。** region->start\_pfn = GPU VA >> PAGE\_SHIFT，pfn 只是沿用了内核「页帧号」的叫法。nr\_pages 是这段 GPU VA 占多少页。物理页在 alloc.pages\[\] 里，页表把「start\_pfn 起的虚拟页」映射到它们。SAME\_VA 时 start\_pfn 等于 mmap 的 vma 起点页号。

**cpu\_alloc / gpu\_alloc 是数据指针，不是函数。** 它们是 kbase\_va\_region 里类型为 struct kbase\_mem\_phy\_alloc \* 的成员，值在分配时由函数调用赋入：

```c
reg->cpu_alloc = kbase_alloc_create(...);                  // 新建，refcount = 1
reg->gpu_alloc = kbase_mem_phy_alloc_get(reg->cpu_alloc);  // 同一对象，refcount = 2
```

「gpu\_alloc = 对 cpu\_alloc 的引用」指两个指针指向同一个 alloc 对象，只是引用计数加 1。只有 Infinite Cache + CPU\_CACHED 时才为 GPU 单独建一个，两边各有页集合。释放时两个指针各 put 一次：共享时计数 2 → 1 → 0，到 0 才还页；分开时各自释放。

**其他入口也会给这两个指针赋值**：MEM\_IMPORT（UMM / USER\_BUF，页不走页池）、MEM\_ALIAS（ALIAS 类型）、JIT（入口不同，页仍从页池拿）。

**SAME\_VA：数值相同，仍是两个空间。** 两个空间是 CPU 虚拟地址空间（进程的，Linux mm / vma 管理，CPU 页表翻译）和 GPU 虚拟地址空间（kctx 的，kbase\_va\_region / zone 管理，GPU 页表翻译，页表根在 AS\_TRANSTAB）。SAME\_VA 让一块内存在两边用同一个数值，CPU 写进描述符的指针 GPU 可直接用；但两张页表各自维护：

- 建立时间不同：GPU 映射在分配 / mmap 时就写好，CPU 页表等首次访问才填。
- 属性可以不同：两边的权限、cache 属性分别设置。
- 拆除分别做：munmap → vm\_close → region put，才去拆 GPU 页表并释放物理页。

### 第 14 页 · GPU MMU

页表是 4 级、存在 DDR；AS 是 GPU 上的翻译槽。MEM\_ALLOC 只写页表，第一次送活时才把页表根装进 AS。

| 术语 | 说明 |
| --- | --- |
| L0–L3 | 4 级页表，GPU VA 按 9+9+9+9+12 位切分 |
| PTE / 大页 | 页表项；L2 级可直接映射 2MB 大页 |
| AS\_TRANSTAB | AS 槽寄存器，存页表根的物理地址 |
| AS\_COMMAND | AS 槽命令寄存器，发 UPDATE / FLUSH / LOCK / UNLOCK |
| retain | kbase\_ctx\_sched\_retain\_ctx：给 kctx 占一个 AS，必要时驱逐旧的 |
| kbase\_mmu\_update | 把 kctx 的页表根写进 AS 寄存器 |

#### 补充 · L2 / L3 页表是什么，kbase\_mmu\_insert\_pages 与 flush（概念补充，未对照源码）

GPU 页表是 4 级：虚拟地址按 9+9+9+9+12 切开，每级一页 4KB、512 项、每项 8 字节。

| 级别 | 索引位 | 表项指向 |
| --- | --- | --- |
| L0 | bit 47–39 | L1 页表页 |
| L1 | bit 38–30 | L2 页表页 |
| L2 | bit 29–21 | L3 页表页，或直接是 2MB 大页 |
| L3 | bit 20–12 | 4KB 物理页（叶子） |

PPT 里「kbase\_mmu\_insert\_pages 写 L3 / L2 页表项」说的就是这两个叶子级：4KB 小页写 L3，2MB 大页写 L2（不再有 L3）。写之前路径上缺的中间级会现分配页表页，取自 kbdev->mem\_pools.small。

「已绑 AS 才 flush」：kctx 占着 AS 时，GPU 的 TLB 可能缓存了旧翻译，写页表后要 kbase\_mmu\_sync\_pgd 同步，再发 AS\_COMMAND FLUSH 作废 TLB；kctx 没占 AS 时 GPU 不在用这棵页表，无需 flush，等它被调度上 AS、AS\_TRANSTAB 指向页表根时自然读到新内容。

### 新增页 · AS 绑定：选槽、换槽与驱逐（PPT 第 16 页）

哪个 kctx 占哪个 AS 槽全由驱动决定，GPU 不通知也不参与。入口是 mali\_kbase\_ctx\_sched.c 的 kbase\_ctx\_sched\_retain\_ctx：只有 refcount 从 0 变成 1、且 kctx 原来的 as\_nr 已失效时，才换槽并调 kbase\_mmu\_update 写硬件。

判断顺序：

1. refcount 加 1 后大于 1：早已占着槽，直接返回，不写硬件。
2. 等于 1，且旧 as\_nr 有效、那个槽在 as\_free 里仍空闲：复用原槽，硬件里还是它的页表根，不写硬件。
3. 否则取 as\_free 里第一个空闲槽：槽上若挂着 refcount 为 0 的旧 kctx，先 kbase\_mmu\_disable 驱逐它并把它的 as\_nr 置 INVALID，再 kbase\_mmu\_update 写 AS\_TRANSTAB / MEMATTR / TRANSCFG 并发 AS\_COMMAND UPDATE。
4. 一个空闲槽都没有：返回 INVALID，JM 调度器调 kbase\_backend\_find\_and\_release\_free\_address\_space 腾出一个槽后重试。

release 时 refcount 减到 0，只把槽标回 as\_free，as\_nr 保留且不写硬件（惰性复用）；fault 过的 kctx 会在这时彻底解绑。as\_nr 没被清掉，就说明这个槽没被别人改写过，这是复用成立的前提。

| 术语 | 说明 |
| --- | --- |
| refcount | kctx 自己的计数：驱动里有多少处正在用它的 AS 绑定。在 runpool 里 +1，每个已拉上槽未完成的 atom +1，刷 TLB、处理 fault 等临时操作 +1；大于 0 时槽不能被抢走 |
| as\_free | 空闲 AS 槽位图 |
| as\_to\_kctx\[\] | 每个槽当前挂着哪个 kctx |
| 惰性复用 | release 不清 as\_nr、不写硬件，下次没人动过就直接复用 |
| 驱逐场景 1 | A 已闲（refcount=0），B 取第一个空闲槽正好取到 A 的旧槽 |
| 驱逐场景 2 | 槽全满：调度器挑一个非特权、refcount 恰为 1（没有 atom 在跑）的 kctx，禁止提交并移出 runpool，腾出槽；正在跑的 kctx 不会被驱逐 |
| 特权 kctx | 带 KCTX\_PRIVILEGED 标志、可独占 AS 的上下文，腾槽时不会被挑中 |

### 第 15 页 · 电源管理与调频

前端用 active\_count 数谁要用 GPU，策略决定想要的状态，状态机写上下电寄存器、等中断确认。调频走 Linux devfreq。

| 术语 | 说明 |
| --- | --- |
| active\_count | 当前需要 GPU 的使用者数量，0→1 上电，1→0 可掉电 |
| coarse\_demand / always\_on | 两种电源策略：按需上电 / 始终上电 |
| POWER\_CHANGED\_ALL | 上下电完成中断 |
| 迟滞 | Shader 空闲后等 2 个 tick（每个 400µs）再关，避免反复上下电 |
| DVFS / devfreq / OPP | 动态调频调压 / Linux 调频框架 / 一组频率 + 电压组合 |
| IPA | 功耗模型，给温控提供 GPU 功耗估算 |

### 第 16 页 · 三路中断与 GPU 复位

中断里只读状态、清中断、排 work。作业超时、MMU 无响应等会触发复位：停提交 → 清现场 → 软复位 → 恢复页表 → 重新调度。

| 术语 | 说明 |
| --- | --- |
| workqueue | Linux 延后执行机制，可以睡眠、分配内存 |
| page fault / bus fault | 地址翻译失败（有时可补页）/ 物理访问总线出错（视为致命） |
| pf\_wq | 每个 AS 一个的 fault 处理 workqueue |
| soft-stop | 让作业在安全点停下，稍后继续 |

### 第 17 页 · 可观测性

硬件计数器只有一套，按四层流水线共享给多个消费者。日常排障主要看 debugfs、sysfs 和 trace。

| 术语 | 说明 |
| --- | --- |
| hwcnt | 硬件性能计数器子系统 |
| virtualizer | 让多个客户端共用一套计数器，启用集取并集 |
| kinstr\_prfcnt | 给用户态工具读计数器的接口（返回一个 fd） |
| Streamline / Perfetto | ARM 与 Android 的性能分析工具 |
| debugfs / sysfs | 内核调试文件系统（/sys/kernel/debug/mali0）/ 设备属性文件 |

## 第四部分 · 关键流程

### 第 18 页 · 设备与上下文的生命周期

probe 建设备、open 建 kfile、SET\_FLAGS 建 kctx、使用、close 拆除。两张初始化表正序初始化、逆序拆除；close 必须先停作业再拆内存。

| 术语 | 说明 |
| --- | --- |
| probe | 驱动与设备匹配后的初始化入口 |
| dev\_init\[\] / context\_init\[\] | 设备 / 上下文的初始化函数表，失败时逆序回滚 |
| 勘误（errata） | 硬件已知缺陷，驱动按 GPU\_ID 打开对应的绕过代码 |
| dummy job 固件 | JM 某些 GPU 的勘误：shader 上电后先跑一个哑 job |
| setup\_state | kfile 状态机，只有 COMPLETE 才能拿到 kctx |
| zap | 销毁 kctx 前清掉它所有在途作业 |

### 第 19 页 · 一次 ioctl 如何穿过各层

下行调用从用户态一路到寄存器；完成从中断回传，经 workqueue 唤醒用户。JM 的提交可能在同一条调用栈里写到硬件。

| 术语 | 说明 |
| --- | --- |
| owner 检查 | 禁止把 fd 交给别的进程使用 |
| copy\_from\_user / copy\_to\_user | 内核与用户态之间安全拷贝参数 |
| event\_wakeup | 唤醒等在 poll 上的用户线程 |

### 第 20 页 · MEM\_ALLOC

先建 region、从页池拿物理页、写页表。SAME\_VA 先返回 cookie，等用户 mmap 后才定 GPU VA。缺页时可增长的 region 会补页重放，否则杀作业。

| 术语 | 说明 |
| --- | --- |
| va\_pages / commit\_pages | 预留的 GPU VA 页数 / 立刻分配的物理页数 |
| cookie | SAME\_VA 下的临时凭证，用户 mmap 它时才真正映射 |
| kbase\_gpu\_mmap | 选定 GPU VA 并写页表 |
| 惰性映射 | CPU 第一次访问时才填 CPU 页表 |
| PF\_GROW | region 标志：允许缺页时增长 |
| UNLOCK 重放 | 补页后解锁 MMU，硬件重新执行刚才失败的访问 |

#### 补充 · commit\_pages 与 GPU 缺页、GROW\_ON\_GPF（概念补充，未对照源码）

MEM\_ALLOC 有两个页数：va\_pages 是 GPU VA 范围（region 的 nr\_pages），commit\_pages 是一开始真正分配并写进页表的物理页数，commit\_pages ≤ va\_pages。

- **普通内存**：commit\_pages == va\_pages，分配时页表写满，GPU 访问不会缺页。
- **可增长内存（GROW\_ON\_GPF）**：commit\_pages 只给一小部分，va\_pages 是上限。GPU 访问到未映射页时 MMU 报 page fault，kbase 在缺页处理里检查 region 是否带 GROW\_ON\_GPF，带就在上限内补页、写页表，再让 GPU 重试；不带就是真错误（越界、悬空指针），只报 fault，不分配。

所以缺页时补多少页由 kbase 的增长策略决定（步长按 region 的 extension 之类字段，以源码为准），不由 commit\_pages 直接带来；commit\_pages 决定初始大小，va\_pages 决定增长上限，是间接关系。

**GROW\_ON\_GPF 的场景**：GPU 自己越写越多、用户态事先算不准的内存，典型是 tiler heap（多边形列表大小取决于场景复杂度）。全部预先 commit 会浪费内存，重新分配会改变地址而 job 已在 GPU 上跑；GROW\_ON\_GPF 让 VA 固定、物理页按需补。限制：只能增长到 va\_pages；补页发生在缺页处理里，GPU 要等待，不适合对延迟敏感的数据；内存紧张时可能补页失败。CSF 下 tiler heap 的增长改由固件发请求、kbase 响应，路径与 JM 不同。

#### 补充 · GROW\_ON\_GPF 增长的页是否计入 used\_pages（概念补充，未对照源码）

**会计入。** used\_pages 记在底层「取物理页」的函数里（kbase\_alloc\_phy\_pages\_helper 一类），不按分配入口记。MEM\_ALLOC 一开始 commit 的页和缺页后增长补的页都经过它，取页时计数一并加上。

缺页增长路径：GPU 缺页 → MMU 中断、page fault worker → 确认 region 带 GROW\_ON\_GPF、算要补多少页 → 从 kctx 页池 / kbdev 页池 / 内核取页，used\_pages 同时加上 → 写页表、让 GPU 重试。

- used\_pages 通常有两份：kctx 级（这个进程用了多少）和设备级（kbdev->memdev，整个驱动用了多少），两处都加，释放时对应减。
- gpu\_memory debugfs 读的就是 used\_pages，所以 tiler heap 这类增长的内存会让它的数字变大。
- 增长改变的是已提交物理页数，GPU VA 范围（va\_pages）不变。
- 增长后一般不自动缩回，除非驱动有显式收缩逻辑，否则一直计着直到 region 释放，这是 gpu\_memory 里 tiler heap 数字可能居高不下的一个原因（是否有收缩机制以源码为准）。

### 第 21 页 · JM 提交全链路

一个 atom 的往返：解依赖 → 调度 → 上槽 → 写寄存器；完成时中断 → 出槽 → job\_done\_wq 收尾 → 用户 read。后三页逐段展开。

| 术语 | 说明 |
| --- | --- |
| pullable 队列 | 有 atom 可拉取的 kctx 队列，按 slot 和优先级分开 |
| job\_done\_wq | 每个 kctx 的完成处理 workqueue |
| base\_jd\_event\_v2 | 用户 read 到的完成事件结构 |

### 第 22 页 · kbase\_js\_sched

逐个 slot 弹出 kctx：首次活跃要电，再拿 AS，最后把 atom 踢进 slot\_rb。拿不到资源就放回队列换下一个。

| 术语 | 说明 |
| --- | --- |
| schedule\_sem | 保证同一时刻只有一个调度者 |
| KCTX\_ACTIVE | kctx 标志：已经为它要过电 |
| active\_kctx\[js\] | 每个 slot 当前服务的 kctx |
| 锁顺序 | 外到内：schedule\_sem → queue\_mutex → jsctx\_mutex → runpool\_mutex → mmu\_hw\_mutex → hwaccess\_lock |

#### 补充 · kbase\_jm\_kick 与 kbase\_js\_pull，以及 ④ 的两层结构（概念补充，未对照源码）

**kbase\_jm\_kick 是入口，kbase\_js\_pull 是它内部取 atom 的一步。**

|  | kbase\_jm\_kick | kbase\_js\_pull |
| --- | --- | --- |
| 层次 | JM，连接调度和硬件后端 | JS，调度队列 |
| 粒度 | 一组 slot（js\_mask） | 一个 kctx 的一个 slot |
| 做什么 | 取 atom 并提交到硬件 | 只取 atom，不碰硬件、不写寄存器 |
| 关系 | 循环调用 js\_pull，再调 kbase\_backend\_run\_atom | 被 jm\_kick 调用 |

js\_pull 会检查依赖、kctx 状态，取不到返回 NULL；jm\_kick 对每个 slot 找活动 kctx，循环 pull + run\_atom 直到槽满或取不到。新 atom 入队、atom 完成腾出 slot、kctx 被调度上来时都会 kick，避免 GPU 空闲。

**PPT 里 ④ 与「展开 ④」不矛盾，是同一步的两层。** 左边 ④ 是 kbase\_js\_sched 直接调用的 kbase\_js\_use\_ctx：kctx 已调度且后端允许就直接用，否则走 kbasep\_js\_schedule\_ctx。右边展开框列的是 kbasep\_js\_schedule\_ctx 内部的步骤（kbase\_ctx\_sched\_retain\_ctx → 失败则 find\_and\_release\_free\_address\_space 再 retain → kbase\_backend\_use\_ctx → active\_kctx\[js\] = kctx），所以标题写了真正干活的函数名。层次是 kbase\_js\_sched → kbase\_js\_use\_ctx → kbasep\_js\_schedule\_ctx → 上述步骤。读者容易把「展开 ④」当成展开左边写的 use\_ctx，标题可改为「展开 ④：use\_ctx → kbasep\_js\_schedule\_ctx」（PPT 尚未修改）。

### 第 23 页 · slot\_rb 状态机与寄存器写入

atom 进 slot\_rb 后要依次满足依赖、保护模式、核已上电等条件才到 READY，然后按顺序写 5 个 NEXT 寄存器，最后写 START。

| 术语 | 说明 |
| --- | --- |
| gpu\_rb\_state | atom 在 slot\_rb 里的状态 |
| protected mode | 保护模式，用于 DRM 加密视频等安全内容 |
| affinity | 允许这个 job 使用的 shader 核掩码 |
| flush reduction | 硬件特性：省掉不必要的 cache flush |

### 第 24 页 · 完成、软停与超时

中断里只做出槽与分类：DONE 正常收尾，STOPPED 重新排队，失败则级联。定时器每 100ms 检查一次，分别在 1、50、55 个 tick 时软停、硬停、复位。

| 术语 | 说明 |
| --- | --- |
| JS\_STATUS / 完成码 | 失败槽的状态寄存器，给出 DONE、STOPPED、TERMINATED 等 |
| evict | 把 HEAD\_NEXT 里还没开始的 atom 从硬件收回 |
| hard-stop | 强行终止作业 |
| tick | 调度定时器的一个周期，默认 100ms |

### 第 25 页 · 对照：CSF 的 CS\_QUEUE\_KICK

CSF 的 kick 只是声明有活，由调度线程异步上槽；之后用户直接敲门铃，内核不在环上。用来反衬 JM 的同步提交。

| 术语 | 说明 |
| --- | --- |
| gpuq-kthread | CSF 的单一调度内核线程 |
| tick / tock | 定时全量重排（约 100ms）/ 事件触发的局部重排 |
| CSG\_REQ | 驱动向固件发组级请求的共享内存字段 |
| ring buffer | 用户态写命令的环形缓冲 |

## 第五部分 · 案例与总结

### 第 26 页 · gpu\_memory 约为 profile 的 2 倍

多半不是泄漏：Infinite Cache 下 CPU 和 GPU 各有一套物理页，gpu\_memory 两份都记。关掉 Infinite Cache 复跑，比值回到约 1 就是记账口径问题。左侧柱子是示意比例，不是实测数据。

| 术语 | 说明 |
| --- | --- |
| gpu\_memory | debugfs 节点，读的是 used\_pages（native 物理页 + 页表页） |
| mem\_profile / profile | 用户态按缓冲逻辑大小上报的内存统计 |
| used\_pages | 驱动对已使用物理页的记账 |
| Infinite Cache | kctx 特性：CPU 可缓存的缓冲给 CPU、GPU 各分配一套物理页 |
| cpu\_alloc / gpu\_alloc | region 的 CPU 侧 / GPU 侧物理页集合，通常是同一个对象 |
| monkey | Android 随机压力测试工具 |

### 第 27 页 · 总结

三句话：kbase 只做四件事；JM 主线是 jd → js → jm\_rb → jm\_hw，完成路径反过来；AS 管翻译、Job Slot 管执行，靠 as\_nr 绑定。读源码可从 kbase\_ioctl、kbase\_jd\_submit、kbase\_js\_sched、kbase\_backend\_slot\_update、kbase\_job\_done 这五个函数入手。

## 补充笔记 · JM 与 CSF 的提交方式和完成通知

两边 GPU 执行、完成通知都是异步的；真正的区别在“谁的线程把活交到硬件”和“完成结果放在哪”。用“同步 / 异步”形容容易误解，更准确的说法是“内联提交”和“延后提交”。

### 提交：内联 vs 延后

| 提交什么 | JM | CSF |
| --- | --- | --- |
| GPU 工作，经内核 | 内联：条件满足时，JOB\_SUBMIT 的 ioctl 线程自己走到 kbase\_job\_hw\_submit 写 JS 寄存器；不满足则由完成中断、job\_done\_wq 或定时器再推 | 延后：CS\_QUEUE\_KICK 只把队列挂进 pending 列表、唤醒 gpuq-kthread 就返回，条件满足也不写硬件 |
| GPU 工作，稳态 | 每个 atom 都要经内核 | 组已在槽上时，用户线程直接敲 mmap 来的门铃页，不经内核 |
| CPU 侧命令（fence 等） | soft job：条件满足时在提交线程里内联执行 | KCPU 队列：条件满足时在 enqueue 线程里内联执行 |

### 完成通知：内核事件 vs 内存同步对象

|  | JM | CSF |
| --- | --- | --- |
| 结果放在哪 | 内核事件队列 event\_list | GPU 内存里的同步对象（CQS / sync object） |
| 通知链 | JOB 中断 → kbase\_job\_done → kbase\_jd\_done → job\_done\_wq → kbase\_event\_post → 唤醒 poll | GPU 写同步对象；用户可直接轮询。要睡眠时经 CSG SYNC\_UPDATE 中断 → kbase\_csf\_event\_signal\_cpu\_only 唤醒 poll |
| read() 得到 | base\_jd\_event\_v2：哪个 atom、成功还是失败；阻塞式 | 只有“有变化，自己去查”或错误通知；非阻塞 |
| 过滤 / 粒度 | 每个 atom 一条，可用 ONLY\_ON\_FAILURE / NEVER / COALESCE 过滤或合并 | 由用户定义，一个同步对象可代表任意多的工作 |

### 跨进程 fence：同一机制，挂接对象不同

两边都是“GPU 完成 → 内核在 CPU 侧 signal dma\_fence → 拿着 sync\_file 的进程（如 SurfaceFlinger）被唤醒”；用户线程都不在 fence 上阻塞。

|  | 条件已满足 | 条件未满足 |
| --- | --- | --- |
| JM：FENCE\_TRIGGER soft job | 在 JOB\_SUBMIT 线程里由 jd\_submit\_atom 当场 signal | 挂在前驱的 dep\_head 上；前驱完成后在 job\_done\_wq 里由 jd\_run\_atom 执行 |
| CSF：KCPU FENCE\_SIGNAL | 在 KCPU\_QUEUE\_ENQUEUE 线程里当场执行 | 停在队列里；同步对象更新后在 kcpu\_wq（高优先级进程用 gpuq-kthread）里执行 |
