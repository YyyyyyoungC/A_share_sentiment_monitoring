# A股情绪指数计算脚本

版本：**v1.1.0**。唯一生产数据源：**Wind Alice 万得金融数据**。本文件内含可保存运行的完整Python代码。

保留原六因子计算、756交易日半秩归一化、无前视逻辑及全部原预警条件；新增周五时间门禁、市场级预聚合计划、增量SHA256缓存、请求登记与预算控制、异常暂停及用户确认。**没有公网抓取代码、非Wind备用源、逐股遍历或自动重试。**

当前没有可调用的Wind Alice连接器，也没有真实工具schema/指标标识/积分价表。本脚本提供的是**本地计划、回执验收、缓存和计算控制层**：由已连接的Wind执行器执行登记的单个请求后导入回执。它不是已经完成真实Wind绑定的在线客户端，不会伪造SDK、API地址或工具参数。本次测试全部使用隔离的合成数据，没有消耗Wind积分。

## 1. 执行规范（原文）

> 执行时间：每周五 18:00（收盘后），盘中数据一律禁用；当日收盘未完成时则暂停生成报告，并向我进行询问。
>
> 数据源：强制使用 Wind Alice 万得金融数据，未经我书面同意不得切换或混用。
>
> 异常处理：
>
> a. 若取数返回为空、积分不足或其他问题，第一时间暂停报告生成，绝不用替代值硬算；
>
> b. 主动与我沟通，说明异常的具体位置（数据源/表/字段/日期）；
>
> c. 提供多个解决方案及利弊说明（至少两个选项）；
>
> d. 必须经我确认后才能继续执行；
>
> e. 严禁自行决定关键问题（包括但不限于：更换数据源、修改权重/阈值/窗口、填补或估算缺失值）。

任何恢复动作都不能豁免盘中禁用、Wind单一来源或模型固定参数。脚本输出异常询问单，实际执行器须立即展示给用户；本地文件不会自行推送消息。本文没有替当前账号设置调度、登录Wind或发送消息。

## 2. 最小调用设计

| 数据集 | 本地标准字段 | 一次性/每周处理 |
|---|---|---|
| index | date,open,high,low,close | 核验000985实体一次；首轮所需历史后，每周补新增日线 |
| market | date,n_eligible,n_expected_traded,n_traded,n_up,n_down,n_flat,n_above_ma20,n_ma20,n_drop5,amount_yuan | 从同口径历史预聚合/获批服务端聚合取得，禁止逐股循环 |
| finance | date,available_date,buy_yuan,repay_yuan | 历史融资纯股票流量和可得性；本地生成五日强度 |
| 每个行业/风格/外围价格面板 | date,close | 只需上周最后可用基点及本周已结束日，不抓756日辅助历史 |
| 已批准的额外资金面板 | date,value | 仅在范围明确且需要时取；资金模块优先复用核心融资 |

字段均为**内部标准格式**，不是Wind工具参数名。`wind_fields`必须填写真实返回字段的映射。`tool`是路由名称；`next_request.json`是请求意图，不能不经真实schema适配就当作工具原始入参发送。

`market`中`amount_yuan`是全部沪深A股股票成交额；其他家数是原非ST、60日成熟股票池。预聚合若不满足这两种明确范围，停止核验，不擅自换分母。`available_date`表示以当日18:00为截止，融资首次已经可得的报告日期；不能拿抓取日期或余额差代替。

周频只指启动次数，**全部核心输入仍为日频**。R、DSV、历史分位、周收益、回撤、周内全部预警均由本地缓存算出，不额外调用技术指标工具。初次建库的历史长度应按756日、20日预热及当周基点确定；历史专项复盘另批范围，不默认全量十几年。

## 3. 保存与初始化

将本文件最后的Python代码块保存为`sentiment_v110.py`。建议Python 3.11以上；验证环境使用Python 3.12.14、pandas 3.0.1、numpy 2.3.5。

```powershell
python -m pip install "pandas>=2.2,<4" "numpy>=1.26,<3"
python sentiment_v110.py init --root wind_cache_v110
```

`init`只生成`config.json`和`calendar_template.json`，不联网、不取数、不扣Wind积分。初始`approved_by_user=false`、`verified=false`、实际字段/指标/报价为空，不能进入付费请求登记。

配置保持原观察清单：3个核心数据集、5个行业、5个风格、2个美国指数，共15个数据集。**15不是保证的调用数**：接口分页、字段覆盖与实际能力会改变请求数；真实支持多代码批量时才可在经审核的Wind执行层合并，并保持请求与返回的可追溯性。当前实现按数据集和缺口分片，未假装支持未知的多代码schema。

不要为运行成功而自行把核验和批准字段改为true。准备就绪后，配置的每个数据集须具备：

- `tool`：实际适用的工具名称，预定义工具能直接返回时不滥用`get_financial_data`。
- `catalog_id`、`entity_or_scope`、`wind_fields`：来自实际Wind目录/schema，非自行猜测。
- `definition_evidence`：股票池、复权、分母、单位、频率、可得性、历史覆盖的证据引用。
- `semantic_contract`：与原公式一致的口径声明；三个核心合同已经固定，不能改名绕过检查。
- `max_rows_per_call`：已核验的分页/批量上限，不用收费试错探测。
- `max_points_per_call`及总`budget`：核验过的单次积分上界和用户批准的总调用/积分上限；未知时禁止`claim`。
- `history_start`：经批准的所需历史起点；`calendar`为相应市场的Wind日历键。

日历结构为一个本地封装：`source`、`request_id`、`covers_from`、`covers_through`、`data`和`sha256`。`data`以`CN`、`US`等市场为键，每项是按日排序的`{"date":"YYYY-MM-DD","close_at":"带时区的实际收盘时间"}`。`sha256`为本脚本`digest(data)`的值。日期和时区只能来自Wind已核验日历，不用工作日推算真实交易日；美国夏令时要按每个日期处理。

日历和字段元信息的首次获取同样可能消耗积分，应先形成有限的一次性设置计划并确认，不能因为脚本需要日历就无预算调用。在目录/schema未核验的当前状态，交付可用于审查和本地初始化，尚不能执行真实取数。

## 4. 运行步骤与Wind执行器协议

下面的日期只是一个周五周期标识，用于说明命令；不是已经取得该周Wind数据。`plan`可以提前准备，`claim`和`report`受真实北京时间门禁约束。

### 步骤一：生成本地最小计划

```powershell
python sentiment_v110.py plan --root wind_cache_v110 --config wind_cache_v110/config.json --calendar wind_calendar.json --friday 2026-09-18
```

计划只覆盖缺失的连续交易日段，按已核验行数上限分片；已有缓存不会重复请求。缓存文件和对应Wind原始回执都校验SHA256及定义版本，失败就暂停。计划冻结配置哈希，同一计划不能偷偷增加积分预算、更改来源或改口径。

### 步骤二：周五18:00后，只登记下一条请求

```powershell
python sentiment_v110.py claim --root wind_cache_v110 --config wind_cache_v110/config.json
```

`claim`持久化本次尝试ID、唯一请求键、预算预留，然后写`next_request.json`。**它本身不发送Wind请求。** 已有未结算请求时再次`claim`会被阻止；超时不能认为没扣积分。下列执行器协议必须同时遵守：

1. 读取一条已经登记的请求意图，按实际Wind schema组装参数，只执行这一次。
2. 成功后保存完整原始响应，转换为下述标准回执；任何不确定的单位、范围或日期都不能猜。
3. 立即调用`ingest`验收；仅验收成功，才可登记下一条请求。不得预先并发发出剩余请求。
4. 空返回、工具错误、积分不足、超时或未完成终盘，立即执行`fail`或导入错误回执，并向用户展示暂停询问。关闭SDK/代理/网关隐藏的自动重试。

### 步骤三：导入同一请求的Wind回执

```powershell
python sentiment_v110.py ingest --root wind_cache_v110 --config wind_cache_v110/config.json --receipt wind_receipt.json
```

`wind_receipt.json`是**适配层封装格式**，不是声称Wind原生就返回这些字段：

| 字段 | 内容 |
|---|---|
| source | 固定为Wind Alice 万得金融数据 |
| provider_request_id | 真实Wind请求标识或可追溯的执行器原始记录标识 |
| attempt_id,request_key,definition_hash | 从已登记请求精确复制，不可另造请求 |
| status,error | 成功为ok；失败保留实际错误说明 |
| information_cutoff | 请求中的北京时间周五18:00，供严格信息可得性验收 |
| pit_verified | 适配层有证据核验截至该时点可得；无证据不得填true |
| all_final | 所有返回记录已完成终盘或正式披露；盘中/待定为false |
| rows | 按请求日期排列、字段集合严格一致的日度数据；缺失不填值 |
| raw_response | 完整供应商原始JSON响应或原始文本，保留错误与元信息 |
| raw_response_sha256 | 本脚本digest(raw_response)，即规范JSON编码后的SHA256 |
| points_used | 从Wind账单/已确认回执核验的实际积分；未知就暂停结算，不能编为0 |

哈希只证明保存内容未被修改，不能证明供应商身份、数据含义或用户授权。适配层必须保存原始响应和转换依据，不能只手工编出一份`rows`再声明已验证。

### 步骤四：全量必需数据验收后生成报告

```powershell
python sentiment_v110.py report --root wind_cache_v110 --config wind_cache_v110/config.json
```

所有待处理请求必须已经缓存；当周任何一天的完整S或上周末基点S不可用，都暂停。保留七模块：情绪指数、中证全指表现、预警、板块线、情绪线、资金流向、外围市场。本周完成后再计划同一周会直接复用归档，不重复取数或发布。

资金模块直接复用核心融资，额外ETF等流量不自动新增调用。外国市场按实际收盘时间截断到周五18:00的信息集。报告只对现有指标作事实描述，不凭价格涨跌推断净流入或外盘因果。

## 5. 异常与恢复

当Wind调用失败但没有可导入的完整响应时：

```powershell
python sentiment_v110.py fail --root wind_cache_v110 --reason "填写真实错误：工具、字段、日期和是否扣费待核实"
```

程序生成`pause.json`并输出询问内容，状态变为`PAUSED_WAIT_USER`，完整S在异常记录中为null/NA，不发布正式报告。给出至少两个选项及利弊：先核对复用原响应；经确认后只重试原请求；等待或取消。重复操作不会覆盖掉最初异常ID而掩盖原始位置。

用户真实确认后，执行器保存以下确认记录，再调用`resume`。这是**确认记录模板，不能由代理自行批准或预填为已确认**：

```json
{
  "ticket_id": "当前pause.json中的异常ID",
  "confirmed_by_user": false,
  "user_text": "等待真实用户的明确选择",
  "confirmed_at": "待填写真实确认时间",
  "decision": "retry_same_request"
}
```

```powershell
python sentiment_v110.py resume --root wind_cache_v110 --approval user_confirmation.json
```

允许的决定：`retry_same_request`（重试原请求）、`reconcile_response`（核实并复用原响应）、`continue_after_fix`（修复原计划内问题后继续）、`cancel_run`（取消）。恢复不会发起调用，只更新状态；未结算尝试的积分预留不会自动归零，重试仍受原预算限制。对配置/数据合同的实质变更，要重新审查和确认，不能把“继续”理解为允许换源、改权重或填缺值。

锁文件存在时禁止第二个进程执行。若进程崩溃，先核实是否有Wind请求已发送、是否扣费、能否复用回执，再人工处理锁；不自动删除锁、重置状态或清缓存来强行继续。

## 6. 文件、边界与验证

```text
wind_cache_v110/
  config.json                 已审查配置，初始模板不可执行
  state.json                  状态、计划、缓存索引、调用与积分台账
  plan.json                   最小缺口计划
  next_request.json           当前唯一已登记请求意图
  receipts/<sha256>.json      包含完整原始响应的标准回执
  objects/<sha256>.json       通过验收的日度记录
  pause.json                  异常位置、选项和待确认问题
  reports/<周五_计划ID>/
    daily_scores.csv
    weekly_report.md
    manifest.json
```

v1.0.0的公网缓存不能导入这个目录。新旧数据不得拼接；旧结果仅保留审计。缺口指未缓存的已知真实交易日，不能以“接口只返回这些日子”作为交易日历。

本次18项合成测试验证了核心函数保持一致、无前视与半秩、门禁/预算/失败暂停、增量缓存、回执链、完整S缺失禁发、七模块和单周幂等。**这不是Wind实盘接入测试，也不是积分节省比例或预测胜率测试。** 公式说明书列出了真实接入尚待完成的字段、口径、价格及执行器验收。

## 7. 完整Python源码

以下代码是完整的单文件本地控制与计算程序；保留旧版计算内核，新增Wind专用工作流。生产入口为文件末尾的`main()`，不会运行保留内核中的旧版报告诊断函数来绕过严格发布门禁。

```python
"""A-share sentiment v1.1.0. Wind-only local calculation and guarded receipt workflow.
No network SDK, guessed Wind API, retries, or non-Wind fallback is included.
"""
from __future__ import annotations
import argparse, hashlib, json, math, os, sys, uuid
from contextlib import contextmanager
from datetime import datetime, timedelta, timezone
from pathlib import Path
import numpy as np
import pandas as pd
VERSION = "1.1.0"
WEIGHTS = {"breadth": .20, "ma20": .15, "momentum": .20,
           "tail5": .15, "downvol": .20, "leverage": .10}
REVERSE = {"tail5", "downvol"}
WINDOW, MIN_HISTORY = 756, 252
CN = timezone(timedelta(hours=8))
MARKET = ["n_eligible", "n_expected_traded", "n_traded", "n_up", "n_down", "n_flat",
          "n_above_ma20", "n_ma20", "n_drop5", "amount_yuan"]
EVENT_DATES = ["2015-06-26", "2015-07-08", "2015-08-24", "2016-01-07",
               "2018-10-11", "2020-02-03", "2022-04-25", "2022-04-26",
               "2022-10-31", "2024-02-02", "2024-02-05", "2024-02-06"]


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

# Wind Alice boundary. These are LOCAL normalized fields, not invented Wind API arguments.
SOURCE = "Wind Alice 万得金融数据"
CORE_CONTRACTS = {
    "index": "CSI_000985_PRICE_OHLC_DAILY_FINAL",
    "market": "CN_SH_SZ_A_NONST_60_PIT_ADJUSTED_COUNTS_AND_ALL_A_AMOUNT",
    "finance": "CN_SH_SZ_A_STOCK_ONLY_BUY_REPAY_PIT_AVAILABILITY",
}
CORE_FIELDS = {"index": ["date","open","high","low","close"],
               "market": ["date"]+MARKET,
               "finance": ["date","available_date","buy_yuan","repay_yuan"]}
ALLOWED_TOOLS = {"get_index_kline", "get_financial_data", "query_economic_indicator_data",
                 "get_fund_performance", "get_fund_holders"}
PANEL_ROLES = {"sector", "style", "funding", "external"}


def canonical(value):
    return json.dumps(value, ensure_ascii=False, sort_keys=True, separators=(",",":"), allow_nan=False).encode("utf-8")


def digest(value): return hashlib.sha256(canonical(value)).hexdigest()


def read_json(path): return json.loads(Path(path).read_text("utf-8-sig"))


def atomic_json(path, value):
    path=Path(path);path.parent.mkdir(parents=True, exist_ok=True)
    temp=path.with_name(path.name+"."+uuid.uuid4().hex+".tmp")
    temp.write_bytes(canonical(value));os.replace(temp,path)


class StopRun(Exception):
    def __init__(self, dataset, field, dates, reason):
        self.location={"source":SOURCE,"dataset":dataset,"field":field,"dates":dates}
        super().__init__(reason)


def guard(ok, dataset, field, dates, reason):
    if not bool(ok): raise StopRun(dataset,field,dates,reason)


def stamp(value):
    t=pd.Timestamp(value)
    guard(t.tzinfo is not None,"metadata","timestamp",str(value),"必须提供含时区时间戳")
    return t.tz_convert(CN)


@contextmanager
def exclusive(root):
    root=Path(root);root.mkdir(parents=True,exist_ok=True);lock=root/"workflow.lock"
    try: fd=os.open(lock,os.O_CREAT|os.O_EXCL|os.O_WRONLY)
    except FileExistsError: raise StopRun("workflow","lock",[],"另一个进程或未清理的崩溃锁存在；先核实，不能自动抢锁")
    try:
        os.write(fd,str(os.getpid()).encode());os.close(fd);yield
    finally: lock.unlink(missing_ok=True)


def get_state(root):
    path=Path(root)/"state.json"
    return read_json(path) if path.exists() else {"status":"IDLE","cache":{},"ledger":[],"approvals":[]}


def stop_record(root, state, exc):
    if state.get("status")=="PAUSED_WAIT_USER" and state.get("ticket"):
        print(json.dumps(state["ticket"],ensure_ascii=False,indent=2));return
    ticket={"ticket_id":uuid.uuid4().hex,"status":"PAUSED_WAIT_USER","S":None,
            "location":getattr(exc,"location",{"source":SOURCE,"dataset":"workflow","field":"validation","dates":[]}),
            "reason":str(exc),"at":datetime.now(CN).isoformat(),
            "options":[
                {"id":"fix_without_call","方案":"核对同一Wind响应、字段口径或缓存后继续","利":"避免再次取数消耗","弊":"需人工核验，可能仍缺数据"},
                {"id":"retry_same_request","方案":"确认原请求结算状态、积分预算后仅重试该请求","利":"可恢复同源缺失数据","弊":"可能再次消耗积分，须明确批准"},
                {"id":"wait_or_cancel","方案":"等待Wind恢复/数据补齐，或取消本次报告","利":"当前不再消耗积分","弊":"报告延后"}],
            "question":"请确认采用哪个方案；确认前暂停取数与报告，不更换数据源或修改模型。"}
    state["status"]="PAUSED_WAIT_USER";state["ticket"]=ticket
    atomic_json(Path(root)/"state.json",state)
    atomic_json(Path(root)/"pause.json",ticket)
    print(json.dumps(ticket,ensure_ascii=False,indent=2))


def settings(path):
    c=read_json(path)
    guard(c.get("source")==SOURCE,"config","source",[],"仅允许Wind Alice")
    guard(c.get("model_version")==VERSION,"config","model_version",[],"配置必须对应v1.1.0")
    guard(c.get("approved_by_user") is True,"config","approved_by_user",[],"计划范围及预算尚未获得用户确认")
    guard(isinstance(c.get("history_start"),str) and len(c["history_start"])==10,"config","history_start",[],"历史起点尚未核验")
    datasets=c.get("datasets",{})
    guard(set(CORE_FIELDS).issubset(datasets),"config","datasets",[],"缺少核心数据集定义")
    guard({"sector","style","external"}.issubset({v.get("role") for v in datasets.values()}),"config","roles",[],"复盘范围尚未定义；资金模块复用核心融资数据")
    for name,d in datasets.items():
        guard(re_safe_name(name),name,"dataset_id",[],"数据集名称仅允许字母数字及下划线")
        guard(d.get("verified") is True and d.get("definition_evidence"),name,"definition",[],"Wind字段及口径未核验，不能试探取数")
        guard(d.get("tool") in ALLOWED_TOOLS,name,"tool",[],"禁止逐股工具、行情快照、分钟线或未登记工具")
        guard(d.get("period")=="daily",name,"period",[],"只降低运行频率，不得将日频输入换成周K")
        guard(d.get("semantic_contract"),name,"semantic_contract",[],"缺少口径合同")
        if name in CORE_FIELDS:
            guard(d["fields"]==CORE_FIELDS[name],name,"fields",[],"核心字段最小集合不符")
            guard(d["semantic_contract"]==CORE_CONTRACTS[name],name,"semantic_contract",[],"预聚合口径不等价于原模型")
        else:
            guard(d.get("role") in PANEL_ROLES and d.get("kind") in {"price","flow"},name,"panel",[],"复盘口径未定义")
            expected=["date","close"] if d["kind"]=="price" else ["date","value"]
            guard(d["fields"]==expected and d.get("unit"),name,"fields/unit",[],"仅取复盘必要字段，必须声明单位")
        guard(set(d.get("wind_fields",{}))==set(d["fields"]) and all(d["wind_fields"].values()),name,"wind_fields",[],"实际Wind字段映射未登记；内部字段名不能当Wind参数")
        guard(d.get("catalog_id") and d.get("entity_or_scope"),name,"catalog_id",[],"缺少实际Wind标识或股票池范围")
        guard(isinstance(d.get("max_rows_per_call"),int) and d["max_rows_per_call"]>0,name,"batch_limit",[],"实际分页/批量上限未核验")
    return c


def re_safe_name(value): return bool(value) and all(ch.isascii() and (ch.isalnum() or ch=="_") for ch in value)


def calendar_data(path, c, friday):
    e=read_json(path)
    guard(e.get("source")==SOURCE and e.get("request_id"),"calendar","source",[],"日历必须有Wind来源与请求证据")
    guard(e.get("sha256")==digest(e.get("data")),"calendar","sha256",[],"日历SHA256校验失败")
    guard(e["covers_from"]<=c["history_start"] and e["covers_through"]>=friday,"calendar","coverage",[friday],"日历覆盖不足，不以缺行推定休市")
    data=e["data"]
    for key,rows in data.items():
        dates=[r["date"] for r in rows]
        guard(dates==sorted(set(dates)),key,"calendar_dates",dates,"日历重复或无序")
        for row in rows: stamp(row["close_at"])
    return e


def cache_rows(root, state, name, definition):
    rows={}
    for ref in state["cache"].get(name,[]):
        p=Path(root)/"objects"/(ref["sha256"]+".json")
        guard(p.exists(),name,"cache",[],"缓存对象缺失")
        raw=p.read_bytes()
        guard(hashlib.sha256(raw).hexdigest()==ref["sha256"],name,"sha256",[],"缓存SHA256校验失败，禁止静默重抓")
        chunk=json.loads(raw)
        guard(chunk["source"]==SOURCE and chunk["definition_hash"]==digest(definition),name,"cache_provenance",[],"缓存来源或口径不一致")
        receipt_path=Path(root)/"receipts"/(chunk["receipt_sha256"]+".json")
        guard(receipt_path.exists(),name,"receipt",[],"缓存缺少原始Wind回执")
        receipt_bytes=receipt_path.read_bytes()
        guard(hashlib.sha256(receipt_bytes).hexdigest()==chunk["receipt_sha256"],name,"receipt_sha256",[],"Wind回执校验失败")
        receipt=json.loads(receipt_bytes)
        guard(receipt.get("source")==SOURCE and receipt.get("status")=="ok" and receipt.get("all_final") is True and receipt.get("rows")==chunk["rows"],name,"receipt_provenance",[],"缓存与原始Wind回执不一致")
        guard(receipt.get("raw_response") and receipt.get("raw_response_sha256")==digest(receipt["raw_response"]),name,"raw_response",[],"供应商原始响应证据缺失或被修改")
        for row in chunk["rows"]:
            day=row["date"]
            guard(day not in rows or rows[day]==row,name,"revisions",[day],"缓存历史修订冲突；须单独确认版本")
            rows[day]=row
    return rows


def plan(root, state, c, cal, friday):
    if friday in state.get("completed_weeks",{}):
        print("该周已经发布；直接使用归档报告，不再次取数或生成。");return
    guard(state["status"] not in {"PAUSED_WAIT_USER","IN_FLIGHT"},"workflow","state",[],"有未处理异常或未结算请求，不能重建计划绕过暂停")
    if state.get("plan") and state["status"]!="COMPLETE":
        guard(state["status"]!="CANCELLED" and state["plan"]["friday"]==friday and state["plan"]["config_hash"]==digest(c),"workflow","active_plan",[friday],"已有未完成/取消的计划，须先人工确认处理")
        print("复用已登记计划，保留已用积分与调用次数；不重建预算。");return
    f=pd.Timestamp(friday)
    guard(f.weekday()==4,"schedule","friday",[friday],"报告周期必须以周五标识")
    cutoff=stamp(friday+"T18:00:00+08:00")
    request_list=[];expected={}
    ordered=list(CORE_FIELDS)+sorted(set(c["datasets"])-set(CORE_FIELDS))
    for name in ordered:
        d=c["datasets"][name]
        sessions=cal["data"].get(d["calendar"],[])
        guard(bool(sessions),name,"calendar",[],"缺少对应市场的Wind日历")
        dates=[r["date"] for r in sessions if c["history_start"]<=r["date"] and stamp(r["close_at"])<=cutoff]
        if name=="finance": dates=dates[:-1]  # T+1 report information set; original factor unchanged.
        if name not in CORE_FIELDS:
            monday=str((f-pd.Timedelta(days=4)).date())
            before=[day for day in dates if day<monday]
            dates=([before[-1]] if before and d["kind"]=="price" else [])+[day for day in dates if day>=monday]
        guard(bool(dates),name,"dates",[],"无可用历史范围")
        expected[name]=dates
        existing=cache_rows(root,state,name,d)
        missing=[day for day in dates if day not in existing]
        # Contiguous missing runs only: do not refetch cached dates inside gaps.
        groups=[];part=[]
        for day in dates:
            if day in missing:
                part.append(day)
                if len(part)==d["max_rows_per_call"]:groups.append(part);part=[]
            elif part:groups.append(part);part=[]
        if part:groups.append(part)
        for group in groups:
            job={"dataset":name,"tool":d["tool"],"catalog_id":d["catalog_id"],
                 "entity_or_scope":d["entity_or_scope"],"period":"daily","fields":d["wind_fields"],
                 "information_cutoff":cutoff.isoformat(),
                 "dates":group,"begin":group[0],"end":group[-1],"definition_hash":digest(d),
                 "max_points":d.get("max_points_per_call")}
            job["request_key"]=digest(job);job["status"]="PENDING";request_list.append(job)
    guard(expected["index"]==expected["market"],"core","calendar",[],"价格与广度交易日必须一致")
    guard(any(day>=str((f-pd.Timedelta(days=4)).date()) for day in expected["index"]),"calendar","week",[friday],"本周无交易日；暂停并确认如何报告")
    p={"version":VERSION,"friday":friday,"cutoff":cutoff.isoformat(),"config_hash":digest(c),
       "calendar_hash":digest(cal),"expected":expected,"jobs":request_list}
    p["plan_id"]=digest(p)
    state["plan"]=p;state["status"]="PLANNED";state["spent_bound"]=0.0;state["calls"]=0
    atomic_json(Path(root)/"state.json",state);atomic_json(Path(root)/"plan.json",p)
    print(json.dumps({"plan_id":p["plan_id"],"new_calls":len(request_list),"new_rows":sum(len(j["dates"]) for j in request_list),
                      "points_upper_bound":sum(j["max_points"] for j in request_list) if all(isinstance(j["max_points"],(int,float)) for j in request_list) else None,
                      "note":"只生成本地计划；未执行Wind调用。"},ensure_ascii=False,indent=2))


def check_frozen(state,c):
    guard("plan" in state,"workflow","plan",[],"先生成计划")
    guard(state["plan"]["config_hash"]==digest(c),"config","hash",[],"计划后配置被修改；不得自动变更口径或预算")
    guard(state["status"] in {"PLANNED","IN_FLIGHT"},"workflow","approval",[],"当前状态不可执行，暂停/取消/已完成状态不能自行继续")


def time_gate(state, now):
    p=state["plan"];now=stamp(now)
    ordinary=now.date().isoformat()==p["friday"] and now>=stamp(p["cutoff"])
    approved=state.get("resume_for_plan")==p["plan_id"] and now>=stamp(p["cutoff"])
    guard(ordinary or approved,"schedule","18:00 Asia/Shanghai",[p["friday"]],"仅周五18:00后运行；盘中或补跑须暂停并询问用户")


def claim(root,state,c,now):
    check_frozen(state,c);time_gate(state,now)
    guard(not any(j["status"]=="IN_FLIGHT" for j in state["plan"]["jobs"]),"workflow","inflight",[],"上次调用结果未知，禁止再次调用")
    pending=[j for j in state["plan"]["jobs"] if j["status"]=="PENDING"]
    if not pending: print("所有数据已缓存；无需付费调用。");return
    j=pending[0];quote=j["max_points"];budget=c.get("budget",{})
    guard(isinstance(quote,(int,float)) and math.isfinite(quote) and quote>=0,j["dataset"],"price_quote",j["dates"],"没有核验单次积分上界，不执行付费请求")
    guard(isinstance(budget.get("max_calls"),int) and state["calls"]<budget["max_calls"],j["dataset"],"call_budget",j["dates"],"调用上限未确认或已达到上限")
    cap=budget.get("max_points")
    guard(isinstance(cap,(int,float)) and math.isfinite(cap) and state["spent_bound"]+quote<=cap,j["dataset"],"point_budget",j["dates"],"积分预算未确认或不足")
    state["calls"]+=1;state["spent_bound"]+=quote
    j["status"]="IN_FLIGHT";j["attempt_id"]=uuid.uuid4().hex
    state["status"]="IN_FLIGHT"
    state["ledger"].append({"plan_id":state["plan"]["plan_id"],"request_key":j["request_key"],"attempt_id":j["attempt_id"],"reserved_points":quote,"time":str(now)})
    atomic_json(Path(root)/"state.json",state)
    atomic_json(Path(root)/"next_request.json",j)
    print("已登记一次调用意图。请由已连接的Wind执行器按真实工具schema执行这一条，并导出回执；本脚本没有发送请求。")


def ingest(root,state,c,receipt_path):
    check_frozen(state,c)
    active=[j for j in state["plan"]["jobs"] if j["status"]=="IN_FLIGHT"]
    guard(len(active)==1,"workflow","attempt",[],"只接收当前已登记请求的回执")
    j=active[0];name=j["dataset"];d=c["datasets"][name];e=read_json(receipt_path)
    # Preserve even an error response before evaluating it; no automatic retries.
    raw=canonical(e);h=hashlib.sha256(raw).hexdigest()
    audit=Path(root)/"receipts"/(h+".json");audit.parent.mkdir(exist_ok=True);audit.write_bytes(raw)
    guard(e.get("source")==SOURCE and e.get("provider_request_id"),name,"source",j["dates"],"缺少Wind原始请求证据")
    guard(e.get("raw_response") and e.get("raw_response_sha256")==digest(e["raw_response"]),name,"raw_response_sha256",j["dates"],"须附完整Wind原始响应及规范JSON哈希，不能只提交手填结果")
    guard(e.get("attempt_id")==j["attempt_id"] and e.get("request_key")==j["request_key"],name,"request_id",j["dates"],"回执与当前请求不匹配")
    guard(e.get("status")=="ok",name,"response",j["dates"],e.get("error","Wind调用失败或积分不足"))
    guard(e.get("definition_hash")==j["definition_hash"],name,"definition",j["dates"],"回执口径不匹配")
    guard(e.get("information_cutoff")==j["information_cutoff"] and e.get("pit_verified") is True,name,"information_cutoff",j["dates"],"必须验证截至报告时点可得，不能用后来发布的数据倒填")
    guard(e.get("all_final") is True,name,"final_close",j["dates"],"收盘未确认完成，禁止使用盘中数据")
    rows=e.get("rows",[])
    guard(bool(rows),name,",".join(d["fields"]),j["dates"],"Wind返回为空")
    guard([r.get("date") for r in rows]==j["dates"],name,"dates",j["dates"],"漏日、重复、乱序或多返回日期，禁止静默截取")
    for r in rows:
        guard(set(r)==set(d["fields"]),name,"columns",[r.get("date")],"字段集合不一致")
        for field,value in r.items():
            guard(value is not None,name,field,[r["date"]],"字段为空，停止报告")
            if field not in {"date","available_date"}:
                guard(isinstance(value,(int,float)) and not isinstance(value,bool) and math.isfinite(value),name,field,[r["date"]],"数值字段无效")
        if name=="index":
            guard(min(r[k] for k in ["open","high","low","close"])>0 and r["high"]>=max(r["open"],r["close"],r["low"]) and r["low"]<=min(r["open"],r["close"],r["high"]),name,"OHLC",[r["date"]],"价格及高低关系无效")
        elif name=="market":
            guard(all(r[k]>=0 and int(r[k])==r[k] for k in MARKET[:-1]),name,"counts",[r["date"]],"家数必须为非负整数")
            guard(r["n_up"]+r["n_down"]+r["n_flat"]==r["n_traded"] and r["n_drop5"]<=r["n_down"] and r["n_above_ma20"]<=r["n_ma20"]<=r["n_traded"]<=r["n_expected_traded"]<=r["n_eligible"],name,"counts/denominators",[r["date"]],"家数或分母关系无效")
            guard(r["amount_yuan"]>0,name,"amount_yuan",[r["date"]],"成交额必须为正")
        elif name=="finance":
            guard(r["buy_yuan"]>=0 and r["repay_yuan"]>=0,name,"buy_yuan/repay_yuan",[r["date"]],"融资买入/偿还额不能为负")
            guard(r["date"]<r["available_date"]<=state["plan"]["friday"],name,"available_date",[r["date"]],"融资必须在交易日后、报告截止前真实可得")
        elif d["kind"]=="price":
            guard(r["close"]>0,name,"close",[r["date"]],"指数价格必须为正")
    points=e.get("points_used")
    guard(isinstance(points,(int,float)) and math.isfinite(points) and 0<=points<=j["max_points"],name,"points_used",j["dates"],"积分结算未知或超出报价；暂停核实")
    chunk={"source":SOURCE,"definition_hash":j["definition_hash"],"receipt_sha256":h,
           "provider_request_id":e["provider_request_id"],"rows":rows}
    ch=digest(chunk);p=Path(root)/"objects"/(ch+".json");p.parent.mkdir(exist_ok=True);p.write_bytes(canonical(chunk))
    state["cache"].setdefault(name,[]).append({"sha256":ch})
    state["spent_bound"]-=j["max_points"]-points
    j["status"]="CACHED";j["points_used"]=points;state["status"]="PLANNED"
    atomic_json(Path(root)/"state.json",state)
    print(f"已缓存 {name} {len(rows)} 行；没有触发下一次调用。")


def resume(root,state,approval_path):
    a=read_json(approval_path);ticket=state.get("ticket",{})
    guard(state["status"]=="PAUSED_WAIT_USER" and a.get("ticket_id")==ticket.get("ticket_id"),"approval","ticket_id",[],"确认必须对应当前异常")
    guard(a.get("confirmed_by_user") is True and a.get("user_text") and a.get("confirmed_at"),"approval","confirmation",[],"需要真实用户确认记录，代理不得自行生成授权")
    decision=a.get("decision")
    guard(decision in {"retry_same_request","reconcile_response","continue_after_fix","cancel_run"},"approval","decision",[],"不允许借恢复确认修改数据源或模型")
    if decision=="retry_same_request":
        for j in state.get("plan",{}).get("jobs",[]):
            if j["status"]=="IN_FLIGHT":j["status"]="PENDING"
        # Unsettled attempt reservations remain consumed. Retry still obeys original caps.
    state["approvals"].append(a);state["status"]="CANCELLED" if decision=="cancel_run" else ("PLANNED" if "plan" in state else "IDLE")
    if "plan" in state:state["resume_for_plan"]=state["plan"]["plan_id"]
    atomic_json(Path(root)/"state.json",state)
    print("已记录用户选择；未执行重试，未修改源、模型或预算。")


def render_wind_report(x,panels,c,friday,plan_id):
    asof=x.index[-1];start=pd.Timestamp(friday)-pd.Timedelta(days=4)
    w=x.loc[start:];prior=x.loc[x.index<start];last=w.iloc[-1]
    # Original function is retained exactly to preserve every original warning condition.
    legacy=weekly_report(x,asof)
    daily=legacy.split("## 周内逐日记录",1)[1].split("## 预警原因",1)[0]
    warning=legacy.split("## 预警原因",1)[1].split("## 市场复盘",1)[0]
    level_summary=next(line for line in legacy.splitlines() if line.startswith("- 预警："))
    path_summary=next(line for line in legacy.splitlines() if line.startswith("- 周内收盘最大回撤"))
    lines=[f"# A股情绪周报 v{VERSION}","",f"唯一数据源：{SOURCE}；周五周期 {friday}；行情截至 {asof.date()}；计划 {plan_id}。","",
           "## 1. 情绪指数","",f"S={fmt(last.S)}，{sentiment_state(last.S)}。上周末S={fmt(prior.S.iloc[-1])}；周均/最低/最高={fmt(w.S.mean())}/{fmt(w.S.min())}/{fmt(w.S.max())}。",
           f"周内最低S日期：{w.S.idxmin().date()}。",
           "","## 2. 中证全指表现","",f"{prior.close.iloc[-1]:.2f} → {last.close:.2f}；周收益 {fmt(last.close/prior.close.iloc[-1]-1,True)}。",path_summary,
           "","## 3. 预警",level_summary,daily,warning]
    titles={"sector":"4. 板块线","style":"5. 情绪线","funding":"6. 资金流向","external":"7. 外围市场"}
    for role,title in titles.items():
        lines += ["","## "+title,""]
        if role=="funding":
            lines += [f"核心融资强度 {fmt(last.leverage,True)}；融资交易窗口末日 {pd.Timestamp(last.finance_trade_date).date()}，滞后 {int(last.finance_age_sessions)} 个交易日。"]
        for name,df in panels.items():
            d=c["datasets"][name]
            if d["role"]!=role:continue
            week=df.loc[df.index>=start];base=df.loc[df.index<start]
            if d["kind"]=="price":
                guard(len(base)>0 and len(week)>0,name,"weekly_base",[friday],"复盘价格缺上周基点或本周观测")
                lines.append(f"- {d['label']}：{base.close.iloc[-1]:.4f} → {week.close.iloc[-1]:.4f}，区间收益 {fmt(week.close.iloc[-1]/base.close.iloc[-1]-1,True)}，截至 {week.index[-1].date()}，单位 {d['unit']}。")
            else:
                guard(len(week)>0,name,"weekly_flow",[friday],"本周资金流量为空")
                lines.append(f"- {d['label']}：本周已披露流量合计 {week.value.sum():.4f} {d['unit']}；截至 {week.index[-1].date()}。")
        lines.append("仅报告已核验口径；相对涨跌不等于资金流入，成交额不是净流入，外围相关走势不自动证明因果。")
    return "\n".join(lines)+"\n"


def report(root,state,c,now):
    check_frozen(state,c);time_gate(state,now);p=state["plan"]
    guard(all(j["status"]=="CACHED" for j in p["jobs"]),"workflow","pending_jobs",[],"仍有未完成或未知结果请求，禁止发布")
    frames={}
    for name,d in c["datasets"].items():
        cached=cache_rows(root,state,name,d);dates=p["expected"][name]
        guard(all(day in cached for day in dates),name,"cache_dates",dates,"缓存存在缺日")
        frames[name]=pd.DataFrame([cached[day] for day in dates])
    manifest={k:True for k in ["point_in_time","includes_delisted","st_history_verified","adjustment_verified","calendar_verified","finance_availability_verified"]}
    manifest.update(universe="CN_SH_SZ_A_NONST_60",source=SOURCE,finance_scope="CN_SH_SZ_A_STOCK_ONLY")
    try: x=calculate(frames["index"],frames["market"],frames["finance"],manifest)
    except ValueError as exc: raise StopRun("index/market/finance","core_validation",p["expected"]["index"],str(exc)) from exc
    start=pd.Timestamp(p["friday"])-pd.Timedelta(days=4);week=x.loc[start:]
    guard(not week.empty and week.S.notna().all(),"core","S",[str(i.date()) for i in week.index],"任一周内核心因子无效：S=NA，停止正式报告；不改权重、不用P替代")
    guard(x.loc[x.index<start].S.notna().iloc[-1],"core","previous_S",[],"上周末S不可用")
    panels={name:date_frame(df) for name,df in frames.items() if name not in CORE_FIELDS}
    result=render_wind_report(x,panels,c,p["friday"],p["plan_id"])
    destination=Path(root)/"reports"/(p["friday"]+"_"+p["plan_id"][:12])
    guard(not destination.exists(),"workflow","report_id",[p["friday"]],"该周该计划已发布，不重复执行")
    destination.mkdir(parents=True)
    x.to_csv(destination/"daily_scores.csv",index_label="date",encoding="utf-8-sig")
    (destination/"weekly_report.md").write_text(result,encoding="utf-8")
    atomic_json(destination/"manifest.json",{"version":VERSION,"source":SOURCE,"plan_id":p["plan_id"],"config_hash":p["config_hash"],"calendar_hash":p["calendar_hash"],"calls":state["calls"],"points_bound":state["spent_bound"],"report_sha256":hashlib.sha256(result.encode()).hexdigest()})
    state.setdefault("completed_weeks",{})[p["friday"]]=str(destination)
    state["status"]="COMPLETE";atomic_json(Path(root)/"state.json",state)
    print(str(destination/"weekly_report.md"))


def initialize(root):
    datasets={}
    for name,fields in CORE_FIELDS.items():
        datasets[name]={"label":name,"fields":fields,"semantic_contract":CORE_CONTRACTS[name],
                        "calendar":"CN","kind":"core","role":name,
                        "tool":"get_index_kline" if name=="index" else "get_financial_data"}
    targets=[("sector_info","sector","中证信息技术"),("sector_energy","sector","中证能源"),
             ("sector_finance","sector","中证金融地产"),("sector_consumer","sector","中证主要消费"),
             ("sector_health","sector","中证医药卫生"),("style_300","style","沪深300"),
             ("style_1000","style","中证1000"),("style_50","style","上证50"),
             ("style_chinext","style","创业板指"),("style_star","style","科创50"),
             ("external_spx","external","标普500"),("external_nasdaq","external","纳斯达克综合指数")]
    for name,role,label in targets:
        datasets[name]={"label":label,"fields":["date","close"],"semantic_contract":None,
                        "calendar":"US" if role=="external" else "CN","kind":"price","role":role,
                        "unit":"指数点","tool":"get_index_kline"}
    for d in datasets.values():
        d.update(verified=False,definition_evidence=None,period="daily",
                 wind_fields={k:None for k in d["fields"]},catalog_id=None,entity_or_scope=None,
                 max_rows_per_call=None,max_points_per_call=None)
    config={"source":SOURCE,"model_version":VERSION,"approved_by_user":False,"history_start":None,
            "budget":{"max_calls":None,"max_points":None},"datasets":datasets}
    path=Path(root)/"config.json"
    guard(not path.exists(),"config","init",[],"已有配置，不覆盖；请审查现有文件")
    atomic_json(path,config)
    atomic_json(Path(root)/"calendar_template.json",{"source":SOURCE,"request_id":None,
        "covers_from":None,"covers_through":None,"sha256":None,"data":{"CN":[],"US":[]}})
    print("已生成15个原观察范围数据集的待核验配置。没有Wind真实字段、报价或数据，不会执行付费调用。")


def main():
    ap=argparse.ArgumentParser(description="Wind-only v1.1 local planner/cache/calculator. No network calls.")
    ap.add_argument("action",choices=["init","plan","claim","ingest","report","resume","fail"])
    ap.add_argument("--root",type=Path,default=Path("wind_cache_v110"))
    ap.add_argument("--config",type=Path)
    ap.add_argument("--calendar",type=Path)
    ap.add_argument("--friday")
    ap.add_argument("--receipt",type=Path)
    ap.add_argument("--approval",type=Path)
    ap.add_argument("--reason",default="Wind取数失败/积分不足/结果未知，请核实当前请求")
    a=ap.parse_args();now=datetime.now(CN)
    try:
        with exclusive(a.root):
            state=get_state(a.root)
            try:
                if a.action=="init":initialize(a.root);return
                if a.action=="resume":resume(a.root,state,a.approval);return
                if a.action=="fail":
                    jobs=state.get("plan",{}).get("jobs",[]);active=next((j for j in jobs if j["status"]=="IN_FLIGHT"),{})
                    raise StopRun(active.get("dataset","workflow"),"response",active.get("dates",[]),a.reason)
                c=settings(a.config)
                if a.action=="plan":plan(a.root,state,c,calendar_data(a.calendar,c,a.friday),a.friday)
                elif a.action=="claim":claim(a.root,state,c,now)
                elif a.action=="ingest":ingest(a.root,state,c,a.receipt)
                elif a.action=="report":report(a.root,state,c,now)
            except Exception as exc:
                stop_record(a.root,state,exc);raise SystemExit(2)
    except StopRun as exc:
        print(json.dumps({"status":"PAUSED_WAIT_USER","location":exc.location,"reason":str(exc),"options":["核实现有执行器状态后再操作","保留现场并取消本次运行"]},ensure_ascii=False));raise SystemExit(2)


if __name__=="__main__":main()

```
