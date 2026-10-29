# 08. 版本兼容性

## pyqlib 0.9 系列变更

### v0.9.0（主线变更）

- 新增 `qlib.rl` 强化学习框架（trainer 阶段）。PR #1125。
- 回测模拟器重构迁移。PR #1262、#1263。
- `TSDataSampler` 内存布局优化，加速 NN 训练。PR #1342。
- Recorder 自动记录未提交代码。PR #1167。
- 新增 `ChangeInstrument` 算子。PR #1005。
- `logging_level` 在 `qlib.init` 全局生效。PR #1368。

### v0.9.2

- 订单执行环境开源。PR #1447。
- yaml 支持 `BASE_CONFIG_PATH` 基配置。PR #1500。

### v0.9.7

- 数据层支持 parquet。PR #1966。
- 引入 `BaseDataHandler` 并统一 fetch 接口。PR #1958。
- `risk_analysis` 新增几何累加模式 `mode="product"`。PR #1938。

### v0.9.8（破坏性安全变更）

**表达式求值改为受限解析器**
- 旧自定义算子须回归测试。
- 证据：`docs/advanced/alpha.rst:83-166`。

**加载历史 artifact 须显式 `trusted=True`**
- `recorder.load_object(name, trusted=True)`
- 迁移指南：`docs/start/artifact_migration.rst`

**配置驱动的本地 `.py` 导入须 `trusted: true`**
- yaml 中 `module_path` 指向本地文件时
- 迁移指南：`docs/start/config_migration.rst`

## 升级检查清单

升级前必须：

1. 读 `docs/start/artifact_migration.rst`
2. 读 `docs/start/config_migration.rst`
3. 回归测试自定义算子
4. 检查所有 `load_object` 调用加 `trusted=True`
5. 检查所有本地 `module_path` 加 `trusted: true`

## 已知未测残留

0.9.8 的受限解析器对旧自定义算子的兼容性，官方未给完整清单。建议以 `tests/ops/test_expression_compatibility.py` 为回归基准自查。

## 版本查询

```python
import qlib
print(qlib.__version__)
```

## 发布说明

完整变更见 `https://github.com/microsoft/qlib/releases`。
