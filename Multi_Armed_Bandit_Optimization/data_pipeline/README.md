# Madison Agent — Data Pipeline (Assignment 3)

This repository contains the **data collection pipeline** (built in **n8n**) for the **Madison Agent: Model Behavior Evaluation System**.

The goal of this pipeline is to **collect, clean, normalize, validate, and save** high-quality data that will be used in **Assignment 4 (agent building)**—without re-scraping or re-calling APIs repeatedly.

---

## 1) Project Context

### Project Name
**Madison Agent — Model Behavior Evaluation System**

### Problem Statement
API / LLM teams often lack an integrated, repeatable way to:
- track new research + ecosystem developments,
- collect ground-truth instruction/response examples,
- and prepare clean evaluation-ready datasets for downstream agents.

This pipeline solves the *data* part of that problem: **it gathers and saves reusable datasets** for evaluation and research-context agents.

---

## 2) What This Pipeline Produces

The pipeline outputs **two files**, stored locally:

1. **`content_feed_*.csv`**
   - A unified, normalized feed from 2 RSS sources (ArXiv + Smol AI)
   - Clean schema: `source`, `title`, `url`, `published_at`, `summary`, `raw`
   - Records are filtered, deduplicated, validated, and date-standardized.

2. **`alpaca_eval_package_*.json`**
   - A packaged JSON export from the **AlpacaEval** dataset (HuggingFace)
   - Contains:
     - `collection_metadata` (stats + schema + validation checks)
     - `records` (flattened instruction/response examples)

> **Why separate files?**  
> The RSS content feed and AlpacaEval examples represent **different data types + different downstream uses**. Keeping them separate prevents schema collisions and keeps the data clean and reusable.

---

## 3) Data Sources (Tier 1)

All sources are **Tier 1** (public, stable, no keys required).

### Source A — ArXiv RSS (cs.AI)
- URL: `https://export.arxiv.org/rss/cs.AI`
- Type: AI research paper feed (titles + summaries/abstract-style text + links)
- Purpose: Research context + emerging evaluation methodologies.

### Source B — Smol AI RSS
- Type: AI news / research updates
- Purpose: Industry context and practical trends.

### Source C — AlpacaEval (HuggingFace dataset)
- Type: Instruction-response pairs (evaluation-style examples)
- Purpose: Baseline examples to support model behavior evaluation and comparisons.

---

## 4) Pipeline Architecture (n8n)

### High-level Flow

**Manual Trigger**
→ **(Parallel)** fetch from sources
→ **Normalize + Validate**
→ **Limit (for clean sample size)**
→ **Merge (append feeds)**
→ **Export to files**

### Conceptual Diagram

- **RSS branch 1 (ArXiv)**  
  RSS Read → Normalize → Validate + Clean → Limit → (to Merge)

- **RSS branch 2 (Smol AI)**  
  RSS Read → Normalize → Validate + Clean → Limit → (to Merge)

- **AlpacaEval branch**  
  HTTP Request → Extract 80 eval examples → Flatten → Package metadata → Export JSON

---

## 5) Node-by-Node Explanation (What Each Stage Does)

### A) Data Collection Nodes
1. **RSS Read nodes**
   - Pull raw RSS entries as items (n8n items)
   - No authentication required

2. **HTTP Request (AlpacaEval)**
   - Pulls dataset content from HuggingFace (public dataset endpoint)

---

### B) Normalization Nodes (per RSS source)

Each RSS source returns fields with different names.
Normalization maps them into a shared schema:

**Unified schema:**
- `source` (string)
- `title` (string)
- `url` (string)
- `published_at` (ISO-8601 string)
- `summary` (string)
- `raw` (stringified JSON)

**Key ideas:**
- `summary` is normalized from the most reliable available field:
  - `summary` → `raw.content` → `raw.description` → `""`
- Dates are standardized to ISO format.
- Raw payload is preserved as **one single JSON string column** to avoid dozens of `raw.*` columns in CSV.

---

### C) Validation + Cleaning Node (RSS feed)

This stage enforces quality:

✅ Drops records if:
- URL is missing  
- Summary is missing or too short (<= 50 chars)  
- Date is invalid  
- Date is older than cutoff (default: 2023-01-01)  
- Duplicate URLs appear

✅ Produces:
- clean items that are ready for export

---

### D) Limit Node (RSS feed)
After validation, each feed is limited to a fixed number of recent usable items (e.g., 60).

> Important: **Limit should happen after validation**, otherwise you risk limiting to low-quality rows.

---

### E) Merge Node (RSS feed)
The clean feeds are appended together into a single stream for export:
- Input 1: ArXiv cleaned
- Input 2: Smol cleaned

---

### F) Export Batch ID Node
Adds a timestamp-style `export_batch_id` so you can trace which execution produced the file.

This is helpful for:
- reproducibility,
- debugging,
- comparing runs (even though the recommended workflow is “run once”).

---

### G) Convert to File + Write File Nodes
- Converts cleaned records into CSV/JSON
- Writes output to disk

---

## 6) Data Quality Strategy

This pipeline was designed around the rubric’s “Quality Over Quantity” principle.

### Quality checks implemented
- **Null/empty summary removal**
- **Minimum summary length requirement**
- **Duplicate URL removal**
- **Invalid date removal**
- **Date standardization (ISO-8601)**
- **Cutoff filter (>= 2023-01-01)**

### Why it matters for agents
Agents perform better when input data is:
- complete and consistent,
- deduplicated,
- easy to index and retrieve,
- stable across runs.

---

## 7) How This Data Will Be Used in Assignment 4

### Content Feed (CSV)
Used by “research/context” style agents to:
- summarize recent papers or news,
- surface trends,
- build lightweight retrieval / search context,
- select items for deeper evaluation tasks.

### AlpacaEval Package (JSON)
Used by evaluation agents to:
- compare model outputs to known good examples,
- run regression-like checks (format, clarity, completeness),
- build a “reference set” for model behavior evaluation workflows.

---

## 8) Local Setup (Reproducible)

### Requirements
- Node.js installed
- n8n installed locally

```bash
npm install -g n8n
n8n start
```

> Tested environment: **n8n 1.64.3** (example install path: `/opt/homebrew/lib/n8n@1.64.3`)

---

## 9) How to Run the Workflow

1. Start n8n:
   ```bash
   n8n start
   ```

2. Open:
   - `http://localhost:5678`

3. Import the workflow JSON (exported from n8n):
   - `File → Import from File`

4. Click **Execute Workflow**

5. Wait for all branches to complete.

6. Locate outputs:
   - `content_feed_*.csv`
   - `alpaca_eval_package_*.json`

✅ **Recommendation:** Run once, save outputs, and reuse them for Assignment 4.

---

## 10) Output Schemas

### A) `content_feed_*.csv`
Columns:
- `source`
- `title`
- `url`
- `published_at` (ISO-8601)
- `summary`
- `raw` (stringified JSON)

### B) `alpaca_eval_package_*.json`
Top-level keys:
- `collection_metadata`
- `records`

Each record includes:
- `record_id`
- `record_type`
- `source`
- `prompt`
- `response`
- `model_used`
- `category` *(heuristic)*
- `dataset_origin`
- `response_length`
- `quality_score` *(placeholder / fixed in this assignment)*
- `instruction_following_score` *(placeholder / fixed)*
- `coherence_score` *(placeholder / fixed)*
- `helpfulness_score` *(placeholder / fixed)*
- `collected_at`
- `quality_flag`

---

## 11) Known Limitations (Intentional)

To keep the pipeline stable and rubric-aligned:

- AlpacaEval “scores” are placeholders (fixed values).  
  **Reason:** Assignment focus is data pipeline, not evaluation scoring.

- AlpacaEval categories are heuristic (derived from prompt keywords).  
  **Reason:** Quick labeling for downstream filtering.

- OpenAI RSS feed was tested but not used in final pipeline because it contained a large number of older records, reducing usefulness for “recent research context”.

---

## 12) Troubleshooting

### RSS returns fewer items than expected
- Some feeds only publish a limited number of items.
- Validation may drop older/empty-summary rows.
- Try moving Limit **after** validation (recommended).

### HTTP 429 Too Many Requests
- Run the workflow once, then reuse outputs (rubric recommended).
- Add an IF node checking `statusCode == 200` before parsing.

### XML parsing errors
- Usually caused by HTML instead of XML in response.
- Verify response headers and status code.
- Ensure you’re passing the correct field (`rawXml`) to the XML node.

### “raw.* columns exploded in CSV”
- Ensure you stringify raw into **one** column:
  - `raw: JSON.stringify(j.raw ?? {})`

---

## 13) Repository Contents (Suggested)

Recommended repo structure:

```
.
├── workflow/
│   └── LastName_FirstName_A3_Workflow.json
├── data/
│   ├── content_feed_*.csv
│   └── alpaca_eval_package_*.json
└── README.md
```

---

## 14) License / Attribution

- ArXiv RSS content belongs to ArXiv/its authors.
- Smol AI RSS content belongs to its publishers/authors.
- AlpacaEval dataset attribution:
  - HuggingFace dataset: `tatsu-lab/alpaca_eval`

This repo stores **metadata and processed excerpts** for educational evaluation workflows.

---

## 15) Contact / Notes

If you’re reviewing this repo as part of coursework:
- The pipeline is designed to be **simple, reproducible, and rubric-aligned**.
- The output files are intended for **Assignment 4 agent development**.

