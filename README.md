<div align="center">

# 🤖 FMCG Sales Promotion Bot

**An end-to-end agentic automation** that turns a raw FMCG sales CSV into a decision-ready promotion insight — in one orchestrated, fully traceable run.

![UiPath](https://img.shields.io/badge/Orchestration-UiPath%20Studio-E3000B?logo=uipath&logoColor=white)
![Claude](https://img.shields.io/badge/AI-Anthropic%20Claude-191919?logo=anthropic&logoColor=white)
![Google Sheets](https://img.shields.io/badge/Output-Google%20Sheets-34A853?logo=google-sheets&logoColor=white)
![JavaScript](https://img.shields.io/badge/Script%20Tasks-JavaScript-F7DF1E?logo=javascript&logoColor=black)
![Status-Academic](https://img.shields.io/badge/Type-Academic%20Project-10B981)

</div>

---

## ✨ At a glance
- 📦 **Domain:** FMCG (Fast-Moving Consumer Goods) retail promotions
- ⚙️ **Orchestration:** UiPath Studio — BPMN Agentic Process, 5 named agents in a linear deterministic sequence
- 🧠 **AI layer:** Anthropic `claude-sonnet-4-6` generates a plain-English executive summary + 2 actionable recommendations
- 📊 **Output:** Timestamped row appended to a shared Google Sheets dashboard after every run
- ⏱️ **Before:** 5–6 hours of manual pivot-table work per promotion cycle → **After:** one automated run

---

## 🧭 Table of contents
- [🔎 Problem](#-problem)
- [🎯 Goal](#-goal)
- [🏗️ Architecture](#️-architecture)
- [🤖 Agent design](#-agent-design)
- [🗃️ Dataset](#️-dataset)
- [📌 Key outputs](#-key-outputs)
- [📁 Repo layout](#-repo-layout)

---

## 🔎 Problem

Category managers in FMCG drown in spreadsheets when reviewing promotions:

| Pain point | Impact |
|---|---|
| Manual extract → pivot → analyze cycle | **5–6 hours** per promotion review |
| Guessed discount depth & timing | **15–20%** uplift lost |
| No structured audit trail | Minimal record of how decisions were reached |

---

## 🎯 Goal

**Turn raw sales data into a decision-ready insight, automatically.**

The bot answers four questions a category manager actually asks — then hands back a written recommendation — all in a single run:

| Step | Question |
|---|---|
| **Measure** | How much did promotions lift sales overall, by category, channel, and country? |
| **Detect** | What kind of promotion works best? Deep (BOGO-style) vs flat, weekend vs weekday, holiday vs non-holiday? |
| **Reason** | What should we do next? (Claude generates the narrative + 2 concrete recommendations) |
| **Deliver** | Where does it land? A timestamped row on a shared Google Sheet — a living dashboard, not a one-off report |

---

## 🏗️ Architecture

Four building blocks wired into **one connected pipeline**:

| Block | Role | Technology |
|---|---|---|
| Orchestration | BPMN process with 5 named agents in a fixed Start → End sequence | UiPath Studio |
| Data In | HTTP GET pulls the sales CSV from a Google Drive download link | Drive + HTTP connector |
| Reasoning | Receives computed KPIs, returns executive summary + recommendations | Anthropic Claude API |
| Data Out | `WriteRow` in APPEND mode adds a timestamped result row | Google Sheets connector |

> Two script tasks (Agents 2 & 3) run JavaScript **in-process** — no external compute, no extra latency.

---

## 🤖 Agent design

**Five agents, each doing one job.**  
The orchestration runs them in a fixed line: `Start → Agent 1 → 2 → 3 → 4 → 5 → End`. Every run is identical and fully traceable.

### Agent 1 — Load CSV Data · *Send Task / HTTP connector*
| | |
|---|---|
| **Input** | Start event trigger |
| **Does** | HTTP GET against a fixed Google Drive download URL |
| **Output** | Raw CSV text stored in `csv_raw_data` |
| **Guardrails** | Single fixed source URL (no arbitrary fetch); error branch defined on task |

---

### Agent 2 — Promotion Analysis · *Script Task / in-process JavaScript*
| | |
|---|---|
| **Input** | `promo_flag`, `units_sold`, `category`, `channel`, `country` |
| **Does** | Parses CSV, splits promoted vs non-promoted rows, computes % uplift overall and grouped four ways |
| **Output** | `scriptResponse`: overall uplift + per-group breakdown |
| **Core logic** | `uplift = (avg(promo) − avg(noPromo)) / avg(noPromo) × 100` |
| **Guardrails** | Empty groups return "0" — never divide by zero; pure function of the CSV (fully reproducible) |

---

### Agent 3 — Trend Detection · *Script Task / in-process JavaScript*
| | |
|---|---|
| **Input** | `discount_pct`, `is_weekend`, `is_holiday` (promoted rows only) |
| **Does** | Buckets units across three binary splits: deep (≥ 0.25) vs flat, weekend vs weekday, holiday vs non-holiday |
| **Output** | `scriptResponse2`: avg units per split |
| **Guardrails** | Skips non-promo rows explicitly; threshold isolated as a single constant (`0.25`) |

---

### Agent 4 — Generate AI Insights · *Send Task / Anthropic Claude*
| | |
|---|---|
| **Input** | `overall_uplift`, `flat_avg_units`, `bogo_avg_units`, `category_breakdown` |
| **Does** | Calls `claude-sonnet-4-6` with a fixed analyst prompt (temp 1, 1 024 tokens) |
| **Output** | 2-sentence executive summary + 2 actionable recommendations as text |
| **Guardrails** | Templated prompt (only numbers vary per run); Claude sees only aggregates, never raw records |

---

### Agent 5 — Write Dashboard Output · *Send Task / Google Sheets*
| | |
|---|---|
| **Input** | Date, overall uplift, per-category uplift values |
| **Does** | `WriteRow` in APPEND mode to `Sheet1!A1:Z1000` (headers off) |
| **Output** | New timestamped row on the shared dashboard sheet |
| **Guardrails** | APPEND only — never overwrites history; bounded range keeps writes predictable |

---

## 🗃️ Dataset

**FMCG Multi-Country Sales Dataset** — public dataset from Kaggle, hosted as a CSV on Google Drive for the bot run.

**Columns the agents read:**

```
units_sold  ·  promo_flag  ·  discount_pct
category   ·  channel     ·  country
is_weekend  ·  is_holiday
```

**How the data is interpreted:**
- `promo_flag = 1` marks a promoted transaction
- `discount_pct >= 0.25` is treated as a deep "BOGO-style" deal; below that is "flat"
- Grouped three ways: by **category**, **channel**, and **country**
- Timing splits: **weekend vs weekday** and **holiday vs non-holiday**

A 5 000-row sample (`Dataset_fmcg_sales_5k.csv`) is included in this repo for reference. The full dataset (~200 MB) is loaded at runtime from Google Drive.

---

## 📌 Key outputs

Each bot run produces:
1. **Uplift metrics** — overall promo lift %, broken down by category, channel, and country
2. **Trend signals** — whether deep vs flat discounts, weekend timing, or holidays move volume more
3. **AI narrative** — a 2-sentence executive summary + 2 concrete next-step recommendations (from Claude)
4. **Dashboard row** — all of the above appended to a shared Google Sheet with a date stamp for longitudinal tracking

---

## 📁 Repo layout

```
FMCG_Sales_Promo_Bot/
├── Agentic Process/
│   ├── Process.bpmn          # BPMN orchestration diagram (UiPath Studio)
│   ├── bindings_v2.json      # connector bindings
│   ├── entry-points.json     # process entry-point config
│   └── project.uiproj        # UiPath project metadata
├── resources/
│   └── solution_folder/      # UiPath solution resources
├── userProfile/              # UiPath user profile data
├── Dataset_fmcg_sales_5k.csv # 5k-row sample dataset
├── FMCG_Sales_Promo_Bot.uipx # UiPath package export
├── FMCG_Sales_Promo_Bot_Presentation.pdf  # Project presentation
└── CPG_Agentic_Process.pdf   # Agentic process architecture diagram
```
