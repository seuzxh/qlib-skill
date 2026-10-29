# 02. 表达式引擎

## 机制

- `Expression` 重载运算符构造 DAG 节点。证据：`qlib/data/base.py:32-140`。
- 求值按日历整数索引加载，非 DateTime。证据：`base.py:142-165`。
- 窗口需求由 `get_extended_window_size` 递归推导。证据：`base.py:222-235`。
- 0.9.8 起表达式为受限语法。证据：`qlib/data/expression_parser.py`。
- 自定义算子须注册：`qlib.init(custom_ops=[...])`。证据：`tests/test_register_ops.py:54`。

## 算子速查

| 需求 | 算子 | 说明 |
|------|------|------|
| 滚动均值 | `Mean($close, N)` | 不是 `RollingMean` |
| N 期前值 | `Ref($close, N)` | N>0 为历史，N<0 为未来 |
| 条件取值 | `If(cond, a, b)` | 替代 fillna 的部分场景 |
| 最大值 | `Max($high, N)` | 滚动窗口 |
| 最小值 | `Min($low, N)` | 滚动窗口 |
| 标准差 | `Std($close, N)` | 滚动窗口 |
| 排名 | `Rank($close, N)` | 截面排名 |
| 相关性 | `Corr($close, $volume, N)` | 滚动相关 |
| 斜率 | `Slope($close, N)` | 线性回归斜率 |
| 变化率 | `ROC($close, N)` | N 期变化率 |

完整算子清单见 `qlib/data/ops.py`。

## 常见坑

| 错误写法 | 正确写法 | 原因 |
|---------|---------|------|
| `Ref($close, 0)` | 避免使用 | 计算不准确，源码有告警 |
| `Ref(Ref($close,-1),1)` | 拆分改写 | 窗口推导不支持嵌套 |
| `0 < $close < 10` | `($close > 0) & ($close < 10)` | 不支持链式比较 |
| `$close[0]` | `[$close][0]` | 表达式不可下标 |
| `$close > 0 and $close < 10` | `($close > 0) & ($close < 10)` | `and/or` 被拒绝 |
| `fillna($close, 0)` | `If(IsNull($close), 0, $close)` | 无 fillna 算子 |
| `RollingMean($close, 5)` | `Mean($close, 5)` | 算子名不同 |

## 性能陷阱

- 求值结果走内存缓存 `H["f"]`，键含表达式串与区间。证据：`base.py:187-203`。
- 深嵌套 DAG 逐层递归 `load`，每层单独缓存与取数。证据：`base.py:142-203`。
- 避免深层嵌套：超过 10 层的表达式建议拆分为中间变量。
- DataHandler 取数用 `col_set=CS_RAW` 可避免 `loc` 拷贝。证据：`handler.py:99-100`。

## 窗口推导规则

`get_extended_window_size` 递归推导表达式所需的历史窗口：

- `Ref($close, N)` 需要 N 期历史
- `Mean($close, N)` 需要 N-1 期历史
- 复合表达式取各分支最大值

**注意**：`Ref(Ref($close,-1),1)` 的推导会错，因为 `Ref` 的 N<0 分支不按常规递归。证据：`base.py:215`。
