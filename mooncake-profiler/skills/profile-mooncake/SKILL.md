---
name: profile-mooncake
description: This skill should be used when the user asks to "profile mooncake", "analyze mooncake performance", "benchmark batch_get_tensor", "find mooncake bottlenecks", "optimize mooncake throughput", or mentions mooncake performance issues. Automatically profiles Mooncake APIs, identifies bottlenecks, and generates optimization recommendations.
version: 1.0.0
---

# Mooncake Performance Profiler

自动化 Mooncake API 性能分析工具，一键完成从测试到优化建议的全流程。

## 功能概述

这个 skill 会自动：
1. ✅ 运行性能测试获取 baseline 吞吐量
2. ✅ 在 C++ 代码中添加详细的 profiling instrumentation
3. ✅ 重新编译并安装 Mooncake
4. ✅ 运行 profiling 测试收集详细数据
5. ✅ 分析瓶颈并生成优化建议报告

## 使用方法

### 基本用法

```bash
/profile-mooncake
```

默认会 profile `batch_get_tensor` API，使用 `mooncake_config.yaml`。

### 带参数

```bash
/profile-mooncake --api batch_get_tensor --config mooncake_config.yaml
```

### 快速模式（跳过 C++ profiling）

```bash
/profile-mooncake --quick
```

## 支持的参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--api` | 要 profile 的 API | `batch_get_tensor` |
| `--config` | Mooncake 配置文件 | `mooncake_config.yaml` |
| `--quick` | 快速模式，跳过 C++ profiling | `false` |
| `--iterations` | 测试迭代次数 | 使用配置文件中的设置 |
| `--keep-profiling` | 保留 C++ profiling 代码 | `false` |

## 工作流程

### 阶段 1: Baseline 测试
- 使用 `performance_test.py tq-mooncake` 运行基准测试
- 记录 PUT/GET 吞吐量
- 识别 Python 层的时间分布

### 阶段 2: C++ Profiling（非 quick 模式）
- 自动在 `real_client.cpp` 中添加 timing instrumentation
- 测量以下阶段：
  - Metadata query
  - Buffer preparation & allocation
  - Batch GET I/O
  - Result processing
- 重新编译和安装

### 阶段 3: 详细测试
- 运行带 profiling 的性能测试
- 收集 C++ 层的详细 timing 数据
- 分析每个 batch 的性能指标

### 阶段 4: 分析与报告
- 识别主要瓶颈（按时间占比排序）
- 生成详细的性能分析报告
- 提供分阶段的优化建议
- 预估优化后的性能提升

## 输出文件

### 主报告
`BATCH_GET_TENSOR_PROFILING_REPORT.md`
- 完整的性能分析
- 瓶颈识别和根因分析
- 分阶段优化建议
- 预期性能提升

### C++ 代码
修改的文件会标注 profiling instrumentation 的位置。

### 终端输出
实时显示：
- 测试进度
- C++ profiling 数据
- 关键性能指标
- 优化建议摘要

## 典型输出示例

```
========================================
batch_get_buffer_internal PROFILING
========================================
Request Info:
  Total keys:           500
  Valid ops:            500
  Successful gets:      500
  Total data:           0.152607 GB

Timing Breakdown:
  Total time:           2.8141 ms
  ├─ Metadata query:    0.564742 ms (20.1%)
  ├─ Buffer prepare:    0.139443 ms (5.0%)
  │  └─ Allocations:    0.051628 ms (1.8%)
  ├─ Batch GET (I/O):   2.09293 ms (74.4%) ← PRIMARY BOTTLENECK
  └─ Result process:    0.016002 ms (0.6%)

Performance Metrics:
  Throughput:           465.825 Gb/s
  Avg latency/key:      0.0056282 ms
========================================
```

## 瓶颈分析

Skill 会自动识别以下类型的瓶颈：

### Python 层瓶颈
- ❌ Merge tensors 开销
- ❌ Validation 开销
- ❌ 数据序列化/反序列化

### C++ 层瓶颈
- ❌ Metadata query 延迟
- ❌ Buffer allocation
- ❌ Batch GET I/O
- ❌ Result processing

### 系统层瓶颈
- ❌ 网络带宽限制
- ❌ RDMA 配置问题
- ❌ 批次串行化

## 优化建议类型

根据识别的瓶颈，skill 会提供：

### 代码级优化
- Zero-copy 策略
- 异步并行处理
- 批次大小调优

### 配置级优化
- RDMA 参数调整
- Buffer size 优化
- 并发度设置

### 架构级优化
- Pipeline 设计
- Prefetch 机制
- 缓存策略

## 适用场景

### 场景 1: 初次性能诊断
```
用户: "mooncake的batch_get_tensor性能不好，帮我看看问题在哪"
→ Skill 自动运行完整 profiling 流程
```

### 场景 2: 优化验证
```
用户: "我改了代码，帮我profile一下看性能提升了多少"
→ Skill 运行新的 profiling 并对比
```

### 场景 3: 快速检查
```
用户: "快速看一下当前的吞吐量"
→ 使用 --quick 模式
```

## 前置要求

### 环境
- Mooncake 已安装并配置
- TransferQueue 环境已设置
- 有 `performance_test.py` 和配置文件

### 权限
- 可以修改 C++ 代码
- 可以编译和安装 Mooncake
- 可以运行 Ray 任务

### 配置文件
需要 `mooncake_config.yaml` 包含：
```yaml
metadata_server: "http://IP:8081/metadata"
master_server_address: "IP:50051"
local_hostname: "localhost"
protocol: "rdma"  # 或 "tcp"
global_segment_size: 536870912000
local_buffer_size: 13421772800
```

## 故障排除

### 问题: 编译失败
**解决方案**: 检查 C++ 代码语法，确保 `<chrono>` 已包含

### 问题: 测试失败
**解决方案**:
- 确认 mooncake_master 正在运行
- 检查网络连接
- 验证配置文件路径

### 问题: 无法写入文件
**解决方案**: 检查目录权限，使用 `--output-dir` 指定可写目录

## 高级用法

### 对比分析
```python
# 先运行 baseline
/profile-mooncake --output baseline.md

# 实施优化后
/profile-mooncake --output optimized.md

# 对比结果
diff baseline.md optimized.md
```

### 自定义 Profiling
如果需要 profile 其他 API，可以参考生成的代码模式手动添加 instrumentation。

### 持续监控
将 skill 集成到 CI/CD 流程中，定期运行性能测试。

## 技术细节

### C++ Instrumentation 模式
```cpp
auto start = std::chrono::high_resolution_clock::now();
// ... 执行操作 ...
auto end = std::chrono::high_resolution_clock::now();
double duration_ms = std::chrono::duration<double, std::milli>(end - start).count();
LOG(INFO) << "Operation took: " << duration_ms << " ms";
```

### 测试命令
```bash
cd /workspace/mh/TransferQueue
GLOG_v=0 GLOG_logtostderr=1 python3 performance_test.py tq-mooncake mooncake_config.yaml
```

### 编译命令
```bash
cd /workspace/mh/Mooncake
make -j8
make install
```

## 报告解读

### 关键指标
- **GET Throughput**: 越高越好，目标 >20 Gb/s
- **PUT/GET Ratio**: 应该接近 1.0
- **Batch Throughput**: 单次 batch 的峰值吞吐量
- **I/O Percentage**: Batch GET I/O 占总时间的比例

### 瓶颈判断标准
- **主要瓶颈**: 占用时间 >30%
- **次要瓶颈**: 占用时间 10-30%
- **可忽略**: 占用时间 <10%

### 优化优先级
1. **P0**: 解决可快速见效的瓶颈（如增大 batch size）
2. **P1**: 实施中等复杂度的优化（如异步并行）
3. **P2**: 长期架构优化（如 C++ 层重构）

## 示例场景

### 完整 Profiling 流程

**用户输入**:
```
profile batch_get_tensor，找出性能瓶颈
```

**Skill 执行**:
1. 检测到 "profile" 和 "batch_get_tensor" 关键词
2. 自动运行 baseline 测试
3. 在 C++ 代码中添加 profiling
4. 重新编译安装
5. 运行详细测试
6. 生成完整报告

**输出**:
```
✅ Baseline: GET 8.72 Gb/s
✅ 主要瓶颈: Merge tensors (29%)
✅ 次要瓶颈: Batch GET I/O (49%)
✅ 优化建议:
   P0: 优化 Merge (预期 +40%)
   P1: 异步并行 (预期 +50%)
📊 详细报告: BATCH_GET_TENSOR_PROFILING_REPORT.md
```

## 相关资源

- Mooncake 文档: https://kvcache-ai.github.io/Mooncake/
- TransferQueue 仓库: /workspace/mh/TransferQueue
- 示例配置: /workspace/mh/TransferQueue/mooncake_config.yaml
- C++ 代码: /workspace/mh/Mooncake/mooncake-store/src/real_client.cpp

## 版本历史

### v1.0.0 (2026-01-02)
- ✅ 初始版本
- ✅ 支持 batch_get_tensor profiling
- ✅ C++ 层自动 instrumentation
- ✅ 详细报告生成
- ✅ 优化建议系统
