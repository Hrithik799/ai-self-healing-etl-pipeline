# AI-Powered Self-Healing ETL Pipeline

An ETL pipeline that doesn't just fail silently on bad data — it diagnoses *why* a record failed, safely auto-corrects unambiguous issues, and escalates anything requiring human judgment, complete with an automated email alert.

Built with **n8n** (workflow orchestration) and **Google Gemini** (AI diagnosis agent), this project explores how far an LLM can safely go in automating the manual data-triage work common in production ETL systems.

![Workflow Canvas](screenshots/workflow-canvas.png)

---

## Why I Built This

Coming from a Systems/Data Engineering background, I've seen firsthand how much manual effort goes into triaging ETL failures — malformed dates, missing fields, type mismatches, duplicate records. Most of these follow recognizable patterns that a human reviewer could classify in seconds, but at scale, that triage work becomes a bottleneck.

This project asks: **can an LLM take over the classification and low-risk correction work, while still knowing exactly when *not* to touch something?**

That last part — knowing what *not* to fix — was the actual engineering problem worth solving, not just wiring an LLM into a pipeline.

---

## Architecture

```
CSV Ingest → Validate Schema → Detect Duplicates → Route: Valid or Invalid
                                                        │
                          ┌─────────────────────────────┴─────────────────────────────┐
                          │                                                             │
                     Valid Rows                                                  Invalid Rows
                          │                                                             │
                          │                                              Gemini Diagnosis Agent
                          │                                                             │
                          │                                          Route: Fixable or Needs Review
                          │                                                        │           │
                          │                                                  Fixable      Needs Review
                          │                                                        │           │
                          │                                                 Apply Auto-Fix   Format Escalation Log
                          │                                                        │           │
                          │                                                        │      Email Alert to Human Reviewer
                          │                                                        │           │
                          └──────────────────┬─────────────────────────────────────┘           │
                                    Merge Clean + Fixed Rows                                    │
                                              │                                                  │
                                    clean_orders_output.csv                          escalation_log.csv
```

### Pipeline Stages

| Stage | What It Does |
|---|---|
| **Ingest** | Reads a CSV of order records from disk |
| **Validate Schema** | Deterministic, rule-based checks: missing required fields, date format, numeric type validity, negative price detection |
| **Detect Duplicates** | Cross-row check for repeated `order_id` values — flags the second+ occurrence, not the first |
| **Route: Valid or Invalid** | Splits records that pass all checks from records with at least one failure |
| **Gemini Diagnosis Agent** | For every *failed* record, an LLM (Gemini 2.5 Flash) classifies the failure and proposes a fix — see prompt design below |
| **Route: Fixable or Needs Review** | Splits the AI's classification into an auto-correctable path and a human-escalation path |
| **Apply Auto-Fix** | Programmatically applies the AI's suggested correction to the flagged field |
| **Format Escalation Log** | Structures unresolved failures into a clean, human-readable record |
| **Email Alert** | Sends a real-time email notification for every row requiring human review |
| **Merge + Load** | Combines originally-valid rows with auto-fixed rows into `clean_orders_output.csv`; escalated rows are written to `escalation_log.csv` |

---

## The Core Design Decision: When Should AI Be Allowed to Fix Something?

This is the part of the project I'd point to first in an interview.

Every failed record is sent to Gemini with a system prompt that enforces two hard rules:

1. **Only mark a fix as `fixable` if confidence is above 0.85 AND the correction is unambiguous.**
   Converting `14/03/2025` → `2025-03-14` is safe — there's exactly one reasonable interpretation.

2. **Never mark something fixable if it requires guessing missing or ambiguous business data.**
   A blank `customer_name` has no safe default. A negative `unit_price` could mean a data-entry error *or* a legitimate return/credit memo — the pipeline has no way to know which, so it always escalates rather than assumes.

### Actual results from testing on 10 sample records (5 intentionally broken):

| Failure | AI Classification | Confidence | Correct? |
|---|---|---|---|
| Date `14/03/2025` | ✅ Fixable → `2025-03-14` | 1.0 | Yes — unambiguous format conversion |
| Missing `customer_name` | ⚠️ Needs Human Review | 0.0 | Yes — correctly refuses to invent a name |
| Quantity `"five"` | ✅ Fixable → `5` | 0.95 | Yes — unambiguous type conversion |
| Negative `unit_price` (-50.00) | ⚠️ Needs Human Review | 0.1 | Yes — correctly flags as ambiguous, not auto-"fixed" to positive |
| Duplicate `order_id` | ⚠️ Needs Human Review | — | Yes — could be a legitimate resend, not silently dropped |

The guardrail isn't just a prompt instruction that sounds good on paper — it held up against real test cases where a naive implementation would have been tempted to "helpfully" guess.

---

## Tech Stack

- **n8n** (self-hosted via Docker) — workflow orchestration
- **Google Gemini API** (`gemini-2.5-flash`) — AI diagnosis agent
- **JavaScript** (n8n Code nodes) — deterministic validation logic, duplicate detection, response parsing, auto-fix application
- **SMTP (Gmail)** — human escalation alerts
- **CSV** — input/output data format

---

## Repository Structure

```
ai-self-healing-etl-pipeline/
├── README.md
├── workflow/
│   └── etl-self-healing-pipeline.json   # Full exported n8n workflow — import directly into n8n
├── sample-data/
│   └── sample_orders_messy.csv          # 10 test records, 5 intentionally broken across all failure categories
├── outputs/
│   ├── clean_orders_output.csv          # Valid + auto-fixed records
│   └── escalation_log.csv               # Records requiring human review, with AI explanations
└── screenshots/
    └── workflow-canvas.png              # Full pipeline visual
```

---

## How to Run It

1. **Install Docker Desktop** and run n8n locally:
   ```
   docker run -it --rm --name n8n -p 5678:5678 \
     -v n8n_data:/home/node/.n8n \
     -v /path/to/local/folder:/home/node/.n8n-files \
     docker.n8n.io/n8nio/n8n
   ```
2. Open `http://localhost:5678`
3. Import `workflow/etl-self-healing-pipeline.json`
4. Add your own **Google Gemini API credential** (free tier available at [aistudio.google.com](https://aistudio.google.com/apikey))
5. Add an **SMTP credential** for email alerts (Gmail App Password recommended)
6. Place `sample-data/sample_orders_messy.csv` in your mounted local folder
7. Update file paths in the "Read/Write Files from Disk" nodes to match your environment
8. Click **Execute Workflow**

---

## What I'd Extend With More Time

- **Streamlit dashboard** for auto-fix rate, escalation rate, and failure category breakdown over time
- **Structured output enforcement** (JSON schema validation on the Gemini response, with retry-on-malformed-output)
- **Google Sheets / database output** instead of local CSV, for shared visibility
- **Confidence threshold tuning** based on a larger labeled dataset of real failure/fix pairs
- **Rate-limit-aware model routing** — route high-volume/simple classifications to a cheaper model tier, reserve larger models for ambiguous cases

---

## Lessons From Building This

- **Free-tier LLM API rate limits are a real constraint**, not just a documentation footnote — I hit this directly during testing and it pushed me to think about model selection (Flash vs Flash Lite vs Pro) as a cost/latency trade-off decision from day one, not an afterthought.
- **The hardest part of "AI-powered automation" isn't calling the model — it's deciding what the model should never be allowed to decide on its own.** The confidence threshold and the missing-data guardrail were more valuable design decisions than the prompt engineering itself.
- **n8n's LangChain-style node structure** (separate "Chain" nodes from "Model" sub-nodes) took some initial troubleshooting to understand, but mirrors real agent-orchestration patterns you'd see in LangGraph or similar frameworks.
