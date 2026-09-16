# A股情绪指数计算脚本

本文提供可复制运行的Python脚本，版本v1.0.0。公式、历史案例、本周实测及引用见《A股情绪指数公式说明书》。配套复算包包含同一脚本、冻结行情与校验信息，可在公网接口不可用时复算。

## 1. 能完成什么

- 从公开接口尝试取得中证全指000985日线，或读取已验证的离线快照。
- 计算20日趋势、下行半波动、过去756日分位和独立价格参考分P。
- 接入合格的个股聚合和融资数据后，按固定权重计算完整情绪指数S。
- 生成日度明细、历史代表日参数、周报及日/周预警；保留周中风险。
- 使用`--with-context`补充预设5个风格指数、5个中证800行业代理；接口缺失时标NA。
- 使用`--diagnostic`输出P的回顾性风险诊断，未来标签不进入正式因子。

**只有价格数据时，完整S始终为NA。P=21.16不是本周完整情绪指数。** 融资汇总和海外市场在说明书中作了额外来源核验，当前脚本不自动抓取这些扩展栏目；也没有内置全市场逐股历史下载器。完整股票池、复权、融资范围和首次披露时间须由合格上游数据提供。

## 2. 运行方式

建议Python 3.11或以上。本次测试环境为Python 3.12.14、pandas 3.0.1、numpy 2.3.5。

```powershell
python -m pip install "pandas>=2.2,<4" "numpy>=1.26,<3"
```

解压《A股情绪体系_复算包.zip》，进入包内目录。下面使用已冻结至2026年9月11日的真实全指日线，**不需要网络**：

```powershell
python sentiment.py --index-csv index.csv --index-meta index_meta.json --as-of 2026-09-11 --diagnostic --out result
```

如要同时联网补充行业/风格，加`--with-context`。它仅为辅助抓取，不能以返回旧数据替代目标周。

```powershell
python sentiment.py --index-csv index.csv --index-meta index_meta.json --as-of 2026-09-11 --diagnostic --with-context --out result
```

在线主行情调用：

```powershell
python sentiment.py --as-of 2026-09-11 --with-context --out result_online
```

公开接口本次出现过连接被关闭、返回旧行情等情况。在线命令可能失败，这不是可承诺稳定的服务；失败时程序报错，使用有来源与哈希的离线数据重跑。脚本取得原始序列后会明确截断到`--as-of`，绝不把今天盘中行计入已结束周。

### 完整模型接入

```powershell
python sentiment.py --index-csv index.csv --index-meta index_meta.json --market-csv market.csv --finance-csv finance.csv --manifest market_manifest.json --calendar-csv calendar.csv --as-of 2026-09-11 --out result_full
```

示例文件头只展示结构，不是历史市场数据；单行示例不能产生有效分位数。所有金额均为人民币元，字段可用空单元格表示缺失，不填任意替代值。

```csv
date,n_eligible,n_expected_traded,n_traded,n_up,n_down,n_flat,n_above_ma20,n_ma20,n_drop5,amount_yuan
```

```csv
date,available_date,buy_yuan,repay_yuan
```

```csv
date
```

`market_manifest.json`模板如下。不要仅为绕过检查把标记写成true；必须先完成其声明的核验。`amount_yuan`是完整沪深A股股票成交额，非当前非ST合格池的局部成交额。

```json
{
  "universe": "CN_SH_SZ_A_NONST_60",
  "source": "填写可追溯的数据集、机构与版本",
  "point_in_time": true,
  "includes_delisted": true,
  "st_history_verified": true,
  "adjustment_verified": true,
  "calendar_verified": true,
  "finance_scope": "CN_SH_SZ_A_STOCK_ONLY",
  "finance_availability_verified": true
}
```

`index_meta.json`必须包含`code="000985"`、`source`和当前CSV文件的`sha256`，复算包内已提供匹配文件。修改CSV后必须重新核验身份与来源，并更新校验和；不允许用上证综指冒充全指。

融资`date`是交易日期，`available_date`是以18:00为截止、首次已经可得的报告日期。现版保守要求公布晚于交易日；计算最近可得五日窗口，最多滞后两个交易日。融资余额变化或包含未知基金范围的汇总，不自动等价于正式融资因子。

## 3. 输出文件

| 文件 | 用途 |
|---|---|
| daily_scores.csv | 原始因子、历史分位、各因子有效历史数量、预热标记、权重覆盖率、P及S |
| weekly_report.md | 当周指数表现、日度记录、最高预警及数据缺口 |
| historical_cases.md | 2015/2018/2022/2024等代表日参数 |
| price_diagnostic.md | 可选回顾性诊断；不可当作样本外胜率 |
| run_manifest.json | 版本、时间、输入哈希、依赖版本及日历检查状态 |
| index_source.json / index.csv | 在线抓取时的原始来源与表格 |
| context.json / 各指数_source.txt | 可选行业、风格快照及各项错误状态 |

默认截止为上一个已经结束的周五。若提供独立日历，会在该周选择实际最后交易日；整周休市或未提供节假日日历时，显式设置已核验的`--as-of`。数据最后一日与目标不符即报错，不能用“接口最后一行”掩盖断更。周中截止只称截至日观察。

日度数据至少需要252个历史有效因子值用于排名，使用最多756个交易日位置；中途缺失率超过5%则暂停分数。首次20日趋势/波动的结构性预热不计入有效历史。完整S任一因子不合格即NA，硬价格预警仍可工作。

## 4. 验证与本次运行状态

配套包包含12项主测试和8项独立审查测试。测试涵盖无前视、公布日期、缺失、计数和覆盖、异常行情、周收益基点、周中警报保留、正向剧烈波动等。完整六因子路径使用合成数据验证控制逻辑；真实历史只验证了公开价格分项，不能宣称完整市场模型已回测成功。

```powershell
python test_sentiment.py
python test_independent.py sentiment
```

本次冻结数据复算应得到：2026-09-11中证全指5769.02、周收益约-1.0483%、P约21.1640、完整S为NA。最近640个全指收盘已与腾讯来源交叉核验一致。行业、融资及美股补充材料和时区限制见说明书。

## 5. 完整Python源码

将下方整个代码块保存为`sentiment.py`即可运行；复算包中提供的是同一份代码。
```python
"""A-share sentiment monitor v1.0. Python 3.9+, pandas, numpy.

Public index data supports P (price reference) only. Full S requires point-in-time
market breadth and financing inputs. No fabricated data or silent reweighting.
"""
from __future__ import annotations

import argparse
import hashlib
import json
import re
import sys
import urllib.request
from datetime import datetime, timedelta, timezone
from pathlib import Path
from concurrent.futures import ThreadPoolExecutor

import numpy as np
import pandas as pd

VERSION = "1.0.0"
WEIGHTS = {"breadth": .20, "ma20": .15, "momentum": .20,
           "tail5": .15, "downvol": .20, "leverage": .10}
REVERSE = {"tail5", "downvol"}
WINDOW, MIN_HISTORY = 756, 252
CN = timezone(timedelta(hours=8))
FIELDS = ["date", "open", "close", "high", "low", "volume", "amount",
          "amplitude", "pct_change", "change", "turnover"]
MARKET = ["n_eligible", "n_expected_traded", "n_traded", "n_up", "n_down", "n_flat",
          "n_above_ma20", "n_ma20", "n_drop5", "amount_yuan"]
EVENT_DATES = ["2015-06-26", "2015-07-08", "2015-08-24", "2016-01-07",
               "2018-10-11", "2020-02-03", "2022-04-25", "2022-04-26",
               "2022-10-31", "2024-02-02", "2024-02-05", "2024-02-06"]
ASSETS = [
    ("风格", "sh000300", "沪深300"), ("风格", "sh000852", "中证1000"),
    ("风格", "sh000016", "上证50"), ("风格", "sz399006", "创业板指"),
    ("风格", "sh000688", "科创50"), ("行业", "sh000935", "中证信息技术"),
    ("行业", "sh000928", "中证能源"), ("行业", "sh000934", "中证金融地产"),
    ("行业", "sh000932", "中证主要消费"), ("行业", "sh000933", "中证医药卫生")]


def require(ok, message):
    if not bool(ok):
        raise ValueError(message)


def date_frame(df, date_column="date"):
    require(date_column in df.columns, f"missing {date_column}")
    x = df.copy()
    x[date_column] = pd.to_datetime(x[date_column], errors="raise").dt.normalize()
    require(not x[date_column].isna().any(), "null date")
    require(not x[date_column].duplicated().any(), "duplicate date")
    return x.sort_values(date_column).set_index(date_column)


def numeric(df, columns):
    for col in columns:
        require(col in df.columns, f"missing column: {col}")
        df[col] = pd.to_numeric(df[col], errors="raise")
        require(not np.isinf(df[col]).any(), f"infinite values: {col}")
    return df


def validate_index(df):
    x = numeric(date_frame(df), ["open", "high", "low", "close"])
    require(x[["open", "high", "low", "close"]].notna().all().all(), "missing OHLC")
    require((x[["open", "high", "low", "close"]] > 0).all().all(), "nonpositive OHLC")
    require((x.high >= x[["open", "close", "low"]].max(axis=1)).all(), "invalid high")
    require((x.low <= x[["open", "close", "high"]].min(axis=1)).all(), "invalid low")
    require(len(x) >= 21, "need at least 21 index observations")
    if "pct_change" in x:
        x = numeric(x, ["pct_change"])
        discrepancy = (x.close.pct_change(fill_method=None)*100-x["pct_change"]).abs().iloc[1:]
        require((discrepancy.dropna() <= .011).all(),
                "return disagrees with provider daily change; check missing dates/instrument")
    return x


def fetch_index(as_of, out):
    url = ("https://push2his.eastmoney.com/api/qt/stock/kline/get?secid=1.000985"
           "&klt=101&fqt=0&beg=20100101&end=" + datetime.now(CN).strftime("%Y%m%d") +
           "&fields1=f1,f2,f3,f4,f5,f6"
           "&fields2=f51,f52,f53,f54,f55,f56,f57,f58,f59,f60,f61")
    req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0"})
    with urllib.request.urlopen(req, timeout=40) as r:
        payload = r.read()
    (out / "index_source.json").write_bytes(payload)
    data = json.loads(payload)
    require(data.get("rc") == 0 and data.get("data"), "provider returned no index data")
    item = data["data"]
    require(str(item.get("code")) == "000985" and item.get("market") == 1,
            "wrong instrument; expected CSI All Share 000985")
    rows = [s.split(",") for s in item["klines"]]
    require(all(len(row) == len(FIELDS) for row in rows), "provider schema changed")
    df = pd.DataFrame(rows, columns=FIELDS)
    df.to_csv(out / "index.csv", index=False, encoding="utf-8-sig")
    return df, {"url": url, "sha256": hashlib.sha256(payload).hexdigest(),
                "fetched_at": datetime.now(CN).isoformat(), "name": item.get("name"),
                "returned_first": df.date.iloc[0], "returned_last": df.date.iloc[-1]}


def rank_past(s, window=WINDOW, minimum=MIN_HISTORY):
    """Midrank percentile relative to strictly prior observations only.

    Window is 756 index trading rows, not last 756 non-null observations.
    Require >=252 valid points and >=95% coverage of the available window.
    """
    values = s.to_numpy(dtype=float)
    result = np.full(len(values), np.nan)
    finite_positions = np.flatnonzero(np.isfinite(values))
    if not len(finite_positions): return pd.Series(result, index=s.index)
    first = finite_positions[0]  # structural warmup before the first factor value
    for i, value in enumerate(values):
        if not np.isfinite(value):
            continue
        past = values[max(first, i-window):i]
        good = past[np.isfinite(past)]
        if len(good) < minimum or len(good) < .95 * len(past):
            continue
        result[i] = 100 * ((good < value).sum() + .5 * (good == value).sum()) / len(good)
    return pd.Series(result, index=s.index)


def join_market(x, market, manifest):
    required_meta = ["point_in_time", "includes_delisted", "st_history_verified",
                     "adjustment_verified", "calendar_verified"]
    require(all(manifest.get(k) is True for k in required_meta),
            "market manifest must attest PIT universe, delisted/ST/adjustment/calendar checks")
    require(manifest.get("universe") == "CN_SH_SZ_A_NONST_60",
            "wrong market universe; expected CN_SH_SZ_A_NONST_60")
    require(bool(manifest.get("source")), "market source description is required")
    m = numeric(date_frame(market), MARKET)
    # Missing cells suppress dependent factors. Invalid cells are errors.
    counts = MARKET[:-1]
    require(((m[counts] >= 0) | m[counts].isna()).all().all(), "negative counts")
    require(((m[counts] % 1 == 0) | m[counts].isna()).all().all(), "fractional counts")
    def check(columns, predicate, message):
        c = m.loc[m[columns].notna().all(axis=1)]
        require(predicate(c).all(), message)
    check(["n_traded","n_up","n_down","n_flat"],
          lambda c: c.n_traded == c.n_up+c.n_down+c.n_flat, "U+D+flat != traded")
    check(["n_eligible","n_expected_traded"],
          lambda c: c.n_eligible >= c.n_expected_traded, "eligible < expected traded")
    check(["n_expected_traded","n_traded"],
          lambda c: c.n_expected_traded >= c.n_traded, "expected traded < observed traded")
    check(["n_drop5","n_down"], lambda c: c.n_drop5 <= c.n_down, "drop5 > down")
    for name in ["n_up","n_down","n_flat","n_drop5"]:
        check([name,"n_traded"], lambda c, k=name: c[k] <= c.n_traded, name+" > traded")
    check(["n_above_ma20","n_ma20"], lambda c: c.n_above_ma20 <= c.n_ma20, "above20 > denominator")
    check(["n_ma20","n_traded"], lambda c: c.n_ma20 <= c.n_traded, "ma20 denominator > traded")
    require(((m.amount_yuan > 0) | m.amount_yuan.isna()).all(), "nonpositive cash amount")
    extra = m.index.difference(x.index)
    # Rows later than requested as-of are ignored; unexplained past non-trading dates are errors.
    require(not any(d <= x.index.max() and d >= x.index.min() for d in extra),
            "market contains dates absent from index trading calendar")
    x = x.join(m[MARKET], how="left")
    x["market_coverage"] = x.n_traded/x.n_expected_traded
    valid = (x.n_traded > 0) & (x.market_coverage >= .95) & x.n_eligible.notna()
    partition = x[["n_up","n_down","n_flat"]].notna().all(axis=1)
    x["breadth"] = ((x.n_up-x.n_down)/x.n_traded).where(valid & partition)
    x["ma20"] = (x.n_above_ma20/x.n_ma20).where(valid & (x.n_ma20/x.n_traded >= .95))
    x["tail5"] = (x.n_drop5/x.n_traded).where(valid)
    x["suspension_share"] = (1-x.n_expected_traded/x.n_eligible).where(x.n_eligible > 0)
    return x


def join_finance(x, finance):
    """Finance rows: date, available_date, buy_yuan, repay_yuan.

    date is the transaction date; available_date is the first report date at
    whose 18:00 Asia/Shanghai cutoff the publication was actually observable.
    No financing-balance-difference approximation is accepted.
    """
    f = numeric(date_frame(finance), ["buy_yuan", "repay_yuan"])
    require("available_date" in f, "finance requires available_date")
    f["available_date"] = pd.to_datetime(f.available_date, errors="raise").dt.normalize()
    require((f.available_date > f.index).all(), "finance availability must be after trade date")
    require(f[["buy_yuan", "repay_yuan", "available_date"]].notna().all().all(),
            "finance rows may be absent but not partly missing")
    require((f[["buy_yuan", "repay_yuan"]] >= 0).all().all(), "negative financing values")
    require("amount_yuan" in x, "finance requires full SH/SZ A-share cash amount")
    f = f.reindex(x.index)
    net5 = (f.buy_yuan-f.repay_yuan).rolling(5, min_periods=5).sum()
    denom = x.amount_yuan.rolling(5, min_periods=5).sum()
    raw = net5 / denom
    avail = f.available_date
    valid_rows = []
    for j in range(4, len(x)):
        published = avail.iloc[j-4:j+1]
        if np.isfinite(raw.iloc[j]) and published.notna().all():
            valid_rows.append((j, published.max(), raw.iloc[j]))
    x["finance_trade_date"] = pd.NaT
    x["finance_age_sessions"] = np.nan
    # At most two exchange sessions of age; age 1 is the normal T+1 observation.
    for i, date in enumerate(x.index):
        candidates = [(j, v) for j, a, v in valid_rows if j < i and a <= date and i-j <= 2]
        if candidates:
            j, v = max(candidates)
            x.loc[date, "leverage"] = v
            x.loc[date, "finance_trade_date"] = x.index[j]
            x.loc[date, "finance_age_sessions"] = i-j
    return x


def calculate(index, market=None, finance=None, manifest=None):
    x = validate_index(index)
    x["r1"] = x.close.pct_change(fill_method=None)
    x["r5"] = x.close.pct_change(5, fill_method=None)
    x["momentum"] = x.close.pct_change(20, fill_method=None)
    x["downvol"] = np.sqrt(252*x.r1.clip(upper=0).pow(2).rolling(20, min_periods=20).mean())
    x["drawdown60"] = x.close/x.close.rolling(60, min_periods=60).max()-1
    for name in ["breadth", "ma20", "tail5", "leverage"]:
        x[name] = np.nan
    if market is not None:
        x = join_market(x, market, manifest or {})
    if finance is not None:
        require((manifest or {}).get("finance_scope") == "CN_SH_SZ_A_STOCK_ONLY"
                and (manifest or {}).get("finance_availability_verified") is True,
                "finance requires stock-only scope and verified publication availability in manifest")
        x = join_finance(x, finance)
    for name in WEIGHTS:
        rank = rank_past(x[name])
        x["rank_"+name] = rank
        x["q_"+name] = 100-rank if name in REVERSE else rank
        x["history_n_"+name] = x[name].shift(1).rolling(WINDOW, min_periods=1).count()
        x["warmup_"+name] = x["history_n_"+name] < WINDOW
    scores = x[["q_"+n for n in WEIGHTS]]
    x["factor_coverage"] = scores.notna().mul(list(WEIGHTS.values()), axis=1).sum(axis=1)
    x["S"] = scores.mul(list(WEIGHTS.values()), axis=1).sum(axis=1, min_count=len(WEIGHTS))
    x["P"] = .5*x.q_momentum+.5*x.q_downvol
    return x


def sentiment_state(value):
    if pd.isna(value): return "无法计算完整情绪指数"
    for upper, label in [(10, "极端低迷/冰点"), (25, "恐慌/低迷"), (45, "偏弱"),
                         (55, "中性"), (75, "偏强"), (90, "亢奋")]:
        if value <= upper: return label
    return "极端亢奋"


def daily_alerts(x):
    output, cold_streak = [], 0
    for date, r in x.iterrows():
        checks = []
        cold_streak = cold_streak+1 if pd.notna(r.S) and r.S <= 10 else 0
        for test, level, text in [
            (r.r1 <= -.05, 3, "全指单日跌幅达到5%"),
            (r.r5 <= -.10, 3, "全指5日跌幅达到10%"),
            (r.tail5 >= .20, 3, "至少20%交易股票单日下跌5%以上"),
            (r.rank_downvol >= 99 and r.r1 <= -.03, 3, "极端下行波动伴随单日跌幅达到3%"),
            (cold_streak >= 2, 3, "完整S连续两日不高于10"),
            (r.r1 <= -.03, 2, "全指单日跌幅达到3%"),
            (r.r5 <= -.06, 2, "全指5日跌幅达到6%"),
            (r.tail5 >= .10, 2, "至少10%交易股票单日下跌5%以上"),
            (r.S <= 20, 2, "完整S不高于20"),
            (r.r1 <= -.02, 1, "全指单日跌幅达到2%"),
            (r.rank_downvol >= 95 and r.r1 < 0, 1, "下行波动处于历史95分位以上且当日下跌"),
            (r.S <= 30, 1, "完整S不高于30"),
            (r.S >= 90, 1, "完整S达到90：过热观察")]:
            if test: checks.append((level, text))
        if r.r1 >= .05: checks.append((2, "全指单日上涨达到5%：剧烈正向波动"))
        elif r.r1 >= .03: checks.append((1, "全指单日上涨达到3%：正向波动观察"))
        output.append({"date": date, "level": max([a for a, _ in checks], default=0),
                       "reasons": "; ".join(dict.fromkeys(b for _, b in checks))})
    return pd.DataFrame(output).set_index("date")


def fmt(value, percent=False, digits=2):
    if pd.isna(value): return "NA"
    return f"{value*100 if percent else value:.{digits}f}" + ("%" if percent else "")


def weekly_report(x, as_of):
    as_of = pd.Timestamp(as_of).normalize()
    require(x.index.max() == as_of, f"stale/misaligned data: latest={x.index.max().date()}, expected={as_of.date()}")
    start = as_of-pd.Timedelta(days=as_of.weekday())
    week = x.loc[start:as_of]
    prev = x.loc[x.index < start]
    require(len(week) > 0 and len(prev) > 0, "no weekly base close")
    base = prev.iloc[-1].close
    last = week.iloc[-1]
    returns = last.close/base-1
    path = pd.concat([pd.Series([base], index=[prev.index[-1]]), week.close])
    maxdd = (path/path.cummax()-1).min()
    amplitude = week.high.max()/week.low.min()-1
    alerts = daily_alerts(x).loc[start:as_of]
    level = int(alerts.level.max())
    extra = []
    if maxdd <= -.08:
        level = 3; extra.append("周内收盘路径最大回撤达到8%（包含上周末基点）")
    elif maxdd <= -.05:
        level = max(level, 2); extra.append("周内收盘路径最大回撤达到5%")
    if amplitude >= .10:
        level = max(level, 2); extra.append("周内高低点振幅达到10%（大幅波动，不等同恐慌）")
    if len(week) > 1 and week.P.min() <= 10:
        extra.append("价格参考分P周内不高于10；仅为价格分项观察，不能认定完整情绪冰点")
    missing = [n for n in WEIGHTS if pd.isna(last["q_"+n])]
    prices_status = ["未触发已配置的价格警报", "黄色观察", "橙色预警", "红色预警"][level]
    quality = "完整S可计算" if not missing else "数据不完整，综合状态不可判定"
    lines = ["# A股情绪量化周报", "",
        f"观察区间：{start.date()}—{as_of.date()}；实际交易日 {len(week)} 天。口径：收盘后18:00（北京时间）。", "",
        f"- 完整情绪指数 S：{fmt(last.S)}；{sentiment_state(last.S)}。",
        f"- 价格趋势/下行波动参考分 P：{fmt(last.P)}；仅覆盖完整模型权重40%，不使用S的情绪分区。",
        f"- 上周末完整S：{fmt(prev.S.iloc[-1])}；本周S均值/最低/最高：{fmt(week.S.mean())} / {fmt(week.S.min())} / {fmt(week.S.max())}。",
        f"- 本周P均值/最低/最高：{fmt(week.P.mean())} / {fmt(week.P.min())} / {fmt(week.P.max())}。",
        f"- 中证全指：上周末 {base:.2f} → 本周末 {last.close:.2f}；周涨跌 {fmt(returns, True)}。",
        f"- 周内收盘最大回撤 {fmt(maxdd, True)}；高低点振幅 {fmt(amplitude, True)}。",
        f"- 预警：{prices_status}。数据质量：{quality}。", "",
        "缺项：" + ("、".join(missing) if missing else "无") + "。缺项绝不以0、50或权重重分配代替。", "",
        "## 周内逐日记录", "",
        "| 日期 | 全指收盘 | 日涨跌 | 20日收益 | 年化下行半波动 | P | 完整S | 预警级别 |",
        "|---|---:|---:|---:|---:|---:|---:|---:|"]
    for date, r in week.iterrows():
        lines.append(f"| {date.date()} | {r.close:.2f} | {fmt(r.r1,True)} | {fmt(r.momentum,True)} | {fmt(r.downvol,True)} | {fmt(r.P)} | {fmt(r.S)} | {int(alerts.loc[date,'level'])} |")
    lines += ["", "## 预警原因", ""]
    detail = [f"- {d.date()}：{r.reasons}。" for d, r in alerts.iterrows() if r.level]
    lines += detail or ["- 已观察到的价格数据未触发日度阈值；不能据此认定全市场没有恐慌。"]
    lines += ["- " + t + "。" for t in extra]
    if missing:
        lines += ["- 数据缺失提示：个股尾部、市场广度与杠杆风险未完成覆盖，综合预警结论保留。"]
    if "suspension_share" in week and week.suspension_share.max() >= .10:
        lines += ["- 可比股票停牌/无成交比例达到10%，广度分母收缩，需人工核验流动性风险。"]
    lines += ["", "## 市场复盘", "",
        "- 板块线：未接入固定行业分类的行业日线/广度；需补行业周收益、超额收益、成交额占比和20日均线上方占比。",
        "- 情绪线：未接入同口径连板、炸板、昨日涨停次日收益及小盘/成长风格日线。",
        ("- 资金流向：最新可得融资5日净买入/对应成交额为"+fmt(last.leverage,True)+
         "，交易数据截至"+str(pd.Timestamp(last.finance_trade_date).date())+"；不代表全市场净流入。")
         if pd.notna(last.leverage) else "- 资金流向：未接入融资净买入及ETF净申赎；成交额与市值变化不能解释为净流入。",
        "- 外围影响：未接入海外收盘与汇率数据；需按发布时间对齐，不推断未经检验的因果关系。", "",
        "以上栏目是数据接入状态。另有经核验的来源时，可在独立复盘段落中补充并引用。"]
    if as_of.weekday() < 4:
        lines.insert(3, "该截止日在周五之前，标为截至日观察；是否为节假日完整周需由交易日历确认。")
        lines = [line.replace("本周末", "报告期末") for line in lines]
    return "\n".join(lines)+"\n"


def historical_table(x):
    rows = ["| 日期 | 全指收盘 | 单日 | 5日 | 20日 | 下行半波动（年化） | 60日收盘回撤 | P | 完整S |",
            "|---|---:|---:|---:|---:|---:|---:|---:|---:|"]
    for day in EVENT_DATES:
        d = pd.Timestamp(day)
        if d not in x.index: continue
        r = x.loc[d]
        rows.append(f"| {day} | {r.close:.2f} | {fmt(r.r1,True)} | {fmt(r.r5,True)} | {fmt(r.momentum,True)} | {fmt(r.downvol,True)} | {fmt(r.drawdown60,True)} | {fmt(r.P)} | {fmt(r.S)} |")
    return "\n".join(rows)+"\n"


def retrospective_diagnostic(x):
    """Future labels for research OUTPUT only. Never called by calculate()."""
    y = pd.DataFrame({"P": x.P, "future20_return": x.close.shift(-20)/x.close-1})
    future_closes = pd.concat([x.close.shift(-k) for k in range(1,21)], axis=1)
    y["future20_min_return"] = future_closes.min(axis=1, skipna=False)/x.close-1
    y = y.dropna()
    y["group"] = pd.cut(y.P, [-.01,10,25,50,75,100],
                        labels=["0—10","10—25","25—50","50—75","75—100"])
    lines = ["# 价格参考分回顾性诊断", "",
             "未来收益仅用于事后标签；不能用于当日计算。交易日样本重叠，不是独立事件数，不是样本外胜率。", "",
             "| P分组（左开右闭，首组含0） | 交易日数 | 后20日收益中位数 | 未来20日内再跌10%的日期比例 |",
             "|---|---:|---:|---:|"]
    for name, group in y.groupby("group", observed=True):
        lines.append(f"| {name} | {len(group)} | {fmt(group.future20_return.median(),True)} | {fmt((group.future20_min_return<=-.10).mean(),True)} |")
    if len(y): lines += ["", f"有效标签区间：{y.index.min().date()}—{y.index.max().date()}。"]
    return "\n".join(lines)+"\n"


def fetch_context(x, as_of, out):
    """Optional fixed assets, exact base/end-date matching, never fill stale prices.
    These are the 5 audited CSI800 industry proxies, not a complete industry ranking.
    """
    start = as_of-pd.Timedelta(days=as_of.weekday())
    base_day = x.loc[x.index < start].index[-1]
    def one(asset):
        group, symbol, name = asset
        url = (f"https://quotes.sina.cn/cn/api/jsonp_v2.php/var%20_{symbol}="
               f"/CN_MarketDataService.getKLineData?symbol={symbol}&scale=240&ma=no&datalen=1023")
        result = {"group": group, "symbol": symbol, "name": name, "url": url}
        try:
            with urllib.request.urlopen(url, timeout=25) as r: payload = r.read()
            (out/(symbol+"_source.txt")).write_bytes(payload)
            result["sha256"] = hashlib.sha256(payload).hexdigest()
            match = re.search(r"=\s*\((\[.*\])\)\s*;?\s*$", payload.decode("utf-8"), re.S)
            require(match is not None, "unexpected JSONP format")
            frame = pd.DataFrame(json.loads(match.group(1))).rename(columns={"day":"date"})
            d = validate_index(frame)
            require(base_day in d.index and as_of in d.index, "required dates absent/stale provider data")
            result.update(base=float(d.loc[base_day,"close"]), end=float(d.loc[as_of,"close"]),
                          weekly_return=float(d.loc[as_of,"close"]/d.loc[base_day,"close"]-1))
        except Exception as exc: result["error"] = str(exc)
        return result
    with ThreadPoolExecutor(max_workers=4) as pool:
        results = list(pool.map(one, ASSETS))
    context = {"as_of": str(as_of.date()), "base_date": str(base_day.date()), "assets": results}
    (out/"context.json").write_text(json.dumps(context, ensure_ascii=False, indent=2), encoding="utf-8")
    lines = ["", "## 已核验行情补充", "", "仅包含预设的5个风格与5个中证800行业代理。缺失项目不参与排名，不代表完整行业覆盖。", "",
             "| 类别 | 指数 | 上周末 | 本周末 | 周涨跌 | 数据状态 |", "|---|---|---:|---:|---:|---|"]
    for r in results:
        lines.append(f"| {r['group']} | [{r['name']}]({r['url']}) | {fmt(r.get('base',np.nan))} | {fmt(r.get('end',np.nan))} | {fmt(r.get('weekly_return',np.nan),True)} | {'缺失：'+r['error'] if 'error' in r else '日期匹配'} |")
    lines += ["", "行业/风格相对收益不等于资金净流入；指数数据不能替代连板、炸板和涨停次日收益。"]
    return "\n".join(lines)+"\n", context


def previous_complete_friday():
    today = datetime.now(CN).date()
    back = (today.weekday()-4) % 7
    # During Friday itself, use preceding week to avoid any intraday ambiguity.
    if back == 0: back = 7
    return pd.Timestamp(today-timedelta(days=back))


def main():
    p = argparse.ArgumentParser(description=__doc__)
    p.add_argument("--as-of", help="Expected final trading date YYYY-MM-DD; default previous completed Friday")
    p.add_argument("--index-csv", type=Path, help="Optional offline 000985 OHLC data")
    p.add_argument("--index-meta", type=Path, help="Offline identity JSON: code, source, sha256")
    p.add_argument("--market-csv", type=Path, help="Point-in-time SH/SZ A-share aggregate data")
    p.add_argument("--finance-csv", type=Path, help="Financing flows with actual availability dates")
    p.add_argument("--manifest", type=Path, help="Required data audit manifest when market inputs are supplied")
    p.add_argument("--calendar-csv", type=Path, help="Independent exchange open dates; column date")
    p.add_argument("--with-context", action="store_true", help="Fetch optional style/industry panel; failure yields NA")
    p.add_argument("--diagnostic", action="store_true", help="Write retrospective P-only future-return diagnostics")
    p.add_argument("--out", type=Path, default=Path("sentiment_output"))
    args = p.parse_args()
    as_of = pd.Timestamp(args.as_of).normalize() if args.as_of else previous_complete_friday()
    if args.calendar_csv and not args.as_of:
        cal = date_frame(pd.read_csv(args.calendar_csv)).index
        last_week = cal[(cal <= as_of) & (cal >= as_of-pd.Timedelta(days=4))]
        require(len(last_week) > 0, "calendar has no sessions in requested week; select --as-of explicitly")
        as_of = last_week.max()
    require(as_of.date() < datetime.now(CN).date(), "as-of must be before today; intraday data is not allowed")
    args.out.mkdir(parents=True, exist_ok=True)
    if args.index_csv:
        payload = args.index_csv.read_bytes()
        require(args.index_meta is not None, "offline index requires --index-meta code/source/sha256")
        identity = json.loads(args.index_meta.read_text("utf-8-sig"))
        require(identity.get("code") == "000985" and identity.get("source"), "offline index identity missing/wrong")
        require(identity.get("sha256") == hashlib.sha256(payload).hexdigest(), "offline index hash mismatch")
        index = pd.read_csv(args.index_csv)
        source = {"local_file": str(args.index_csv.resolve()), "sha256": hashlib.sha256(payload).hexdigest(),
                  "identity": identity}
    else:
        index, source = fetch_index(as_of, args.out)
    index = index.loc[pd.to_datetime(index.date) <= as_of].copy()
    if args.calendar_csv:
        calendar = date_frame(pd.read_csv(args.calendar_csv)).index
        actual = pd.DatetimeIndex(pd.to_datetime(index.date))
        expected = calendar[(calendar >= actual.min()) & (calendar <= as_of)]
        require(actual.sort_values().equals(expected.sort_values()), "index dates disagree with exchange calendar")
    market = pd.read_csv(args.market_csv) if args.market_csv else None
    finance = pd.read_csv(args.finance_csv) if args.finance_csv else None
    manifest = json.loads(args.manifest.read_text("utf-8-sig")) if args.manifest else None
    if args.market_csv: require(manifest is not None, "--market-csv requires --manifest")
    x = calculate(index, market, finance, manifest)
    report = weekly_report(x, as_of)
    context = None
    if args.with_context:
        supplement, context = fetch_context(x, as_of, args.out)
        if any(a["group"] == "行业" and "error" not in a for a in context["assets"]):
            report = report.replace("未接入固定行业分类的行业日线/广度；需补行业周收益、超额收益、成交额占比和20日均线上方占比。",
                                    "部分行业代理日线已取得，见补充表；仍缺完整行业广度、成交额占比和20日均线上方占比。")
        if any(a["group"] == "风格" and "error" not in a for a in context["assets"]):
            report = report.replace("未接入同口径连板、炸板、昨日涨停次日收益及小盘/成长风格日线。",
                                    "风格代理日线见补充表；仍缺同口径连板、炸板和昨日涨停次日收益。")
        report += supplement
    x.to_csv(args.out/"daily_scores.csv", encoding="utf-8-sig", index_label="date")
    (args.out/"weekly_report.md").write_text(report, encoding="utf-8")
    (args.out/"historical_cases.md").write_text(historical_table(x), encoding="utf-8")
    if args.diagnostic:
        (args.out/"price_diagnostic.md").write_text(retrospective_diagnostic(x), encoding="utf-8")
    run = {"model_version": VERSION, "as_of": str(as_of.date()), "source": source,
           "window": WINDOW, "minimum_history": MIN_HISTORY, "weights": WEIGHTS,
           "full_S_available": bool(pd.notna(x.S.iloc[-1])), "input_manifest": manifest,
           "observed_rows": len(x), "first": str(x.index.min().date()), "last": str(x.index.max().date()),
           "generated_at": datetime.now(CN).isoformat(), "python": sys.version,
           "pandas": pd.__version__, "numpy": np.__version__,
           "calendar_checked": bool(args.calendar_csv),
           "context_available": context is not None,
           "additional_input_hashes": {str(v.resolve()): hashlib.sha256(v.read_bytes()).hexdigest()
                 for v in [args.market_csv,args.finance_csv,args.manifest,args.calendar_csv,args.index_meta] if v}}
    (args.out/"run_manifest.json").write_text(json.dumps(run, ensure_ascii=False, indent=2), encoding="utf-8")
    print(report)


if __name__ == "__main__":
    try:
        main()
    except Exception as exc:
        raise SystemExit(f"DATA / EXECUTION ERROR: {exc}") from exc

```
