# GraphicBuffer 和 Gralloc 架构分析

## 一、核心组件关系

```
┌─────────────────────────────────────────────────────────────┐
│                    GraphicBuffer                             │
│  (高级封装，提供完整的缓冲区生命周期管理)                      │
└──────────────┬──────────────────────────────────────────────┘
               │ 使用
               ├──────────────────────────────────────────────┐
               │                                              │
┌──────────────▼──────────────┐    ┌───────────────────────▼──────────────┐
│  GraphicBufferAllocator     │    │   GraphicBufferMapper                 │
│  (缓冲区分配器)              │    │   (缓冲区映射器)                       │
└──────────────┬───────────────┘    └──────────────┬──────────────────────┘
               │                                    │
               │ 使用                                │ 使用
               │                                    │
┌──────────────▼──────────────┐    ┌──────────────▼──────────────────────┐
│   GrallocAllocator          │    │   GrallocMapper                      │
│   (抽象分配接口)             │    │   (抽象映射接口)                      │
└──────────────┬──────────────┘    └──────────────┬──────────────────────┘
               │                                    │
               │ 实现                                │ 实现
               │                                    │
    ┌──────────┼──────────┐            ┌──────────┼──────────┐
    │          │          │            │          │          │
┌───▼───┐ ┌───▼───┐ ┌───▼───┐    ┌───▼───┐ ┌───▼───┐ ┌───▼───┐
│Gralloc│ │Gralloc│ │Gralloc│    │Gralloc│ │Gralloc│ │Gralloc│
│  2    │ │  3    │ │  4/5  │    │  2    │ │  3    │ │  4/5  │
└───────┘ └───────┘ └───────┘    └───────┘ └───────┘ └───────┘
    │          │          │            │          │          │
    └──────────┴──────────┴────────────┴──────────┴──────────┘
                          │
                          ▼
              ┌────────────────────────┐
              │   Gralloc HAL          │
              │   (硬件抽象层)          │
              └────────────────────────┘
```

## 二、核心数据结构

### 2.1 GraphicBuffer 核心数据

```cpp
// GraphicBuffer 的继承关系
class GraphicBuffer
    : public ANativeObjectBase<ANativeWindowBuffer, GraphicBuffer, RefBase>,
      public Flattenable<GraphicBuffer>
{
    // ANativeObjectBase 通过多重继承同时继承自：
    // 1. ANativeWindowBuffer (提供 C 接口兼容性)
    // 2. RefBase (提供引用计数功能)
    
    // 继承自 ANativeWindowBuffer 的字段（通过 ANativeObjectBase）
    int width;              // 缓冲区宽度
    int height;             // 缓冲区高度
    int stride;             // 行步长（每行的像素数，不是字节数）
    int format;             // 像素格式 (PixelFormat)
    uint64_t usage;         // 使用标志位
    uint32_t layerCount;    // 图层数量
    buffer_handle_t handle;  // 底层句柄
    
    // GraphicBuffer 特有字段
    GraphicBufferMapper& mBufferMapper;  // 映射器引用
    uint8_t mOwner;                     // 所有权：ownNone/ownHandle/ownData
    status_t mInitCheck;                // 初始化状态
    uint64_t mId;                       // 唯一ID
    uint32_t mGenerationNumber;         // 生成号
    uint32_t mTransportNumFds;          // 传输时需要的文件描述符数量
    uint32_t mTransportNumInts;        // 传输时需要的整数数量
};
```

**ANativeObjectBase 的作用**：
- 将 C 风格的 `ANativeWindowBuffer` 结构体转换为 C++ 引用计数对象
- 通过多重继承同时提供 `ANativeWindowBuffer` 接口和 `RefBase` 引用计数
- 桥接 C 接口（`ANativeWindowBuffer*`）和 C++ 智能指针（`sp<GraphicBuffer>`）
```

### 2.2 GraphicBufferMapper 核心数据

```cpp
class GraphicBufferMapper {
    enum Version {
        GRALLOC_2 = 2,
        GRALLOC_3,
        GRALLOC_4,
        GRALLOC_5,
    };
    
    struct LockResult {
        void* address;           // 映射后的虚拟地址
        int32_t bytesPerPixel;   // 每个像素的字节数（如 RGBA_8888 是 4 字节）
        int32_t bytesPerStride;  // 每行（stride：每行像素数）的总字节数 = stride × bytesPerPixel
    };
    
private:
    std::unique_ptr<const GrallocMapper> mMapper;  // 底层 Gralloc 实现
    Version mMapperVersion;                        // 当前使用的版本
};
```

**注意**：`bytesPerPixel` 和 `bytesPerStride` 的区别
- `bytesPerPixel`: 单个像素的大小（字节/像素）
- `bytesPerStride`: 每行的总大小（字节/行），包含填充区域
- 关系：`bytesPerStride = stride（像素数）× bytesPerPixel`

### 2.3 GraphicBufferAllocator 核心数据

```cpp
class GraphicBufferAllocator {
    struct AllocationRequest {
        bool importBuffer;           // 是否导入缓冲区
        uint32_t width;              // 宽度
        uint32_t height;             // 高度
        PixelFormat format;          // 像素格式
        uint32_t layerCount;         // 图层数
        uint64_t usage;             // 使用标志
        std::string requestorName;   // 请求者名称
        std::vector<AdditionalOptions> extras;  // 额外选项
    };
    
    struct AllocationResult {
        status_t status;             // 状态码
        buffer_handle_t handle;      // 分配的句柄
        uint32_t stride;             // 行步长
    };
    
    struct alloc_rec_t {
        uint32_t width;
        uint32_t height;
        uint32_t stride;
        PixelFormat format;
        uint32_t layerCount;
        uint64_t usage;
        size_t size;
        std::string requestorName;
    };
    
private:
    GraphicBufferMapper& mMapper;              // 映射器引用
    std::unique_ptr<const GrallocAllocator> mAllocator;  // 底层分配器
    static KeyedVector<buffer_handle_t, alloc_rec_t> sAllocList;  // 分配记录
};
```

### 2.4 GrallocMapper 抽象接口

```cpp
class GrallocMapper {
    // 核心操作
    virtual status_t importBuffer(const native_handle_t* rawHandle, 
                                  buffer_handle_t* outBufferHandle) = 0;
    virtual void freeBuffer(buffer_handle_t bufferHandle) = 0;
    
    // 锁定/解锁
    virtual status_t lock(buffer_handle_t bufferHandle, uint64_t usage, 
                          const Rect& bounds, int acquireFence, 
                          void** outData, int32_t* outBytesPerPixel,
                          int32_t* outBytesPerStride) = 0;
    virtual int unlock(buffer_handle_t bufferHandle) = 0;
    
    // 元数据查询（gralloc4+）
    virtual status_t getWidth(buffer_handle_t bufferHandle, uint64_t* outWidth) = 0;
    virtual status_t getHeight(buffer_handle_t bufferHandle, uint64_t* outHeight) = 0;
    virtual status_t getUsage(buffer_handle_t bufferHandle, uint64_t* outUsage) = 0;
    virtual status_t getPlaneLayouts(buffer_handle_t bufferHandle,
                                     std::vector<ui::PlaneLayout>* outPlaneLayouts) = 0;
    // ... 更多元数据接口
};
```

### 2.5 GrallocAllocator 抽象接口

```cpp
class GrallocAllocator {
    // 分配缓冲区
    virtual status_t allocate(std::string requestorName, uint32_t width, 
                              uint32_t height, PixelFormat format, 
                              uint32_t layerCount, uint64_t usage,
                              uint32_t* outStride, buffer_handle_t* outBufferHandles,
                              bool importBuffers = true) = 0;
    
    virtual GraphicBufferAllocator::AllocationResult allocate(
            const GraphicBufferAllocator::AllocationRequest&) = 0;
};
```

## 三、主要接口

### 3.1 GraphicBuffer 主要接口

#### 构造和初始化
```cpp
// 创建空缓冲区（用于反序列化）
GraphicBuffer();

// 分配新缓冲区
GraphicBuffer(uint32_t width, uint32_t height, PixelFormat format,
              uint32_t layerCount, uint64_t usage, std::string requestorName);

// 从已有句柄创建
GraphicBuffer(const native_handle_t* inHandle, HandleWrapMethod method,
              uint32_t width, uint32_t height, PixelFormat format,
              uint32_t layerCount, uint64_t usage, uint32_t stride);
```

#### 缓冲区操作
```cpp
// 锁定缓冲区获取 CPU 访问
status_t lock(uint32_t usage, void** vaddr, 
              int32_t* outBytesPerPixel = nullptr,
              int32_t* outBytesPerStride = nullptr);
status_t lock(uint32_t usage, const Rect& rect, void** vaddr, ...);

// 锁定 YCbCr 格式
status_t lockYCbCr(uint32_t usage, android_ycbcr* ycbcr);

// 异步锁定（带 fence）
status_t lockAsync(uint32_t usage, const Rect& rect, void** vaddr, int fenceFd, ...);

// 解锁缓冲区
status_t unlock();
status_t unlockAsync(int* fenceFd);

// 重新分配缓冲区
status_t reallocate(uint32_t width, uint32_t height, PixelFormat format,
                   uint32_t layerCount, uint64_t usage);
```

#### 查询接口
```cpp
uint32_t getWidth() const;
uint32_t getHeight() const;
uint32_t getStride() const;
uint64_t getUsage() const;
PixelFormat getPixelFormat() const;
uint32_t getLayerCount() const;
```

#### 序列化接口
```cpp
size_t getFlattenedSize() const;
size_t getFdCount() const;
status_t flatten(void*& buffer, size_t& size, int*& fds, size_t& count) const;
status_t unflatten(void const*& buffer, size_t& size, 
                   int const*& fds, size_t& count);
```

### 3.2 GraphicBufferMapper 主要接口

```cpp
// 导入缓冲区（从其他进程或 HAL）
status_t importBuffer(const native_handle_t* rawHandle, uint32_t width,
                     uint32_t height, uint32_t layerCount, PixelFormat format,
                     uint64_t usage, uint32_t stride, buffer_handle_t* outHandle);

// 释放缓冲区
status_t freeBuffer(buffer_handle_t handle);

// 锁定缓冲区
ui::Result<LockResult> lock(buffer_handle_t handle, int64_t usage, 
                            const Rect& bounds, base::unique_fd&& acquireFence = {});

// 解锁缓冲区
status_t unlock(buffer_handle_t handle, base::unique_fd* outFence = nullptr);

// 查询缓冲区是否支持
status_t isSupported(uint32_t width, uint32_t height, PixelFormat format,
                     uint32_t layerCount, uint64_t usage, bool* outSupported);

// 元数据查询（gralloc4+）
status_t getWidth(buffer_handle_t bufferHandle, uint64_t* outWidth);
status_t getHeight(buffer_handle_t bufferHandle, uint64_t* outHeight);
status_t getUsage(buffer_handle_t bufferHandle, uint64_t* outUsage);
status_t getPlaneLayouts(buffer_handle_t bufferHandle,
                        std::vector<ui::PlaneLayout>* outPlaneLayouts);
```

### 3.3 GraphicBufferAllocator 主要接口

```cpp
// 分配缓冲区（新接口）
AllocationResult allocate(const AllocationRequest& request);

// 分配缓冲区（旧接口）
status_t allocate(uint32_t width, uint32_t height, PixelFormat format,
                  uint32_t layerCount, uint64_t usage,
                  buffer_handle_t* handle, uint32_t* stride,
                  std::string requestorName);

// 分配原始句柄（不导入）
status_t allocateRawHandle(uint32_t width, uint32_t height, PixelFormat format,
                          uint32_t layerCount, uint64_t usage,
                          buffer_handle_t* handle, uint32_t* stride,
                          std::string requestorName);

// 释放缓冲区
status_t free(buffer_handle_t handle);

// 查询总分配大小
uint64_t getTotalSize() const;
```

## 四、主要流程

### 4.1 缓冲区分配流程

```
应用层请求分配缓冲区
    │
    ▼
GraphicBuffer::GraphicBuffer(width, height, format, layerCount, usage)
    │
    ▼
GraphicBuffer::initWithSize()
    │
    ▼
GraphicBufferAllocator::allocate()
    │
    ├─► 验证参数（宽度、高度、格式等）
    │
    ├─► 根据 Mapper 版本选择对应的 Allocator
    │   └─► Gralloc5Allocator / Gralloc4Allocator / ...
    │
    ├─► GrallocAllocator::allocate()
    │   │
    │   ├─► 调用 HAL IAllocator::allocate()
    │   │   └─► 硬件分配器分配物理内存
    │   │
    │   └─► 返回 native_handle_t（原始句柄）
    │
    ├─► 如果需要导入（importBuffer = true，默认）
    │   └─► GraphicBufferMapper::importBuffer()
    │       │
    │       ├─► 调用 GrallocMapper::importBuffer()
    │       │   └─► HAL IMapper::importBuffer()
    │       │       ├─► 验证句柄有效性
    │       │       ├─► 在 HAL 中注册缓冲区
    │       │       ├─► 映射文件描述符到当前进程
    │       │       └─► 创建进程内可用的 buffer_handle_t
    │       │
    │       └─► 验证缓冲区大小（validateBufferSize）
    │           └─► 确保缓冲区参数匹配
    │
    │   注意：importBuffer 将 native_handle_t（原始句柄）转换为
    │         buffer_handle_t（进程内句柄），使缓冲区可以在当前进程使用
    │
    ├─► 记录分配信息到 sAllocList
    │
    └─► 返回 AllocationResult
        │
        ▼
GraphicBuffer 保存 handle 和元数据
```

### 4.2 缓冲区锁定流程

```
应用层调用 GraphicBuffer::lock()
    │
    ▼
GraphicBuffer::lockAsync()
    │
    ├─► 根据 Gralloc 版本解析 bytesPerPixel/bytesPerStride
    │   ├─► Gralloc2: 从 format 计算
    │   ├─► Gralloc3: 从 lock() 返回值获取
    │   └─► Gralloc4+: 从 PlaneLayout 元数据获取
    │
    ▼
GraphicBufferMapper::lock()
    │
    ▼
GrallocMapper::lock()
    │
    ├─► 根据版本调用对应的实现
    │   ├─► Gralloc5Mapper::lock()
    │   │   └─► AIMapper::lock() [AIDL]
    │   │
    │   ├─► Gralloc4Mapper::lock()
    │   │   └─► IMapper::lock() [HIDL/AIDL]
    │   │
    │   └─► Gralloc2Mapper::lock()
    │       └─► IMapper::lock() [HIDL]
    │
    ├─► HAL 层执行锁定操作
    │   ├─► 验证 usage 标志
    │   ├─► 等待 acquireFence（如果提供）
    │   ├─► 映射物理内存到虚拟地址空间
    │   └─► 返回虚拟地址
    │
    └─► 返回 LockResult（包含地址和步长信息）
        │
        ▼
应用层获得可访问的虚拟地址
```

### 4.3 缓冲区解锁流程

```
应用层调用 GraphicBuffer::unlock()
    │
    ▼
GraphicBufferMapper::unlock()
    │
    ▼
GrallocMapper::unlock()
    │
    ├─► 根据版本调用对应的实现
    │   └─► HAL IMapper::unlock()
    │
    ├─► HAL 层执行解锁操作
    │   ├─► 取消内存映射
    │   ├─► 同步缓存（如果需要）
    │   └─► 返回 releaseFence（同步栅栏）
    │
    └─► 返回状态和 fence
```

### 4.4 缓冲区导入流程（跨进程）

```
进程 A 创建 GraphicBuffer
    │
    ├─► 分配缓冲区，获得 buffer_handle_t
    │
    └─► 序列化（flatten）
        ├─► 写入元数据（width, height, format, usage等）
        ├─► 写入 handle 的 fds 和 ints
        └─► 通过 Binder 传输到进程 B
            │
            ▼
进程 B 接收数据
    │
    ├─► 反序列化（unflatten）
    │   ├─► 读取元数据
    │   └─► 重建 native_handle_t
    │
    └─► GraphicBuffer::initWithHandle()
        │
        └─► GraphicBufferMapper::importBuffer()
            │
            ├─► 验证缓冲区参数
            │
            └─► GrallocMapper::importBuffer()
                │
                └─► HAL IMapper::importBuffer()
                    │
                    └─► 创建进程 B 内的新 buffer_handle_t
                        （引用相同的物理内存）
```

### 4.5 版本选择流程

```
系统启动 / GraphicBufferMapper 初始化
    │
    ▼
GraphicBufferMapper::GraphicBufferMapper()
    │
    ├─► 尝试加载 Gralloc5Mapper
    │   └─► Gralloc5Mapper::isLoaded()
    │       └─► 检查 AIDL IAllocator 服务是否存在
    │           └─► 成功 → 使用 GRALLOC_5
    │
    ├─► 失败则尝试 Gralloc4Mapper
    │   └─► Gralloc4Mapper::isLoaded()
    │       └─► 检查 HIDL/AIDL 服务是否存在
    │           └─► 成功 → 使用 GRALLOC_4
    │
    ├─► 失败则尝试 Gralloc3Mapper
    │   └─► Gralloc3Mapper::isLoaded()
    │       └─► 检查 HIDL 服务是否存在
    │           └─► 成功 → 使用 GRALLOC_3
    │
    └─► 失败则尝试 Gralloc2Mapper
        └─► Gralloc2Mapper::isLoaded()
            └─► 检查 HIDL 服务是否存在
                └─► 成功 → 使用 GRALLOC_2
                └─► 失败 → LOG_ALWAYS_FATAL
```

## 五、版本差异

### 5.1 Gralloc 版本特性对比

| 特性 | Gralloc2 | Gralloc3 | Gralloc4 | Gralloc5 |
|------|---------|----------|----------|----------|
| **接口技术** | HIDL | HIDL | HIDL/AIDL混合 | AIDL |
| **isSupported()** | ❌ | ✅ | ✅ | ✅ |
| **元数据系统** | ❌ | ❌ | ✅ 扩展元数据 | ✅ 扩展元数据 |
| **bytesPerPixel** | 需计算 | ✅ lock返回 | ✅ PlaneLayout | ✅ PlaneLayout |
| **PlaneLayout** | ❌ | ❌ | ✅ | ✅ |
| **Android版本** | 8.0+ | 9.0+ | 10+ | 12+ |

### 5.2 关键版本差异实现

#### bytesPerPixel/bytesPerStride 获取方式

**Gralloc2** (需手动计算):
```cpp
// GraphicBuffer.cpp:377-383
if (mapperVersion == GraphicBufferMapper::GRALLOC_2) {
    legacyBpp = bytesPerPixel(format);  // 从格式计算
    if (legacyBpp > 0) {
        legacyBps = stride * legacyBpp;
    }
}
```

**Gralloc3** (从 lock 返回值获取):
```cpp
// lock() 直接返回 bytesPerPixel 和 bytesPerStride
LockResult result = mapper->lock(...);
// result.bytesPerPixel 和 result.bytesPerStride 已填充
```

**Gralloc4+** (从 PlaneLayout 元数据获取):
```cpp
// GraphicBuffer.cpp:384-388
if (mapperVersion >= GraphicBufferMapper::GRALLOC_4) {
    auto planeLayout = getBufferMapper().getPlaneLayouts(handle);
    resolveLegacyByteLayoutFromPlaneLayout(planeLayout.value(), 
                                          &legacyBpp, &legacyBps);
}
```

## 六、关键设计模式

### 6.1 单例模式
- `GraphicBufferMapper`: 全局唯一实例
- `GraphicBufferAllocator`: 全局唯一实例

### 6.2 策略模式
- 不同 Gralloc 版本作为不同的策略实现
- 运行时根据硬件支持选择策略

### 6.3 适配器模式
- `GraphicBufferMapper` 适配不同的 `GrallocMapper` 实现
- `GraphicBufferAllocator` 适配不同的 `GrallocAllocator` 实现

### 6.4 桥接模式（ANativeObjectBase）
- **目的**: 将 C 风格的 `ANativeWindowBuffer` 结构体桥接到 C++ 对象系统
- **实现**: 通过多重继承同时继承 `ANativeWindowBuffer` 和 `RefBase`
- **作用**:
  - 允许 C 代码通过 `ANativeWindowBuffer*` 指针访问
  - 允许 C++ 代码通过 `sp<GraphicBuffer>` 智能指针管理生命周期
  - 统一引用计数：C 接口的 `incRef/decRef` 调用转发到 C++ 的 `RefBase`
- **类型转换**:
  ```cpp
  // C++ 智能指针 → C 指针
  sp<GraphicBuffer> buffer = ...;
  ANativeWindowBuffer* anb = buffer.get();  // 直接转换
  
  // C 指针 → C++ 智能指针
  ANativeWindowBuffer* anb = ...;
  GraphicBuffer* gb = GraphicBuffer::from(anb);
  sp<GraphicBuffer> buffer = gb;  // 通过 RefBase 管理
  ```

### 6.5 所有权管理
```cpp
enum {
    ownNone   = 0,  // 不拥有，仅包装
    ownHandle = 1,  // 拥有句柄（从其他进程导入）
    ownData   = 2,  // 拥有数据（自己分配）
};
```

## 七、使用示例

### 7.1 分配和使用缓冲区

```cpp
// 1. 分配缓冲区
sp<GraphicBuffer> buffer = new GraphicBuffer(
    1920, 1080, HAL_PIXEL_FORMAT_RGBA_8888, 
    1, GRALLOC_USAGE_SW_WRITE_OFTEN, "MyApp");

// 2. 检查分配是否成功
if (buffer->initCheck() != NO_ERROR) {
    // 处理错误
    return;
}

// 3. 锁定缓冲区进行写入
void* vaddr;
status_t err = buffer->lock(GRALLOC_USAGE_SW_WRITE_OFTEN, &vaddr);
if (err == NO_ERROR) {
    // 4. 写入数据
    // ... 操作 vaddr 指向的内存 ...
    
    // 5. 解锁缓冲区
    buffer->unlock();
}
```

### 7.2 跨进程传递缓冲区

```cpp
// 进程 A: 序列化
Parcel parcel;
buffer->flatten(parcel);

// 通过 Binder 发送 parcel

// 进程 B: 反序列化
sp<GraphicBuffer> buffer = new GraphicBuffer();
buffer->unflatten(parcel);
// buffer 现在可以在这个进程中使用
```

## 八、总结

### 8.1 核心职责划分

- **GraphicBuffer**: 高级封装，提供完整的缓冲区生命周期管理
- **GraphicBufferAllocator**: 负责缓冲区的分配和释放
- **GraphicBufferMapper**: 负责缓冲区的映射、锁定、解锁和元数据查询
- **GrallocMapper/Allocator**: 抽象接口，适配不同版本的 HAL

### 8.2 设计优势

1. **版本兼容性**: 通过策略模式支持多个 Gralloc 版本
2. **接口统一**: 上层应用无需关心底层 HAL 版本
3. **生命周期管理**: 清晰的缓冲区所有权管理
4. **跨进程支持**: 通过序列化/反序列化支持跨进程传递

### 8.3 关键流程

1. **分配**: Allocator → HAL → 物理内存分配 → 导入到进程
2. **锁定**: Mapper → HAL → 内存映射 → 返回虚拟地址
3. **解锁**: Mapper → HAL → 取消映射 → 同步缓存
4. **传递**: 序列化 → Binder → 反序列化 → 导入到目标进程

