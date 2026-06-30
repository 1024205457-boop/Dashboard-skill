# AI 驱动的周度数据更新工作流 — 设计方法论

## 概述

本文档总结了一套 AI（Claude Code）协助人类完成周度数据采集、处理、写入 Excel 的工作流设计方法论。核心理念：**AI 是实习生，人类是 mentor**——AI 负责机械执行和初步校验，人类负责业务判断和最终确认。

---

## 一、工作流架构设计

### 分层流水线（6步）

```
update-all.sh
├── Step 1: 核对前一周数据（verify-prev-week.py）
├── Step 2: 采集当周数据（5个fetch脚本并行）
├── Step 3: Mario 异步任务（提交+轮询+下载CSV）
├── Step 4: 采集4周滚动数据（fetch-4week.js）
├── Step 5: 写入当周 Excel（write-weekly.py）
└── Step 6: 写入4周 Excel（write-4week.py）
```

### 设计原则

1. **采集和写入分离**：fetch 脚本只产出 JSON/CSV 到 `input/`，write 脚本只读 `input/` 写 `output/`。中间态可审查、可重跑。
2. **幂等性**：同一参数重复执行结果一致。写入前先定位目标行/列，已有数据覆盖而非追加。
3. **并行不依赖**：5个 fetch 脚本互不依赖，可并行执行。Mario 异步任务提交后不阻塞其他采集。
4. **先验证再写入**：Step 1 校验上周数据完整性，有异常则中止。
5. **最小化写入范围**：4周脚本只对当前周写入新的 BI 加微数据，历史周保留 Excel 已有值。

### 为什么不做成一个大脚本

- 调试时可以单步重跑（只重跑出错的 step）
- 不同 step 的失败原因不同（cookie 过期 vs API 超时 vs 公式错误）
- AI 修改时能精确定位问题模块，不影响其他

---

## 二、提示词（Prompt）怎么写

### 给 AI 的指令结构

```
1. 明确目标：「更新 0608 这周的四个大表 + OKR」
2. 提供约束：「先刷新 cookie」「21日观察期未满的不要写」
3. 验证条件：「跑完后对比上周，环比变化超过50%要报告」
4. 异常处理：「如果 BI 查询返回 null，重试3次」
```

### 关键原则

| 原则 | 说明 | 反例 |
|------|------|------|
| 给公式不给结论 | 告诉 AI「加微转化率=转化uv/加微uv」 | 不要说「大约4%~6%」 |
| 给边界不给自由 | 「col14=转化/加微, col15=转化/激活, col16=转化/参课」 | 不要说「算一下转化率」 |
| 给校验不给信任 | 「写入后读回来验证数值一致」 | 不要说「写完就行」 |
| 给来源不给推测 | 「流量池加微=BI查询(emonTagName=纯新用户+小程序)」 | 不要说「加微数据你自己找」 |

### 容易踩坑的表述

- **「更新一下」** → AI 可能只更新本周，忽略4周滚动。应明确「四个大表+OKR，包含4周滚动」
- **「算一下转化率」** → AI 可能用错分母。应写清「转化率=A/B，A来自X字段，B来自Y字段」
- **「数据不对你看看」** → AI 会瞎猜。应给出「上周实际值是X，你写的是Y，差异在Z」

---

## 三、各环节的作用与 AI 能力边界

### Step 1: 核对前一周数据

| 项目 | AI做 | 人做 |
|------|------|------|
| 读取Excel已有数据 | ✅ | |
| 对比JSON源数据 | ✅ | |
| 发现数值偏差 | ✅ | |
| 判断偏差是否合理（如21日观察期尚未满） | | ✅ |
| 决定是否覆盖修正 | | ✅ |

### Step 2: 采集当周数据

| 项目 | AI做 | 人做 |
|------|------|------|
| 执行 fetch 脚本 | ✅ | |
| 检查返回数据非空 | ✅ | |
| 判断 cookie 是否过期并刷新 | ✅（执行刷新脚本）| ✅（扫码） |
| 判断数据量级是否合理 | ✅（与上周对比）| ✅（最终确认） |
| 判断数据为什么下降 | ⚠️（给出假设）| ✅（确认原因） |

### Step 3: Mario 异步任务

| 项目 | AI做 | 人做 |
|------|------|------|
| 提交 Mario SQL 任务 | ✅ | |
| 轮询等待完成 | ✅ | |
| 校验 paramsValue 匹配（防止下载旧记录）| ✅ | |
| 解析 CSV（过滤21日行）| ✅ | |
| 判断 Mario SQL 逻辑是否正确 | | ✅ |

### Step 4-6: 数据写入

| 项目 | AI做 | 人做 |
|------|------|------|
| 按公式计算指标 | ✅ | |
| 写入 Excel 指定单元格 | ✅ | |
| insert_cols 后修复历史公式 | ✅ | |
| 确认不完整数据是否要写入 | | ✅ |
| 确认异常数值是否写入（如环比-80%）| | ✅ |

---

## 四、AI 必须做的事（非做不可）

### 1. 写入前确认数据完整性

```
如果数据不满7天（周一到周日），必须问用户：
「这周数据只到X月X日，还差N天，是否继续写入？」
```

**教训**：曾把周中不完整数据直接写入，拼团成功用户 1264（实际3930），所有指标严重偏低。

### 2. Mario CSV 必须过滤观察期

```javascript
// 每组合3行（7日/14日/21日），只取21日
if (row[观察期字段] === '21日') { ... }
```

**教训**：不过滤会导致购买用户数 3 倍 overcounting。

### 3. Mario 轮询校验参数匹配

```javascript
function isMatchingRecord(rec) {
  const params = rec.paramsValue || '';
  return params.includes(START) && params.includes(END);
}
```

**教训**：`marioLatestQuery` 可能返回旧的成功记录（范围不同），导致分销某周无数据。

### 4. insert_cols 后修复历史公式

```python
# openpyxl insert_cols 不会更新被推移列中的公式引用
fix_formulas_after_insert(ws, insert_col, max_row, max_col)
```

**教训**：不修复会导致"这周和上周流失率一样"。

### 5. 公式分母必须明确指定

```python
# 不同转化率的分母不同！
col14 = 转化uv / 加微uv      # 加微转化率
col15 = 转化uv / 激活uv      # UV转化率
col16 = 转化uv / 参课uv      # 参课转化率
```

**教训**：三列全用同一个分母（领取），导致三个率完全相同。

### 6. Pipe chart API 多 series 聚合

```javascript
// 当筛选多个 user_type 时，API 返回多个 series
// 必须按 name 聚合，不能只取 series[0]
for (const s of series) {
  if (s.name.includes('成单uv')) orderUv += sumSeries(s);
  if (s.name.includes('转化uv')) zhuanhuaUv += sumSeries(s);
}
```

### 7. BI 查询 null 值重试

BI API 偶尔返回 null（服务端缓存未就绪），需自动重试 2-3 次，间隔 3 秒。

---

## 五、AI 绝对不能做的事

### 1. 不能猜 API 参数

- Pipe selects 每一位代表什么筛选，必须由人指定或从抓包获取
- Mario SQL 模板 ID、paramsValue 格式必须精确，不能"差不多"
- BI 查询模板 138KB，字段名/过滤条件/日期格式必须与模板严格一致

### 2. 不能跳过核对直接写入

即使上次跑通了，这次也必须：
- 检查 cookie 是否过期（约一周过期）
- 检查数据是否覆盖完整一周
- 对比上周数据，环比异常要报告

### 3. 不能自行决定"数据正常"

AI 可以报告「本周小课包订单 3627，上周 6337，环比 -42.8%」，但不能自行判断这是否合理。可能原因：
- 推送停了（业务变动）
- 21日观察期未满（正常）
- Cookie 过期导致采集为空（Bug）

这些需要人来判断。

### 4. 不能假设列结构不变

Excel 列随时可能被人手动调整。每次写入前：
- 确认目标 sheet 和行/列位置
- 用 header 或日期定位，不硬编码行号

### 5. 不能用错误的时间范围

- 本周数据：周一~周日（7天）
- 4周滚动：最近4个完整周
- 21日观察期：weekEnd + 21天后才是有效的21日转化数据
- 续报转化率的"观察周"≠当前周，是当前周减去21天后找到的那个完整周

### 6. 不能在同一位置卡住超过3次

遇到同一个错误反复出现（如 Playwright 版本不匹配、API 持续 502），立刻告知用户寻求帮助，不要死磕。

---

## 六、异常检测与报告模板

AI 完成数据写入后，应生成如下格式的报告：

```
===== 周度数据更新报告 (2026-06-15) =====

正常指标:
  拼团成功用户: 3,842 (上周 4,012, 环比 -4.2%) ✅
  企微好友新增: 1,523 (上周 1,491, 环比 +2.1%) ✅

异常指标（环比变化 >30%）:
  ⚠️ 小课包订单: 3,627 (上周 6,337, 环比 -42.8%)
  ⚠️ bkqw-纯新用户: 297 (上周 1,194, 环比 -75.1%)

待确认:
  - 21日观察期未满的周: 0608, 0615（转化数据偏低属正常）
  - 流量池加微: BI返回null已重试成功，最终值 523

操作:
  [x] cookie 刷新
  [x] 5个fetch脚本执行完成
  [x] Mario 27717/28543 任务完成并下载
  [x] write-weekly.py 写入完成
  [x] write-4week.py 写入完成
```

---

## 七、Cookie 管理策略

```
Cookie 约一周过期。流程：
1. 更新前先用 refresh-cookie.js 刷新
2. 脚本启动 Playwright 打开登录页
3. 人工扫码（唯一需要人工介入的环节）
4. 脚本自动保存 pipe-cookie.txt / emon-cookie.txt / bi-cookie.txt
5. 如果中途某个 fetch 返回 401/403，说明 cookie 已过期，需重新刷新
```

AI 可以识别 cookie 过期的信号（HTTP 状态码、返回空数据、"未登录"错误信息），但不能自动完成扫码登录。

---

## 八、数据源与指标对应关系（核心知识）

| 指标 | 数据源 | 关键注意点 |
|------|--------|-----------|
| 拼团/新用户/卡型 | Pipe chart/detail API | selects 参数每位含义不同 |
| 转化漏斗 | Mario 26833 CSV | 需等几分钟完成 |
| 流量池指标 | Pipe API (chart) | 领取/激活/参课/完课/转化 各有不同分母 |
| 腾讯z/分销转化 | Mario 28543 CSV + BI 加微 | CSV过滤21日 + BI活码名精确匹配 |
| 体验渠道转化占比 | Mario 27717 CSV | 同上，必须过滤21日 |
| 企微好友新增/流失 | Emon API | groupIds=[1,75,428,949] |
| 公众号累计关注 | 微信公众号 API | 需要 mp-cookie + mp-token |
| 流量池加微 | BI 查询 | emonTagName=纯新用户+小程序，不是 biChannels 总计 |

---

## 九、工作流演进原则

### 何时新增脚本 vs 修改现有

- **新指标、新数据源** → 新脚本（如 fetch-okr.js）
- **同数据源的新筛选条件** → 在现有 fetch 脚本中加参数
- **写入目标变了** → 修改 write 脚本，不动 fetch

### 何时引入自动化 vs 保持手动

- **机械重复、无需判断** → 自动化（如 fetch + write）
- **需要业务判断** → 人工确认点（如异常数据是否写入）
- **一次性操作** → 临时脚本（tmp-*.js），不入主流程

### 如何安全地修改计算逻辑

1. 先读当前代码，理解现有逻辑
2. 明确要改的公式和位置
3. 改完后用已知数据验证（拿上周的 input JSON 重跑 write，对比 Excel 输出）
4. 不要连带"优化"不相关的代码

---

## 十、总结：人机协作的黄金法则

```
                AI 的职责                          人的职责
┌─────────────────────────────┐    ┌─────────────────────────────┐
│ 执行采集脚本                 │    │ 扫码登录（cookie刷新）       │
│ 按公式计算指标               │    │ 确认异常数据是否合理         │
│ 写入 Excel 指定位置          │    │ 判断业务原因（推送停了？）   │
│ 发现环比异常并报告           │    │ 决定不完整数据是否写入       │
│ 重试失败的 API 调用          │    │ 提供新指标的公式和来源       │
│ 校验 Mario 参数匹配         │    │ 最终审核写入结果             │
│ 修复公式引用偏移             │    │ 指定 API 参数含义           │
└─────────────────────────────┘    └─────────────────────────────┘

黄金法则：AI 永远不假设，只报告和等待确认。
```

---
---

# AI-Driven Weekly Data Update Workflow — Design Methodology

## Overview

This document summarizes the design methodology for an AI (Claude Code) assisted weekly data collection, processing, and Excel writing workflow. Core philosophy: **AI is the intern, the human is the mentor** — AI handles mechanical execution and preliminary validation, while the human handles business judgment and final confirmation.

---

## 1. Workflow Architecture

### Layered Pipeline (6 Steps)

```
update-all.sh
├── Step 1: Verify previous week's data (verify-prev-week.py)
├── Step 2: Collect current week's data (5 fetch scripts in parallel)
├── Step 3: Mario async tasks (submit + poll + download CSV)
├── Step 4: Collect 4-week rolling data (fetch-4week.js)
├── Step 5: Write current week Excel (write-weekly.py)
└── Step 6: Write 4-week Excel (write-4week.py)
```

### Design Principles

1. **Separation of collection and writing**: Fetch scripts only output JSON/CSV to `input/`; write scripts only read `input/` and write to `output/`. Intermediate state is inspectable and re-runnable.
2. **Idempotency**: Same parameters produce the same result. Writes locate target rows/columns first and overwrite rather than append.
3. **Parallel without dependencies**: 5 fetch scripts are independent and run in parallel. Mario async task submission doesn't block other collection.
4. **Verify before writing**: Step 1 validates previous week's data integrity; aborts on anomalies.
5. **Minimal write scope**: The 4-week script only writes new BI data for the current week; historical weeks retain existing Excel values.

### Why Not One Big Script

- Allows single-step re-runs during debugging (only re-run the failed step)
- Different steps fail for different reasons (cookie expiry vs API timeout vs formula errors)
- AI can precisely locate the problem module without affecting others

---

## 2. How to Write Prompts

### Instruction Structure for AI

```
1. Clear objective: "Update the four main tables + OKR for week 0608"
2. Constraints: "Refresh cookie first" "Don't write if 21-day observation period incomplete"
3. Validation criteria: "After completion, compare with last week; report if WoW change exceeds 50%"
4. Error handling: "If BI query returns null, retry 3 times"
```

### Key Principles

| Principle | Explanation | Anti-pattern |
|-----------|-------------|--------------|
| Give formulas, not conclusions | Tell AI "conversion rate = conversion_uv / add_friend_uv" | Don't say "about 4%~6%" |
| Give boundaries, not freedom | "col14=conversion/add_friend, col15=conversion/activation, col16=conversion/class_attendance" | Don't say "calculate the conversion rates" |
| Give validation, not trust | "Read back after writing to verify values match" | Don't say "just write it" |
| Give sources, not guesses | "Pool add_friend = BI query (emonTagName=new_user+mini_program)" | Don't say "find the add_friend data yourself" |

### Common Prompt Pitfalls

- **"Update it"** → AI might only update current week, ignoring 4-week rolling. Should specify "four tables + OKR, including 4-week rolling"
- **"Calculate the conversion rate"** → AI might use wrong denominator. Should write "rate = A/B, where A comes from field X, B comes from field Y"
- **"The data is wrong, check it"** → AI will guess randomly. Should provide "last week's actual value is X, you wrote Y, the discrepancy is in Z"

---

## 3. Role of Each Step & AI Capability Boundaries

### Step 1: Verify Previous Week Data

| Task | AI does | Human does |
|------|---------|------------|
| Read existing Excel data | ✅ | |
| Compare against JSON source data | ✅ | |
| Detect numerical discrepancies | ✅ | |
| Judge whether discrepancy is acceptable (e.g., 21-day observation period not yet complete) | | ✅ |
| Decide whether to overwrite and correct | | ✅ |

### Step 2: Collect Current Week Data

| Task | AI does | Human does |
|------|---------|------------|
| Execute fetch scripts | ✅ | |
| Check returned data is non-empty | ✅ | |
| Detect cookie expiry and trigger refresh | ✅ (runs refresh script) | ✅ (scans QR code) |
| Judge if data magnitude is reasonable | ✅ (compare to last week) | ✅ (final confirmation) |
| Judge why data dropped | ⚠️ (provide hypothesis) | ✅ (confirm the reason) |

### Step 3: Mario Async Tasks

| Task | AI does | Human does |
|------|---------|------------|
| Submit Mario SQL tasks | ✅ | |
| Poll until completion | ✅ | |
| Validate paramsValue match (prevent downloading stale records) | ✅ | |
| Parse CSV (filter for 21-day rows) | ✅ | |
| Judge whether Mario SQL logic is correct | | ✅ |

### Step 4-6: Data Writing

| Task | AI does | Human does |
|------|---------|------------|
| Calculate metrics by formula | ✅ | |
| Write to specified Excel cells | ✅ | |
| Fix historical formulas after insert_cols | ✅ | |
| Confirm whether to write incomplete data | | ✅ |
| Confirm whether to write anomalous values (e.g., -80% WoW) | | ✅ |

---

## 4. What AI MUST Do (Non-Negotiable)

### 1. Confirm Data Completeness Before Writing

```
If data doesn't cover 7 days (Monday to Sunday), must ask user:
"This week's data only goes to [date], missing N days. Proceed with writing?"
```

**Lesson learned**: Once wrote incomplete mid-week data directly. Group-buy users showed 1,264 (actual was 3,930). All metrics were severely low.

### 2. Mario CSV Must Filter by Observation Period

```javascript
// Each combination has 3 rows (7-day/14-day/21-day), only take 21-day
if (row[observationField] === '21日') { ... }
```

**Lesson learned**: Without filtering, purchase user counts are 3x overcounted.

### 3. Mario Polling Must Validate Parameter Match

```javascript
function isMatchingRecord(rec) {
  const params = rec.paramsValue || '';
  return params.includes(START) && params.includes(END);
}
```

**Lesson learned**: `marioLatestQuery` may return old successful records (different date range), causing missing data for certain weeks.

### 4. Fix Historical Formulas After insert_cols

```python
# openpyxl insert_cols does NOT update formula references in shifted columns
fix_formulas_after_insert(ws, insert_col, max_row, max_col)
```

**Lesson learned**: Without fixing, "this week and last week have the same churn rate."

### 5. Formula Denominators Must Be Explicitly Specified

```python
# Different conversion rates have different denominators!
col14 = conversion_uv / add_friend_uv    # Add-friend conversion rate
col15 = conversion_uv / activation_uv    # UV conversion rate
col16 = conversion_uv / class_attendance_uv  # Class-attendance conversion rate
```

**Lesson learned**: All three columns used the same denominator (claim_uv), resulting in three identical rates.

### 6. Pipe Chart API Multi-Series Aggregation

```javascript
// When filtering multiple user_types, API returns multiple series
// Must aggregate by name, cannot just take series[0]
for (const s of series) {
  if (s.name.includes('order_uv')) orderUv += sumSeries(s);
  if (s.name.includes('conversion_uv')) conversionUv += sumSeries(s);
}
```

### 7. BI Query Null Retry

BI API occasionally returns null (server-side cache not ready). Must auto-retry 2-3 times with 3-second intervals.

---

## 5. What AI Must NEVER Do

### 1. Never Guess API Parameters

- Pipe selects: each position represents a different filter dimension; must be specified by the human or captured from network requests
- Mario SQL template IDs and paramsValue format must be exact — no "close enough"
- BI query template is 138KB; field names, filter conditions, and date formats must strictly match the template

### 2. Never Skip Verification and Write Directly

Even if last run succeeded, this run must still:
- Check if cookie has expired (expires roughly weekly)
- Check if data covers a complete week
- Compare to last week's data; report anomalous WoW changes

### 3. Never Unilaterally Decide "Data Looks Normal"

AI can report "This week's small-course orders: 3,627; last week: 6,337; WoW: -42.8%", but cannot independently judge whether this is acceptable. Possible causes:
- Push notifications stopped (business decision)
- 21-day observation period incomplete (normal)
- Cookie expired causing empty collection (bug)

These require human judgment.

### 4. Never Assume Column Structure Is Unchanged

Excel columns may be manually adjusted at any time. Before every write:
- Confirm target sheet and row/column positions
- Locate by header or date, never hardcode row numbers

### 5. Never Use Wrong Time Ranges

- Current week data: Monday~Sunday (7 days)
- 4-week rolling: the most recent 4 complete weeks
- 21-day observation period: weekEnd + 21 days before the 21-day conversion data is valid
- Renewal conversion rate "observation week" ≠ current week; it's the complete week found by subtracting 21 days from current week

### 6. Never Get Stuck at the Same Point More Than 3 Times

When the same error keeps recurring (e.g., Playwright version mismatch, API persistent 502), immediately inform the user and ask for help. Don't keep hammering.

---

## 6. Anomaly Detection & Reporting Template

After completing data writes, AI should generate a report in this format:

```
===== Weekly Data Update Report (2026-06-15) =====

Normal metrics:
  Group-buy users: 3,842 (last week 4,012, WoW -4.2%) ✅
  WeCom friend adds: 1,523 (last week 1,491, WoW +2.1%) ✅

Anomalous metrics (WoW change >30%):
  ⚠️ Small-course orders: 3,627 (last week 6,337, WoW -42.8%)
  ⚠️ bkqw-new users: 297 (last week 1,194, WoW -75.1%)

Pending confirmation:
  - Weeks with incomplete 21-day observation: 0608, 0615 (low conversion data is normal)
  - Pool add-friend: BI returned null, retry succeeded, final value 523

Operations:
  [x] Cookie refresh
  [x] 5 fetch scripts completed
  [x] Mario 27717/28543 tasks completed and downloaded
  [x] write-weekly.py completed
  [x] write-4week.py completed
```

---

## 7. Cookie Management Strategy

```
Cookies expire approximately weekly. Flow:
1. Refresh with refresh-cookie.js before updates
2. Script launches Playwright and opens login page
3. Human scans QR code (the only manual intervention point)
4. Script automatically saves pipe-cookie.txt / emon-cookie.txt / bi-cookie.txt
5. If any fetch returns 401/403 mid-run, cookie has expired — need to re-refresh
```

AI can recognize cookie expiry signals (HTTP status codes, empty data returns, "not logged in" error messages), but cannot complete QR code login automatically.

---

## 8. Data Source to Metric Mapping (Core Knowledge)

| Metric | Data Source | Key Considerations |
|--------|-------------|-------------------|
| Group-buy/New users/Card types | Pipe chart/detail API | Each position in selects has different meaning |
| Conversion funnel | Mario 26833 CSV | Takes several minutes to complete |
| Traffic pool metrics | Pipe API (chart) | Claim/activation/class/completion/conversion each has different denominator |
| Tencent-z/Distribution conversion | Mario 28543 CSV + BI add-friend | CSV filter 21-day + BI QR-code name exact match |
| Trial channel conversion share | Mario 27717 CSV | Same as above, must filter 21-day |
| WeCom friend add/churn | Emon API | groupIds=[1,75,428,949] |
| WeChat Official Account followers | WeChat MP API | Requires mp-cookie + mp-token |
| Traffic pool add-friend | BI query | emonTagName=new_user+mini_program, NOT biChannels total |

---

## 9. Workflow Evolution Principles

### When to Add New Scripts vs Modify Existing

- **New metric, new data source** → New script (e.g., fetch-okr.js)
- **New filter on same data source** → Add parameters to existing fetch script
- **Write target changed** → Modify write script, leave fetch alone

### When to Automate vs Keep Manual

- **Mechanical repetition, no judgment needed** → Automate (fetch + write)
- **Requires business judgment** → Manual confirmation point (e.g., whether to write anomalous data)
- **One-off operation** → Temporary script (tmp-*.js), not in main flow

### How to Safely Modify Calculation Logic

1. Read current code first, understand existing logic
2. Clearly identify the formula and position to change
3. After modification, validate with known data (re-run write with last week's input JSON, compare Excel output)
4. Don't "optimize" unrelated code along the way

---

## 10. Summary: Golden Rules of Human-AI Collaboration

```
              AI's Responsibilities                    Human's Responsibilities
┌─────────────────────────────────────┐  ┌─────────────────────────────────────┐
│ Execute collection scripts           │  │ QR code login (cookie refresh)      │
│ Calculate metrics by formula         │  │ Confirm if anomalous data is valid  │
│ Write to specified Excel positions   │  │ Judge business reasons (push stopped?) │
│ Detect WoW anomalies and report      │  │ Decide whether to write incomplete data │
│ Retry failed API calls               │  │ Provide formulas and sources for new metrics │
│ Validate Mario parameter matching    │  │ Final review of written results     │
│ Fix formula reference offsets        │  │ Specify API parameter meanings      │
└─────────────────────────────────────┘  └─────────────────────────────────────┘

Golden Rule: AI never assumes — it only reports and waits for confirmation.
```
