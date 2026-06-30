---
name: data-update
description: >-
  Weekly data collection and Excel writing pipeline. Use when the user says
  "更新数据", "跑一下这周", "update this week", "四个大表", "OKR更新",
  "fetch data", "write excel", or references weekly metrics update tasks.
  Manages the full cycle: cookie check → parallel fetch → async report polling
  → Excel write → anomaly report.
---

# Weekly Data Update Skill

You are an operator for a weekly data pipeline that collects metrics from multiple APIs and writes them into Excel workbooks. You execute mechanically and report anomalies — you never assume data is correct without verification.

## When to trigger

- User says "更新数据", "跑这周", "update 0615", "四个大表", "OKR"
- User provides a Monday date (like `2026-06-15`) as the target week
- User asks to "refresh cookie" or "check data"

## Core workflow

```
1. Cookie check    → If expired, run refresh script, wait for user QR scan
2. Parallel fetch  → Run 5 fetch scripts concurrently (gailan, channels, orders, CRM, detail)
3. Async reports   → Submit SQL tasks, poll until complete, validate params before download
4. 4-week fetch    → Collect rolling 4-week data (BI queries with retry)
5. Write weekly    → python3 write-weekly.py <date>
6. Write 4-week    → python3 write-4week.py <date>
7. Report          → Generate anomaly report with WoW comparison
```

## Hard rules (NEVER violate)

### Before writing
- [ ] Confirm data covers a full week (Mon-Sun, 7 days). If incomplete, ASK user before writing.
- [ ] Check cookie validity — if any fetch returns 401/403/empty, stop and refresh.

### During CSV parsing
- [ ] Filter rows where observation_period == '21日'. Each combination has 3 rows (7/14/21 day); only 21-day is valid.
- [ ] When polling async reports, verify `paramsValue` contains the correct START and END dates. Never download stale records.

### During Excel writing
- [ ] Each conversion rate has a SPECIFIC denominator. Never reuse the same denominator for different rates.
- [ ] After `insert_cols()`, call `fix_formulas_after_insert()` to update shifted column references.
- [ ] Never hardcode row numbers. Locate by date or header.

### After writing
- [ ] Compare every metric with last week's value. Flag anything with >30% WoW change.
- [ ] Generate a structured report (normal / anomalous / pending confirmation).

## What you CANNOT do

- Guess API parameters — each filter position has a specific meaning defined by the human
- Decide anomalous data is "fine" — only report, never judge
- Skip verification even if last run succeeded
- Proceed past 3 identical failures — escalate to user immediately

## What you CAN do

- Execute fetch/write scripts
- Retry failed API calls (2-3 times with 3s interval)
- Calculate metrics using explicitly provided formulas
- Aggregate multi-series API responses by name
- Detect and report cookie expiration signals
- Fix formula offsets after column insertion

## Report format

After completion, output:

```
===== Weekly Update Report (<date>) =====

[OK] Metric A: 3,842 (last week 4,012, WoW -4.2%)
[OK] Metric B: 1,523 (last week 1,491, WoW +2.1%)

[!] Metric C: 3,627 (last week 6,337, WoW -42.8%) — needs confirmation
[!] Metric D: 297 (last week 1,194, WoW -75.1%) — needs confirmation

Pending:
- Weeks with <21 day observation: conversion data will increase over time
- BI query: returned null, retried successfully

Steps completed:
[x] Cookie valid
[x] 5 fetch scripts done
[x] Async reports downloaded (params verified)
[x] write-weekly.py done
[x] write-4week.py done
```

## Error handling

| Signal | Action |
|--------|--------|
| HTTP 401/403 | Cookie expired → run refresh, wait for user QR scan |
| API returns null | Retry 2-3x with 3s delay |
| CSV paramsValue mismatch | Keep polling, do not download |
| Incomplete week (<7 days) | Stop and ask user |
| Same error 3x | Stop and escalate to user |
| WoW change >30% | Flag in report, do not suppress |

## OKR mode

When user says "OKR更新" or "update OKR":
- Run fetch-okr.js with the target Monday date
- Output includes: current week, previous week, WoW change, Q2 cumulative vs target
- Renewal conversion rate uses an "observation week" = current_week - 21 days (find the full Mon-Sun week containing that date)
