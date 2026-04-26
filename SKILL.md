---
name: tushare-proxy
version: "1.0.0"
description: 使用第三方 Tushare API 代理编写 Python 股票数据代码。当用户要求获取A股行情、基本面、财务数据、指数，或提到 tushare/股票数据/行情接口 时使用此 skill。提供 proxy 连接模板、API 选择决策树、完整代码示例。
user-invocable: true
argument-hint: 'tushare-proxy 查茅台日线 | tushare-proxy 测试连通性'
allowed-tools: Bash, Read, Write
license: MIT
---

# Tushare Proxy Skill

教会 agent 使用第三方代理的 Tushare API 编写 Python 代码获取股票数据。

## 隐私规则

token 从 `TUSHARE_PROXY_TOKEN` 环境变量读取，绝不打印、不硬编码。检查环境变量时只报告"已设置/未设置"。

## 1. 标准连接代码（每次必须）

编写任何 tushare 数据获取代码，开头固定以下 preamble：

```python
import os
import tushare as ts

token = os.environ.get("TUSHARE_PROXY_TOKEN")
if not token:
    raise RuntimeError("TUSHARE_PROXY_TOKEN 未设置")
url = os.environ.get("TUSHARE_PROXY_URL")
if not url:
    raise RuntimeError("TUSHARE_PROXY_URL 未设置")

pro = ts.pro_api(token)
pro._DataApi__token = token
pro._DataApi__http_url = url
```

> `_DataApi__token` / `_DataApi__http_url` 是 name-mangled 私有属性，依赖 tushare 内部实现。两行缺一不可。

## 2. API 决策树

用户想获取什么 → 用什么接口：

| 用户需求 | 接口 | 调用示例 |
|----------|------|----------|
| 股票基本信息/列表 | `stock_basic` | `pro.stock_basic(exchange='SSE', list_status='L')` |
| 日线行情 (OHLCV) | `daily` | `pro.daily(ts_code='000001.SZ', start_date='20240101', end_date='20240115')` |
| 周线/月线行情 | `weekly` / `monthly` | 参数同 daily |
| 每日估值指标 (PE/PB) | `daily_basic` | `pro.daily_basic(ts_code='000001.SZ', start_date='20240101', end_date='20240115')` |
| 指数日线 | `index_daily` | `pro.index_daily(ts_code='000001.SH', start_date='20240101', end_date='20240115')` |
| 利润表 | `income` | `pro.income(ts_code='000001.SZ', start_date='20230101', end_date='20231231')` |
| 资产负债表 | `balancesheet` | 参数同 income |
| 现金流量表 | `cashflow` | 参数同 income |
| 复权因子 | `adj_factor` | `pro.adj_factor(ts_code='000001.SZ', start_date='20240101', end_date='20240115')` |
| 交易日历 | `trade_cal` | `pro.trade_cal(exchange='SSE', start_date='20240101', end_date='20240131')` |
| 停复牌信息 | `suspend_d` | `pro.suspend_d(ts_code='000001.SZ', start_date='20240101', end_date='20240115')` |
| 涨跌停价格 | `stk_limit` | `pro.stk_limit(ts_code='000001.SZ', start_date='20240101', end_date='20240115')` |
| 股票曾用名 | `namechange` | `pro.namechange(ts_code='000001.SZ')` |

## 3. 股票代码格式

```
ts_code = "代码.交易所后缀"
```

| 交易所 | 后缀 | exchange 参数值 | 示例 |
|--------|------|-----------------|------|
| 上交所 | `.SH` | `'SSE'` | `600000.SH` 浦发银行 |
| 深交所 | `.SZ` | `'SZSE'` | `000001.SZ` 平安银行 |
| 北交所 | `.BJ` | `'BSE'` | `430047.BJ` |

**注意** `ts_code` 后缀 (`.SH`/`.SZ`/`.BJ`) 和 `exchange` 参数值 (`'SSE'`/`'SZSE'`/`'BSE'`) 不同，不要混用。

指数代码同理：上证综指 `000001.SH`，深证成指 `399001.SZ`，沪深300 `000300.SH`，中证500 `000905.SH`。

## 4. 重要约定

- **日期**: 所有日期用 `YYYYMMDD` 字符串（如 `'20240115'`），不是 datetime 对象
- **交易日期**: 取最近交易日先调 `trade_cal` 查 `is_open=1` 的最大日期，不要直接用 `today`
- **复权**: `daily` 返回不复权价格，后复权需要 × `adj_factor` 的前复权因子
- **分页**: 加 `limit` 和 `offset` 翻页，如 `limit=5000, offset=5000` 取第二页
- **fields**: 不传返回全部字段，传 `fields='ts_code,trade_date,close'` 只取指定列
- **积分限制**: 三方代理积分有限，`moneyflow`/`hk_daily`/`us_daily` 等高级接口可能不可用

## 5. `pro_bar()` 快捷接口

一行取行情+复权，比 `daily` + `adj_factor` 手动合并更方便：

```python
# 后复权日线
df = ts.pro_bar(ts_code='000001.SZ', adj='qfq', start_date='20240101', end_date='20240115')
# adj: None=不复权, 'qfq'=前复权, 'hfq'=后复权

# 批量多只股票
df = ts.pro_bar(ts_code='000001.SZ,600000.SH', adj='qfq', start_date='20240101', end_date='20240115')
```

## 6. 完整分析场景模板

当用户要求"分析某股票近期走势"时，输出以下模式的代码：

```python
import os
import tushare as ts
import pandas as pd

# --- 连接（固定 preamble）---
token = os.environ.get("TUSHARE_PROXY_TOKEN")
if not token:
    raise RuntimeError("TUSHARE_PROXY_TOKEN 未设置")
url = os.environ.get("TUSHARE_PROXY_URL")
if not url:
    raise RuntimeError("TUSHARE_PROXY_URL 未设置")
pro = ts.pro_api(token)
pro._DataApi__token = token
pro._DataApi__http_url = url

# --- 取数据 ---
code = "600519.SH"  # 贵州茅台
df = ts.pro_bar(ts_code=code, adj='qfq', start_date='20240101', end_date='20240630')
df = df.sort_values('trade_date')

# --- 简单分析 ---
df['ma5'] = df['close'].rolling(5).mean()
df['ma20'] = df['close'].rolling(20).mean()
print(f"{code} 最近5日收盘:\n{df[['trade_date','close','ma5','ma20']].tail()}")
```

## 7. 异常处理

```python
from requests.exceptions import Timeout, ConnectionError as ReqConnectionError

try:
    df = pro.xxx(...)
    if df is None or len(df) == 0:
        print("返回空数据，可能日期无交易或代码错误")
except Timeout:
    print("请求超时，检查网络或 URL")
except ReqConnectionError:
    print("无法连接，检查 TUSHARE_PROXY_URL")
except Exception as e:
    msg = str(e).lower()
    if "token" in msg or "permission" in msg:
        print(f"认证/权限错误: {e}")
    else:
        print(f"接口调用失败: {e}")
```

## 8. 首次连通性验证（可选）

首次使用或怀疑连接有问题时，运行：

```bash
if [ -n "${TUSHARE_PROXY_TOKEN:-}" ]; then echo "TUSHARE_PROXY_TOKEN: 已设置"; else echo "TUSHARE_PROXY_TOKEN: 未设置"; fi
if [ -n "${TUSHARE_PROXY_URL:-}" ]; then echo "TUSHARE_PROXY_URL: 已设置"; else echo "TUSHARE_PROXY_URL: 未设置"; fi
```

然后跑简单接口测试。只报告成功/失败和记录数，不输出 token。

## 9. 文档链接

- 全部 API 列表: https://tushare.pro/document/2
- 具体接口文档: https://tushare.pro/document/2?doc_id=<编号>
- Python SDK 指南: https://tushare.pro/document/1?doc_id=40
