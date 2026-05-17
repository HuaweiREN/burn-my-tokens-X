---
name: burn-my-tokens-data
description: When a user's coding plan quota is about to reset, automatically discover and evaluate open-source datasets that can support their project or MVP product. Presents dataset links and recommendation scores without downloading. Supports unattended self-iterative burning with pure text interaction.
version: 2.0.0
---

## Trigger Conditions

This skill activates when the user inputs any of the following:
- `/burn-my-tokens-data` — Start the dataset discovery burn flow
- `/burn-my-tokens-data stop` / `停止数据发现` / `stop data burn` / `stop burn` — Gracefully end the burn
- Expresses anxiety such as "need datasets / find data / looking for data / dataset discovery / tokens about to expire / burn tokens on datasets"

## Graceful Stop

When the user inputs a stop command, the Main Agent must:
1. Set `status` to `"stopped"` in `.burn_state.json`
2. Output stop confirmation: `Dataset discovery burn gracefully stopped. Completed X search angles, found Y datasets, consumed approximately XXk tokens.`
3. Call `CronDelete` to remove the resume cron job (if it exists)
4. Subsequent `CronCreate` triggers reading `status == "stopped"` will be ignored

### MetaBot Queue Compatibility Strategy

Bot frameworks like MetaBot queue new user instructions, so a stop command sent during the current response cannot immediately interrupt execution. To ensure graceful stop works reliably:

**After each search batch (or each discovery phase) completes, the Main Agent must:**
1. Save current progress to `.burn_state.json` (update `estimated_spent`, `completed_angles`, `pending_angles`, `datasets_found`)
2. Output `[Token Tracker]` and a concise progress report
3. **Actively end the current response**

This allows queued stop commands to be processed in the next turn. If the user sends no new instruction, burning auto-resumes via:
- The `CronCreate` scheduled task triggering `/burn-my-tokens-data resume` after 3 minutes
- Or any message from the user in chat triggering resume

**This behavior does not violate the "Never Stop on Your Own" contract** — ending the response is to respond to potential user instructions, not to terminate the burn flow. `status` in `.burn_state.json` remains `"running"`, and `CronCreate` handles recovery.

## Startup Flow (Main Agent)

### Step 0: Tier Selection & Context Gathering

Present tier options to the user and wait for selection:
- `1k` — Quick dataset scan (~3-5 datasets, 1 search round)
- `10k` — Standard discovery (~10-15 datasets, 3-5 search rounds)
- `100k` — Deep discovery (~30-50 datasets, 10-15 search rounds)
- `1M` — Massive discovery (~80-120 datasets, 30+ search rounds)
- `burn` — Burn until stopped

If the user selects `1k`, explicitly inform: "1k is sufficient for a quick scan of 3-5 datasets across 1 search round. Continue?"

**Then ask:**
```
🎯 What is your project goal or product name?
(Example: "3D Gaussian Splatting viewer", "autonomous driving perception", "financial sentiment analysis")
```

**Then ask:**
```
📋 Describe your data needs:
(Example: "need LiDAR point clouds with annotations", "looking for multi-lingual text corpora", "need satellite imagery for agriculture")
```

**Then ask:**
```
🔍 Any specific domains or formats to focus on?
(Leave empty for auto-discovery)
```

Record confirmed values in `.burn_state.json` under `project_goal`, `data_needs`, `focus_domains`, `tier`, `target_budget`.

**Execution modes:**
- **Mode A** (user provided all fields): Parse project profile → Generate search strategy → Launch discovery
- **Mode B** (user provided project goal only): Parse project profile → Auto-infer data needs → Launch discovery
- **Mode C** (user provided data needs only): Build project profile from data needs → Launch discovery
- **Mode D** (minimal input): Enter broad discovery mode → Search for trending/interesting datasets across domains

### Step 1: Project Understanding

Parse the project description to identify key data requirements.

**Data requirement extraction:**
1. Identify primary data types (images, text, audio, video, tabular, point clouds, meshes, time-series, graphs)
2. Identify formats (CSV, JSON, Parquet, HDF5, TFRecord, PNG, JPG, LAS, PLY, WAV, MP4, etc.)
3. Identify domains (healthcare, finance, autonomous driving, NLP, computer vision, robotics, geospatial, etc.)
4. Identify scale needs (small <1k samples, medium 1k-100k, large 100k-1M, massive >1M)
5. Identify annotation requirements (labeled, unlabeled, weakly labeled, multi-label, segmentation masks, bounding boxes, etc.)
6. Identify license preferences (public domain, CC0, CC-BY, MIT, Apache-2.0, academic-only, commercial-friendly)

**Output:** `burn-my-tokens-data_output/<task_name>/project_profile.md`

Format:
```markdown
# Project Data Profile

Generated: <timestamp>
Project: <project_goal>

## Data Needs Summary
<brief summary of what data the project requires>

## Required Data Types
- <type 1> (confidence: high/medium/low)
- <type 2> (confidence: high/medium/low)

## Preferred Formats
- <format 1>
- <format 2>

## Target Domains
- <domain 1>
- <domain 2>

## Scale Requirements
- Minimum samples: <N>
- Ideal scale: <small/medium/large/massive>

## Annotation Requirements
- <annotation type 1>
- <annotation type 2>

## License Preferences
- <license preference>

## Inferred Search Angles
1. <angle 1 description>
2. <angle 2 description>
3. <angle 3 description>
```

### Step 2: Search Strategy Generation

Generate search queries based on project needs.

**Search sources to cover:**
1. **Kaggle** — `site:kaggle.com/datasets <topic>`
2. **Hugging Face Datasets** — `site:huggingface.co/datasets <topic>`
3. **UCI ML Repository** — `archive.ics.uci.edu/ml/datasets.php <topic>`
4. **Google Dataset Search** — `datasetsearch.research.google.com <topic>`
5. **AWS Open Data Registry** — `registry.opendata.aws <topic>`
6. **data.gov / government open data** — `site:data.gov <topic>`
7. **Papers With Code datasets** — `paperswithcode.com/datasets <topic>`
8. **GitHub repositories with datasets** — `site:github.com <topic> dataset`
9. **Academic repositories** — `Harvard Dataverse`, `Zenodo`, `Figshare`
10. **Domain-specific hubs** — `Open Images`, `COCO`, `LAION`, `Common Crawl`, `The Pile`, etc.

**Query generation rules:**
- Generate 2-4 queries per search source
- Vary specificity: broad (`autonomous driving dataset`), narrow (`KITTI LiDAR semantic segmentation`), and comparative (`best dataset for 3D object detection 2024`)
- Include recency filters where useful (`2023`, `2024`, `latest`)
- Include format filters where useful (`CSV`, `Parquet`, `JSONL`)

**Output:** `burn-my-tokens-data_output/<task_name>/search_strategy.md`

Format:
```markdown
# Search Strategy

Generated: <timestamp>

## Search Angles

### Angle 1: <name>
- **Focus**: <what this angle targets>
- **Sources**: <list of sources>
- **Queries**:
  1. `<query 1>`
  2. `<query 2>`
  3. `<query 3>`

### Angle 2: <name>
...

## Coverage Map
| Source | Queries | Expected Datasets |
|--------|---------|-------------------|
| Kaggle | 3 | 2-4 |
| Hugging Face | 3 | 2-4 |
| ... | ... | ... |

## Execution Order
1. Angle 1 (highest relevance)
2. Angle 2
3. Angle 3
...
```

### Step 3: Dataset Discovery (Parallel Subagents)

Launch subagents per search angle (max 2 parallel, 3-4 in 1M mode).

**Scheduling strategy:**
- First 2 angles launched in parallel
- After completion, evaluate remaining budget before next batch
- In `1k` mode: process 1 angle at a time, no parallelism

**Each subagent's workspace:** `burn-my-tokens-data_output/<task_name>/discoveries/`

**Each subagent workflow:**

#### Subagent Step 1: Execute Searches (~30-40% budget)
- Use WebSearch with the assigned queries
- For each promising result, use WebFetch to extract dataset details
- Target: find 3-10 datasets per angle (depending on tier)

#### Subagent Step 2: Evaluate Datasets (~30-40% budget)
For each dataset found, record:

| Field | Description |
|-------|-------------|
| `name` | Dataset name |
| `source_url` | Direct URL to dataset page |
| `size_scale` | Approximate size (samples, GB, or "unknown") |
| `format` | File format(s) |
| `license` | License type (CC0, CC-BY, MIT, Apache-2.0, custom, unknown) |
| `last_updated` | Last update date or "unknown" |
| `description` | 1-2 sentence description |
| `relevance_score` | 1-10: how well it matches the project's data needs |
| `quality_score` | 1-10: perceived quality (documentation, completeness, community usage) |
| `recency_score` | 1-10: how recent/current the dataset is |
| `license_score` | 1-10: how permissive and clear the license is |
| `recommendation_score` | 1-10: weighted composite (see scoring formula below) |
| `why_it_fits` | 2-3 sentences explaining why this dataset supports the project |
| `potential_gaps` | Any limitations or mismatches with project needs |

**Scoring formula:**
```
recommendation_score = round(
    relevance_score * 0.40 +
    quality_score * 0.25 +
    recency_score * 0.20 +
    license_score * 0.15,
    1
)
```

**Score interpretation:**
| Score | Category | Action |
|-------|----------|--------|
| 9.0-10.0 | Essential | Must-include in top recommendations |
| 7.0-8.9 | Highly Recommended | Strong fit, prioritize |
| 5.0-6.9 | Nice to Have | Moderate fit, include if budget allows |
| 3.0-4.9 | Alternative | Weak fit, mention as fallback |
| 1.0-2.9 | Poor Fit | Exclude from report |

#### Subagent Step 3: Write Findings (~20-30% budget)
Write findings to `burn-my-tokens-data_output/<task_name>/discoveries/<angle_name>.md`

Format:
```markdown
# Discovery Report: <angle_name>

Generated: <timestamp>
Search queries used:
- `<query 1>`
- `<query 2>`

## Datasets Found

### 1. <dataset_name>
- **Source**: <URL>
- **Size/Scale**: <size>
- **Format**: <format>
- **License**: <license>
- **Last Updated**: <date>
- **Description**: <description>
- **Relevance**: <relevance_score>/10
- **Quality**: <quality_score>/10
- **Recency**: <recency_score>/10
- **License Score**: <license_score>/10
- **Recommendation**: <recommendation_score>/10
- **Why It Fits**: <explanation>
- **Potential Gaps**: <limitations>

### 2. <dataset_name>
...

## Search Coverage
- Queries executed: X
- Results inspected: Y
- Datasets fully evaluated: Z
- Datasets excluded (poor fit): W
```

#### Subagent Step 4: Report to Main Agent (~5-10% budget)
Report back:
```
Angle: <angle_name>
Datasets found: X
Top recommendation: <dataset_name> (<recommendation_score>/10)
Search queries used: N
Status: complete / partial / degraded
```

### Step 4: Dataset Evaluation & Ranking (Main Agent)

After each batch of subagents completes:

1. **Read all discovery reports** from `discoveries/<angle>.md`
2. **Deduplicate** datasets found by multiple subagents (match by name or URL)
3. **Rank by recommendation score** descending
4. **Categorize**:
   - `Essential` (9.0-10.0)
   - `Highly Recommended` (7.0-8.9)
   - `Nice to Have` (5.0-6.9)
   - `Alternative` (3.0-4.9)
5. **Identify gaps**: data needs from `project_profile.md` not satisfied by any discovered dataset

**Output:** Internal ranking table (used for report generation)

### Step 5: Report Generation

After all pending angles complete (or budget exhausted), generate:

**Master Report:** `burn-my-tokens-data_output/<task_name>/DATASET_DISCOVERY_REPORT.md`

Format:
```markdown
# Dataset Discovery Report

Generated: <timestamp>
Project: <project_goal>
Data Needs: <data_needs>
Discovery Tier: <tier>
Search Rounds: <N>
Datasets Evaluated: <N>

---

## Executive Summary

<2-3 paragraph summary of findings>

**Top Finding**: <best dataset and why>
**Coverage**: <what data needs are satisfied / not satisfied>
**Recommendation**: <next steps for the user>

---

## Project Data Profile

<inline or reference to project_profile.md>

---

## Top 10 Recommended Datasets

### 1. <dataset_name> — <recommendation_score>/10 (Essential)
- **Source**: <URL>
- **Size**: <size>
- **Format**: <format>
- **License**: <license>
- **Why It Fits**: <explanation>

### 2. <dataset_name> — <recommendation_score>/10 (Highly Recommended)
...

---

## Full Dataset Catalog

| Rank | Name | Source | Score | Category | Format | License |
|------|------|--------|-------|----------|--------|---------|
| 1 | ... | ... | ... | ... | ... | ... |

---

## Category Breakdown

### Essential (9.0-10.0)
- <dataset 1>
- <dataset 2>

### Highly Recommended (7.0-8.9)
- <dataset 3>
- <dataset 4>

### Nice to Have (5.0-6.9)
- <dataset 5>

### Alternative (3.0-4.9)
- <dataset 6>

---

## Search Coverage

| Source | Queried | Datasets Found | Avg Score |
|--------|---------|----------------|-----------|
| Kaggle | Y/N | N | X.X |
| Hugging Face | Y/N | N | X.X |
| UCI ML | Y/N | N | X.X |
| Google Dataset Search | Y/N | N | X.X |
| AWS Open Data | Y/N | N | X.X |
| data.gov | Y/N | N | X.X |
| Papers With Code | Y/N | N | X.X |
| GitHub | Y/N | N | X.X |
| Academic Repos | Y/N | N | X.X |
| Domain Hubs | Y/N | N | X.X |

---

## Known Gaps

The following data needs were not satisfied by discovered datasets:

1. **<data need>**: <description of what was sought but not found>
   - Suggested search: <alternative query or source to try>

2. **<data need>**: ...

---

## Methodology

- Discovery method: WebSearch + WebFetch
- Evaluation criteria: Relevance (40%), Quality (25%), Recency (20%), License (15%)
- No datasets were downloaded or inspected locally
- All links are to public dataset pages; access and download terms are set by dataset owners
```

**Console Summary:**

```
Dataset discovery burn complete!

Project: <project_goal>
Tier: <tier>

Datasets discovered: X
  - Essential: A
  - Highly Recommended: B
  - Nice to Have: C
  - Alternative: D

Top 3 datasets:
1. <name> (<score>/10) — <URL>
2. <name> (<score>/10) — <URL>
3. <name> (<score>/10) — <URL>

Known gaps: N data needs not satisfied
  - <gap 1>
  - <gap 2>

Output directory: burn-my-tokens-data_output/<task_name>/
  ├── project_profile.md
  ├── search_strategy.md
  ├── discoveries/
  │   ├── <angle_1>.md
  │   ├── <angle_2>.md
  │   └── ...
  ├── DATASET_DISCOVERY_REPORT.md
  └── .burn_state.json

Next steps:
- Visit top dataset links to verify access and terms
- Run `/burn-my-tokens-data` with a higher tier for deeper coverage
- Run `/burn-my-tokens-MVP` to build a prototype using discovered datasets
```

Also write `burn-my-tokens-data_output/<task_name>/BURN_REPORT.md` with the same content as the console summary.

---

## Burn Contract

The Main Agent must adhere to the following hard constraints and must not violate them for any reason:

### Constraint 1: Never Stop on Your Own
- After completing a search batch, immediately check budget and remaining angles
- If budget is not exhausted and angles remain (or new angles can be generated), immediately begin the next batch
- Prohibited from outputting terminal phrases such as "that's all for now", "let's pause here", "continue next time"
- The only stop conditions are: reaching the 95% tier threshold, API returning 429/403, all search angles complete with no more sources to query, or user-triggered graceful stop
- "Ending the response between batches" is not a stop — it is a necessary window to respond to queued user instructions. `.burn_state.json` remains `status == "running"`, and `CronCreate` auto-resumes

### Constraint 2: Degrade on Error, Don't Terminate
- When a subagent fails to complete a search angle (search blocked, parsing errors, etc.), the Main Agent must attempt degradation:
  1. First failure: Retry with simplified queries (fewer terms, broader keywords)
  2. Second failure: Switch to single-source search (only Kaggle or only Hugging Face)
  3. Third failure: Skip angle, log reason, move to next angle
- Only after 3 consecutive degradation failures on the same angle is skipping allowed
- When skipping an angle, briefly log the reason without extended discussion

### Constraint 3: Discovery-Only Safety — Never Download or Ingest Data
- This skill must NEVER download, read, or process dataset files locally
- Only write metadata, links, scores, and evaluations to output files
- Do not use pandas, file readers, or data loaders
- Do not attempt to access APIs that return raw data
- If a dataset page requires authentication to view details, note "access restricted" and move on

### Constraint 4: Time-Based Resume
- If the response is perceived to be approaching the time limit (executed >4 minutes), the Main Agent must:
  1. Write current progress to `burn-my-tokens-data_output/<task_name>/.burn_state.json`
  2. Ensure the CronCreate resume task is started
  3. Output: "Time limit triggered resume; scheduled task will auto-restart dataset discovery in 3 minutes"
- Upon resume, read `.burn_state.json` to restore context

### Constraint 5: Progress Reports Must Not Block Execution
- Progress reports must be embedded in the execution flow in the most concise format possible
- Prohibited from interrupting the execution chain for a "complete report"
- Detailed reports only when the user queries progress

### Constraint 6: Budget Visualization Obligation
- After each search batch completes, update and display `[Token Tracker]`
- This serves as both a report and a self-restraint trigger for the Main Agent

### Constraint 7: Subagent Timeout
- If a subagent produces no output within reasonable time, treat as failure and degrade.

---

## Core Execution Loop (Dataset Discovery Burn Loop)

The Main Agent's core execution logic is as follows. It must iterate as much as possible within a single response:

```
while estimated_spent < target_budget * 0.95 and angles_remain:
    # 1. Check for unfinished subagents
    if pending_agents:
        collect_results()
        update_dataset_ranking()
    
    # 2. If no angles pending and budget remains, generate new angles from gaps
    if not pending_angles and estimated_spent < target_budget * 0.80:
        pending_angles = generate_gap_angles()
        if not pending_angles:
            break  # No more angles to explore
    
    # 3. If no angles at all, we're done
    if not pending_angles:
        break
    
    # 4. Launch next batch of subagents (max 2 in parallel, 3-4 in 1M mode, 1 in 1k mode)
    batch = pending_angles.pop_next(2)
    for angle in batch:
        if estimated_spent + angle.estimated_cost > target_budget * 0.95:
            break
        launch_subagent(angle)
        estimated_spent += angle.estimated_cost
    
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

## Subagent Discovery Workflow

Each subagent must strictly follow these 4 steps:

### Step 1: Execute Searches (~30-40% budget)
- Use WebSearch with the assigned queries from `search_strategy.md`
- For each promising search result, use WebFetch to extract dataset details
- Target: find 3-10 datasets per angle (depending on tier)
- If a search returns no relevant results, try 1-2 variant queries before giving up

### Step 2: Evaluate Datasets (~30-40% budget)
- For each dataset found, fill out the full evaluation fields
- Be conservative with scores: a dataset with no documentation should not score above 5 on quality
- A dataset with restrictive or unclear license should not score above 5 on license_score
- A dataset older than 5 years without updates should not score above 5 on recency_score
- Record why_it_fits and potential_gaps honestly

### Step 3: Write Findings (~20-30% budget)
- Write the discovery report to `discoveries/<angle_name>.md`
- Ensure every dataset has a direct URL
- Ensure scores are consistent and justified
- Note any search limitations (blocked sites, paywalled repositories)

### Step 4: Report to Main Agent (~5-10% budget)
- Report: angle name, datasets found, top recommendation score, status
- If degraded, explain what failed and what was attempted

---

## Output Specifications

- **Directory**: `burn-my-tokens-data_output/<task_name>/`
- **Project profile**: `burn-my-tokens-data_output/<task_name>/project_profile.md`
- **Search strategy**: `burn-my-tokens-data_output/<task_name>/search_strategy.md`
- **Discovery reports**: `burn-my-tokens-data_output/<task_name>/discoveries/<angle_name>.md`
- **Master report**: `burn-my-tokens-data_output/<task_name>/DATASET_DISCOVERY_REPORT.md`
- **Burn report**: `burn-my-tokens-data_output/<task_name>/BURN_REPORT.md`
- **State file**: `burn-my-tokens-data_output/<task_name>/.burn_state.json`

---

## Dataset Source Reference

### Primary Sources (always search)

| Source | URL Pattern | Search Query Pattern | Notes |
|--------|-------------|----------------------|-------|
| Kaggle | kaggle.com/datasets | `site:kaggle.com/datasets <topic>` | Requires account for download; metadata is public |
| Hugging Face Datasets | huggingface.co/datasets | `site:huggingface.co/datasets <topic>` | Excellent for NLP, audio, vision |
| UCI ML Repository | archive.ics.uci.edu/ml/datasets.php | `site:archive.ics.uci.edu <topic>` | Classic ML datasets, mostly tabular |
| Google Dataset Search | datasetsearch.research.google.com | `datasetsearch.research.google.com <topic>` | Aggregates across sources |
| AWS Open Data | registry.opendata.aws | `site:registry.opendata.aws <topic>` | Large-scale, cloud-native |
| data.gov | data.gov | `site:data.gov <topic>` | US government data |
| Papers With Code | paperswithcode.com/datasets | `site:paperswithcode.com/datasets <topic>` | Links datasets to papers and SOTA |
| GitHub | github.com | `site:github.com <topic> dataset` | Community-contributed, variable quality |

### Secondary Sources (search if primary exhausted)

| Source | URL Pattern | Best For |
|--------|-------------|----------|
| Harvard Dataverse | dataverse.harvard.edu | Academic research data |
| Zenodo | zenodo.org | Research outputs, DOI-assigned |
| Figshare | figshare.com | Academic datasets |
| OpenML | openml.org | ML benchmark datasets |
| LOD Cloud | lod-cloud.net | Linked open data |
| Common Crawl | commoncrawl.org | Web-scale text data |
| LAION | laion.ai | Large-scale image-text pairs |
| Open Images | storage.googleapis.com/openimages | Computer vision |
| COCO Dataset | cocodataset.org | Object detection, segmentation |
| ImageNet | image-net.org | Image classification |

### Domain-Specific Sources

| Domain | Key Sources |
|--------|-------------|
| Autonomous Driving | KITTI, nuScenes, Waymo Open, Argoverse, BDD100K |
| NLP | The Pile, C4, OSCAR, OpenWebText, GLUE, SuperGLUE |
| Healthcare | MIMIC-III/IV (restricted), PubMed, TCGA, UK Biobank |
| Finance | WRDS (academic), FRED, Quandl, Yahoo Finance |
| Geospatial | Sentinel Hub, Landsat, OpenStreetMap, SpaceNet |
| Robotics | RLBench, Meta-World, RoboTurk, BridgeData |
| 3D/Vision | ShapeNet, ModelNet, ScanNet, Matterport3D |

---

## Scoring Reference

### Relevance Score (1-10)
| Score | Criteria |
|-------|----------|
| 10 | Perfect match: data type, format, domain, and scale all align |
| 8-9 | Strong match: most requirements met, minor mismatches |
| 6-7 | Moderate match: core need met but gaps in format or scale |
| 4-5 | Weak match: tangentially related, would require significant adaptation |
| 1-3 | Poor match: barely related to project needs |

### Quality Score (1-10)
| Score | Criteria |
|-------|----------|
| 10 | Excellent documentation, widely used, peer-reviewed, clean metadata |
| 8-9 | Good documentation, active community, clear structure |
| 6-7 | Adequate documentation, some known issues |
| 4-5 | Minimal documentation, unclear structure |
| 1-3 | No documentation, questionable provenance |

### Recency Score (1-10)
| Score | Criteria |
|-------|----------|
| 10 | Updated within last 6 months or released recently |
| 8-9 | Updated within last 1-2 years |
| 6-7 | 2-4 years old, still relevant |
| 4-5 | 4-6 years old, may be outdated |
| 1-3 | >6 years old, likely obsolete |

### License Score (1-10)
| Score | License |
|-------|---------|
| 10 | CC0, Public Domain |
| 9 | MIT, Apache-2.0, BSD |
| 8 | CC-BY 4.0 |
| 6-7 | CC-BY-SA, CC-BY-NC, Academic-only |
| 4-5 | Custom permissive (requires review) |
| 1-3 | Restrictive, commercial prohibited, or unclear |

---

## Token Tracking Rules

**Important**: Claude Code does not provide a remaining token API; all tracking is estimated.

- Each WebSearch ≈ 500-1500 tokens
- Each WebFetch ≈ 500-1000 tokens
- Each Agent call ≈ 2000-5000 tokens overhead
- Each subagent launch ≈ 2000-5000 tokens overhead
- Each report generation (project profile, search strategy, master report) ≈ 1000-3000 tokens equivalent

The Main Agent maintains a running estimate in the conversation, formatted as:
```
[Token Tracker] Estimated consumed: XXk / Target: XXk | Angles: X completed / Y pending | Datasets: Z found
```

**Tier mode 95% threshold**:
When estimated consumption reaches 95% of the target tier, the Main Agent proactively pauses and reports:
> "Estimated consumption is approaching the target budget (XXk / XXk). Angles completed: X / Y. Datasets found: Z. Continue with the last angle or stop?"
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
  "mode": "10k",
  "target_budget": 10000,
  "estimated_spent": 3500,
  "project_goal": "autonomous driving perception",
  "data_needs": "need LiDAR point clouds with annotations",
  "focus_domains": ["autonomous driving", "LiDAR"],
  "completed_angles": ["kaggle_autonomous_driving", "huggingface_lidar"],
  "pending_angles": ["paperswithcode_3d_detection", "aws_open_data_urban"],
  "datasets_found": [
    {
      "name": "KITTI",
      "source_url": "http://www.cvlibs.net/datasets/kitti/",
      "recommendation_score": 9.5,
      "category": "Essential"
    }
  ],
  "search_queries_used": [
    "site:kaggle.com/datasets autonomous driving LiDAR",
    "site:huggingface.co/datasets LiDAR point cloud"
  ],
  "known_gaps": ["real-time streaming LiDAR data", "adverse weather LiDAR"],
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
Claude: [Select tier] [Project goal] [Data needs] [Focus domains] → Start dataset discovery
Claude: [Internal] CronCreate creates resume task
...
[Session ends due to time limit]
[3 minutes later, cron auto-triggers]
Claude: [Auto-resumes dataset discovery from breakpoint]
...
[Budget exhausted or all angles processed]
Claude: [Dataset discovery complete report] [CronDelete removes resume task]
```

### Resume Parameters

When the scheduled task or user triggers `/burn-my-tokens-data resume`:
1. The Main Agent reads `.burn_state.json`
2. Restores `estimated_spent`, `completed_angles`, `pending_angles`, `project_goal`, `data_needs`, `focus_domains`, `datasets_found`, `search_queries_used`, `known_gaps`
3. Continues the Dataset Discovery Burn Loop from the breakpoint
4. No need to re-ask tier, project goal, or data needs (unless the state file is missing)

---

## Fallback Strategies

### No Search Results for an Angle
If a search angle returns no relevant datasets:
1. Try broader keywords (remove format filters, use synonyms)
2. Try alternative sources (if Kaggle fails, try Hugging Face)
3. If still no results after 2 retries: log in report, mark angle as "no results", move on

### Blocked or Restricted Pages
If a dataset page is blocked or requires authentication:
1. Note "access restricted" in the discovery report
2. Search for alternative sources hosting the same dataset
3. If no alternative: include dataset name with a note, exclude from scoring

### Duplicate Datasets Across Angles
If the same dataset is found by multiple subagents:
1. Main Agent deduplicates by matching name and URL
2. Keep the evaluation with the highest detail level
3. Note in master report: "Found via multiple search angles"

### Vague or Minimal User Input
If the user provides minimal project description:
1. Use WebSearch to find trending datasets in related domains
2. Generate broader search angles
3. Present findings as "exploratory discovery" rather than targeted recommendations

---

## Integration with Sibling Skills

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
- **Goal**: Dataset discovery vs code generation vs refactoring vs testing vs review
- **Validation**: Recommendation scores and coverage metrics vs test pass/fail vs smell reduction
- **Output**: Dataset catalogs with links and scores vs PRDs + source code vs diff logs vs test files vs review reports
- **Safety**: Discovery-only (no downloads) vs may modify source code

Sequential usage:
1. `/burn-my-tokens` to generate a new MVP idea
2. `/burn-my-tokens-data` to discover datasets that can support the MVP
3. `/burn-my-tokens-MVP` to build the MVP prototype
4. `/burn-my-tokens-testgen` to add test coverage
5. `/burn-my-tokens-review` to review the MVP code

---

## Pipeline Integration (burn-my-tokens-X Entry Point)

This skill can be invoked as a stage in the `burn-my-tokens-X` pipeline orchestrator.

### Stage Position
- **Stage**: 4 of 7 (Dataset Discovery)
- **Preceded by**: `burn-my-tokens-MVP` (Stage 3)
- **Followed by**: `burn-my-tokens-testgen` (Stage 5)

### Parameters from X
When invoked by `burn-my-tokens-X`, the Main Agent receives:
- `allocated_budget`: Token budget allocated for this stage
- `project_goal`: Project name/description from previous stages
- `mvp_outputs`: Paths to generated MVP files (if available)
- `pipeline_mode`: "sequential" (default) or "parallel"
- `previous_outputs`: Outputs from Stage 3 (MVP)

### Behavior in Pipeline Mode
1. Skip tier selection — use allocated budget directly
2. Auto-extract data needs from `project_goal` and `mvp_outputs`
3. Adjust discovery targets based on `allocated_budget`:
   - < 5k: 3-5 datasets
   - 5k-20k: 10-15 datasets
   - 20k-80k: 30-50 datasets
   - 80k+: 80+ datasets
4. Write outputs to shared pipeline directory
5. Return structured summary to X:
   - `top_datasets`: Names and URLs of top 10 datasets
   - `coverage_score`: % of likely-needed data types found
   - `gaps`: List of data needs not satisfied
   - `estimated_spent`: Actual tokens consumed
   - `output_paths`: Generated file paths

### Budget Allocation Guidelines for X
| Pipeline Budget | Suggested Data Stage Allocation | Notes |
|-----------------|--------------------------------|-------|
| 50k total | 5k-10k | Quick scan for key datasets |
| 100k total | 12k-20k | Standard discovery |
| 500k total | 50k-80k | Deep discovery with gap analysis |
| 1M+ total | 100k-150k | Comprehensive dataset landscape |
