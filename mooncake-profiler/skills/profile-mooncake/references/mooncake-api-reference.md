# Mooncake API 完整参考手册

**版本**: v1.0
**更新日期**: 2026-01-02

## 目录

- [1. Python API](#1-python-api)
  - [1.1 MooncakeDistributedStore](#11-mooncakedistributedstore)
  - [1.2 基本操作 API](#12-基本操作-api)
  - [1.3 批量操作 API](#13-批量操作-api)
  - [1.4 高级 API](#14-高级-api)
- [2. C++ API](#2-c-api)
  - [2.1 RealClient](#21-realclient)
  - [2.2 PyClient](#22-pyclient)
  - [2.3 BufferHandle](#23-bufferhandle)
- [3. TransferQueue API](#3-transferqueue-api)
  - [3.1 MooncakeStorageClient](#31-mooncakestorageclient)
  - [3.2 MooncakeStorageManager](#32-mooncakestoragemanager)
- [4. 配置参数](#4-配置参数)
- [5. 错误码](#5-错误码)

---

## 1. Python API

### 1.1 MooncakeDistributedStore

主要的 Python 接口类，提供分布式对象存储功能。

#### 初始化

```python
from mooncake.store import MooncakeDistributedStore

store = MooncakeDistributedStore()
```

#### setup() - 初始化存储

**签名**:
```python
def setup(
    local_hostname: str,
    metadata_server: str,
    global_segment_size: int = 512 * 1024 * 1024,
    local_buffer_size: int = 128 * 1024 * 1024,
    protocol: str = "tcp",
    rdma_devices: str = "",
    master_server_addr: str = "127.0.0.1:50051"
) -> int
```

**参数**:
- `local_hostname` (str): 本地主机名或 IP 地址
- `metadata_server` (str): 元数据服务器地址
  - HTTP: `"http://IP:PORT/metadata"`
  - etcd: `"etcd://IP:PORT"`
  - Redis: `"redis://IP:PORT"`
- `global_segment_size` (int): 全局内存段大小（字节）
  - 默认: 512 MB
  - 推荐: 512 MB - 2 GB
- `local_buffer_size` (int): 本地缓冲区大小（字节）
  - 默认: 128 MB
  - 推荐: 128 MB - 512 MB
- `protocol` (str): 传输协议
  - `"tcp"`: TCP 协议（默认）
  - `"rdma"`: RDMA 协议（需要硬件支持）
- `rdma_devices` (str): RDMA 设备名称（逗号分隔）
  - 示例: `"mlx5_0,mlx5_1"`
  - 留空则自动发现
- `master_server_addr` (str): Master 服务器地址
  - 格式: `"IP:PORT"`
  - 默认: `"127.0.0.1:50051"`

**返回值**:
- `0`: 成功
- `负数`: 失败（参考错误码）

**示例**:
```python
ret = store.setup(
    local_hostname="10.0.0.1",
    metadata_server="http://10.0.0.1:8081/metadata",
    global_segment_size=1024 * 1024 * 1024,  # 1 GB
    local_buffer_size=256 * 1024 * 1024,      # 256 MB
    protocol="rdma",
    rdma_devices="",  # 自动发现
    master_server_addr="10.0.0.1:50051"
)

if ret != 0:
    raise RuntimeError(f"Setup failed with code {ret}")
```

#### setup_dummy() - 初始化 Dummy 客户端

用于测试或本地开发。

**签名**:
```python
def setup_dummy(
    mem_pool_size: int,
    local_buffer_size: int,
    server_address: str,
    ipc_socket_path: str = ""
) -> int
```

**参数**:
- `mem_pool_size` (int): 内存池大小
- `local_buffer_size` (int): 本地缓冲区大小
- `server_address` (str): 服务器地址
- `ipc_socket_path` (str): IPC socket 路径（可选）

**示例**:
```python
ret = store.setup_dummy(
    mem_pool_size=1024 * 1024 * 1024,
    local_buffer_size=128 * 1024 * 1024,
    server_address="127.0.0.1:50051"
)
```

---

### 1.2 基本操作 API

#### put_tensor() - 存储单个 Tensor

**签名**:
```python
def put_tensor(key: str, tensor: torch.Tensor) -> int
```

**参数**:
- `key` (str): 对象的唯一标识符
- `tensor` (torch.Tensor): 要存储的 PyTorch Tensor

**返回值**:
- `0`: 成功
- `负数`: 失败

**示例**:
```python
import torch

tensor = torch.randn(1024, 1024, dtype=torch.float32)
ret = store.put_tensor("my_tensor", tensor)

if ret == 0:
    print("Tensor stored successfully")
```

**注意事项**:
- Tensor 必须是连续的（contiguous）
- 支持的数据类型: float32, float16, int32, int64, bool
- Key 长度限制: < 256 字符

#### get_tensor() - 获取单个 Tensor

**签名**:
```python
def get_tensor(key: str) -> Optional[torch.Tensor]
```

**参数**:
- `key` (str): 对象的标识符

**返回值**:
- `torch.Tensor`: 成功时返回 Tensor
- `None`: 失败或不存在

**示例**:
```python
tensor = store.get_tensor("my_tensor")

if tensor is not None:
    print(f"Retrieved tensor shape: {tensor.shape}")
else:
    print("Tensor not found or error occurred")
```

#### remove() - 删除对象

**签名**:
```python
def remove(key: str) -> int
```

**参数**:
- `key` (str): 要删除的对象标识符

**返回值**:
- `0`: 成功
- `负数`: 失败

**示例**:
```python
ret = store.remove("my_tensor")
if ret == 0:
    print("Tensor removed successfully")
```

#### isExist() - 检查对象是否存在

**签名**:
```python
def isExist(key: str) -> int
```

**参数**:
- `key` (str): 对象标识符

**返回值**:
- `1`: 存在
- `0`: 不存在
- `-1`: 错误

**示例**:
```python
exists = store.isExist("my_tensor")
if exists == 1:
    print("Tensor exists")
elif exists == 0:
    print("Tensor does not exist")
else:
    print("Error checking existence")
```

#### getSize() - 获取对象大小

**签名**:
```python
def getSize(key: str) -> int
```

**参数**:
- `key` (str): 对象标识符

**返回值**:
- `>= 0`: 对象大小（字节）
- `-1`: 错误或不存在

**示例**:
```python
size = store.getSize("my_tensor")
if size >= 0:
    print(f"Tensor size: {size / (1024**2):.2f} MB")
```

---

### 1.3 批量操作 API

#### batch_put_tensor() - 批量存储 Tensors

**签名**:
```python
def batch_put_tensor(
    keys: List[str],
    tensors: List[torch.Tensor]
) -> List[int]
```

**参数**:
- `keys` (List[str]): 对象标识符列表
- `tensors` (List[torch.Tensor]): Tensor 列表

**返回值**:
- `List[int]`: 每个操作的结果（0=成功，负数=失败）

**示例**:
```python
keys = ["tensor_0", "tensor_1", "tensor_2"]
tensors = [
    torch.randn(100, 100, dtype=torch.float32),
    torch.randn(200, 200, dtype=torch.float32),
    torch.randn(300, 300, dtype=torch.float32)
]

results = store.batch_put_tensor(keys, tensors)

for i, (key, result) in enumerate(zip(keys, results)):
    if result == 0:
        print(f"✓ {key} stored successfully")
    else:
        print(f"✗ {key} failed with code {result}")
```

**性能提示**:
- 批量操作比单个操作快 10-100 倍
- 推荐批次大小: 100-500
- 超大批次（>1000）可能导致内存压力

#### batch_get_tensor() - 批量获取 Tensors

**签名**:
```python
def batch_get_tensor(keys: List[str]) -> List[Optional[torch.Tensor]]
```

**参数**:
- `keys` (List[str]): 对象标识符列表

**返回值**:
- `List[Optional[torch.Tensor]]`: Tensor 列表（None 表示失败或不存在）

**示例**:
```python
keys = ["tensor_0", "tensor_1", "tensor_2"]
tensors = store.batch_get_tensor(keys)

for key, tensor in zip(keys, tensors):
    if tensor is not None:
        print(f"✓ {key}: shape={tensor.shape}, dtype={tensor.dtype}")
    else:
        print(f"✗ {key}: not found or error")
```

**注意事项**:
- 返回的 Tensors 顺序与 keys 顺序一致
- 失败的项返回 None，但不影响其他项
- 内部会自动分批处理（batch_size=500）

#### batchIsExist() - 批量检查存在性

**签名**:
```python
def batchIsExist(keys: List[str]) -> List[int]
```

**参数**:
- `keys` (List[str]): 对象标识符列表

**返回值**:
- `List[int]`: 每个 key 的存在性（1=存在，0=不存在，-1=错误）

**示例**:
```python
keys = ["tensor_0", "tensor_1", "tensor_2"]
exists = store.batchIsExist(keys)

for key, exist in zip(keys, exists):
    status = {1: "exists", 0: "not found", -1: "error"}
    print(f"{key}: {status.get(exist, 'unknown')}")
```

---

### 1.4 高级 API

#### register_buffer() - 注册内存缓冲区

用于零拷贝操作。

**签名**:
```python
def register_buffer(buffer: int, size: int) -> int
```

**参数**:
- `buffer` (int): 缓冲区指针（通常来自 `tensor.data_ptr()`）
- `size` (int): 缓冲区大小（字节）

**返回值**:
- `0`: 成功
- `负数`: 失败

**示例**:
```python
tensor = torch.empty(1024, 1024, dtype=torch.float32)
buffer_ptr = tensor.data_ptr()
buffer_size = tensor.numel() * tensor.element_size()

ret = store.register_buffer(buffer_ptr, buffer_size)
if ret == 0:
    print("Buffer registered successfully")
```

#### unregister_buffer() - 取消注册缓冲区

**签名**:
```python
def unregister_buffer(buffer: int) -> int
```

**参数**:
- `buffer` (int): 缓冲区指针

**返回值**:
- `0`: 成功
- `负数`: 失败

#### get_into() - 零拷贝获取

直接将数据读取到预分配的缓冲区，避免拷贝。

**签名**:
```python
def get_into(key: str, buffer: int, size: int) -> int
```

**参数**:
- `key` (str): 对象标识符
- `buffer` (int): 目标缓冲区指针（必须已注册）
- `size` (int): 缓冲区大小

**返回值**:
- `>= 0`: 实际读取的字节数
- `负数`: 失败

**示例**:
```python
# 预分配 tensor
tensor = torch.empty(1024, 1024, dtype=torch.float32)
buffer_ptr = tensor.data_ptr()
buffer_size = tensor.numel() * tensor.element_size()

# 注册缓冲区
store.register_buffer(buffer_ptr, buffer_size)

# 零拷贝读取
bytes_read = store.get_into("my_tensor", buffer_ptr, buffer_size)

if bytes_read > 0:
    print(f"Read {bytes_read} bytes into buffer")
    # tensor 现在包含数据，无需额外拷贝

# 清理
store.unregister_buffer(buffer_ptr)
```

#### batch_get_into() - 批量零拷贝获取

**签名**:
```python
def batch_get_into(
    keys: List[str],
    buffers: List[int],
    sizes: List[int]
) -> List[int]
```

**参数**:
- `keys` (List[str]): 对象标识符列表
- `buffers` (List[int]): 缓冲区指针列表（必须已注册）
- `sizes` (List[int]): 缓冲区大小列表

**返回值**:
- `List[int]`: 每个操作读取的字节数（负数表示失败）

**示例**:
```python
# 预分配多个 tensors
tensors = [torch.empty(100, 100, dtype=torch.float32) for _ in range(3)]
buffer_ptrs = [t.data_ptr() for t in tensors]
buffer_sizes = [t.numel() * t.element_size() for t in tensors]

# 注册所有缓冲区
for ptr, size in zip(buffer_ptrs, buffer_sizes):
    store.register_buffer(ptr, size)

# 批量零拷贝读取
keys = ["tensor_0", "tensor_1", "tensor_2"]
bytes_read = store.batch_get_into(keys, buffer_ptrs, buffer_sizes)

for key, read in zip(keys, bytes_read):
    if read > 0:
        print(f"✓ {key}: read {read} bytes")
    else:
        print(f"✗ {key}: failed")

# 清理
for ptr in buffer_ptrs:
    store.unregister_buffer(ptr)
```

**性能对比**:
```python
# 普通 batch_get_tensor (有拷贝)
# 吞吐量: ~10-15 Gb/s

# batch_get_into (零拷贝)
# 吞吐量: ~20-30 Gb/s (提升 2-3x)
```

#### put_from() - 零拷贝存储

**签名**:
```python
def put_from(key: str, buffer: int, size: int) -> int
```

**参数**:
- `key` (str): 对象标识符
- `buffer` (int): 源缓冲区指针（必须已注册）
- `size` (int): 数据大小

**返回值**:
- `0`: 成功
- `负数`: 失败

#### batch_put_from() - 批量零拷贝存储

**签名**:
```python
def batch_put_from(
    keys: List[str],
    buffers: List[int],
    sizes: List[int]
) -> List[int]
```

**参数**:
- `keys` (List[str]): 对象标识符列表
- `buffers` (List[int]): 源缓冲区指针列表
- `sizes` (List[int]): 数据大小列表

**返回值**:
- `List[int]`: 每个操作的结果

#### removeByRegex() - 正则删除

**签名**:
```python
def removeByRegex(pattern: str) -> int
```

**参数**:
- `pattern` (str): 正则表达式模式

**返回值**:
- `>= 0`: 删除的对象数量
- `负数`: 失败

**示例**:
```python
# 删除所有以 "temp_" 开头的对象
count = store.removeByRegex("^temp_.*")
print(f"Removed {count} temporary objects")
```

#### removeAll() - 删除所有对象

**签名**:
```python
def removeAll() -> int
```

**返回值**:
- `>= 0`: 删除的对象数量
- `负数`: 失败

**警告**: 此操作不可逆，谨慎使用！

#### close() - 关闭存储

**签名**:
```python
def close() -> None
```

**示例**:
```python
store.close()
# 之后不能再使用此 store 实例
```

---

## 2. C++ API

### 2.1 RealClient

C++ 层的实际客户端实现。

#### 核心方法

##### batch_get_buffer()

**签名**:
```cpp
std::vector<std::shared_ptr<BufferHandle>> batch_get_buffer(
    const std::vector<std::string> &keys
);
```

**功能**: 批量获取数据的底层实现。

**流程**:
1. 查询元数据（BatchQuery）
2. 准备缓冲区（allocate）
3. 执行批量获取（BatchGet）
4. 处理结果

**性能特征**:
- 单次 batch 吞吐量: 430-466 Gb/s（RDMA）
- 元数据查询: 18-26% 时间
- 实际 I/O: 68-76% 时间
- 缓冲区准备: 5% 时间

##### batch_get_buffer_internal()

内部实现，包含详细的性能分析（如果启用 profiling）。

**时间分布**（典型值，500 keys）:
```
Total: 2.8-3.0 ms
├─ Metadata query:  0.5-0.8 ms (20%)
├─ Buffer prepare:  0.15 ms (5%)
│  └─ Allocations:  0.05 ms (1.8%)
├─ Batch GET (I/O): 2.0-2.2 ms (74%)
└─ Result process:  0.02 ms (0.5%)
```

##### register_buffer()

**签名**:
```cpp
int register_buffer(void *buffer, size_t size);
```

**功能**: 注册内存区域用于零拷贝操作（RDMA）。

##### batch_get_into()

**签名**:
```cpp
std::vector<int64_t> batch_get_into(
    const std::vector<std::string> &keys,
    const std::vector<void *> &buffers,
    const std::vector<size_t> &sizes
);
```

**功能**: 零拷贝批量获取。

---

### 2.2 PyClient

Python 绑定的基类。

**主要职责**:
- GIL 管理
- Python ↔ C++ 对象转换
- 错误处理

---

### 2.3 BufferHandle

表示一个数据缓冲区。

**主要方法**:
```cpp
class BufferHandle {
public:
    void* ptr();           // 获取缓冲区指针
    size_t size();         // 获取缓冲区大小
    void release();        // 释放缓冲区
};
```

---

## 3. TransferQueue API

### 3.1 MooncakeStorageClient

TransferQueue 中对 Mooncake 的封装。

**位置**: `transfer_queue/storage/clients/mooncake_client.py`

#### 初始化

```python
from transfer_queue.storage.clients.mooncake_client import MooncakeStorageClient

config = {
    "metadata_server": "http://10.0.0.1:8081/metadata",
    "master_server_address": "10.0.0.1:50051",
    "local_hostname": "localhost",
    "protocol": "rdma",
    "global_segment_size": 512 * 1024 * 1024,
    "local_buffer_size": 128 * 1024 * 1024,
    "device_name": ""
}

client = MooncakeStorageClient(config)
```

#### get() - 获取数据

**签名**:
```python
def get(
    self,
    keys: List[str],
    shapes: Optional[List] = None,
    dtypes: Optional[List] = None
) -> List[Any]
```

**参数**:
- `keys` (List[str]): 要获取的 key 列表
- `shapes` (List, optional): Tensor 的 shape 列表
- `dtypes` (List, optional): Tensor 的 dtype 列表

**返回值**:
- `List[Any]`: 数据列表（Tensor 或 NonTensorData）

**内部实现**:
```python
def get(self, keys, shapes=None, dtypes=None):
    # 1. 分类项目（tensor vs non-tensor）
    # 2. 批量获取 tensors (_batch_get_tensors)
    # 3. 批量获取 non-tensors
    # 4. 合并结果
    return results
```

**性能特征**（15.67 GB 数据）:
```
Total time: 10.07s
├─ Classify items:     0.003s (0.0%)
├─ Tensor operations:  9.76s (96.9%)
├─ Non-tensor ops:     0.30s (3.0%)
└─ Other overhead:     0.006s (0.1%)
```

#### put() - 存储数据

**签名**:
```python
def put(
    self,
    keys: List[str],
    data: List[Any]
) -> List[int]
```

**参数**:
- `keys` (List[str]): 对象标识符列表
- `data` (List[Any]): 数据列表

**返回值**:
- `List[int]`: 结果列表

#### _batch_get_tensors() - 批量获取 Tensors（内部）

**签名**:
```python
def _batch_get_tensors(
    self,
    keys: List[str],
    shapes: List,
    dtypes: List
) -> List[torch.Tensor]
```

**内部流程**:
```python
# BATCH_SIZE_LIMIT = 500
for batch in chunks(keys, BATCH_SIZE_LIMIT):
    # 1. 调用 self._store.batch_get_tensor(batch_keys)
    # 2. 验证 shape 和 dtype
    # 3. 累积结果
```

**性能优化机会**:
- 增大 BATCH_SIZE_LIMIT (500 → 2000-5000)
- 异步并行处理多个 batch
- 使用 batch_get_into 实现零拷贝

---

### 3.2 MooncakeStorageManager

KVStorageManager 的 Mooncake 实现。

**位置**: `transfer_queue/storage/managers/kv_storage_manager.py`

#### get_data() - 获取数据

**签名**:
```python
async def get_data(
    self,
    metadata: BatchMeta
) -> TensorDict
```

**参数**:
- `metadata` (BatchMeta): 批次元数据

**返回值**:
- `TensorDict`: 数据字典

**时间分布**（14.26s 总时间）:
```
├─ Generate keys:      0.01s (0.1%)
├─ Get shape/type:     0.04s (0.3%)
├─ Storage client.get: 10.07s (70.6%) ← 主要瓶颈
└─ Merge tensors:      4.14s (29.0%)  ← 次要瓶颈
```

**优化建议**:
1. 优化 client.get (P1)
2. 优化 merge tensors (P0 - 最高优先级)

---

## 4. 配置参数

### 4.1 Mooncake 配置

**完整配置示例** (`mooncake_config.yaml`):

```yaml
# 元数据服务器
metadata_server: "http://10.144.204.167:8081/metadata"
# 可选: "etcd://IP:PORT" 或 "redis://IP:PORT"

# Master 服务器
master_server_address: "10.144.204.167:50051"

# 本地主机名
local_hostname: "localhost"

# 传输协议
protocol: "rdma"  # 或 "tcp"

# 内存配置
global_segment_size: 536870912000  # 512 GB
local_buffer_size: 13421772800     # 12.5 GB

# RDMA 设备（可选，留空自动发现）
device_name: ""

# 副本配置（可选）
replication_factor: 1
```

### 4.2 性能调优参数

#### 内存配置

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `global_segment_size` | 512MB - 2GB | 越大越好，但受内存限制 |
| `local_buffer_size` | 128MB - 512MB | 根据并发度调整 |

#### 批次大小

| 场景 | 推荐值 | 说明 |
|------|--------|------|
| 小对象 (<1MB) | 1000-5000 | 减少元数据开销 |
| 中对象 (1-10MB) | 100-500 | 平衡延迟和吞吐 |
| 大对象 (>10MB) | 10-100 | 避免内存压力 |

#### 协议选择

| 协议 | 延迟 | 吞吐量 | 使用场景 |
|------|------|--------|----------|
| TCP | 高 (~1ms) | 中 (~10 Gb/s) | 开发、调试 |
| RDMA | 低 (~10μs) | 高 (~100 Gb/s) | 生产环境 |

### 4.3 环境变量

```bash
# Transfer Engine 指标
export MC_TE_METRIC=1              # 启用 TE 指标

# Logging
export GLOG_v=0                    # glog 详细级别 (0-3)
export GLOG_logtostderr=1          # 输出到 stderr

# RDMA 配置
export MC_IB_PCI_RELAXED_ORDERING=0  # 禁用 PCI relaxed ordering
```

---

## 5. 错误码

### 5.1 通用错误码

| 错误码 | 名称 | 说明 |
|--------|------|------|
| `0` | SUCCESS | 成功 |
| `-1` | GENERIC_ERROR | 通用错误 |
| `-200` | INVALID_PARAMS | 参数无效 |
| `-600` | CLIENT_NOT_INITIALIZED | 客户端未初始化 |

### 5.2 对象操作错误

| 错误码 | 名称 | 说明 |
|--------|------|------|
| `-100` | OBJECT_NOT_FOUND | 对象不存在 |
| `-101` | OBJECT_ALREADY_EXISTS | 对象已存在 |
| `-102` | REPLICA_IS_NOT_READY | 副本未就绪 |

### 5.3 网络错误

| 错误码 | 名称 | 说明 |
|--------|------|------|
| `-300` | NETWORK_ERROR | 网络错误 |
| `-301` | TIMEOUT | 超时 |
| `-302` | CONNECTION_FAILED | 连接失败 |

### 5.4 内存错误

| 错误码 | 名称 | 说明 |
|--------|------|------|
| `-400` | OUT_OF_MEMORY | 内存不足 |
| `-401` | BUFFER_TOO_SMALL | 缓冲区太小 |
| `-402` | ALLOCATION_FAILED | 分配失败 |

### 5.5 内部错误

| 错误码 | 名称 | 说明 |
|--------|------|------|
| `-500` | INTERNAL_ERROR | 内部错误 |
| `-501` | NOT_IMPLEMENTED | 未实现 |

---

## 6. 最佳实践

### 6.1 性能优化

#### 使用批量操作

```python
# ❌ 不推荐 - 单个操作
for key, tensor in zip(keys, tensors):
    store.put_tensor(key, tensor)

# ✅ 推荐 - 批量操作
store.batch_put_tensor(keys, tensors)
```

#### 使用零拷贝 API

```python
# ❌ 不推荐 - 有拷贝开销
tensors = store.batch_get_tensor(keys)

# ✅ 推荐 - 零拷贝
tensors = [torch.empty(shape, dtype=dtype) for shape, dtype in ...]
buffer_ptrs = [t.data_ptr() for t in tensors]
buffer_sizes = [t.numel() * t.element_size() for t in tensors]

for ptr, size in zip(buffer_ptrs, buffer_sizes):
    store.register_buffer(ptr, size)

store.batch_get_into(keys, buffer_ptrs, buffer_sizes)

for ptr in buffer_ptrs:
    store.unregister_buffer(ptr)
```

#### 选择合适的批次大小

```python
# 根据对象大小调整
if avg_object_size < 1 * 1024 * 1024:  # < 1MB
    batch_size = 2000
elif avg_object_size < 10 * 1024 * 1024:  # < 10MB
    batch_size = 500
else:  # > 10MB
    batch_size = 50
```

### 6.2 错误处理

```python
# 检查初始化结果
ret = store.setup(...)
if ret != 0:
    raise RuntimeError(f"Setup failed: {ret}")

# 检查批量操作结果
results = store.batch_put_tensor(keys, tensors)
failed_indices = [i for i, r in enumerate(results) if r != 0]

if failed_indices:
    failed_keys = [keys[i] for i in failed_indices]
    print(f"Failed to store: {failed_keys}")
    # 重试或处理错误
```

### 6.3 资源管理

```python
try:
    store = MooncakeDistributedStore()
    store.setup(...)

    # 使用 store
    store.put_tensor(...)

finally:
    # 确保关闭
    store.close()
```

### 6.4 性能监控

```python
import time

# 监控吞吐量
start = time.time()
results = store.batch_get_tensor(keys)
duration = time.time() - start

total_bytes = sum(t.numel() * t.element_size() for t in results if t)
throughput_gbps = (total_bytes * 8 / 1e9) / duration

print(f"Throughput: {throughput_gbps:.2f} Gb/s")
```

---

## 7. 故障排除

### 问题 1: 连接失败

**症状**:
```
ERROR: Failed to setup: -600
```

**解决方案**:
1. 检查 mooncake_master 是否运行
2. 验证网络连接
3. 检查防火墙设置

### 问题 2: 低吞吐量

**症状**:
```
GET Throughput: < 5 Gb/s (expected > 20 Gb/s)
```

**解决方案**:
1. 使用 RDMA 协议而非 TCP
2. 增大批次大小
3. 使用零拷贝 API
4. 检查网络带宽

### 问题 3: 内存不足

**症状**:
```
ERROR: Allocation failed: -400
```

**解决方案**:
1. 减小 batch_size
2. 减小 local_buffer_size
3. 增加系统内存

---

**文档版本**: v1.0
**最后更新**: 2026-01-02
**维护者**: Mooncake Team
