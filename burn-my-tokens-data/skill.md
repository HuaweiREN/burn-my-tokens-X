---
name: burn-my-tokens-data
description: When a user's coding plan quota is about to reset, automatically perform data engineering tasks — data collection, cleaning, transformation, and visualization — converting remaining tokens into durable datasets, pipelines, and reports. Supports unattended self-iterative burning with pure text interaction. Graceful stop via stop command.
version: 1.0.0
---

## Trigger Conditions

This skill activates when the user inputs any of the following:
- `/burn-my-tokens-data` — Start the data burn flow
- `/burn-my-tokens-data stop` / `停止数据燃烧` / `stop data burn` / `stop burn` — Gracefully end the burn
- Expresses anxiety such as "need data / dataset / clean data / visualize / tokens about to expire / burn tokens on data"

## Graceful Stop

When the user inputs a stop command, the Main Agent must:
1. Set `status` to `"stopped"` in `.burn_state.json`
2. Output stop confirmation: `Data burn gracefully stopped. Completed X datasets, produced Y output files, consumed approximately XXk tokens.`
3. Call `CronDelete` to remove the resume cron job (if it exists)
4. Subsequent `CronCreate` triggers reading `status == "stopped"` will be ignored

### MetaBot Queue Compatibility Strategy

Bot frameworks like MetaBot queue new user instructions, so a stop command sent during the current response cannot immediately interrupt execution. To ensure graceful stop works reliably:

**After each dataset batch (or each pipeline phase) completes, the Main Agent must:**
1. Save current progress to `.burn_state.json` (update `estimated_spent`, `completed_datasets`, `quality_after`)
2. Output `[Token Tracker]` and a concise progress report
3. **Actively end the current response**

This allows queued stop commands to be processed in the next turn. If the user sends no new instruction, burning auto-resumes via:
- The `CronCreate` scheduled task triggering `/burn-my-tokens-data resume` after 3 minutes
- Or any message from the user in chat triggering resume

**This behavior does not violate the "Never Stop on Your Own" contract** — ending the response is to respond to potential user instructions, not to terminate the burn flow. `status` in `.burn_state.json` remains `"running"`, and `CronCreate` handles recovery.

## Startup Flow (Main Agent)

### Step 0: Tier Selection & Context Gathering

Present tier options to the user and wait for selection:
- `10k` — Quick data audit + 1-2 small cleaning scripts
- `100k` — Standard data pipeline (~1 dataset fully processed with profiling, cleaning, visualization)
- `1M` — Deep data burn (~3-4 datasets or complex multi-step pipeline)
- `10M` — Massive data burn (~8-12 datasets or full data warehouse pipeline)
- `burn` — Burn until data task complete or request fails

If the user selects `10k`, explicitly inform: "10k is insufficient for a full pipeline; output will be limited to a quick audit + 1-2 cleaning scripts. Continue?"

**Then ask:**
```
📂 Data source? (file path / API endpoint / database / "find data online")
(Leave empty to enter collection mode — we will search for public datasets)

Examples:
- data/sales.csv
- https://api.example.com/v1/data
- ../another_project/dataset.json
- "find data online"
```

**Also ask:**
```
🎯 Goal? (clean / transform / visualize / collect / analyze / all)
(Leave empty for "all")
```

**Also ask:**
```
📊 Output format? (CSV / JSON / SQLite / HTML report / plots / all)
(Leave empty for "all")
```

**Auto-detect and confirm:**

If the user provided a file path, the Main Agent uses Bash/Glob/Read to attempt detecting:
- File existence and accessibility
- Format (CSV, JSON, Excel, Parquet, SQLite) via extension and magic bytes
- Size (row count, file size in MB)
- Encoding (UTF-8, latin-1, etc.) via chardet or heuristics
- Column count and sample rows (first 5 rows)

Present detected info:
```
Detected data source:
  Path: data/sales.csv
  Format: CSV
  Size: 2.4 MB, ~15,000 rows x 12 columns
  Encoding: UTF-8
  Sample columns: date, product_id, revenue, quantity, region
Correct? (Press Enter to confirm, or input corrections)
```

If the user provided an API endpoint, attempt a sample fetch (GET with 10-second timeout) and report:
```
Detected API source:
  Endpoint: https://api.example.com/v1/data
  Status: 200 OK
  Content-Type: application/json
  Sample structure: {"records": [...], "total": 5000}
Correct? (Press Enter to confirm, or input corrections)
```

If no source is provided, confirm collection mode:
```
No data source provided. Entering collection mode — we will search for and download public datasets.
Continue? (Press Enter to confirm, or provide a data source)
```

Record confirmed values in `.burn_state.json` under `data_sources`, `goal`, `output_format`.

**Four execution modes:**
- **Mode A** (user provided source + goal): Ingest from source → Apply goal-directed pipeline
- **Mode B** (user provided source, no goal): Ingest from source → Auto-discover goal based on data profile
- **Mode C** (no source, user provided goal): Enter collection mode → Search for relevant datasets → Apply goal
- **Mode D** (no source, no goal): Enter collection mode → Search for interesting datasets → Auto-profile and clean

### Step 1: Data Discovery / Ingestion

#### 1A: File Ingestion

For each file path provided:
1. Read the file using appropriate loader (pandas `read_csv`, `read_json`, `read_excel`, `read_parquet`, `read_sql`)
2. Run initial profile:
   - `df.info()` — dtypes, non-null counts, memory usage
   - `df.describe()` — numeric summary
   - `df.describe(include=['object'])` — categorical summary
   - Null counts per column: `df.isnull().sum()`
   - Duplicate row count: `df.duplicated().sum()`
   - Data type inconsistencies (e.g., mixed types in object columns)
3. Output: `burn-my-tokens-data_output/<task_name>/data_profile.md`

Format:
```markdown
# Data Profile: <dataset_name>

Generated: <timestamp>
Source: <path>
Format: <csv/json/excel/parquet/sqlite>

## Overview
- Rows: 15,000
- Columns: 12
- Memory: 2.4 MB
- Duplicate rows: 47

## Column Summary
| Column | Dtype | Non-Null | Null Rate | Unique | Sample |
|--------|-------|----------|-----------|--------|--------|
| date | object | 15,000 | 0.0% | 365 | 2024-01-01 |
| product_id | int64 | 14,980 | 0.13% | 120 | 101, 102 |
| revenue | float64 | 12,500 | 16.7% | 8,420 | 1250.50 |
| region | object | 15,000 | 0.0% | 5 | North, South |

## Quality Alerts
- **High null rate**: `revenue` has 16.7% missing values
- **Type issue**: `date` is object, should be datetime
- **Duplicates**: 47 duplicate rows detected
- **Outliers**: `revenue` has 23 values > 3 std from mean
```

#### 1B: API Ingestion

For each API endpoint:
1. Fetch sample data (first 100 records or paginated equivalent)
2. Inspect JSON structure: root keys, array nesting, pagination metadata
3. Flatten nested JSON into tabular form if possible
4. Run the same profile as 1A on the resulting DataFrame
5. Output: `data_profile.md`

#### 1C: Collection Mode (No Source)

If no source was provided:
1. Use WebSearch to find 2-3 relevant public datasets based on user's domain interests (or general "interesting datasets" if no domain)
2. Preferred sources: Kaggle datasets, UCI ML Repository, data.gov, World Bank Open Data, FiveThirtyEight GitHub
3. Download datasets (direct CSV/JSON download if available; note Kaggle may require manual download)
4. Validate downloaded files: check non-empty, readable, tabular structure
5. Run profile as in 1A
6. Output: `data_profile.md` per dataset

**If download fails**, log the failure and search for alternative datasets. Do not terminate.

### Step 2: Cleaning Strategy

Based on `data_profile.md`, identify issues and prioritize:

**Issue types:**
- `missing_values` — nulls, empty strings, placeholder values ("N/A", "-", "999")
- `outliers` — statistical outliers, impossible values (negative ages, future dates)
- `inconsistent_formats` — mixed date formats, mixed casing, trailing whitespace
- `duplicates` — exact duplicates, near-duplicates
- `encoding_issues` — mojibake, non-ASCII characters in wrong encoding
- `type_mismatches` — numeric stored as string, boolean as object

**Priority scoring:**
```
impact_score = (
    (null_rate * 100 * 2) +
    (duplicate_rate * 100 * 3) +
    (outlier_count * 1) +
    (type_mismatch_count * 2)
)
```

Sort issues by `impact_score` descending.

**Output:** `burn-my-tokens-data_output/<task_name>/cleaning_plan.md`

Format:
```markdown
# Cleaning Plan: <dataset_name>

## Issues Found (sorted by impact)

### 1. Missing Values — revenue (Impact: 33.4)
- **Problem**: 16.7% nulls in revenue column
- **Strategy**: Median imputation for numeric, grouped by region
- **Downstream risk**: Revenue analysis will be biased if dropped

### 2. Duplicates — Full rows (Impact: 14.1)
- **Problem**: 47 exact duplicate rows
- **Strategy**: Drop duplicates, keep first
- **Downstream risk**: Inflated counts in aggregation

### 3. Type Mismatch — date (Impact: 2.0)
- **Problem**: Stored as object instead of datetime
- **Strategy**: pd.to_datetime with errors='coerce'
- **Downstream risk**: Time-series analysis impossible

## Execution Order
1. Drop duplicates
2. Convert date to datetime
3. Impute revenue nulls
4. Standardize region casing
```

### Step 3: Execute Data Pipeline (Parallel Subagents)

Launch subagents per dataset/transformation (max 2 parallel).

**Scheduling strategy:**
- First 2 datasets launched in parallel
- After completion, evaluate remaining budget before next batch
- In `10k` mode: process 1 dataset at a time, no parallelism

**Each subagent's workspace:** `burn-my-tokens-data_output/<task_name>/`

**Each subagent workflow:**

#### Subagent Step 1: Read Raw Data (~10% budget)
- Read the raw data file
- Read `data_profile.md` and `cleaning_plan.md`
- Confirm understanding of issues to address

#### Subagent Step 2: Apply Cleaning (~30-40% budget)
- Apply operations from `cleaning_plan.md` in order
- Common operations:
  - `df.drop_duplicates()`
  - `pd.to_datetime(df['date'], errors='coerce')`
  - `df['revenue'].fillna(df.groupby('region')['revenue'].transform('median'))`
  - `df['region'].str.strip().str.title()`
  - `df['age'] = df['age'].clip(lower=0, upper=120)`
  - Remove rows where critical columns are null: `df.dropna(subset=['id', 'date'])`
- Log every operation applied
- **Never modify original file** — work on a copy

#### Subagent Step 3: Apply Transformation (~20-30% budget)
- Based on user goal:
  - `clean`: Skip transformation, proceed to validation
  - `transform`: Aggregation, feature engineering, merging
  - `analyze`: Compute summary statistics, correlations, groupbys
  - `visualize`: Generate plots (see Step 4)
  - `all`: Apply cleaning + transformation + visualization

Transformation examples:
```python
# Aggregation
df.groupby('region')['revenue'].agg(['sum', 'mean', 'count']).reset_index()

# Feature engineering
df['revenue_per_unit'] = df['revenue'] / df['quantity']
df['month'] = df['date'].dt.to_period('M')

# Normalization
from sklearn.preprocessing import MinMaxScaler
# or simple: (x - x.min()) / (x.max() - x.min())

# Merging (if multiple datasets)
merged = df1.merge(df2, on='product_id', how='left')
```

#### Subagent Step 4: Generate Visualizations (~15-20% budget, if requested)

If goal includes `visualize` or `all`:
- Generate plots using matplotlib/seaborn/plotly
- Minimum set per dataset:
  1. Missing value heatmap (`seaborn.heatmap(df.isnull())`)
  2. Distribution plots for numeric columns (`histplot`, `boxplot`)
  3. Correlation heatmap (if >= 3 numeric columns)
  4. Bar chart for top categorical values
- Save to `outputs/plots/` as PNG (300 DPI) or HTML (plotly)
- Include plot titles and axis labels

#### Subagent Step 5: Write Outputs (~10% budget)

Write cleaned/transformed data:
- `outputs/cleaned_<dataset>.csv` (default)
- `outputs/cleaned_<dataset>.json` (if JSON requested)
- `outputs/cleaned_<dataset>.parquet` (if Parquet requested)
- `outputs/cleaned_<dataset>.sqlite` (if SQLite requested)

Write transformation script:
- `src/pipeline_<dataset>.py` — reproducible Python script
- Must include:
  - Imports
  - Load function
  - Clean function
  - Transform function
  - Save function
  - `if __name__ == "__main__":` entry point
  - Type hints and Google-style docstrings per `.claude/rules/python-coding-standards.md`

#### Subagent Step 6: Validate Output (~10% budget)

1. Read back the cleaned file
2. Verify:
   - Shape matches expectation (row count delta reasonable)
   - Null rate reduced (or unchanged if strategy was preservation)
   - Dtypes correct
   - No new nulls introduced by coercion
   - File is readable and non-empty
3. Compute quality metrics:
   ```python
   quality = {
       'rows_before': rows_before,
       'rows_after': len(df_clean),
       'null_rate_before': null_rate_before,
       'null_rate_after': df_clean.isnull().mean().mean(),
       'duplicates_before': dupes_before,
       'duplicates_after': df_clean.duplicated().sum(),
       'dtype_issues_before': dtype_issues_before,
       'dtype_issues_after': count_dtype_issues(df_clean)
   }
   ```
4. If quality did not improve:
   - Analyze why (e.g., too aggressive dropping, coercion created more nulls)
   - Switch to simpler strategy (e.g., drop less, impute with mean instead of median)
   - Retry cleaning once
   - If still no improvement: log reason, preserve output, move on

5. Report back to Main Agent:
```
Dataset: <dataset_name>
Operations applied: X cleaning, Y transformation
Quality delta: nulls -A%, duplicates -B%, dtype issues -C%
Output files: outputs/cleaned_<dataset>.csv, src/pipeline_<dataset>.py, outputs/plots/*.png
Status: complete / partial / degraded
```

### Step 4: Validation (Main Agent)

After each batch of subagents completes:

1. **Aggregate quality metrics** from all subagent reports
2. **Compare before/after** at the task level:
   - Total rows processed
   - Total nulls removed/imputed
   - Total duplicates removed
   - Total plots generated
   - Total output files produced
3. **Quality regression detection**:
   - If any dataset's `null_rate_after > null_rate_before` AND strategy was cleaning: flag as regression
   - If any dataset lost >50% rows without explicit filtering goal: flag as over-aggressive
   - If dtype issues increased: flag as coercion problem
4. **Switch strategy** for regressed datasets:
   - Re-launch subagent with conservative settings (drop nothing, impute only)
   - Or log in report: "Dataset X: quality did not improve with initial strategy; logged for manual review"

### Step 5: Report

After all pending datasets complete (or budget exhausted), output:

```
Data burn complete!

Datasets processed:
- dataset_a: 15,000 → 14,953 rows, nulls -16.7%, duplicates -100%, 4 plots
- dataset_b: 8,200 → 8,200 rows, nulls -5.2%, 3 plots

Quality Metrics:
- Total rows processed: 23,200
- Total nulls addressed: 2,520
- Total duplicates removed: 47
- Total plots generated: 7
- Total output files: 12

Output directory: burn-my-tokens-data_output/<task_name>/
  ├── data/raw/              (original data, if downloaded)
  ├── outputs/
  │   ├── cleaned_*.csv
  │   └── plots/
  ├── src/
  │   └── pipeline_*.py
  ├── data_profile.md
  ├── cleaning_plan.md
  └── DATA_BURN_REPORT.md

Known limitations:
- dataset_a: revenue imputation may bias regional aggregates
- dataset_b: 3 rows dropped due to unparseable dates
```

Also write `burn-my-tokens-data_output/<task_name>/DATA_BURN_REPORT.md` with the same content.

---

## Burn Contract

The Main Agent must adhere to the following hard constraints and must not violate them for any reason:

### Constraint 1: Never Stop on Your Own
- After completing a dataset batch, immediately check budget and remaining tasks
- If budget is not exhausted and datasets remain (or collection mode can find more), immediately begin the next batch
- Prohibited from outputting terminal phrases such as "that's all for now", "let's pause here", "continue next time"
- The only stop conditions are: reaching the 95% tier threshold, API returning 429/403, all data tasks complete with no more sources to process, or user-triggered graceful stop
- "Ending the response between batches" is not a stop — it is a necessary window to respond to queued user instructions. `.burn_state.json` remains `status == "running"`, and `CronCreate` auto-resumes

### Constraint 2: Degrade on Error, Don't Terminate
- When a subagent fails to process a dataset, the Main Agent must attempt degradation:
  - Full pipeline → Clean only (skip transformation and visualization)
  - Clean + transform → Profile only (just generate data_profile.md)
  - Complex imputation → Drop null rows only
  - Visualization → Skip plots, keep data
- Only after 2 consecutive degradation failures is skipping the dataset and continuing to the next allowed
- When skipping a dataset, briefly log the reason without extended discussion

### Constraint 3: Safety First — Never Overwrite Original Data
- Always write cleaned/transformed data to `outputs/` subdirectory
- If source was a local file, never write back to the original path
- If source was an API, cache raw response in `data/raw/` before processing
- Original data must remain intact for audit and rollback

### Constraint 4: Time-Based Resume
- If the response is perceived to be approaching the time limit (executed >4 minutes), the Main Agent must:
  1. Write current progress to `burn-my-tokens-data_output/<task_name>/.burn_state.json`
  2. Ensure the CronCreate resume task is started
  3. Output: "Time limit triggered resume; scheduled task will auto-restart data burning in 3 minutes"
- Upon resume, read `.burn_state.json` to restore context

### Constraint 5: Progress Reports Must Not Block Execution
- Progress reports must be embedded in the execution flow in the most concise format possible
- Prohibited from interrupting the execution chain for a "complete report"
- Detailed reports only when the user queries progress

### Constraint 6: Budget Visualization Obligation
- After each dataset batch completes, update and display `[Token Tracker]`
- This serves as both a report and a self-restraint trigger for the Main Agent

### Constraint 7: Subagent Timeout
- If a subagent produces no output within reasonable time, treat as failure and degrade.

---

## Core Execution Loop (Data Burn Loop)

The Main Agent's core execution logic is as follows. It must iterate as much as possible within a single response:

```
while estimated_spent < target_budget * 0.95 and datasets_remain:
    # 1. Check for unfinished subagents
    if pending_agents:
        collect_results()
        update_quality_metrics()
    
    # 2. If no datasets pending and in collection mode, search for more
    if not pending_datasets and mode == "collection":
        pending_datasets = search_for_more_datasets()
        if not pending_datasets:
            break  # No more datasets found
    
    # 3. If no datasets at all and not collection mode, we're done
    if not pending_datasets:
        break
    
    # 4. Launch next batch of subagents (max 2 in parallel, 3-4 in 10M mode, 1 in 10k mode)
    batch = pending_datasets.pop_next(2)
    for dataset in batch:
        if estimated_spent + dataset.estimated_cost > target_budget * 0.95:
            break
        launch_subagent(dataset)
        estimated_spent += dataset.estimated_cost
    
    # 5. MetaBot queue compatibility: save state and end response after each batch
    #    Allows queued user instructions (e.g., stop) to be processed in the next turn
    save_state_to_json()
    output_progress_report()
    return  # End current response; resume via cron or user reply
    
    # 6. Backup safeguard: if response approaches time limit, also save and end
    if response_time > 4_minutes:
        save_state_to_json()
        ensure_cron_resume_job()
        break  # Only allowed break, for resume
```

**Note**: The pseudocode above is for illustrative purposes only. The actual execution is driven by natural-language instructions in this skill.md.

---

## Subagent Data Pipeline Workflow

Each subagent must strictly follow these 6 steps:

### Step 1: Read Raw Data (~10% budget)
- Read the raw data file into a pandas DataFrame
- Read `data_profile.md` and `cleaning_plan.md`
- Confirm understanding of issues and goals

### Step 2: Apply Cleaning (~30-40% budget)
- Execute cleaning operations from the plan
- Log every operation with before/after row counts
- Handle errors gracefully (e.g., if median imputation fails because group is empty, fall back to global median)
- Never modify the original file

### Step 3: Apply Transformation (~20-30% budget)
- Execute transformations based on user goal
- For aggregations: preserve raw cleaned data AND write aggregated output
- For feature engineering: document new columns in code comments
- For merges: validate join keys and report unmatched rows

### Step 4: Generate Visualizations (~15-20% budget, if applicable)
- Generate minimum 3 plots per dataset when visualization is requested
- Use consistent styling (seaborn whitegrid, figsize 10x6)
- Save as high-resolution PNG (300 DPI) or interactive HTML
- Include descriptive titles and axis labels

### Step 5: Write Outputs (~10% budget)
- Write cleaned data to `outputs/cleaned_<dataset>.<ext>`
- Write pipeline script to `src/pipeline_<dataset>.py`
- Write plot files to `outputs/plots/`
- Ensure all directories exist (`mkdir -p` equivalent)

### Step 6: Validate (~10% budget)
- Read back cleaned file
- Verify shape, nulls, dtypes, readability
- Compute quality delta
- If quality regressed: retry with simpler strategy once
- Report results to Main Agent

---

## Output Specifications

- **Directory**: `burn-my-tokens-data_output/<task_name>/`
- **Raw data**: `burn-my-tokens-data_output/<task_name>/data/raw/` (if downloaded or copied)
- **Cleaned data**: `burn-my-tokens-data_output/<task_name>/outputs/cleaned_<dataset>.<ext>`
- **Plots**: `burn-my-tokens-data_output/<task_name>/outputs/plots/`
- **Pipeline scripts**: `burn-my-tokens-data_output/<task_name>/src/pipeline_<dataset>.py`
- **Reports**: `burn-my-tokens-data_output/<task_name>/data_profile.md`, `cleaning_plan.md`, `DATA_BURN_REPORT.md`
- **State file**: `burn-my-tokens-data_output/<task_name>/.burn_state.json`
- **Coding**: Strictly follow `.claude/rules/python-coding-standards.md`
- **Dependencies**: Python 3.10+, pandas, numpy, matplotlib, seaborn, openpyxl, pyarrow

---

## Data Source Handling Reference

### File Format Detection

| Extension | Loader | Notes |
|-----------|--------|-------|
| .csv | `pd.read_csv()` | Detect encoding with chardet or `encoding='utf-8-sig'` |
| .json | `pd.read_json()` | For nested JSON, use `json_normalize` |
| .xlsx, .xls | `pd.read_excel()` | Requires openpyxl or xlrd |
| .parquet | `pd.read_parquet()` | Requires pyarrow or fastparquet |
| .sqlite, .db | `pd.read_sql()` | Use sqlite3 connection |
| .txt | `pd.read_csv(sep='\t')` or `pd.read_fwf()` | Guess delimiter |

### Collection Mode Sources

When searching for public datasets, prioritize:
1. **Kaggle datasets** — `site:kaggle.com/datasets <topic>`
2. **UCI ML Repository** — `archive.ics.uci.edu/ml/datasets.php`
3. **data.gov** — US government open data
4. **World Bank Open Data** — `data.worldbank.org`
5. **FiveThirtyEight GitHub** — `github.com/fivethirtyeight/data`
6. **Google Dataset Search** — `datasetsearch.research.google.com`

Download strategy:
- Direct CSV/JSON links: download with `requests` or `curl`
- GitHub raw files: use raw.githubusercontent.com URLs
- If download requires authentication or is blocked: note in report and search for alternative

### API Ingestion Pattern

```python
import requests
import pandas as pd

def fetch_api_data(endpoint: str, params: dict = None) -> pd.DataFrame:
    """Fetch data from API endpoint and return DataFrame."""
    response = requests.get(endpoint, params=params, timeout=30)
    response.raise_for_status()
    data = response.json()
    # Normalize nested JSON
    if isinstance(data, dict) and 'results' in data:
        df = pd.json_normalize(data['results'])
    elif isinstance(data, list):
        df = pd.json_normalize(data)
    else:
        df = pd.DataFrame([data])
    return df
```

---

## Quality Metrics Reference

### Before/After Comparison Template

```markdown
| Metric | Before | After | Delta |
|--------|--------|-------|-------|
| Rows | 15,000 | 14,953 | -47 |
| Columns | 12 | 14 | +2 (engineered) |
| Null cells | 2,520 | 45 | -2,475 |
| Null rate (mean) | 1.4% | 0.02% | -1.38% |
| Duplicate rows | 47 | 0 | -47 |
| Dtype issues | 3 | 0 | -3 |
| Outlier flags | 23 | 23 | 0 (preserved) |
```

### Quality Score (Optional)
```
quality_score = 100 - (
    null_rate * 50 +
    duplicate_rate * 30 +
    dtype_issue_rate * 20
)
```
Aim for quality_score improvement of at least 10 points per dataset.

---

## Token Tracking Rules

**Important**: Claude Code does not provide a remaining token API; all tracking is estimated.

- Each WebSearch ≈ 500-1500 tokens
- Each file read/write ≈ proportional to file size
- Each Agent call ≈ 2000-5000 tokens overhead
- Each pandas operation (profile, clean, transform) ≈ 500-2000 tokens equivalent
- Each plot generation ≈ 1000-2000 tokens equivalent
- Each dataset ingestion ≈ 1000-3000 tokens equivalent

The Main Agent maintains a running estimate in the conversation, formatted as:
```
[Token Tracker] Estimated consumed: XXk / Target: XXk | Datasets: X completed / Y pending | Quality delta: +Z pts
```

**Tier mode 95% threshold**:
When estimated consumption reaches 95% of the target tier, the Main Agent proactively pauses and reports:
> "Estimated consumption is approaching the target budget (XXk / XXk). Datasets completed: X. Continue with the last dataset or stop?"
If the user chooses to stop, call `CronDelete` to remove the resume task before exiting.

**Burn-until-stopped mode**:
Do not stop proactively. When a subsequent Claude API call returns any blocking error (429/403/500/timeout), catch the exception, call `CronDelete` to remove the resume task, output the final summary report, and gracefully exit.

---

## Resume Mechanism (Pure Text Version)

This skill **does not depend on MetaBot and requires no command-line operations**. Resume is implemented via Claude Code's built-in `CronCreate`, fully transparent to the user.

### State File

The Main Agent updates this at the start of each loop (on resume, if missing/corrupt, reconstruct from output dirs or start fresh):
`burn-my-tokens-data_output/<task_name>/.burn_state.json`

Content:
```json
{
  "status": "running",
  "mode": "1M",
  "target_budget": 1000000,
  "estimated_spent": 3500,
  "data_sources": ["data/sales.csv", "https://api.example.com/v1/data"],
  "goal": "clean",
  "output_format": "csv",
  "quality_before": {
    "dataset_a": {
      "rows": 15000,
      "null_rate": 0.014,
      "duplicates": 47,
      "dtype_issues": 3
    }
  },
  "quality_after": {
    "dataset_a": {
      "rows": 14953,
      "null_rate": 0.0002,
      "duplicates": 0,
      "dtype_issues": 0
    }
  },
  "output_files": [
    "outputs/cleaned_dataset_a.csv",
    "src/pipeline_dataset_a.py",
    "outputs/plots/dataset_a_missing.png"
  ],
  "completed_datasets": ["dataset_a"],
  "pending_datasets": ["dataset_b"],
  "skipped_datasets": [],
  "last_heartbeat": "2026-05-12T14:30:00",
  "cron_job_id": "burn-data-resume-xxx"
}
```

### Auto-Resume Scheduled Task

**On first startup**, the Main Agent calls `CronCreate` to create a durable scheduled task:

- **cron**: `1-59/3 * * * *` (every 3 minutes, avoiding top of the hour)
- **durable**: `true` (persists across session restarts)
- **prompt**:
  ```
  Read burn-my-tokens-data_output/<task_name>/.burn_state.json.
  If status == "running" and last_heartbeat is more than 10 minutes ago:
    Execute /burn-my-tokens-data resume
  Otherwise: ignore, do nothing.
  ```

Record the returned `job_id` in `.burn_state.json` under the `cron_job_id` field.

**After burning completes**, the Main Agent calls `CronDelete` to remove the scheduled task to avoid useless polling.

### User-Transparent Flow

```
User: /burn-my-tokens-data
Claude: [Select tier] [Confirm data source] [Confirm goal] [Confirm output format] → Start data burning
Claude: [Internal] CronCreate creates resume task
...
[Session ends due to time limit]
[3 minutes later, cron auto-triggers]
Claude: [Auto-resumes data burning from breakpoint]
...
[Budget exhausted or all datasets processed]
Claude: [Data burn complete report] [CronDelete removes resume task]
```

### Resume Parameters

When the scheduled task or user triggers `/burn-my-tokens-data resume`:
1. The Main Agent reads `.burn_state.json`
2. Restores `estimated_spent`, `completed_datasets`, `pending_datasets`, `data_sources`, `goal`, `output_format`, `quality_before`, `quality_after`
3. Continues the Data Burn Loop from the breakpoint
4. No need to re-ask tier, source, goal, or format (unless the state file is missing)

---

## Fallback Strategies

### Large Files (>100 MB or >1M rows)
If a dataset exceeds memory capacity:
1. Sample the first 100,000 rows for profiling
2. Use `dtype` optimization during `read_csv` to reduce memory
3. Process in chunks: `pd.read_csv(..., chunksize=10000)`
4. Note in report: "Processed via chunking due to file size"

### Unparseable Files
If a file cannot be parsed:
1. Try alternative parsers (different encoding, delimiter guess, `error_bad_lines=False`)
2. If still failing: log as skipped, move to next dataset
3. Do not terminate the burn flow

### Empty or Trivial Datasets
If a dataset has <10 rows or <2 columns:
1. Profile it anyway
2. Skip cleaning (nothing to clean)
3. Generate a minimal report
4. Note: "Dataset is trivial; minimal processing applied"

### API Failures
If an API is unreachable or returns errors:
1. Retry once with 10-second timeout
2. If still failing: log as skipped, search for alternative data source if in collection mode
3. Do not terminate the burn flow

---

## Integration with burn-my-tokens Family

This skill is a sibling to:
- `burn-my-tokens` (new project generation)
- `burn-my-tokens-refactor` (code refactoring)
- `burn-my-tokens-testgen` (test generation)
- `burn-my-tokens-review` (code review)

They share:
- The same burn contract philosophy (never stop, degrade on error, time-based resume)
- The same state file pattern (`.burn_state.json`)
- The same CronCreate resume mechanism
- The same output directory convention (`burn-my-tokens-data_output/`)

They differ in:
- **Goal**: Data engineering vs code generation vs refactoring vs testing vs review
- **Validation**: Before/after quality metrics vs test pass/fail vs smell reduction
- **Output**: Datasets + pipelines + plots vs PRDs + source code vs diff logs vs test files vs review reports

Sequential usage:
1. `/burn-my-tokens` to generate a new data-heavy MVP
2. `/burn-my-tokens-data` to collect, clean, and visualize datasets for it
3. `/burn-my-tokens-testgen` to add test coverage
4. `/burn-my-tokens-refactor` to clean up the pipeline code
