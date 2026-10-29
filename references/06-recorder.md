# 06. Recorder 与实验管理

## 层级结构

```text
ExpManager → Experiment → Recorder
```

实现为 `MLflowExpManager`（`qlib/workflow/expm.py:317`）。证据：`docs/component/recorder.rst:23-38`。

## 入口对象 R

```python
from qlib.workflow import R
```

证据：`docs/component/recorder.rst:48`。

## 开实验

```python
with R.start_exp(experiment_name="my_exp", recorder_name="run_001"):
    # 训练与回测
    pass
```

或：

```python
R.start(experiment_name="my_exp", recorder_name="run_001", resume=True)
```

证据：`qlib/workflow/__init__.py:98,39`。

## 记录参数与指标

```python
R.log_params(**{"learning_rate": 0.01, "max_depth": 8})
R.log_metrics(step=0, **{"train_loss": 0.5, "valid_loss": 0.6})
```

证据：`__init__.py:550,575`。

## 保存产物

```python
R.save_objects(**{"params.pkl": model, "pred.pkl": pred_df})
```

证据：`__init__.py:482`。

## 加载产物

```python
recorder = R.get_recorder()
model = recorder.load_object("params.pkl", trusted=True)  # 0.9.8+ 必须
```

证据：`__init__.py:393`、`workflow/recorder.py:96,423`。

## 记录模板

| 模板 | 作用 | 源码路径 |
|------|------|---------|
| `SignalRecord` | 存 pred.pkl/label.pkl | `record_temp.py:168` |
| `SigAnaRecord` | 计算 IC/RankIC | `record_temp.py:302` |
| `PortAnaRecord` | 组合分析 | `record_temp.py:365` |
| `MultiPassPortAnaRecord` | 多轮组合分析 | `record_temp.py:582` |

## SignalRecord 示例

```yaml
record:
  - class: SignalRecord
    module_path: qlib.workflow.record_temp
    kwargs:
      model: <MODEL>
      dataset: <DATASET>
```

## SigAnaRecord 示例

```yaml
- class: SigAnaRecord
  module_path: qlib.workflow.record_temp
  kwargs:
    ana_long_short: False
    ann_scaler: 252
```

## 已知限制

产物基于 pickle，环境不一致会加载失败。证据：`docs/component/recorder.rst:166-169`。

## 0.9.8 安全变更

`load_object` 默认受限，可执行对象须 `trusted=True`。证据：`recorder.py:96`；`docs/start/artifact_migration.rst`。
