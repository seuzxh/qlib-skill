# 07. 常见错误与反模式

## 表达式相关

| # | 错误现象 | 正确做法 | 出处 |
|---|---|---|---|
| 1 | 用 `Ref($close,-1)` 取"昨日"特征 | 负数 N 是未来数据；特征用正 N | `qlib/data/ops.py:789` |
| 2 | 用 `Ref($close,0)` 取首日 | 该用法计算不准，源码有告警 | `ops.py:816-819` |
| 3 | 嵌套 `Ref(Ref($close,-1),1)` | 窗口推导不支持，拆分改写 | `qlib/data/base.py:215` |
| 4 | 写 `0<$close<10` 或 `$close[0]` | 用 `($close>0)&($close<10)`、`[$close][0]` | `docs/advanced/alpha.rst:126,142` |
| 5 | 用 `and/or` 连表达式 | 用 `&`/`|` 加括号或 `If` | `alpha.rst:141-144` |
| 6 | 找 `fillna`/`RollingMean` | 算子名是 `Mean`；填充用处理器 `Fillna` | `qlib/data/ops.py:827`；`processor.py:179` |

## 数据相关

| # | 错误现象 | 正确做法 | 出处 |
|---|---|---|---|
| 7 | 标签写 `Ref($close,-1)/$close-1` | 用 `Ref($close,-2)/Ref($close,-1)-1`（T+1 成交） | `docs/component/data.rst:488` |
| 8 | `fit_end_time` 覆盖测试期 | 归一化拟合区间只含训练段 | `qlib/data/dataset/processor.py:198-199` |
| 9 | benchmark 与 market 混用 | benchmark 是单只指数（SH000300）；market 是股票池（csi300） | `docs/component/strategy.rst:198-204` |

## 回测相关

| # | 错误现象 | 正确做法 | 出处 |
|---|---|---|---|
| 10 | 主用 `backtest_daily` 做回测 | 用 `qlib.backtest.backtest`；前者为 legacy | `qlib/contrib/evaluate.py:146-147` |
| 11 | 策略构造传 `model`+`dataset` | 已弃用；统一传 `signal` | `qlib/contrib/strategy/signal_strategy.py:58-61` |
| 12 | `drop_raw=True` 后取 raw label | raw label 不可得，需保留或重新计算 | `record_temp.py:190-194` |
| 13 | 预测分数做尺度归一 | 策略自行处理，qlib 不归一 | `strategy.rst:131-135` |

## 配置相关

| # | 错误现象 | 正确做法 | 出处 |
|---|---|---|---|
| 14 | 照搬文档 `module_path: qlib.contrib.strategy.strategy` | 实际为 `qlib.contrib.strategy`；以仓库 yaml 为准 | `docs/component/workflow.rst:52` 对 `examples/.../workflow_config_lightgbm_Alpha158.yaml:15` |

## 环境相关

| # | 错误现象 | 正确做法 | 出处 |
|---|---|---|---|
| 15 | 在 qlib 源码目录里 import qlib | 报 `No module named qlib.data._libs.rolling`；换目录或 `python setup.py build_ext --inplace` | `docs/FAQ/FAQ.rst:74-100` |
| 16 | Windows 下直接跑 `D.features` | 多进程须包 `if __name__ == "__main__"` | `docs/FAQ/FAQ.rst:15-51` |
