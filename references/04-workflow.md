# 04. 工作流与 yaml 配置

## qrun 入口

`qrun` → `qlib/cli/run.py` 的 `workflow()`（`run.py:86`），内部调 `task_train(config["task"])`（`run.py:147`）。

## yaml 顶层 section

```yaml
qlib_init:
  provider_uri: "~/.qlib/qlib_data/cn_data"
  region: cn

market: &market csi300
benchmark: &benchmark SH000300

data_handler_config: &data_handler_config
  start_time: 2008-01-01
  end_time: 2020-08-01
  fit_start_time: 2008-01-01
  fit_end_time: 2014-12-31
  instruments: *market

port_analysis_config: &port_analysis_config
  strategy:
    class: TopkDropoutStrategy
    module_path: qlib.contrib.strategy
    kwargs:
      signal: <PRED>
      topk: 50
      n_drop: 5
  backtest:
    start_time: 2017-01-01
    end_time: 2020-08-01
    account: 100000000
    benchmark: *benchmark
    exchange_kwargs:
      limit_threshold: 0.095
      deal_price: close
      open_cost: 0.0005
      close_cost: 0.0015
      min_cost: 5

task:
  model:
    class: LGBModel
    module_path: qlib.contrib.model.gbdt
    kwargs:
      loss: mse
      colsample_bytree: 0.8879
      learning_rate: 0.0421
      subsample: 0.8789
      lambda_l1: 205.6999
      lambda_l2: 580.9768
      max_depth: 8
      num_leaves: 210
      num_threads: 20
  dataset:
    class: DatasetH
    module_path: qlib.data.dataset
    kwargs:
      handler:
        class: Alpha158
        module_path: qlib.contrib.data.handler
        kwargs: *data_handler_config
      segments:
        train: [2008-01-01, 2014-12-31]
        valid: [2015-01-01, 2016-12-31]
        test: [2017-01-01, 2020-08-01]
  record:
    - class: SignalRecord
      module_path: qlib.workflow.record_temp
      kwargs:
        model: <MODEL>
        dataset: <DATASET>
    - class: SigAnaRecord
      module_path: qlib.workflow.record_temp
      kwargs:
        ana_long_short: False
        ann_scaler: 252
    - class: PortAnaRecord
      module_path: qlib.workflow.record_temp
      kwargs:
        config: *port_analysis_config
```

## 锚点与引用

- `&market` 定义锚点
- `*market` 引用锚点

证据：`docs/component/workflow.rst:121`。

## init_instance_by_config

按 `class`/`module_path`/`kwargs` 实例化：

```python
from qlib.utils import init_instance_by_config

model = init_instance_by_config({
    "class": "LGBModel",
    "module_path": "qlib.contrib.model.gbdt",
    "kwargs": {"loss": "mse"}
})
```

证据：`qlib/utils/mod.py:176`。

## 占位符

| 占位符 | 替换时机 | 替换为 |
|--------|---------|--------|
| `<PRED>` | 运行时 | 预测信号 |
| `<MODEL>` | 运行时 | 训练后的模型 |
| `<DATASET>` | 运行时 | 数据集对象 |

证据：`trainer.py:63`。

## BASE_CONFIG_PATH

0.9.2 起支持继承基础配置：

```yaml
BASE_CONFIG_PATH: "base_config.yaml"
```

证据：`run.py:110-133`。

## sys.path 扩展

```yaml
sys_path:
  - "./custom_modules"
sys_rel_path:
  - "./relative_modules"
```

证据：`run.py:30-49`。

## trusted: true（0.9.8+）

本地 `.py` 的 `module_path` 须加 `trusted: true`：

```yaml
model:
  class: MyModel
  module_path: ./my_model.py
  trusted: true
  kwargs: {}
```

嵌套组件各自声明。证据：`docs/component/workflow.rst:246-272`。
