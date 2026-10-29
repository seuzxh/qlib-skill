# 05. 回测与评估

## 主接口

```python
from qlib.backtest import backtest

portfolio_metric_dict, indicator_dict = backtest(
    start_time="2017-01-01",
    end_time="2020-08-01",
    strategy=strategy_config,
    executor=executor_config,
    benchmark="SH000300",
    account=100000000,
    exchange_kwargs={
        "limit_threshold": 0.095,
        "deal_price": "close",
        "open_cost": 0.0005,
        "close_cost": 0.0015,
        "min_cost": 5,
    }
)
```

证据：`qlib/backtest/__init__.py:217-226`。

## 返回值

- `portfolio_metric_dict`：按频率键取值，`portfolio_metric_dict.get("1day")` 得 `(report_df, positions_df)`。
- `indicator_dict`：按频率键取值。

证据：`__init__.py:259-263`。

## 执行器配置

默认 `SimulatorExecutor(time_per_step="day", generate_portfolio_metrics=True)`。证据：`qlib/workflow/record_temp.py:433-440`。

## exchange_kwargs 键

| 键 | 说明 | 默认值 |
|---|------|--------|
| `limit_threshold` | 涨跌停阈值 | None |
| `deal_price` | 成交价 | close |
| `open_cost` | 开仓费率 | 0.0005 |
| `close_cost` | 平仓费率 | 0.0015 |
| `min_cost` | 最低费用 | 5 |
| `volume_threshold` | 成交量限制 | None |

证据：`qlib/backtest/exchange.py:44-50`。

## 禁止使用的接口

`backtest_daily` 是 legacy 兼容接口，禁止新代码使用。证据：`qlib/contrib/evaluate.py:146-147`。

## PortfolioMetrics

```python
from qlib.backtest.report import PortfolioMetrics

pm = PortfolioMetrics()
pm.update_portfolio_metrics_record(...)
```

证据：`qlib/backtest/report.py:153`。

## Indicator

```python
from qlib.backtest.report import Indicator

ind = Indicator()
ind.update_order_indicators(trade_info)
ind.record(trade_start_time)
```

指标集：amount、deal_amount、ffr、pa、pos 等。证据：`report.py:255-268`。

## 指标聚合

```python
from qlib.contrib.evaluate import indicator_analysis

indicator_analysis(df, method="mean")  # 或 amount_weighted / value_weighted
```

证据：`qlib/contrib/evaluate.py:96-119`。

## risk_analysis

```python
from qlib.contrib.evaluate import risk_analysis

analysis_df = risk_analysis(
    r=report["return"] - report["bench"] - report["cost"],
    N=252,
    freq="day",
    mode="sum"  # 或 "product"（0.9.7+）
)
```

输出指标：mean、std、annualized_return、information_ratio、max_drawdown。证据：`qlib/contrib/evaluate.py:26`。

## 超额收益口径

- 不含费：`report["return"] - report["bench"]`
- 含费：`report["return"] - report["bench"] - report["cost"]`

证据：`docs/component/strategy.rst:246-251`。

## 累计指标

按求和累加，避免指数扭曲。证据：`docs/component/report.rst:23-24`。

## 个股利润归因

见 `qlib/backtest/profit_attribution.py`。

## 图形报告

```python
from qlib.contrib.report import GRAPH_NAME_LIST
```

证据：`docs/component/report.rst:47-49`。
