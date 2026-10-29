---
name: qlib
description: |
  qlib 量化框架架构规范与最佳实践。MANDATORY: 所有 qlib 相关代码必须先读本文件，再按需查 references。

  触发场景：
  - 数据层：DataHandler、Expression、Operator、Dataset、Alpha158/Alpha360、数据准备
  - 模型层：Model、ModelFT、训练、预测、信号生成
  - 策略层：BaseStrategy、TopkDropoutStrategy、WeightStrategyBase、信号策略
  - 执行层：backtest、SimulatorExecutor、Exchange、交易规则
  - 评估层：PortfolioMetrics、Indicator、risk_analysis、收益归因
  - 记录层：R.start_exp、log_params、log_metrics、save_objects、SignalRecord/SigAnaRecord/PortAnaRecord
  - 工作流：qrun、yaml 配置、init_instance_by_config、占位符 <MODEL>/<DATASET>/<PRED>、代码式工作流
  - 示例：找官方 sample、批量跑 benchmark、嵌套执行器多频率、组合优化、滚动/在线训练、高频、强化学习
  - 排查：表达式结果异常、数据泄露、性能慢、回测失真

  禁止触发：
  - 纯 pandas/numpy/sklearn 代码，与 qlib 无关
  - 文档翻译、代码格式化、重命名
  - 非量化任务
---

# qlib 架构规范

本技能包固化 qlib 的分层架构铁律与高频踩坑点。写任何 qlib 代码前必须读完本文件；命中细则时再读对应 references。

## 六层分层铁律

qlib 分六层，每层职责单一，禁止跨层直接调用。

| 层 | 职责 | 关键类 | 边界 |
|---|---|---|---|
| 数据层 | 原始数据存取与表达式计算 | `LocalProvider`、`Expression`、`DataHandler` | 只产出 DataFrame，不知模型存在 |
| 模型层 | 训练与预测 | `Model.fit/predict`、`ModelFT.finetune` | 入参只有 `Dataset`，不碰 Provider |
| 策略层 | 信号转为交易决策 | `BaseStrategy.generate_trade_decision`、`TopkDropoutStrategy` | 只消费 `Signal`，不直接读行情 |
| 执行层 | 撮合与账户 | `BaseExecutor`、`SimulatorExecutor`、`backtest()` | 只执行 `TradeDecision`，不预测 |
| 记录层 | 实验跟踪 | `QlibRecorder`、`Experiment`、`Recorder` | 只记录产物，不参与计算 |
| 评估层 | 报告与指标 | `PortfolioMetrics`、`Indicator`、`risk_analysis` | 只读回测结果 |

**跨层调用即错误。** 数据流向单向：`Dataset → Model.fit(dataset)`。策略不取数，预测经 `Signal` 注入。

## 表达式引擎铁律

- `Ref($close, N)`：`N>0` 为 N 期前，`N<0` 为未来数据。特征只用正 N。
- `Ref($close, 0)` 计算不准确，源码有告警。禁止用于取首日。
- 嵌套 `Ref(Ref($close,-1),1)` 窗口推导会错。拆分改写。
- 无 `RollingMean` 算子。滚动均值算子名是 `Mean`。
- 无 `fillna` 算子。填充用处理器 `Fillna` 或算子 `If`。
- 不支持链式比较。写 `($close > 0) & ($close < 10)`，不写 `0 < $close < 10`。
- 表达式对象不可下标。`[$close][0]` 合法，`$close[0]` 非法。
- Python `and/or` 连接表达式被拒绝。用 `&`、`|` 加括号，或用 `If`。
- 表达式与 DataFrame 不混用。表达式在 `D.features` 或 DataLoader 内求值，出数后才是 pandas。

## 数据格式铁律

- 价格为前复权。原始价 = `$close / $factor`。
- 停牌日 OHLCV 与 factor 置 NaN。
- `fit_start_time`/`fit_end_time` 是处理器拟合区间，**不得包含测试期信息**。
- 官方标签约定 `Ref($close,-2)/Ref($close,-1)-1`，因 T 日收盘后 T+1 才能买。
- `provider_uri` 默认 `~/.qlib/qlib_data/cn_data`。

## 工作流铁律

- yaml 顶层 section 固定：`qlib_init`、`market`、`benchmark`、`data_handler_config`、`port_analysis_config`、`task`。
- 锚点 `&market` / 引用 `*market` 复用配置。
- `init_instance_by_config` 按 `class`/`module_path`/`kwargs` 实例化。
- 占位符 `<PRED>`、`<MODEL>`、`<DATASET>` 运行时替换。
- 0.9.8 起本地 `.py` 的 `module_path` 须加 `trusted: true`。

## 回测铁律

- 主接口 `qlib.backtest.backtest(start_time, end_time, strategy, executor, ...)`。
- 返回 `(portfolio_metric_dict, indicator_dict)`，按频率键取值。
- `backtest_daily` 是 legacy 兼容接口，禁止新代码使用。
- 超额收益口径：`report["return"] - report["bench"]`（不含费），再减 `report["cost"]`（含费）。

## Recorder 铁律

- 导入：`from qlib.workflow import R`。
- `R.start_exp(experiment_name=..., recorder_name=...)` 开实验。
- `R.log_params(**kwargs)`、`R.log_metrics(step=None, **kwargs)`。
- `R.save_objects(**{"params.pkl": model})` 存产物。
- 0.9.8 起 `load_object` 默认受限，可执行对象须 `trusted=True`。

## 版本兼容铁律

- 0.9.8 表达式求值改为受限解析器。旧自定义算子须回归测试。
- 0.9.8 加载历史 artifact 须显式 `trusted=True`。
- 升级前必须读 `docs/start/artifact_migration.rst` 与 `docs/start/config_migration.rst`。

## 使用流程

1. 读 SKILL.md，确认任务落在哪一层。
2. 命中细则时读对应 references 文件。
3. 写代码时对照规范，不凭空捏造 API。
4. 交付时在回复末尾写 `### qlib 技能包引用`，披露应用了哪条规范、防住了什么坑。

## references 索引

| 文件 | 内容 |
|------|------|
| `01-architecture.md` | 六层分层详解、跨层铁律、数据流向、源码路径 |
| `02-expression.md` | 表达式语法、算子清单、窗口推导、性能陷阱 |
| `03-data-provider.md` | Provider 结构、存储格式、DataHandler 语义、处理器类型 |
| `04-workflow.md` | qrun、yaml 配置、占位符、模块导入、嵌套配置 |
| `05-backtest-evaluate.md` | backtest 接口、PortfolioMetrics、Indicator、risk_analysis、归因 |
| `06-recorder.md` | R 对象、实验层级、log 接口、record 模板、产物加载 |
| `07-antipatterns.md` | 16 条反模式对照表，含错误现象、正确做法、出处 |
| `08-version-compat.md` | 0.9 系列变更点、迁移指南、已知风险 |
| `09-samples.md` | 官方 examples 24 目录地图、代码式工作流模板、嵌套执行器片段、benchmark 基线指标 |
