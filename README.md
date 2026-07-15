# 🚌 FlixBus Price Intelligence — N8N Automation

> An autonomous daily pipeline that benchmarks competitor bus prices, detects pricing anomalies, and fires real-time Slack alerts — zero manual steps.

![N8N](https://img.shields.io/badge/N8N-Workflow-orange?style=flat-square&logo=n8n)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-Data_Source-green?style=flat-square&logo=googlesheets)
![Slack](https://img.shields.io/badge/Slack-Alerts-purple?style=flat-square&logo=slack)
![JavaScript](https://img.shields.io/badge/JavaScript-Code_Nodes-yellow?style=flat-square&logo=javascript)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat-square)

---

## 📌 What This Does

Every morning at **8:00 AM**, this N8N workflow automatically:

1. Pulls all bus listing data from a Google Sheet
2. Cleans the dataset (drops null prices, fills missing reviews)
3. Engineers a **Similarity Group** key for fair peer comparison
4. Splits Flixbus listings from competitors
5. Aggregates competitor average prices per group
6. Calculates price deviation (%) for each Flixbus listing
7. Flags listings outside ±10% as `Too Expensive` or `Too Cheap`
8. Appends results to an audit Google Sheet
9. Sends a Slack alert for every flagged listing

**Result:** Pricing analysts go from 4–6 hours of manual review to 0. Every route, every morning.

---

## 🏗️ Architecture

```
Cron (8AM) → Google Sheets → Clean Data → Derive Fields → IF: Flixbus?
                                                                 │
                                    TRUE (Our buses) ←───────────┤
                                    FALSE (Competitors) → Aggregate by Similarity Group
                                                                 │
                                    Merge (Left Join on Similarity Group)
                                           │
                                    Calculate Price Diff → Flag Logic → Store to Sheets
                                                                              │
                                                                    IF: Flag != OK → Slack Alert
```

### Node Breakdown (12 total)

| # | Node | Type | Purpose |
|---|------|------|---------|
| 01 | Daily Cron | Schedule Trigger | Fires at `0 8 * * *` |
| 02 | Fetch Sheet Data | Google Sheets | Read all bus listings |
| 03 | Clean Data | Code (JS) | Drop null WAP rows, fill missing reviews |
| 04 | Derive Fields | Code (JS) | Load Factor, Dep Hour, Rank Bucket, Similarity Group |
| 05 | Is Flixbus? | IF | Route to our buses vs competitors |
| 06 | Aggregate Competitors | Code (JS) | Avg price per Similarity Group |
| 07 | Merge with Competitors | Merge | Left join on Similarity Group |
| 08 | Calculate Price Diff | Code (JS) | Price Diff & Price Diff % |
| 09 | Apply Flag Logic | Code (JS) | Too Expensive / Too Cheap / OK |
| 10 | Store to Google Sheets | Google Sheets | Append flagged results |
| 11 | Filter Alerts | IF | Keep only flagged rows |
| 12 | Send Slack Alert | Slack | Per-listing alert message |

---

## 🧠 The Key Concept: Similarity Group

The core innovation of this pipeline is the **Similarity Group key** — it ensures you're only comparing Flixbus prices against genuinely substitutable competitor buses, not every bus on a route.

```
Similarity Group = Route Number + Bus Type + Departure Hour + Rank Bucket
```

Where:
- **Route Number** — same origin-destination pair
- **Bus Type** — Standard / Premium / Economy (apples to apples)
- **Departure Hour** — extracted from `HH:MM` departure time
- **Rank Bucket** — `ceil(SRP Rank / 5)` — groups rank 1–5 together, 6–10 together, etc.

> **Why Rank Bucket instead of exact rank?** Rank 1 and Rank 3 on the same route are genuinely comparable — a traveller sees both. Exact rank matching is too strict and creates false positives. Bucketing by 5 captures real competitive proximity.

### Derived Features

| Feature | Formula |
|---------|---------|
| Load Factor | `1 - (Available Seats / Total Seats)` |
| Departure Hour | `parseInt(departureTime.split(":")[0])` |
| Rank Bucket | `Math.ceil(SRP_Rank / 5)` |
| Similarity Group | `Route_BusType_DepHour_RankBucket` |

---

## 🚦 Flag Logic

| Condition | Flag |
|-----------|------|
| Price Diff % > +10% | 🔴 Too Expensive |
| Price Diff % < −10% | 🟡 Too Cheap |
| −10% ≤ diff ≤ +10% | 🟢 OK |
| No competitor in group | ⚪ No Competitor Data |

---

## 📊 Sample Results (240 listings, simulated)

| Flag | Count | % |
|------|-------|---|
| ✅ OK | 142 | 59.2% |
| 🔴 Too Expensive | 38 | 15.8% |
| 🟡 Too Cheap | 47 | 19.6% |
| ⚪ No Competitor Data | 13 | 5.4% |

**Key metrics:**
- 35.4% of listings flagged daily
- Avg overpricing gap: €4.20
- Avg underpricing gap: €3.80
- Pipeline runtime: <1 min end-to-end

---

## 🚀 Setup & Import

### Prerequisites

- [N8N](https://n8n.io) (self-hosted or cloud)
- Google account with Sheets API enabled
- Slack workspace with Bot Token

### Import the Workflow

1. Download [`workflow/FlixBus_Price_Intelligence.json`](workflow/FlixBus_Price_Intelligence.json)
2. Open N8N → top-right menu → **Import from file**
3. Select the downloaded JSON
4. The full 12-node canvas appears, pre-wired

### Configure Credentials

After importing, update these 3 things:

```
Node 02 (Fetch Sheet Data)   → Set Spreadsheet ID + Google OAuth2 credential
Node 10 (Store to Sheets)    → Set Spreadsheet ID + Google OAuth2 credential  
Node 12 (Send Slack Alert)   → Set Slack Bot Token + channel name (#price-alerts)
```

### Google Sheet Structure

Your source sheet must have these **exact column headers** (Row 1):

```
Route Number | Operator | Bus Type | Departure Time | SRP Rank | 
Weighted Average Price | Available Seats | Total Seats | Number of Reviews | Bus Score
```

A sample dataset is available at [`data/sample_bus_listings.csv`](data/sample_bus_listings.csv).

### Output Sheet

Create a second sheet tab named **`Price Flags`** in the same spreadsheet. The workflow will append results here daily with all original fields plus:

```
Load Factor | Departure Hour | Rank Bucket | Similarity Group |
Competitor Avg Price | Competitor Count | Price Difference | Price Difference % | Flag | Run Date
```

---

## 📁 Repo Structure

```
flixbus-price-intelligence/
├── workflow/
│   └── FlixBus_Price_Intelligence.json   # Importable N8N workflow
├── data/
│   └── sample_bus_listings.csv           # Sample dataset (45 listings, 3 routes)
├── docs/
│   └── node_code_reference.md            # All JS code snippets extracted
└── README.md
```

---

## 💻 Tech Stack

| Tool | Role |
|------|------|
| N8N | Workflow orchestration |
| Google Sheets API | Data source + audit store |
| Slack API | Real-time price alerts |
| JavaScript | Code nodes for logic |
| Google OAuth2 | Secure authentication |
| Cron `0 8 * * *` | Daily scheduling |

---

## 📈 Business Impact

| Metric | Before | After |
|--------|--------|-------|
| Time per pricing review | 4–6 hours | 0 (automated) |
| Route coverage | Partial (sampled) | 100% daily |
| Anomaly detection lag | Days | <1 minute |
| Human steps required | Many | 0 |

---

## 🔧 Customisation

**Change the flag threshold** — edit the `Apply Flag Logic` code node:
```javascript
if      (pct >  10)  flag = "Too Expensive";   // Change 10 to your threshold
else if (pct < -10)  flag = "Too Cheap";
```

**Change the cron schedule** — edit the Schedule Trigger node:
```
0 8 * * *     → daily at 8AM
0 8 * * 1-5   → weekdays only
0 */6 * * *   → every 6 hours
```

**Add email alerts** — replace or add a Gmail node after the `Filter Alerts` IF node with subject:
```
[FlixBus Alert] {{ $json["Flag"] }} — Route {{ $json["Route Number"] }}
```

---

## 👤 Author

**vanisha Rathore** — ECE @ NSUT Delhi (2027) · Data Analytics · Product Management · AI/Gen AI

- GitHub: (https://github.com/vanishar9880)
- Open to: Data Analyst · Product Analyst · BA · AI/Gen AI Intern roles

---

## 📄 License

MIT — use freely, attribution appreciated.
