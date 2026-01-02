# Mooncake Profiler Plugin

一键式 Mooncake 性能分析工具，自动完成从测试到优化建议的全流程。

## 功能

- ✅ 自动运行性能测试
- ✅ C++ 层自动 instrumentation
- ✅ 详细瓶颈分析
- ✅ 生成优化建议报告
- ✅ 预估性能提升

## 安装

插件已安装在：
```
/root/.claude/plugins/local/mooncake-profiler/
```

## 使用

### 基本用法
```bash
/profile-mooncake
```

### 带参数
```bash
/profile-mooncake --api batch_get_tensor --config mooncake_config.yaml
```

### 快速模式
```bash
/profile-mooncake --quick
```

## 触发条件

Skill 会在以下情况自动激活：
- 用户提到 "profile mooncake"
- 用户询问 "mooncake 性能"
- 用户提到 "batch_get_tensor 瓶颈"
- 用户询问 "优化 mooncake"

## 输出

### 主要报告
- `BATCH_GET_TENSOR_PROFILING_REPORT.md` - 完整分析报告

### 实时输出
- 测试进度
- C++ profiling 数据
- 关键性能指标
- 优化建议摘要

## 示例

**用户**: "帮我 profile 一下 batch_get_tensor，看看哪里慢"

**Skill 自动执行**:
1. 运行 baseline 测试
2. 添加 C++ profiling
3. 重新编译
4. 收集详细数据
5. 生成优化建议

**输出**:
```
GET Throughput: 8.72 Gb/s
主要瓶颈: Merge tensors (29%)
优化建议: 使用 zero-copy，预期 +40% 性能
```

## 版本

- **v1.0.0** - 初始版本（2026-01-02）

## 依赖

- Mooncake 环境
- TransferQueue
- Python 3.12+
- C++ 编译工具链

## License

MIT
