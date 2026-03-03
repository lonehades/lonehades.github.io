---
layout: post
title: "Real‑time Cashflow Automation for Small Clinics with OpenClaw + Notion"
date: 2026-03-03 11:00:00 +0900
categories: [AI Agent, Automation, Finance]
tags: [OpenClaw, Notion, Cashflow, Accounting, Shortcuts]
---

# Real‑time Cashflow Automation for Small Clinics with OpenClaw + Notion

Small clinics rarely have dedicated accounting staff. But without a **3‑month cashflow forecast**, survival becomes guesswork. I built a fully automated pipeline that turns raw bank/card SMS into a **real‑time ledger** and **budget‑vs‑actual dashboard**.

---

## The Pipeline (End‑to‑End)

1. **Bank/Card SMS alerts**
2. **iPhone Shortcuts** captures SMS and writes to Notion
3. **Notion Transactions DB** stores raw entries
4. **OpenClaw** parses/normalizes transactions into accounting‑ready data
5. **Notion Cashflow Dashboard** shows real‑time budget vs actual
6. **Monthly Budget/Plan DB** updates automatically

It’s essentially **automated daily cashflow** with zero manual bookkeeping.

---

## Why This Matters

- **Real‑time visibility** into cashflow and burn
- **Budget vs actual** always current
- **Forecasting** becomes credible (at least 3 months ahead)
- Founders and managers can focus on operations—not ledger work

---

## Key Components

- **iPhone Shortcuts** for SMS capture
- **Notion DB** for transactions + budget + dashboard
- **OpenClaw** for parsing, normalization, and analytics

> Android users can build this even faster because SMS routing is more open. If you have OpenClaw, you can ask it to generate the full automation workflow.

---

## Practical Tips

- Keep a **clean transaction schema** in Notion (amount, category, method, timestamp)
- Use **OpenClaw rules** for normalization (merchant → category, refunds, transfers)
- Design the dashboard around **weekly and monthly cashflow health**

---

## Full Korean Write‑up

If you want the deep technical breakdown and screenshots, here’s the original KR post:

- https://lonehades.github.io/.../%ED%98%84%EA%B8%88%ED%9D.../

---

*This system has already been deployed in a real clinic environment. If you want a more detailed technical guide (English), I can publish a full walk‑through.*
