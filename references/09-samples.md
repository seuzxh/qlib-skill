# 09. 官方 Samples 速查

官方 examples 目录：`https://github.com/microsoft/qlib/tree/main/examples`。写 qlib 代码前先在此找最接近的 sample，不要从零搭。

**版本铁律**：sample 必须与本机安装的 qlib revision 对齐。main、tagged release、PyPI 三者有差异。证据：`examples/README.md`。

## 一、examples 目录地图

### 入门与工作流

| 路径 | 作用 | 入口 | 何时参考 |
|------|------|------|---------|
| `workflow_by_code.py` | 代码式工作流（不写 yaml） | 直接运行 | 需要在 Python 中串联训练/记录/回测 |
| `workflow_by_code.ipynb` | 同上的 notebook 版 | jupyter | 交互式研究 |
| `tutorial/detailed_workflow.ipynb` | 端到端教程：数据探查→表达式→自定义模型→策略→回测 | jupyter | 第一次理解全链路 |
| `run_all_model.py` | 批量跑全部 benchmark 的 fire CLI | `python run_all_model.py run ...` | 批量复现/对比模型 |

### 模型基准 benchmarks/

24 个模型，每个目录含 `workflow_config_<model>_<dataset>.yaml`，直接 `qrun`：

| 类别 | 模型 |
|------|------|
| 树模型 | LightGBM、XGBoost、CatBoost、DoubleEnsemble |
| 前馈/线性 | Linear、MLP |
| RNN 系 | LSTM、GRU、ALSTM、KRNN、TCN |
| Transformer/注意力 | Transformer、GATs、Localformer、TFT、TabNet、TRA、Sandwich |
| 其他 | IGMTF、SFM、ADARNN、HIST、TCTS、GeneralPtNN、ADD |

LightGBM 目录额外含进阶配置：`workflow_config_lightgbm_multi_freq.yaml`（多频率数据源）、`workflow_config_lightgbm_configurable_dataset.yaml`（可配置数据集）、`multi_freq_handler.py`、`features_sample.py`（特征采样）。

### 动态模型

| 路径 | 作用 |
|------|------|
| `benchmarks_dynamic/DDG-DA/` | 概念漂移下的动态模型，`workflow.py` 为入口，含独立 Makefile 与缓存目录，升级前读其 README 的 trust 说明 |
| `benchmarks_dynamic/baseline/` | DDG-DA 的滚动基线 |

### 回测与组合

| 路径 | 作用 | 关键类/文件 |
|------|------|------------|
| `nested_decision_execution/` | 日/30分/5分多层嵌套执行 | `NestedExecutor` + `TWAPStrategy`/`SBBStrategyEMA` |
| `portfolio/` | 增强指数组合优化 | `EnhancedIndexingStrategy`，需自配指数权重与风险模型 |
| `highfreq/` | 高频数据与高频预测 | `highfreq_handler.py`、`highfreq_ops.py`（自定义算子样例） |
| `orderbook_data/` | 订单簿数据集构建 | `create_dataset.py`、`example.py` |

### 数据与训练工程

| 路径 | 作用 | 关键点 |
|------|------|--------|
| `data_demo/` | Provider 缓存与内存复用 | `data_cache_demo.py`、`data_mem_resuse_demo.py` |
| `rolling_process_data/` | 滚动窗口数据处理 | DataHandler-based DataLoader 出原始特征，Processor 出滚动相关特征，避免重复生成 |
| `model_rolling/` | 滚动训练任务管理 | `task_manager_rolling.py`，`RollingTaskExample` |
| `online_srv/` | 在线更新/在线管理 | `update_online_pred.py`、`online_management_simulate.py`、`rolling_online_management.py`（需 MongoDB） |
| `hyperparameter/LightGBM/` | 超参搜索样例 | 调参流程参考 |
| `model_interpreter/` | 模型解释 | `feature.py` 特征重要性 |

### 强化学习

| 路径 | 作用 |
|------|------|
| `rl/simple_example.ipynb` | `qlib.rl` 最小示例（0.9.0+） |
| `rl_order_execution/` | RL 订单执行，`exp_configs/` + `scripts/` |

## 二、代码式工作流最小模板

yaml 能做的一切都能用代码做。标准序列见 `examples/workflow_by_code.py`：

```python
import qlib
from qlib.constant import REG_CN
from qlib.utils import init_instance_by_config, flatten_dict
from qlib.workflow import R
from qlib.workflow.record_temp import SignalRecord, SigAnaRecord, PortAnaRecord
from qlib.tests.data import GetData
from qlib.tests.config import CSI300_BENCH, CSI300_GBDT_TASK

if __name__ == "__main__":
    provider_uri = "~/.qlib/qlib_data/cn_data"
    GetData().qlib_data(target_dir=provider_uri, region=REG_CN, exists_skip=True)
    qlib.init(provider_uri=provider_uri, region=REG_CN)

    model = init_instance_by_config(CSI300_GBDT_TASK["model"])
    dataset = init_instance_by_config(CSI300_GBDT_TASK["dataset"])

    # dataset 可独立使用：train/valid/test 三段
    # print(dataset.prepare("train").head())

    port_analysis_config = {
        "executor": {
            "class": "SimulatorExecutor",
            "module_path": "qlib.backtest.executor",
            "kwargs": {"time_per_step": "day", "generate_portfolio_metrics": True},
        },
        "strategy": {
            "class": "TopkDropoutStrategy",
            "module_path": "qlib.contrib.strategy.signal_strategy",
            "kwargs": {"signal": (model, dataset), "topk": 50, "n_drop": 5},
        },
        "backtest": {
            "start_time": "2017-01-01",
            "end_time": "2020-08-01",
            "account": 100000000,
            "benchmark": CSI300_BENCH,
            "exchange_kwargs": {
                "freq": "day",
                "limit_threshold": 0.095,
                "deal_price": "close",
                "open_cost": 0.0005,
                "close_cost": 0.0015,
                "min_cost": 5,
            },
        },
    }

    with R.start(experiment_name="workflow"):
        R.log_params(**flatten_dict(CSI300_GBDT_TASK))
        model.fit(dataset)
        R.save_objects(**{"params.pkl": model})

        recorder = R.get_recorder()
        SignalRecord(model, dataset, recorder).generate()   # pred.pkl / label.pkl
        SigAnaRecord(recorder).generate()                    # IC / RankIC
        PortAnaRecord(recorder, port_analysis_config, "day").generate()
```

**铁律**：
- 三个 record 模板的调用顺序固定：Signal → SigAna → PortAna。后两者依赖前者产物。
- 代码式里 `signal` 可传 `(model, dataset)` 元组；生产/恢复场景传已算好的预测 DataFrame。
- `GetData().qlib_data` 只用于 demo；生产环境用 scripts 下载好的数据。

## 三、批量跑 benchmark

```bash
cd examples
# 查看参数
python run_all_model.py run --help
# 跑单个模型（fire CLI，会为每个模型建独立 exp 目录并调 qrun）
python run_all_model.py run --models LightGBM
```

入口是 `fire.Fire(ModelRunner)`，内部拼 `qrun <yaml> <exp_folder>`。证据：`run_all_model.py:365,403`。

## 四、嵌套执行器（多频率）配置骨架

日频选股 + 30 分钟调仓 + 5 分钟 TWAP 执行，三层 `inner_executor` 递归嵌套。见 `examples/nested_decision_execution/workflow.py:161-212`：

```python
executor_config = {
    "class": "NestedExecutor",
    "module_path": "qlib.backtest.executor",
    "kwargs": {
        "time_per_step": "day",
        "inner_executor": {
            "class": "NestedExecutor",
            "module_path": "qlib.backtest.executor",
            "kwargs": {
                "time_per_step": "30min",
                "inner_executor": {
                    "class": "SimulatorExecutor",
                    "module_path": "qlib.backtest.executor",
                    "kwargs": {"time_per_step": "5min"},
                },
                "inner_strategy": {
                    "class": "TWAPStrategy",
                    "module_path": "qlib.contrib.strategy.rule_strategy",
                },
            },
        },
    },
}
```

- 规则策略在 `qlib.contrib.strategy.rule_strategy`（`TWAPStrategy`、`SBBStrategyEMA`），不在 signal_strategy。
- 多频率回测产物按频率取：`rec.load_object("portfolio_analysis/report_normal_30minute.pkl")`。
- 嵌套时外层策略只见外层执行器，这是分层铁律的直接体现。

## 五、benchmark 基线指标（校验用）

复现实验时用于判断结果是否正常。csi300 + Alpha158 + LightGBM（2017-01 至 2020-08，来自 `benchmarks/README.md`）：

| 指标 | 参考值 |
|------|--------|
| IC | 0.0448 |
| ICIR | 0.3660 |
| Rank IC | 0.0469 |
| Annualized Return | 0.0901 |
| Information Ratio | 1.0164 |
| Max Drawdown | -0.1038 |

跨 OS 年化收益差异 <2% 属正常。证据：`examples/README.md` NOTE。

## 六、sample 使用安全要点

- 0.9.8+ 本地 `.py` 组件在 yaml 中引用时各自声明 `trusted: true`；包导入不需要。证据：`examples/README.md`。
- `online_srv` 三个 CLI 有 `--trusted`，默认 False；`main`/`first_run` 会重置实验数据，先备份。
- 不要对拒绝的 artifact 加载自动重试加 trust；先核实写入方与存储权限。
- portfolio sample 的指数权重数据是手工构造的，非官方公开数据源。
