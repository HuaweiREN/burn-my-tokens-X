# burn-my-tokens-X — Product Requirements Document

## 1. Overview

**Product Name**: burn-my-tokens-X
**Family**: burn-my-tokens
**Version**: 1.0.0
**Status**: L3 MVP
**Date**: 2026-05-12

### 1.1 Purpose

burn-my-tokens-X is the orchestrator skill for the burn-my-tokens family. It runs the complete 7-stage pipeline — idea → research → MVP → data → testgen → review → refactor — in sequence, with automatic budget allocation, cross-stage output propagation, and iterative refinement loops. It converts expiring coding plan quotas into end-to-end product development cycles, not just single deliverables.

### 1.2 Target User

- Technical founders who want to go from concept to polished codebase in one burn session
- Product teams that need research-backed MVPs with tests and data pipelines
- Developers who want a full quality cycle (code + tests + review + refactor) on existing or new projects
- Anyone with substantial remaining tokens who wants maximum output density

### 1.3 Key Differentiators

| Dimension | burn-my-tokens-X | Individual burn skills | Traditional CI/CD pipeline |
|-----------|------------------|----------------------|---------------------------|
| **Scope** | 7 stages end-to-end | Single stage only | No ideation/research |
| **Budget intelligence** | Auto-allocation + hard caps per stage | Manual tier selection | No token awareness |
| **Iteration** | Multi-iteration with context carry-over | Single pass | Linear only |
| **Output chaining** | Stage N output feeds Stage N+1 input | Isolated outputs | Fixed steps |
| **Stagnation detection** | Auto-pivot after 2 flat iterations | N/A | N/A |
| **Autonomy** | Unattended burning with auto-resume | Unattended (single stage) | Requires manual triggers |

## 2. User Stories

### 2.1 Primary User Stories

**US-1: Full product cycle from idea to production**
As a founder, I want to run `/burn-my-tokens-X` with 100k tokens so that I get a validated idea, research-backed PRD, runnable MVP, datasets, tests, code review, and refactored code in one session.

**US-2: Time-constrained pipeline**
As a user with limited time, I want to run a 10k quick pipeline so that I get lightweight outputs across all 7 stages without deep dives.

**US-3: Continuous improvement loop**
As a builder, I want the pipeline to loop back after Stage 7 so that the second iteration refines the first based on review issues and refactor metrics.

**US-4: Budget-conscious execution**
As a user with a fixed token budget, I want per-stage hard caps so that no single stage consumes more than 50% of my total budget.

**US-5: Partial pipeline**
As a user with an existing MVP, I want to skip idea/research stages so that the remaining budget is redistributed to testgen, review, and refactor.

**US-6: Graceful stop**
As a user in a chat bot environment, I want to send "stop" and have the pipeline halt cleanly after the current stage batch, preserving all iteration progress.

### 2.2 System User Stories

**SYS-1: Cross-stage state management**
As the system, I want `stage_outputs` persisted in `.burn_state.json` so that each stage receives the necessary context from previous stages.

**SYS-2: Budget enforcement**
As the system, I want to track `stage_spent` vs `stage_budgets` so that no stage exceeds its allocation.

**SYS-3: Stagnation detection**
As the system, I want to detect when 2 consecutive iterations show no meaningful improvement so that I can prompt the user to continue, pivot, or stop.

## 3. Functional Requirements

### 3.1 Startup Flow (FR-1)

**FR-1.1**: The system shall present 5 tier options: 10k, 100k, 1M, 10M, burn.  
**FR-1.2**: The system shall collect: project domain or idea area (optional), total pipeline budget (optional), stages to skip (optional).  
**FR-1.3**: The system shall auto-detect existing pipeline state and sibling burn outputs.  
**FR-1.4**: The system shall support 4 execution modes (A/B/C/D) based on user inputs.  
**FR-1.5**: The system shall record all inputs in `.burn_state.json`.

### 3.2 Budget Allocation (FR-2)

**FR-2.1**: Default allocation ratios: idea 15%, research 10%, mvp 30%, data 10%, testgen 10%, review 10%, refactor 15%.  
**FR-2.2**: Skipped stage budgets shall be redistributed proportionally to remaining stages.  
**FR-2.3**: Hard cap per stage: default 100k tokens; if finite target_budget, no stage may exceed 50% of total.  
**FR-2.4**: 10k mode override: only stages 1-3, with idea 40%, research 30%, mvp 30%.

### 3.3 State Initialization (FR-3)

**FR-3.1**: The system shall create the output directory structure with `iteration_N/stage_M_skillname/` folders.  
**FR-3.2**: The system shall initialize `.burn_state.json` with `current_iteration: 1`, `current_stage: 1`, `stage_budgets`, `stage_spent`, `stage_outputs`.  
**FR-3.3**: The system shall create a `CronCreate` resume task on first startup.

### 3.4 Stage Orchestration (FR-4)

**FR-4.1**: Stages shall execute in order: 1(idea) → 2(research) → 3(mvp) → 4(data) → 5(testgen) → 6(review) → 7(refactor).  
**FR-4.2**: Skipped stages shall advance immediately with budget redistribution.  
**FR-4.3**: Each stage shall embed the corresponding individual skill's workflow using the same subagent patterns.  
**FR-4.4**: Stage outputs shall be propagated to the next stage via `stage_outputs` in state file.

### 3.5 Subagent Batching (FR-5)

**FR-5.1**: The system shall launch up to 2 parallel subagents per stage batch.  
**FR-5.2**: Before launching each task, the system shall check both total budget and stage budget caps.  
**FR-5.3**: If a task would exceed either cap, the system shall break the batch.

### 3.6 Iteration Loop (FR-6)

**FR-6.1**: After Stage 7 completes, the system shall write `iteration_N_summary.md`.  
**FR-6.2**: In burn mode: always continue to next iteration.  
**FR-6.3**: In tier mode: continue if `estimated_spent < target_budget * 0.95`.  
**FR-6.4**: Context carry-over: best idea becomes seed, research insights inform deeper analysis, MVP directory becomes refinement base, coverage/health/smell metrics set baselines for next iteration.  
**FR-6.5**: If 2 consecutive iterations show no meaningful improvement, prompt user: "Pipeline has stagnated for 2 iterations. Continue, pivot domain, or stop?"

### 3.7 Progress Reporting (FR-7)

**FR-7.1**: Token tracker shall display: Pipeline Iteration X/Y | Stage N/7 | Total spent / Target | Stage spent / Stage cap.  
**FR-7.2**: Progress reports shall be concise and embedded in execution flow.  
**FR-7.3**: The user shall be able to query "progress" or "status" at any time.

### 3.8 Final Reporting (FR-8)

**FR-8.1**: After pipeline completes, write `PIPELINE_REPORT.md` with metadata, iteration summaries, overall metrics, key deliverables, known limitations, and next steps.

### 3.9 Graceful Stop (FR-9)

**FR-9.1**: Stop commands set `status = "stopped"` in `.burn_state.json`.  
**FR-9.2**: Stop triggers `CronDelete` and outputs confirmation with iteration and stage count.  
**FR-9.3**: The system ends response between stage batches to allow queued stop commands to process.

### 3.10 Resume (FR-10)

**FR-10.1**: On first startup, create `CronCreate` resume task (every 3 minutes).  
**FR-10.2**: Resume restores `current_iteration`, `current_stage`, `stage_budgets`, `stage_spent`, `stage_outputs`, `completed_iterations`.  
**FR-10.3**: On completion, call `CronDelete` to clean up.

## 4. Non-Functional Requirements

### 4.1 Performance

**NFR-1**: Each stage batch should complete within reasonable token estimates (see Token Tracking).  
**NFR-2**: The system should save state and end response if execution exceeds 4 minutes.  
**NFR-3**: Iteration summary generation should use 1k-3k tokens.

### 4.2 Quality

**NFR-4**: All generated code in Stage 3 must follow `.claude/rules/python-coding-standards.md`.  
**NFR-5**: Stage outputs must be properly propagated; no stage may start without required context.  **NFR-6**: If in burn mode, iteration reports must show meaningful deltas, not flat metrics.

### 4.3 Safety

**NFR-7**: No stage shall exceed its allocated budget (hard cap enforcement).  
**NFR-8**: Review stage (Stage 6) must be read-only; refactor stage (Stage 7) must only modify after baseline test run.  
**NFR-9**: Data stage (Stage 4) must never overwrite original data files.

### 4.4 Usability

**NFR-10**: Pipeline output structure must be predictable and documented.  
**NFR-11**: Token tracker must be displayed after every stage batch.  
**NFR-12**: `PIPELINE_REPORT.md` must serve as a single source of truth for pipeline results.

## 5. System Architecture

### 5.1 Component Diagram

```
+-------------------+         +-------------------+         +-------------------+
|   User Input      |>--------|   Main Agent      |<--------|   .burn_state.json |
|  (/burn-my-tokens-|         |  (orchestrator)   |         |  (state + budgets) |
|   X [stop])       |         +-------------------+         +-------------------+
+-------------------+              |    |    |
                                   |    |    |
                        +----------+    |    +--------------+
                        |               |                   |
               +--------v--------+  +---v-----------+  +----v----------+
               | Budget Allocator|  |  Subagent 1   |  |  Subagent 2   |
               | (stage budgets) |  | (stage task)  |  | (stage task)  |
               +--------+--------+  +-------+-------+  +-------+-------+
                        |                  |                   |
               +--------v--------+         |                   |
               | Stage Sequencer |         |                   |
               | (1→2→3→4→5→6→7)|         |                   |
               +--------+--------+         |                   |
                        |                  |                   |
               +--------v--------+         |                   |
               | Output Propagator|        |                   |
               | (stage_outputs) |         |                   |
               +--------+--------+         |                   |
                        |                  |                   |
                        +------------------+-------------------+
                                           |
                                 +---------v---------+
                                 | Iteration Loop    |
                                 | (carry-over       |
                                 |  context)         |
                                 +-------------------+
```

### 5.2 Data Flow

1. User triggers skill → Main Agent collects inputs → Allocates stage budgets → Writes `.burn_state.json`
2. Main Agent initializes output directory structure → Creates CronCreate resume task
3. For each iteration:
   a. Stage 1 (idea): generates concepts → writes to `stage_1_idea/` → passes best idea to Stage 2
   b. Stage 2 (research): deep research → writes to `stage_2_research/` → passes insights to Stage 3
   c. Stage 3 (mvp): builds MVP → writes to `stage_3_mvp/` → passes project dir to Stages 4-7
   d. Stage 4 (data): discovers/cleans data → writes to `stage_4_data/` → passes datasets to Stage 5
   e. Stage 5 (testgen): generates tests → writes to `stage_5_testgen/` → passes coverage to Stage 6
   f. Stage 6 (review): reviews code → writes to `stage_6_review/` → passes issues to Stage 7
   g. Stage 7 (refactor): refactors code → writes to `stage_7_refactor/` → passes metrics to next iteration
4. After Stage 7: write iteration summary, check loop condition, carry over context
5. On completion: write `PIPELINE_REPORT.md`, call `CronDelete`

## 6. Output Specifications

### 6.1 Directory Structure

```
burn-my-tokens-X_output/
└── <pipeline_name>/
    ├── .burn_state.json
    ├── PIPELINE_REPORT.md
    ├── iteration_1/
    │   ├── stage_1_idea/
    │   │   ├── domain_analysis.md
    │   │   ├── raw_ideas/
    │   │   ├── IDEA_CARDS.md
    │   │   ├── TOP_PICKS.md
    │   │   └── IDEA_BURN_REPORT.md
    │   ├── stage_2_research/
    │   │   ├── research_outline.md
    │   │   ├── findings/
    │   │   ├── synthesis_notes.md
    │   │   ├── RESEARCH_REPORT.md
    │   │   ├── validation_report.md
    │   │   └── RESEARCH_BURN_REPORT.md
    │   ├── stage_3_mvp/
    │   │   ├── research.md
    │   │   ├── PRD.md
    │   │   ├── src/
    │   │   ├── tests/
    │   │   ├── requirements.txt
    │   │   ├── main.py
    │   │   ├── PROGRESS.md
    │   │   └── MVP_BURN_REPORT.md
    │   ├── stage_4_data/
    │   │   ├── data_profile.md
    │   │   ├── cleaning_plan.md
    │   │   ├── outputs/
    │   │   ├── src/pipeline_*.py
    │   │   └── DATA_BURN_REPORT.md
    │   ├── stage_5_testgen/
    │   │   ├── coverage_gap.md
    │   │   ├── test_strategy.md
    │   │   ├── tests/
    │   │   └── TEST_BURN_REPORT.md
    │   ├── stage_6_review/
    │   │   ├── analysis_report.md
    │   │   ├── review_plan.md
    │   │   ├── review_reports/
    │   │   ├── MASTER_REVIEW.md
    │   │   └── REVIEW_BURN_REPORT.md
    │   └── stage_7_refactor/
    │       ├── smell_report.md
    │       ├── refactor_plan.md
    │       ├── refactor_logs/
    │       └── REFACTOR_BURN_REPORT.md
    ├── iteration_2/
    │   └── ...
    └── stage_summaries/
        ├── iteration_1_summary.md
        ├── iteration_2_summary.md
        └── ...
```

### 6.2 File Formats

- All report files are Markdown (.md)
- Source code follows Python 3.10+ standards
- State file is JSON with comprehensive stage tracking

### 6.3 State File Schema

```json
{
  "status": "running | stopped | completed",
  "mode": "10k | 100k | 1M | 10M | burn",
  "target_budget": 100000,
  "estimated_spent": 0,
  "current_iteration": 1,
  "current_stage": 1,
  "stage_budgets": {
    "idea": 15000,
    "research": 10000,
    "mvp": 30000,
    "data": 10000,
    "testgen": 10000,
    "review": 10000,
    "refactor": 15000
  },
  "stage_spent": {
    "idea": 0,
    "research": 0,
    "mvp": 0,
    "data": 0,
    "testgen": 0,
    "review": 0,
    "refactor": 0
  },
  "stage_outputs": {
    "idea": {"top_ideas": [], "best_idea_name": "", "concept_brief": ""},
    "research": {"report_path": "", "key_insights": []},
    "mvp": {"project_dir": "", "tech_stack": []},
    "data": {"dataset_report": "", "cleaned_data_paths": []},
    "testgen": {"coverage_delta": 0, "current_coverage": 0},
    "review": {"health_score": 0, "top_issues": []},
    "refactor": {"smells_before": 0, "smells_after": 0}
  },
  "completed_iterations": [],
  "skipped_stages": [],
  "domain": "",
  "pipeline_name": "",
  "last_heartbeat": "ISO8601",
  "cron_job_id": null
}
```

## 7. Error Handling

### 7.1 Stage Failure Recovery

| Failure | Action |
|---------|--------|
| Critical stage fails (idea, mvp) | Offer user: retry, skip to next iteration, or stop |
| Non-critical stage fails (data, testgen, review, refactor) | Log warning, advance to next stage |
| Budget exhausted mid-stage | Complete current batch, save state, do not start new stage |
| State file missing on resume | Attempt reconstruction from output dirs; ask user if impossible |

### 7.2 Degradation by Stage

| Stage | Degradation Chain |
|-------|-------------------|
| MVP (Stage 3) | L3 → L2 → L1 |
| Data (Stage 4) | Full pipeline → Clean only → Profile only |
| TestGen (Stage 5) | Property-based → Example-based → Simple happy path |
| Review (Stage 6) | Full review → Summary review → Issue list only |
| Refactor (Stage 7) | Full refactor → Smaller extractions → Renaming only → Skip |

### 7.3 Abort Conditions

The system shall abort (with explanatory message) only when:
- Critical stage fails and user chooses to stop
- Total budget exhausted
- API returns persistent 403/429 errors
- User triggers graceful stop
- 2 consecutive stagnant iterations and user chooses to stop

## 8. Token Budget Estimates

| Tier | Target Budget | Estimated Output |
|------|---------------|------------------|
| 10k | 10,000 tokens | Stages 1-3 only, lightweight |
| 100k | 100,000 tokens | 1 full iteration, all 7 stages (reduced depth) |
| 1M | 1,000,000 tokens | 1-2 iterations, all stages thoroughly |
| 10M | 10,000,000 tokens | 3-5 iterations, comprehensive |
| burn | unlimited | Unlimited iterations until stopped |

### Per-Stage Estimates

| Stage | Tokens per unit | Unit |
|-------|-----------------|------|
| Idea | 3k-8k | per angle |
| Research | 2k-6k | per subtopic |
| MVP | 5k-15k | per project |
| Data | 3k-8k | per dataset |
| TestGen | 2k-6k | per module |
| Review | 3k-8k | per module |
| Refactor | 3k-8k | per module |

## 9. Success Metrics

### 9.1 Output Quality Metrics

- **Stage completion rate**: % of stages producing their expected key output (target: 100% for critical stages)
- **Cross-stage propagation success**: % of stages that received valid context from previous stage (target: 100%)
- **Iteration improvement delta**: measurable improvement in coverage, health score, or smell count between iterations (target: >0)

### 9.2 Process Metrics

- **Budget adherence**: % of stages staying within allocated budget (target: 100%)
- **Resume success rate**: % of interrupted pipelines resuming correctly (target: 100%)
- **Pipeline runtime**: iterations completed per 100k tokens (target: >= 1)

## 10. Future Enhancements (Post-MVP)

- **FE-1**: Dynamic budget reallocation — shift unused budget from completed stages to struggling ones
- **FE-2**: Parallel stage execution — run non-dependent stages (e.g., data + testgen) simultaneously
- **FE-3**: Custom stage definitions — allow users to define their own stage sequences
- **FE-4**: Integration with external CI/CD — trigger GitHub Actions after MVP stage
- **FE-5**: Multi-project pipelines — generate multiple MVPs in parallel within one pipeline
- **FE-6**: Visual pipeline dashboard — HTML report showing stage progress, metrics, and blockers
- **FE-7**: A/B iteration comparison — compare Iteration N vs N+1 with diff views

## 11. Dependencies

- Claude Code with WebSearch, WebFetch, Agent, CronCreate, CronDelete capabilities
- File system write access to `burn-my-tokens-X_output/`
- All individual burn skills available as implementation references
- Python 3.10+ environment for stages that generate code

## 12. Open Questions

1. Should X support custom user-defined stages beyond the 7 standard ones?
2. How should we handle pipelines where the user wants to start mid-pipeline (e.g., skip to review of existing project)?
3. Should iteration context carry-over include full file contents or just summaries to manage token usage?
4. How do we best visualize pipeline progress for users monitoring a long burn session?

---

**Document Owner**: burn-my-tokens family
**Last Updated**: 2026-05-12
**Change Log**: See skill version history
