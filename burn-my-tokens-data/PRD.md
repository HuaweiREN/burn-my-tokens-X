# PRD: burn-my-tokens-data

**Version:** 1.0.0  
**Date:** 2026-05-12  
**Status:** L3 MVP

## Problem Statement

Users have remaining coding plan tokens that are about to expire. Instead of letting them go to waste, they want to convert those tokens into tangible data engineering assets: cleaned datasets, transformation pipelines, visualizations, and data quality reports. There is no existing skill in the burn family that targets data-specific tasks.

## Solution

**burn-my-tokens-data** is a Claude Code skill that burns remaining tokens to perform data engineering tasks: data collection, cleaning, transformation, and visualization. It follows the exact same burn framework as its siblings (burn-my-tokens, burn-my-tokens-refactor, burn-my-tokens-testgen) but is specialized for data workflows.

## User Stories

1. As a data analyst, I want to clean a messy CSV before my tokens expire, so I get a production-ready dataset.
2. As a researcher, I want to find and download public datasets on a topic, so I can start analysis without manual searching.
3. As an engineer, I want automated data quality reports and visualizations, so I can understand my data before modeling.
4. As a token-anxious user, I want the skill to keep working until my budget is exhausted or I say stop, so no tokens go to waste.

## Implementation Decisions

### Architecture
- Main Agent: Orchestrates the burn loop, manages budget, launches subagents
- Data Subagents: Each handles one dataset or one transformation pipeline
- Output directory: `burn-my-tokens-data_output/<task_name>/`

### Tier System
- `10k`: Quick data audit + 1-2 small cleaning scripts (~1 file)
- `100k`: Standard data pipeline (~1 dataset fully processed with profiling, cleaning, visualization)
- `1M`: Deep data burn (~3-4 datasets or complex multi-step pipeline)
- `10M`: Massive data burn (~8-12 datasets or enterprise-grade pipeline)
- `burn`: Burn until data task complete or request fails

### Data Safety
- NEVER overwrite original data files
- Always write cleaned/transformed outputs to `outputs/` subdirectory
- Keep raw data in `data/raw/` or reference original path
- Generate before/after quality metrics for every operation

### Supported Data Sources
- Local files: CSV, JSON, Excel (.xlsx), Parquet, SQLite
- APIs: REST endpoints with sample fetch + inspect
- No source: WebSearch for public datasets (Kaggle, UCI, data.gov, etc.)

### Supported Operations
- Profiling: pandas `.info()`, `.describe()`, null counts, dtypes, ydata-profiling style reports
- Cleaning: missing value imputation, deduplication, type conversion, outlier handling, encoding fixes
- Transformation: aggregation, feature engineering, merging/joining, normalization, pivoting
- Visualization: matplotlib/seaborn/plotly histograms, correlation heatmaps, time series plots, distribution comparisons

### Quality Metrics
- Null rate before/after
- Duplicate rate before/after
- Dtype consistency score
- Row count delta
- Column count delta
- Outlier flag count

## Testing Decisions

- Each subagent validates its own output (read back and check shape, nulls, basic stats)
- Main Agent runs cross-validation: compare before/after metrics
- If quality did not improve: switch strategy and retry once

## Out of Scope

- Real-time streaming data pipelines (batch only)
- Database write operations (read-only from DB, write to files)
- Distributed computing (Spark/Dask — single-machine pandas only)
- Production deployment (local files and reports only)

## Acceptance Criteria

- [ ] Skill activates on `/burn-my-tokens-data` or data-anxiety triggers
- [ ] Tier selection works (10k/100k/1M/10M/burn)
- [ ] Context gathering asks for data source, goal, and output format
- [ ] Auto-detects file format, size, encoding when path provided
- [ ] Enters collection mode when no source provided (WebSearch)
- [ ] Generates `data_profile.md` after ingestion
- [ ] Generates `cleaning_plan.md` with prioritized issues
- [ ] Executes pipeline via parallel subagents (max 2)
- [ ] Each subagent writes cleaned data + transformation script + plots
- [ ] Validates output and compares before/after metrics
- [ ] Generates final summary report
- [ ] Never stops on its own (burn contract)
- [ ] Degrades on error, does not terminate
- [ ] Time-based resume with CronCreate
- [ ] Budget visualization after each batch
- [ ] Graceful stop on user command
- [ ] State file tracks data_sources, quality_before, quality_after, output_files
