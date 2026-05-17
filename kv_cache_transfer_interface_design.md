# KV Cache 传输接口设计文档

## 一、现有接口分析

### 1.1 接口概览

根据代码分析，昇腾NPU后端KV cache传输涉及三个核心接口：

| 接口名称 | 功能描述 | 数据方向 | 适用场景 |
|---------|---------|---------|---------|
| `transfer_kv_dim_exchange` | KV数据批量复制 | H2D / D2H | 全量加载/卸载 |
| `load_cache_to_device_buffer_mla` | MLA稀疏缓存加载 | H2D | 稀疏加载（通用布局） |
| `load_cache_to_device_buffer_dsv4_mla` | DSv4稀疏缓存加载 | H2D | 稀疏加载（DSv4布局） |

### 1.2 `transfer_kv_dim_exchange` 接口分析

#### 1.2.1 当前接口定义（Python层）

```python
def transfer_kv_dim_exchange(
    device_indices: torch.Tensor,      # device端token索引
    host_indices: torch.Tensor,        # host端token索引
    device_k: torch.Tensor,            # device端K buffer
    host_k: torch.Tensor,              # host端K buffer
    device_v: torch.Tensor,            # device端V buffer
    host_v: torch.Tensor,              # host端V buffer
    device_index_k: Optional[torch.Tensor] = None,  # device端index_k buffer（可选）
    host_index_k: Optional[torch.Tensor] = None,    # host端index_k buffer（可选）
    page_size: int = 128,              # 页大小
    direction: TransferDirection = TransferDirection.H2D,  # 传输方向
    flags: TransferFlag = TransferFlag.FAST2D,       # 传输标志
):
```

#### 1.2.2 当前接口定义（C++层）

```cpp
void transfer_kv_dim_exchange(
    at::Tensor &device_k,      // device端K buffer
    at::Tensor &host_k,        // host端K buffer  
    at::Tensor &device_v,      // device端V buffer
    at::Tensor &host_v,        // host端V buffer
    const at::Tensor &device_indices,  // device索引
    const at::Tensor &host_indices,    // host索引
    int64_t page_size,         // 页大小
    int64_t direction,         // 传输方向 (1=H2D, 2=D2H)
    int64_t flags              // 传输标志
);
```

#### 1.2.3 上下文用法

该接口在 `memory_pool_host.py` 中被以下类调用：

**MHATokenToKVPoolHost**（MHA架构）:
- `load_to_device_per_layer()` - layer_id == 0 时调用，H2D方向
- `backup_from_device_all_layer()` - D2H方向
- 支持布局：`page_first_direct`

**MLATokenToKVPoolHost**（MLA架构）:
- `load_to_device_per_layer()` - layer_id == 0 时调用，H2D方向
- 支持布局：`page_first_kv_split`

#### 1.2.4 内存注册机制

当前NPU平台使用 `pin_memory` 方式进行内存注册：

```python
def alloc_with_pin_memory(
    dims,
    dtype: torch.dtype,
    device: str,
    pin_memory: bool,
    allocator: None,
) -> torch.Tensor:
    buffer = torch.empty(dims, dtype=dtype, device=device, pin_memory=pin_memory)
    return buffer
```

注册时机：在 `init_kv_buffer()` 初始化时完成。

---

### 1.3 `load_cache_to_device_buffer` 接口分析

#### 1.3.1 当前接口定义

```python
def load_cache_to_device_buffer_mla(
    top_k_tokens: torch.Tensor,           # Top-K token列表
    device_buffer_tokens: torch.Tensor,   # device buffer中的token映射
    host_cache_locs: torch.Tensor,        # host cache位置
    device_buffer_locs: torch.Tensor,     # device buffer位置
    host_cache: torch.Tensor,             # host cache数据
    device_buffer: torch.Tensor,          # device buffer数据
    top_k_device_locs: torch.Tensor,      # Top-K在device的位置
    req_pool_indices: torch.Tensor,       # 请求池索引
    seq_lens: torch.Tensor,               # 序列长度
    lru_slots: torch.Tensor,              # LRU槽位
    item_size_bytes: int,                 # 每个item的字节数
    num_top_k: int,                       # Top-K数量
    hot_buffer_size: int,                 # 热缓冲区大小
    page_size: int = 1,                   # 页大小
    block_size: int = 256,                # 块大小
    num_real_reqs: torch.Tensor | None = None,  # 实际请求数
) -> None:
```

#### 1.3.2 上下文用法

该接口在 `hisparse_coordinator.py` 中被调用，用于HiSparse场景的稀疏KV缓存加载：

```python
swap_in_fn = load_cache_to_device_buffer_dsv4_mla if self.is_dsv4_hisparse else load_cache_to_device_buffer_mla
swap_in_fn(
    top_k_tokens=top_k_result,
    device_buffer_tokens=self.req_device_buffer_tokens[layer_id],
    host_cache_locs=self.req_to_host_pool,
    ...
)
```

---

## 二、需求分析

### 2.1 新增需求背景

在昇腾NPU服务器上部署时，需要支持以下场景：

| 需求场景 | 描述 | 技术挑战 |
|---------|------|---------|
| **内存池化** | decode节点内实现KV cache内存池化管理 | 需要支持内存池分配/释放、碎片整理 |
| **外挂内存服务器** | 支持远程内存设备作为KV cache存储后端 | 需要支持远程内存访问、网络传输 |
| **多级存储** | 支持本地内存 + 远程内存的多级存储架构 | 需要支持存储层级管理、数据迁移策略 |

### 2.2 现有接口的局限性

1. **`transfer_kv_dim_exchange`**：
   - 仅支持本地host内存，无法访问远程内存
   - 内存注册方式固定，不支持动态注册
   - 缺乏存储后端抽象，难以扩展

2. **`load_cache_to_device_buffer`**：
   - 仅支持CPU host缓存，不支持外挂存储
   - 缺乏缓存策略控制（如预取、替换策略）
   - 与特定存储后端耦合紧密

---

## 三、接口改动设计

### 3.1 设计原则

1. **向后兼容性**：保持原有接口不变，通过新增参数扩展功能
2. **存储抽象**：引入存储后端抽象层，解耦具体存储实现
3. **内存池化支持**：支持内存池的分配、释放和管理
4. **扩展性**：预留扩展点，支持未来新增存储后端

---

### 3.2 `transfer_kv_dim_exchange` 接口改动

#### 3.2.1 新增参数说明

| 参数名 | 类型 | 说明 | 默认值 |
|-------|------|------|-------|
| `storage_backend` | `StorageBackend` | 存储后端抽象接口 | `None` |
| `memory_pool_id` | `int` | 内存池ID | `-1`（不使用内存池） |
| `stream` | `torch.npu.Stream` | 自定义流 | `None`（使用默认流） |
| `async_mode` | `bool` | 是否异步传输 | `False` |
| `prefetch_hint` | `int` | 预取提示（0=无, 1=顺序, 2=随机） | `0` |

#### 3.2.2 改动后接口定义

**Python层：**

```python
from enum import Enum
from typing import Optional, Protocol
from dataclasses import dataclass

class TransferDirection(Enum):
    H2D = 1
    D2H = 2

class TransferFlag(Enum):
    FAST2D = 2
    ASYNC = 4  # 新增：异步传输

class StorageBackend(Protocol):
    """存储后端抽象接口"""
    
    def read(self, offset: int, size: int) -> torch.Tensor:
        """从存储后端读取数据"""
        ...
    
    def write(self, offset: int, data: torch.Tensor) -> None:
        """向存储后端写入数据"""
        ...
    
    def register_memory(self, buffer: torch.Tensor) -> int:
        """注册内存区域，返回注册ID"""
        ...
    
    def unregister_memory(self, handle: int) -> None:
        """注销已注册的内存区域"""
        ...

@dataclass
class TransferRequest:
    """传输请求上下文"""
    request_id: int = 0
    priority: int = 0  # 0-10，值越大优先级越高
    deadline_ms: int = 0  # 截止时间，0表示无限制

def transfer_kv_dim_exchange(
    # 原有参数
    device_indices: torch.Tensor,
    host_indices: torch.Tensor,
    device_k: torch.Tensor,
    host_k: torch.Tensor,
    device_v: torch.Tensor,
    host_v: torch.Tensor,
    device_index_k: Optional[torch.Tensor] = None,
    host_index_k: Optional[torch.Tensor] = None,
    page_size: int = 128,
    direction: TransferDirection = TransferDirection.H2D,
    flags: TransferFlag = TransferFlag.FAST2D,
    
    # 新增参数
    storage_backend: Optional[StorageBackend] = None,
    memory_pool_id: int = -1,
    stream: Optional[torch.npu.Stream] = None,
    async_mode: bool = False,
    prefetch_hint: int = 0,
    transfer_request: Optional[TransferRequest] = None,
):
    """
    在L1和L2 radix cache场景下，在device和host之间执行KV数据的批量复制。
    
    新增特性：
    - 支持外挂存储后端：通过storage_backend参数支持远程内存服务器
    - 支持内存池化：通过memory_pool_id参数指定使用的内存池
    - 支持异步传输：通过async_mode和flags支持异步操作
    - 支持传输优先级：通过transfer_request控制传输调度
    
    Args:
        storage_backend: 存储后端抽象接口，为None时使用本地内存
        memory_pool_id: 内存池ID，-1表示不使用内存池管理
        stream: 自定义NPU流，用于异步传输
        async_mode: 是否启用异步模式
        prefetch_hint: 预取模式提示：0=无预取, 1=顺序访问, 2=随机访问
        transfer_request: 传输请求上下文，包含优先级和截止时间
    """
    pass
```

**C++层改动：**

```cpp
namespace sglang {
namespace npu_kernel {

// 传输标志位扩展
constexpr int64_t KV_TRANS_FLAG_1D = 1 << 0;
constexpr int64_t KV_TRANS_FLAG_2D = 1 << 1;
constexpr int64_t KV_TRANS_FLAG_ASYNC = 1 << 2;  // 新增
constexpr int64_t KV_TRANS_FLAG_PREFETCH_SEQ = 1 << 3;  // 新增：顺序预取
constexpr int64_t KV_TRANS_FLAG_PREFETCH_RAND = 1 << 4;  // 新增：随机预取

// 传输请求上下文
struct TransferRequest {
    int64_t request_id = 0;
    int64_t priority = 0;
    int64_t deadline_ms = 0;
};

// 存储后端抽象接口
class StorageBackend {
public:
    virtual ~StorageBackend() = default;
    
    // 读取数据到buffer
    virtual void read(int64_t offset, int64_t size, void* buffer) = 0;
    
    // 从buffer写入数据
    virtual void write(int64_t offset, int64_t size, const void* buffer) = 0;
    
    // 注册内存区域
    virtual int64_t register_memory(void* buffer, int64_t size) = 0;
    
    // 注销内存区域
    virtual void unregister_memory(int64_t handle) = 0;
    
    // 查询是否为远程存储
    virtual bool is_remote() const { return false; }
};

// 扩展后的接口
HOST_API void transfer_kv_dim_exchange(
    at::Tensor &device_k,
    at::Tensor &host_k,
    at::Tensor &device_v,
    at::Tensor &host_v,
    const at::Tensor &device_indices,
    const at::Tensor &host_indices,
    int64_t page_size,
    int64_t direction,
    int64_t flags,
    
    // 新增参数
    StorageBackend* storage_backend = nullptr,
    int64_t memory_pool_id = -1,
    c10_npu::NPUStream stream = nullptr,
    const TransferRequest* transfer_request = nullptr
);

}  // namespace npu_kernel
}  // namespace sglang
```

---

### 3.3 `load_cache_to_device_buffer` 接口改动

#### 3.3.1 新增参数说明

| 参数名 | 类型 | 说明 | 默认值 |
|-------|------|------|-------|
| `storage_backend` | `StorageBackend` | 存储后端抽象接口 | `None` |
| `memory_pool` | `HiCacheMemoryPool` | 内存池实例 | `None` |
| `cache_policy` | `CachePolicy` | 缓存策略枚举 | `CachePolicy.LRU` |
| `async_mode` | `bool` | 是否异步加载 | `False` |
| `enable_prefetch` | `bool` | 是否启用预取 | `False` |

#### 3.3.2 改动后接口定义

```python
from enum import Enum
from typing import Optional, Protocol

class CachePolicy(Enum):
    """缓存替换策略"""
    LRU = 1      # 最近最少使用
    LFU = 2      # 最不经常使用
    FIFO = 3     # 先进先出
    CLOCK = 4    # 时钟算法

class HiCacheMemoryPool(Protocol):
    """HiCache内存池接口"""
    
    def alloc(self, size: int) -> int:
        """分配内存，返回起始偏移"""
        ...
    
    def free(self, offset: int, size: int) -> None:
        """释放内存"""
        ...
    
    def get_available_size(self) -> int:
        """获取可用内存大小"""
        ...
    
    def register_buffer(self, buffer: torch.Tensor) -> int:
        """注册外部buffer到内存池"""
        ...

def load_cache_to_device_buffer_mla(
    # 原有参数
    top_k_tokens: torch.Tensor,
    device_buffer_tokens: torch.Tensor,
    host_cache_locs: torch.Tensor,
    device_buffer_locs: torch.Tensor,
    host_cache: torch.Tensor,
    device_buffer: torch.Tensor,
    top_k_device_locs: torch.Tensor,
    req_pool_indices: torch.Tensor,
    seq_lens: torch.Tensor,
    lru_slots: torch.Tensor,
    item_size_bytes: int,
    num_top_k: int,
    hot_buffer_size: int,
    page_size: int = 1,
    block_size: int = 256,
    num_real_reqs: Optional[torch.Tensor] = None,
    
    # 新增参数
    storage_backend: Optional[StorageBackend] = None,
    memory_pool: Optional[HiCacheMemoryPool] = None,
    cache_policy: CachePolicy = CachePolicy.LRU,
    async_mode: bool = False,
    enable_prefetch: bool = False,
    prefetch_depth: int = 2,
) -> None:
    """
    MLA架构下的稀疏KV缓存加载。
    
    新增特性：
    - 支持外挂存储后端：通过storage_backend访问远程内存
    - 支持内存池化：通过memory_pool管理device buffer分配
    - 支持多种缓存策略：通过cache_policy配置替换算法
    - 支持预取：通过enable_prefetch和prefetch_depth启用数据预取
    
    Args:
        storage_backend: 存储后端抽象接口
        memory_pool: 内存池实例，用于管理device buffer
        cache_policy: 缓存替换策略
        async_mode: 是否异步加载
        enable_prefetch: 是否启用预取
        prefetch_depth: 预取深度
    """
    pass

def load_cache_to_device_buffer_dsv4_mla(
    # 原有参数保持不变
    top_k_tokens: torch.Tensor,
    device_buffer_tokens: torch.Tensor,
    host_cache_locs: torch.Tensor,
    device_buffer_locs: torch.Tensor,
    host_cache: torch.Tensor,
    device_buffer: torch.Tensor,
    top_k_device_locs: torch.Tensor,
    req_pool_indices: torch.Tensor,
    seq_lens: torch.Tensor,
    lru_slots: torch.Tensor,
    item_size_bytes: int,
    num_top_k: int,
    hot_buffer_size: int,
    page_size: int = 1,
    block_size: int = 256,
    num_real_reqs: Optional[torch.Tensor] = None,
    
    # 新增参数（与通用版本一致）
    storage_backend: Optional[StorageBackend] = None,
    memory_pool: Optional[HiCacheMemoryPool] = None,
    cache_policy: CachePolicy = CachePolicy.LRU,
    async_mode: bool = False,
    enable_prefetch: bool = False,
    prefetch_depth: int = 2,
) -> None:
    """
    DSv4架构下的稀疏KV缓存加载（页对齐device布局）。
    参数说明同load_cache_to_device_buffer_mla。
    """
    pass
```

---

## 四、内存池化设计

### 4.1 内存池架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      KV Cache 内存池                           │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │   Device    │    │    Host     │    │   Remote    │        │
│  │   Pool      │    │   Pool      │    │   Pool      │        │
│  │ (NPU Memory)│    │ (DRAM)      │    │ (External)  │        │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘        │
│         │                  │                  │                │
│         └────────┬─────────┴──────────────────┘                │
│                  ▼                                            │
│         ┌─────────────────┐                                   │
│         │  Pool Manager   │  ← 统一内存池管理器                 │
│         │                 │     • 分配/释放                     │
│         │                 │     • 碎片整理                     │
│         │                 │     • 迁移调度                     │
│         └────────┬────────┘                                   │
│                  │                                            │
│                  ▼                                            │
│         ┌─────────────────┐                                   │
│         │   Allocator     │  ← 内存分配器                     │
│         │   (Page-based)  │     • 页分配                     │
│         │                 │     • 对齐处理                     │
│         └─────────────────┘                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 内存池接口设计

```python
class HiCacheMemoryPool:
    """HiCache内存池实现"""
    
    def __init__(
        self,
        device: str,
        size: int,
        page_size: int = 128,
        enable_memory_saver: bool = False,
    ):
        self.device = device
        self.size = size
        self.page_size = page_size
        self.page_num = size // page_size
        
        # 空闲页位图
        self.free_pages = torch.ones(self.page_num, dtype=torch.bool, device=device)
        
        # 页映射表：逻辑页 -> 物理页
        self.page_map = torch.arange(self.page_num, dtype=torch.int64, device=device)
        
        # 内存buffer
        self.buffer = None
    
    def init_buffer(self, dtype: torch.dtype, kv_dim: int):
        """初始化KV缓存buffer"""
        dims = (self.page_num, self.page_size, kv_dim)
        self.buffer = torch.empty(dims, dtype=dtype, device=self.device)
        return self.buffer
    
    def alloc(self, num_pages: int) -> Optional[torch.Tensor]:
        """分配连续的物理页"""
        # 找到连续的空闲页块
        free_mask = self.free_pages.cpu().numpy()
        # ... 实现连续页分配算法
        pass
    
    def free(self, page_indices: torch.Tensor) -> None:
        """释放物理页"""
        self.free_pages[page_indices] = True
    
    def get_buffer_view(self, page_indices: torch.Tensor) -> torch.Tensor:
        """获取指定页的buffer视图"""
        return self.buffer[page_indices]
    
    def migrate_pages(self, page_indices: torch.Tensor, target_pool: "HiCacheMemoryPool") -> None:
        """将页迁移到目标内存池"""
        # 1. 读取数据
        # 2. 写入目标池
        # 3. 更新页映射
        # 4. 释放原页
        pass
```

---

## 五、外挂内存服务器支持

### 5.1 存储后端实现示例

```python
class RemoteMemoryBackend(StorageBackend):
    """远程内存服务器后端"""
    
    def __init__(self, server_address: str, port: int):
        self.server_address = server_address
        self.port = port
        self.connections = {}  # 连接池
        self.registered_buffers = {}  # 注册的buffer映射
    
    def _get_connection(self, pool_id: int = 0):
        """获取或创建连接"""
        if pool_id not in self.connections:
            # 建立网络连接
            # self.connections[pool_id] = create_connection(...)
            pass
        return self.connections[pool_id]
    
    def read(self, offset: int, size: int) -> torch.Tensor:
        """从远程服务器读取数据"""
        conn = self._get_connection()
        # 发送读取请求
        # data = conn.read(offset, size)
        # return torch.frombuffer(data, dtype=torch.float16)
        pass
    
    def write(self, offset: int, data: torch.Tensor) -> None:
        """向远程服务器写入数据"""
        conn = self._get_connection()
        # conn.write(offset, data.numpy())
        pass
    
    def register_memory(self, buffer: torch.Tensor) -> int:
        """注册内存区域到远程服务器"""
        handle = len(self.registered_buffers)
        self.registered_buffers[handle] = buffer
        # 通知服务器注册
        return handle
    
    def unregister_memory(self, handle: int) -> None:
        """注销内存区域"""
        del self.registered_buffers[handle]
    
    def is_remote(self) -> bool:
        return True
```

### 5.2 多层存储架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                      多层存储架构                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Decode Node                                                       │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  L1 Cache (Device)                                         │   │
│   │  • 低延迟访问                                              │   │
│   │  • 小容量                                                  │   │
│   └──────────────────┬────────────────────────────────────────┘   │
│                      │  miss                                      │
│                      ▼                                            │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  L2 Cache (Host DRAM)                                      │   │
│   │  • 中等延迟                                                │   │
│   │  • 大容量                                                  │   │
│   └──────────────────┬────────────────────────────────────────┘   │
│                      │  miss                                      │
│                      ▼                                            │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  L3 Cache (Remote Memory Server)                            │   │
│   │  • 高延迟                                                  │   │
│   │  • 超大容量                                                │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、接口调用流程

### 6.1 全量加载流程（H2D）

```
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ 1. 准备阶段      │ -> │ 2. 内存注册      │ -> │ 3. 执行传输      │
├──────────────────┤    ├──────────────────┤    ├──────────────────┤
│ • 计算需要加载   │    │ • 注册host buffer│    │ • 调用transfer_  │
│   的token范围    │    │ • 注册device     │    │   kv_dim_exchange│
│ • 获取indices    │    │   buffer         │    │ • 等待完成       │
└──────────────────┘    └──────────────────┘    └──────────────────┘
                                                      │
                                                      ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ 6. 更新状态      │ <- │ 5. 缓存管理      │ <- │ 4. 内存同步      │
├──────────────────┤    ├──────────────────┤    ├──────────────────┤
│ • 更新LRU状态    │    │ • 更新缓存映射   │    │ • 同步索引映射   │
│ • 更新统计信息   │    │ • 记录加载状态   │    │ • 刷新设备缓存   │
└──────────────────┘    └──────────────────┘    └──────────────────┘
```

### 6.2 稀疏加载流程

```
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ 1. Top-K查询    │ -> │ 2. 缓存命中判断  │ -> │ 3. 缺失数据加载  │
├──────────────────┤    ├──────────────────┤    ├──────────────────┤
│ • 执行检索      │    │ • 检查device     │    │ • 调用load_cache │
│ • 获取候选token │    │   buffer         │    │   _to_device     │
└──────────────────┘    │ • 检查host cache │    │ • 异步预取       │
                        └──────────────────┘    └──────────────────┘
                                                      │
                                                      ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ 6. 完成推理      │ <- │ 5. 执行Attention│ <- │ 4. 更新LRU      │
├──────────────────┤    ├──────────────────┤    ├──────────────────┤
│ • 返回结果      │    │ • 使用加载的KV   │    │ • 更新slot状态   │
│                 │    │   计算attention │    │ • 触发淘汰策略   │
└──────────────────┘    └──────────────────┘    └──────────────────┘
```

---

## 七、部署建议

### 7.1 配置参数

| 参数名 | 说明 | 推荐值（NPU） |
|-------|------|--------------|
| `page_size` | 页大小（token数） | 128 |
| `host_to_device_ratio` | host/device容量比 | 4-8 |
| `enable_memory_pool` | 是否启用内存池化 | True |
| `cache_policy` | 缓存替换策略 | LRU |
| `enable_prefetch` | 是否启用预取 | True |
| `prefetch_depth` | 预取深度 | 2-4 |

### 7.2 性能优化建议

1. **异步传输**：decode阶段启用 `async_mode=True`，隐藏数据传输延迟
2. **批量处理**：合并多个请求的KV传输，减少传输开销
3. **预取策略**：根据访问模式选择顺序/随机预取模式
4. **内存注册**：提前注册常用内存区域，避免运行时注册开销

---

## 八、总结

| 改动项 | 原设计 | 新设计 | 收益 |
|-------|-------|-------|------|
| 存储后端 | 固定本地内存 | 抽象接口+多后端 | 支持外挂内存服务器 |
| 内存管理 | 静态分配 | 内存池化 | 提高内存利用率 |
| 传输模式 | 同步 | 同步+异步 | 隐藏传输延迟 |
| 缓存策略 | 固定LRU | 可配置策略 | 适配不同访问模式 |
| 预取支持 | 无 | 多级预取 | 减少缓存miss |

通过以上改动，KV cache传输接口将具备更好的扩展性和性能，能够满足昇腾NPU服务器上内存池化和外挂内存服务器的需求。