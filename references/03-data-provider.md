# 03. 数据格式与 Provider

## 存储约定

```text
~/.qlib/qlib_data/cn_data/
├── calendars/
│   └── day.txt              # 交易日历
├── instruments/
│   ├── csi300.txt           # 股票池及存续区间
│   └── csi500.txt
└── features/
    └── sh600000/
        ├── close.day.bin    # 单字段二进制序列
        ├── open.day.bin
        └── ...
```

证据：`docs/component/data.rst:571-604`。

## 字段命名

- 字段名以 `$` 开头：`$close`、`$open`、`$high`、`$low`、`$volume`、`$factor`。
- 价格为前复权。原始价 = `$close / $factor`。
- 停牌日 OHLCV 与 factor 置 NaN。

证据：`docs/component/data.rst:316`、`data.rst:55-56`、`data.rst:197`。

## Provider 结构

`D` 是 `Wrapper` 代理，按配置委派：

| Provider | 职责 | 源码路径 |
|---------|------|---------|
| `LocalCalendarProvider` | 日历 | `data.py:639` |
| `LocalInstrumentProvider` | 股票池 | `data.py:680` |
| `LocalFeatureProvider` | 特征 | `data.py:728` |
| `LocalExpressionProvider` | 表达式 | `data.py:835` |

## 区域默认值

| 区域 | trade_unit | limit_threshold |
|------|-----------|-----------------|
| cn | 100 | 0.095 |
| us | 1 | None |

证据：`qlib/config.py:316-329`。

## DataHandler 语义

- `fit_start_time`/`fit_end_time`：处理器拟合区间。**不得包含测试期信息**。
- `DataHandlerLP` 产出三份数据：
  - `DK_R`：raw
  - `DK_I`：infer
  - `DK_L`：learn

证据：`handler.py:51-54,392-394`。

## 处理器类型

| 类型 | 含义 | learn 数据 = shared + ? |
|------|------|------------------------|
| `infer_process_type` / `PTYPE_I` | independent | learn 处理器 |
| `learn_process_type` / `PTYPE_A` | append | infer + learn 处理器 |

证据：`handler.py:425-434`。

`Alpha158` 默认 `process_type=PTYPE_A`，learn 处理器为 `DropnaLabel` 与 `CSZScoreNorm(label)`。证据：`qlib/contrib/data/handler.py:38-39,109`。

## 标签约定

官方标签：`Ref($close,-2)/Ref($close,-1)-1`

原因：T 日收盘后 T+1 才能买，T+1 收盘才能确定收益。证据：`docs/component/data.rst:488`。

## 常用处理器

| 处理器 | 作用 | 源码路径 |
|-------|------|---------|
| `Fillna` | 填充 NaN | `processor.py:179` |
| `DropnaLabel` | 删除 label 为 NaN 的行 | `processor.py` |
| `CSZScoreNorm` | 截面 ZScore 归一化 | `processor.py` |
| `CSRankNorm` | 截面排名归一化 | `processor.py` |
| `RobustZScoreNorm` | 稳健 ZScore | `processor.py` |
