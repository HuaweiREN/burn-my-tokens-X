---
name: burn-my-tokens-X
description: When a user's coding plan quota is about to reset, run the complete burn-my-tokens pipeline — idea → research → MVP → data → testgen → review → refactor — and loop back for continuous improvement. Supports budget allocation across stages and unattended self-iterative burning.
version: 1.0.0
---

## Trigger Conditions

This skill activates when the user inputs any of the following:
- `/burn-my-tokens-X` — Start the pipeline burn flow
- `/burn-my-tokens-X stop` / `停止管道燃烧` / `stop pipeline burn` / `stop burn` — Gracefully end the burn
- Expresses anxiety such as "run full pipeline / burn tokens on everything / complete burn cycle / tokens about to expire / run all burn stages"

## Graceful Stop

When the user inputs a stop command, the Main Agent must:
1. Set `status` to `"stopped"` in `.burn_state.json`
2. Output stop confirmation: `Pipeline burn gracefully stopped. Completed X iterations, Y stages, consumed approximately XXk tokens.`
3. Call `CronDelete` to remove the resume cron job (if it exists)
4. Subsequent `CronCreate` triggers reading `status == "stopped"` will be ignored

### MetaBot Queue Compatibility Strategy

Bot frameworks like MetaBot queue new user instructions, so a stop command sent during the current response cannot immediately interrupt execution. To ensure graceful stop works reliably:

**After each stage completes (or each subagent batch within a stage), the Main Agent must:**
1. Save current progress to `.burn_state.json` (update `estimated_spent`, `current_stage`, `current_iteration`, `stage_outputs`, `completed_iterations`)
2. Output `[Token Tracker]` and a concise progress report
3. **Actively end the current response**

This allows queued stop commands to be processed in the next turn. If the user sends no new instruction, burning auto-resumes via:
- The `CronCreate` scheduled task triggering `/burn-my-tokens-X resume` after 3 minutes
- Or any message from the user in chat triggering resume

**This behavior does not violate the "Never Stop on Your Own" contract** — ending the response is to respond to potential user instructions, not to terminate the burn flow. `status` in `.burn_state.json` remains `"running"`, and `CronCreate` handles recovery.

## Startup Flow (Main Agent)

### Step 0: Tier Selection & Context Gathering

Present tier options to the user and wait for selection:
- `10k` — Quick pipeline run (Stages 1-3 only, lightweight)
- `100k` — Standard pipeline (1 full iteration, all 7 stages)
- `1M` — Deep pipeline (1-2 iterations, all stages thoroughly)
- `10M` — Massive pipeline (3-5 iterations, comprehensive)
- `burn` — Burn until stopped (unlimited iterations)

If the user selects `10k`, explicitly inform: "10k runs only Stages 1-3 (idea → research → MVP) with lightweight outputs. Continue?"

**Then ask:**
```
🎯 What is your project domain or idea area?
(Leave empty for autonomous discovery)

Examples:
- B2B SaaS for veterinary clinics
- AI-powered creative tools for indie developers
- Sustainable packaging for e-commerce
```

**Then ask:**
```
💰 Total pipeline budget? (tokens)
(Leave empty for burn-until-stopped mode)

Examples:
- 1000000
- 1000000
- burn
```

**Then ask:**
```
⚙️ Any stages to skip? (idea/research/mvp/data/testgen/review/refactor/none)
(Leave empty to run all stages)
```

**Auto-detect and confirm:**

The Main Agent uses Bash/Glob to attempt detecting:
- Existing burn state from a previous run: `burn-my-tokens-X_output/*/.burn_state.json`
- Existing projects in sibling burn output directories
- Current working directory for potential MVP target

Present detected info:
```
Detected:
- Previous pipeline state: <found / not found>
- Existing burn outputs: <list of sibling burn output directories>
- Working directory: <cwd>
Correct? (Press Enter to confirm, or input corrections)
```

Record confirmed values in `.burn_state.json` under `domain`, `target_budget`, `skipped_stages`, `mode`.

**Four execution modes:**
- **Mode A** (user provided domain + budget + no skips): Full pipeline with budget allocation
- **Mode B** (user provided domain + budget + skips): Partial pipeline with reallocated budget
- **Mode C** (user provided domain, no budget): Burn-until-stopped mode, unlimited iterations
- **Mode D** (no domain provided): Autonomous discovery — X will pick trending/underserved domains via WebSearch

### Budget Allocation

Given `target_budget` (or treating as infinite in burn mode), allocate across stages using default ratios:

| Stage | Default Ratio | Purpose |
|-------|--------------|---------|
| idea | 15% | Domain analysis, raw ideation, evaluation, refinement |
| research | 10% | Deep research on top idea, competitive landscape, citations |
| mvp | 30% | PRD, core development, tests, verification run |
| data | 10% | Dataset discovery, cleaning, profiling, visualization |
| testgen | 10% | Coverage analysis, test generation, validation |
| review | 10% | Code analysis, structured review reports, health scoring |
| refactor | 15% | Smell detection, safe refactoring, diff logs, validation |

**Budget formulas:**
```
stage_budgets[stage] = floor(target_budget * default_ratio[stage])
total_allocated = sum(stage_budgets.values())
```

If user skips stages, redistribute skipped budgets proportionally to remaining stages using the same ratio weights.

**Hard cap per stage (even in burn mode):**
- Default: 1,000,000 tokens per stage
- If target_budget is specified and finite: `min(stage_budget, target_budget * 0.5)` — no single stage may consume more than 50% of total budget

**100k mode override:**
- Only stages 1-3 run
- idea: 40%, research: 30%, mvp: 30%
- All other stages are forced-skipped

### Step 1: Initialize State & Output Structure

Create the output directory structure:
```
burn-my-tokens-X_output/<pipeline_name>/
├── .burn_state.json
├── PIPELINE_REPORT.md
├── iteration_1/
│   ├── stage_1_idea/
│   ├── stage_2_research/
│   ├── stage_3_mvp/
│   ├── stage_4_data/
│   ├── stage_5_testgen/
│   ├── stage_6_review/
│   └── stage_7_refactor/
├── iteration_2/
│   └── ...
└── stage_summaries/
    ├── iteration_1_summary.md
    └── ...
```

Initialize `.burn_state.json`:
```json
{
  "status": "running",
  "mode": "1M",
  "target_budget": 1000000,
  "estimated_spent": 0,
  "current_iteration": 1,
  "current_stage": 1,
  "stage_budgets": {
    "idea": 150000,
    "research": 100000,
    "mvp": 300000,
    "data": 100000,
    "testgen": 100000,
    "review": 100000,
    "refactor": 150000
  },
  "stage_outputs": {
    "idea": {"top_ideas": [], "output_dir": ""},
    "research": {"report_path": ""},
    "mvp": {"project_dir": ""},
    "data": {"dataset_report": ""},
    "testgen": {"coverage_delta": 0},
    "review": {"health_score": 0},
    "refactor": {"smells_after": 0}
  },
  "completed_iterations": [],
  "skipped_stages": [],
  "domain": "",
  "last_heartbeat": "2026-05-12T14:30:00",
  "cron_job_id": null
}
```

### Step 2: CronCreate Resume Task

**On first startup**, call `CronCreate` to create a durable scheduled task:

- **cron**: `1-59/3 * * * *` (every 3 minutes, avoiding top of the hour)
- **durable**: `true` (persists across session restarts)
- **prompt**:
  ```
  Read burn-my-tokens-X_output/<pipeline_name>/.burn_state.json.
  If status == "running" and last_heartbeat is more than 10 minutes ago:
    Execute /burn-my-tokens-X resume
  Otherwise: ignore, do nothing.
  ```

Record the returned `job_id` in `.burn_state.json` under `cron_job_id`.

---

## Pipeline Stage Orchestration

X does not "call" other skills via the skill system. Instead, X embeds the orchestration logic for each stage, using the same subagent patterns as individual skills. Each stage section references the corresponding individual skill's workflow as the "implementation reference."

### Stage 1: Idea Generation (burn-my-tokens-idea)

**Implementation reference**: See `burn-my-tokens-idea` skill for full ideation workflow.
**X-specific behavior**:
- Budget: 15% of total (or user-adjusted)
- Inputs: Domain from user or previous iteration
- Outputs: Top 3 ideas with scores and concept briefs
- Next stage input: Best idea name + concept brief
- Output directory: `burn-my-tokens-X_output/<pipeline_name>/iteration_N/stage_1_idea/`

**Subagent workflow (embedded):**
1. **Domain Analysis** (~20% of stage budget): Use WebSearch to research domain gaps, pain points, underserved segments, emerging trends. Write `domain_analysis.md`.
2. **Idea Generation** (~40% of stage budget): Launch subagents per niche angle (max 2 parallel). Each subagent generates 5-8 raw ideas passing the NICHE test (Narrow, Innovative, Concrete, Heavy, Edge). Write `raw_ideas/<angle>.md`.
3. **Idea Evaluation** (~20% of stage budget): Main Agent reads all raw ideas, scores each on 5 axes (feasibility, novelty, market_size, differentiation, personal_fit), validates top ideas via WebSearch. Write `IDEA_CARDS.md`.
4. **Idea Refinement** (~15% of stage budget): Select top 3-5 ideas, produce structured concept briefs with problem, solution, target user, differentiation, MVP scope, risks. Write `TOP_PICKS.md`.
5. **Report** (~5% of stage budget): Write `IDEA_BURN_REPORT.md`.

**Output propagation to Stage 2:**
- Pass `best_idea_name`, `concept_brief`, `problem_statement`, `mvp_scope`, `tech_stack_suggestion`

### Stage 2: Research (burn-my-tokens-research)

**Implementation reference**: See `burn-my-tokens-research` skill for full research workflow.
**X-specific behavior**:
- Budget: 10% of total (or user-adjusted)
- Inputs: Best idea name + concept brief from Stage 1
- Outputs: Comprehensive research report with citations
- Next stage input: Research insights + validated assumptions
- Output directory: `burn-my-tokens-X_output/<pipeline_name>/iteration_N/stage_2_research/`

**Subagent workflow (embedded):**
1. **Research Scoping** (~10% of stage budget): Formulate 3-7 core research questions based on the idea. Decompose into sub-topics. Write `research_outline.md`.
2. **Information Gathering** (~50% of stage budget): Launch subagents per sub-topic (max 2 parallel). Each subagent executes WebSearch, WebFetch for detailed extraction, cross-references sources, and writes `findings/<subtopic>.md`.
3. **Synthesis** (~20% of stage budget): Read all findings, identify patterns, gaps, contradictions, implications. Write `synthesis_notes.md`.
4. **Report Generation** (~15% of stage budget): Write `RESEARCH_REPORT.md` with executive summary, key findings, market/tech/competitive analysis, trends, gaps, methodology, and sources.
5. **Validation** (~5% of stage budget): Check coverage of research questions, citation completeness, source diversity. Write `validation_report.md`.

**Output propagation to Stage 3:**
- Pass `research_insights`, `validated_assumptions`, `competitive_landscape`, `tech_enablers`, `market_sizing`

### Stage 3: MVP Generation (burn-my-tokens-MVP)

**Implementation reference**: See `burn-my-tokens` (MVP) skill for full project generation workflow.
**X-specific behavior**:
- Budget: 30% of total (or user-adjusted)
- Inputs: Best idea + concept brief + research insights
- Outputs: L3 MVP project with PRD, source code, tests, verification
- Next stage input: Project directory path
- Output directory: `burn-my-tokens-X_output/<pipeline_name>/iteration_N/stage_3_mvp/`

**Subagent workflow (embedded):**
1. **Web Research** (~15% of stage budget): Research technical solutions and feasibility specific to the idea. Write `research.md`.
2. **Write PRD** (~15% of stage budget): Write `PRD.md` with Problem Statement, Solution, User Stories, Implementation Decisions, Testing Decisions, Out of Scope, Acceptance Criteria.
3. **Core Development** (~45% of stage budget): Implement MVP code according to PRD. Follow `.claude/rules/python-coding-standards.md`. Include `requirements.txt` and runnable `main.py`. Write to `src/`.
4. **Unit Tests** (~10% of stage budget): pytest covering core functionality. Tests must pass. Write to `tests/`.
5. **Verification Run** (~10% of stage budget): Actually run `main.py` or core entry point. If run fails, auto-degrade to L2 (keep code skeleton and PRD, document failure in `PROGRESS.md`).
6. **Write PROGRESS.md** (~5% of stage budget): Record milestones, known issues, how to run.

**Output propagation to Stage 4:**
- Pass `project_dir`, `tech_stack`, `data_requirements`

### Stage 4: Data Engineering (burn-my-tokens-data)

**Implementation reference**: See `burn-my-tokens-data` skill for full data engineering workflow.
**X-specific behavior**:
- Budget: 10% of total (or user-adjusted)
- Inputs: Project directory + data requirements from Stage 3
- Outputs: Discovered datasets, cleaned data, pipeline scripts, visualizations
- Next stage input: Dataset report + cleaned data paths
- Output directory: `burn-my-tokens-X_output/<pipeline_name>/iteration_N/stage_4_data/`

**Subagent workflow (embedded):**
1. **Data Discovery / Ingestion** (~20% of stage budget): Search for public datasets (Kaggle, UCI, data.gov, World Bank, FiveThirtyEight) relevant to the project domain. Download and profile. Write `data_profile.md`.
2. **Cleaning Strategy** (~15% of stage budget): Identify issues (missing values, outliers, duplicates, type mismatches). Prioritize by impact score. Write `cleaning_plan.md`.
3. **Execute Data Pipeline** (~50% of stage budget): Launch subagents per dataset (max 2 parallel). Each subagent applies cleaning, transformation, visualization, writes pipeline script, validates output. Write to `outputs/` and `src/pipeline_*.py`.
4. **Validation** (~10% of stage budget): Aggregate quality metrics, detect regressions, switch strategy if needed.
5. **Report** (~5% of stage budget): Write `DATA_BURN_REPORT.md`.

**Output propagation to Stage 5:**
- Pass `dataset_report`, `cleaned_data_paths`, `pipeline_scripts`

### Stage 5: Test Generation (burn-my-tokens-testgen)

**Implementation reference**: See `burn-my-tokens-testgen` skill for full test generation workflow.
**X-specific behavior**:
- Budget: 10% of total (or user-adjusted)
- Inputs: Project directory from Stage 3
- Outputs: Generated tests, coverage delta
- Next stage input: Coverage report + test files
- Output directory: `burn-my-tokens-X_output/<pipeline_name>/iteration_N/stage_5_testgen/`

**Subagent workflow (embedded):**
1. **Coverage Analysis** (~15% of stage budget): Run `pytest --cov` if available. Identify critical uncovered paths. Write `coverage_gap.md`.
2. **Test Strategy** (~10% of stage budget): Prioritize uncovered paths by risk tier. Select test types (unit, boundary, property-based, integration). Write `test_strategy.md`.
3. **Generate Tests** (~55% of stage budget): Launch subagents per module (max 2 parallel). Each subagent reads source, generates pytest tests with type hints and Google-style docstrings, mocks external dependencies, includes boundary and error path tests. Writes to `tests/test_<module>.py`.
4. **Run & Fix** (~15% of stage budget): Run generated tests. If fail: analyze, fix test (not source), re-run. Degrade after 2 fix attempts.
5. **Report** (~5% of stage budget): Write `TEST_BURN_REPORT.md` with before/after coverage, modules processed, known limitations.

**Output propagation to Stage 6:**
- Pass `coverage_report`, `test_files`, `current_coverage`

### Stage 6: Code Review (burn-my-tokens-review)

**Implementation reference**: See `burn-my-tokens-review` skill for full code review workflow.
**X-specific behavior**:
- Budget: 10% of total (or user-adjusted)
- Inputs: Project directory from Stage 3 + test files from Stage 5
- Outputs: Structured review reports, health scores, top issues
- Next stage input: Review report + health score + top issues
- Output directory: `burn-my-tokens-X_output/<pipeline_name>/iteration_N/stage_6_review/`

**Subagent workflow (embedded):**
1. **Code Analysis** (~20% of stage budget): Enumerate source files. Build complexity hotspots, coupling analysis, documentation gaps, type hint gaps, potential bugs, performance issues, security issues. Run radon/ruff/bandit if available. Write `analysis_report.md`.
2. **Issue Prioritization** (~10% of stage budget): Rank issues by severity matrix (likelihood x impact). Group into batches: Critical → High → Medium → Low. Write `review_plan.md`.
3. **Generate Reviews** (~50% of stage budget): Launch subagents per module (max 2 parallel). Each subagent reads source, performs deep review with exact line numbers, severity ratings, rationale, concrete fix suggestions with code examples. Calculates module health score. Writes `review_reports/<module>_review.md`.
4. **Consolidation** (~15% of stage budget): Merge all reviews into `MASTER_REVIEW.md`. Calculate overall code health score. Identify top 10 issues.
5. **Report** (~5% of stage budget): Write `REVIEW_BURN_REPORT.md`.

**Output propagation to Stage 7:**
- Pass `master_review_path`, `overall_health_score`, `top_issues`, `review_reports_dir`

### Stage 7: Refactoring (burn-my-tokens-refactor)

**Implementation reference**: See `burn-my-tokens-refactor` skill for full refactoring workflow.
**X-specific behavior**:
- Budget: 15% of total (or user-adjusted)
- Inputs: Project directory from Stage 3 + review reports from Stage 6
- Outputs: Refactored code, diff logs, smell reduction metrics
- Next stage input: Refactored project directory + smell metrics
- Output directory: `burn-my-tokens-X_output/<pipeline_name>/iteration_N/stage_7_refactor/`

**Subagent workflow (embedded):**
1. **Smell Detection** (~15% of stage budget): Run radon cc/mi, jscpd, pylint, ruff. Build smell report with severity scores. Write `smell_report.md`.
2. **Prioritization** (~10% of stage budget): Rank smells by impact x ease_of_fix. Group into batches: Quick Wins → Complexity Reduction → Duplication Elimination → Deep Cleanup. Write `refactor_plan.md`.
3. **Baseline Test Run** (~5% of stage budget): Run existing test suite. Record baseline. If tests fail, warn user.
4. **Execute Refactoring** (~50% of stage budget): Launch subagents per batch (max 2 parallel). Each subagent reads source, plans changes, applies safe refactors (extract function, rename, remove duplication, simplify conditionals, reorganize imports), runs tests, handles failures with rollback and conservative retry (max 3 attempts). Writes `refactor_logs/<module>.diff`.
5. **Validation Loop** (~15% of stage budget): Re-run full test suite. Re-run smell detection on refactored modules. Compare metrics. If smells_after < smells_before, continue. If stalled for 2 batches, switch strategy.
6. **Report** (~5% of stage budget): Write `REFACTOR_BURN_REPORT.md`.

**Output propagation to next iteration:**
- Pass `refactored_project_dir`, `smells_before`, `smells_after`, `maintainability_delta`

---

## Core Execution Loop (Pipeline Burn Loop)

The Main Agent's core execution logic is as follows. It must iterate as much as possible within a single response:

```
while (target_budget == "burn" or estimated_spent < target_budget * 0.95):
    # 1. Check for unfinished subagents within current stage
    if pending_agents:
        collect_results()
        update_stage_outputs()
    
    # 2. If current stage is complete, advance to next stage
    if current_stage_complete:
        write_stage_summary()
        current_stage += 1
        
        # If all 7 stages complete, check loop condition
        if current_stage > 7:
            write_iteration_summary()
            completed_iterations.append(current_iteration)
            
            # Loop check: should we start next iteration?
            if target_budget == "burn" or estimated_spent < target_budget * 0.95:
                current_iteration += 1
                current_stage = 1
                # Pass outputs from Stage 7 back to Stage 1 context
                carry_over_context()
                output_iteration_report()
            else:
                break  # Budget exhausted, stop
    
    # 3. If stage is skipped, advance immediately
    if current_stage in skipped_stages:
        current_stage += 1
        continue
    
    # 4. Launch next batch of subagents for current stage (max 2 parallel)
    stage_budget_remaining = stage_budgets[stage_name] - stage_spent[stage_name]
    batch = get_next_batch(current_stage, max_parallel=2)
    for task in batch:
        if target_budget != "burn" and estimated_spent + task.estimated_cost > target_budget * 0.95:
            break
        if stage_spent[stage_name] + task.estimated_cost > stage_budget_remaining:
            break
        launch_subagent(task)
        estimated_spent += task.estimated_cost
        stage_spent[stage_name] += task.estimated_cost
    
    # 5. MetaBot queue compatibility: save state and end response after each batch
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

## Loop Behavior

### Iteration Advancement

After Stage 7 completes successfully:
1. Write `iteration_N_summary.md` to `stage_summaries/`
2. Append iteration record to `completed_iterations` in `.burn_state.json`
3. Check if loop should continue:
   - **Burn mode**: Always continue (increment `current_iteration`, reset `current_stage` to 1)
   - **Tier mode**: Continue if `estimated_spent < target_budget * 0.95`
   - **Stop command received**: Break loop, set `status = "stopped"`
4. If continuing:
   - Increment `current_iteration`
   - Reset `current_stage` to 1
   - Carry over context from previous iteration:
     - Best idea from previous iteration becomes "seed idea" for Stage 1
     - Research insights inform deeper domain analysis
     - MVP project directory becomes base for next iteration's refinement
     - Data pipelines inform new data requirements
     - Coverage metrics set baseline for next testgen
     - Health score and top issues inform next review focus
     - Smell metrics set baseline for next refactor
   - Output iteration report:
     ```
     Pipeline Iteration N Complete!
     
     Iteration Summary:
     - Idea: <best idea>
     - Research: <key insight>
     - MVP: <project dir>
     - Data: <datasets processed>
     - TestGen: <coverage before> → <coverage after>
     - Review: <health score>
     - Refactor: <smells before> → <smells after>
     
     Starting Iteration N+1...
     ```

### Iteration Refinement Strategy

Each subsequent iteration should build on the previous:
- **Iteration 1**: Greenfield — generate new idea, build from scratch
- **Iteration 2+**: Refinement — deepen research, extend MVP features, enrich datasets, increase coverage, fix review issues, reduce smells further

If an iteration produces no meaningful improvement (all delta metrics flat or negative), X should:
1. Log the stagnation in `PIPELINE_REPORT.md`
2. Attempt a pivot: use WebSearch to find a new angle or adjacent domain
3. If 2 consecutive iterations show no improvement, prompt user: "Pipeline has stagnated for 2 iterations. Continue, pivot domain, or stop?" If the user chooses to stop, call `CronDelete` to remove the resume task before exiting.

---

## Burn Contract

The Main Agent must adhere to the following hard constraints and must not violate them for any reason:

### Constraint 1: Never Stop on Your Own
- After completing a stage (or subagent batch), immediately check budget and remaining stages/iterations
- If budget is not exhausted and stages remain, immediately begin the next batch
- Prohibited from outputting terminal phrases such as "that's all for now", "let's pause here", "continue next time"
- The only stop conditions are: all stages complete AND budget exhausted (or burn mode stopped), API returning 429/403, or user-triggered graceful stop
- "Ending the response between batches" is not a stop — it is a necessary window to respond to queued user instructions. `.burn_state.json` remains `status == "running"`, and `CronCreate` auto-resumes

### Constraint 2: Degrade on Error, Don't Terminate
- When a subagent fails within a stage, the Main Agent must attempt degradation following that stage's specific degradation rules
- For MVP stage: L3 → L2 → L1
- For Data stage: Full pipeline → Clean only → Profile only
- For TestGen stage: Property-based → Example-based → Simple happy path
- For Review stage: Full review → Summary review → Issue list only
- For Refactor stage: Full refactor → Smaller extractions → Renaming only → Skip
- Only after 2-3 consecutive degradation failures (per stage rules) is skipping the task and continuing allowed
- When skipping, briefly log the reason without extended discussion

### Constraint 3: Time-Based Resume
- If the response is perceived to be approaching the time limit (executed >4 minutes), the Main Agent must:
  1. Write current progress to `burn-my-tokens-X_output/<pipeline_name>/.burn_state.json`
  2. Ensure the CronCreate resume task is started
  3. Output: "Time limit triggered resume; scheduled task will auto-restart pipeline burning in 3 minutes"
- Upon resume, read `.burn_state.json` to restore context

### Constraint 4: Progress Reports Must Not Block Execution
- Progress reports must be embedded in the execution flow in the most concise format possible
- Prohibited from interrupting the execution chain for a "complete report"
- Detailed reports only when the user queries progress

### Constraint 5: Budget Visualization Obligation
- After each stage (or batch) completes, update and display `[Token Tracker]`
- This serves as both a report and a self-restraint trigger for the Main Agent

### Constraint 6: Output Propagation
- Relevant outputs from Stage N must be passed as inputs to Stage N+1
- The Main Agent must maintain `stage_outputs` in `.burn_state.json` and reference it when launching subagents
- Never start a stage without the necessary context from previous stages (unless the stage is skipped)

### Constraint 7: Budget Enforcement
- Never exceed a stage's allocated budget (hard cap)
- Track `stage_spent` per stage in `.burn_state.json`
- If a stage approaches its cap (95%), pause and report: "Stage X is approaching its budget cap (XXk / XXk). Continue with last task or advance to next stage?"
- In burn mode, each stage still has the default 1M cap unless overridden

### Constraint 8: Stage Sequencing
- Never skip a stage unless explicitly requested by user via `skipped_stages`
- Stages must execute in order: 1 → 2 → 3 → 4 → 5 → 6 → 7
- If a stage is skipped, its budget is redistributed; its outputs are marked as "skipped" in `stage_outputs`

### Constraint 9: State Preservation
- `.burn_state.json` is the single source of truth for pipeline state
- Update it after every significant event: stage start, batch complete, subagent report, iteration end
- Include `last_heartbeat` timestamp updated on every save
- Preserve state across sessions via CronCreate resume

### Constraint 10: Subagent Timeout
- If a subagent produces no output within reasonable time, treat as failure and degrade.

---

## Token Tracking Rules

**Important**: Claude Code does not provide a remaining token API; all tracking is estimated.

- Each WebSearch ≈ 500-1500 tokens
- Each WebFetch ≈ 500-1500 tokens
- Each file read/write ≈ proportional to file size
- Each Agent call ≈ 2000-5000 tokens overhead
- Stage 1 (idea) ≈ 3000-8000 tokens per angle
- Stage 2 (research) ≈ 2000-6000 tokens per subtopic
- Stage 3 (mvp) ≈ 5000-15000 tokens per project
- Stage 4 (data) ≈ 3000-8000 tokens per dataset
- Stage 5 (testgen) ≈ 2000-6000 tokens per module
- Stage 6 (review) ≈ 3000-8000 tokens per module
- Stage 7 (refactor) ≈ 3000-8000 tokens per module
- Iteration summary/report generation ≈ 1000-3000 tokens

The Main Agent maintains a running estimate in the conversation, formatted as:
```
[Token Tracker] Pipeline Iteration X/Y | Stage: <stage_name> (N/7) | Estimated consumed: XXk / Target: XXk | Stage spent: XXk / Stage cap: XXk | Total spent: XXk
```

**Tier mode 95% threshold**:
When estimated consumption reaches 95% of the target tier, the Main Agent proactively pauses and reports:
> "Estimated consumption is approaching the target budget (XXk / XXk). Iteration: X, Stage: Y. Continue with the last stage or stop?"
If the user chooses to stop, call `CronDelete` to remove the resume task before exiting.

**Burn-until-stopped mode**:
Do not stop proactively. When a subsequent Claude API call returns any blocking error (429/403/500/timeout), catch the exception, call `CronDelete` to remove the resume task, output the final pipeline summary report, and gracefully exit.

---

## Resume Mechanism (Pure Text Version)

This skill **does not depend on MetaBot and requires no command-line operations**. Resume is implemented via Claude Code's built-in `CronCreate`, fully transparent to the user.

### State File

The Main Agent updates this at the start of each loop (on resume, if missing/corrupt, reconstruct from output dirs or start fresh):
`burn-my-tokens-X_output/<pipeline_name>/.burn_state.json`

Content:
```json
{
  "status": "running",
  "mode": "10M",
  "target_budget": 10000000,
  "estimated_spent": 0,
  "current_iteration": 1,
  "current_stage": 1,
  "stage_budgets": {
    "idea": 1500000,
    "research": 1000000,
    "mvp": 3000000,
    "data": 1000000,
    "testgen": 1000000,
    "review": 1000000,
    "refactor": 1500000
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
    "idea": {"top_ideas": [], "output_dir": "", "best_idea_name": "", "concept_brief": ""},
    "research": {"report_path": "", "key_insights": [], "validated_assumptions": []},
    "mvp": {"project_dir": "", "tech_stack": [], "data_requirements": []},
    "data": {"dataset_report": "", "cleaned_data_paths": [], "pipeline_scripts": []},
    "testgen": {"coverage_delta": 0, "test_files": [], "current_coverage": 0},
    "review": {"health_score": 0, "top_issues": [], "review_reports_dir": ""},
    "refactor": {"smells_before": 0, "smells_after": 0, "maintainability_delta": 0}
  },
  "completed_iterations": [],
  "skipped_stages": [],
  "domain": "",
  "pipeline_name": "",
  "last_heartbeat": "2026-05-12T14:30:00",
  "cron_job_id": null
}
```

### Auto-Resume Scheduled Task

**On first startup**, the Main Agent calls `CronCreate` to create a durable scheduled task:

- **cron**: `1-59/3 * * * *` (every 3 minutes, avoiding top of the hour)
- **durable**: `true` (persists across session restarts)
- **prompt**:
  ```
  Read burn-my-tokens-X_output/<pipeline_name>/.burn_state.json.
  If status == "running" and last_heartbeat is more than 10 minutes ago:
    Execute /burn-my-tokens-X resume
  Otherwise: ignore, do nothing.
  ```

Record the returned `job_id` in `.burn_state.json` under `cron_job_id`.

**After burning completes**, the Main Agent calls `CronDelete` to remove the scheduled task to avoid useless polling.

### User-Transparent Flow

```
User: /burn-my-tokens-X
Claude: [Select tier] [Confirm domain] [Confirm budget] [Confirm skipped stages] → Start pipeline
Claude: [Internal] CronCreate creates resume task
...
Claude: [Stage 1 complete] → [Stage 2 complete] → ... → [Stage 7 complete]
Claude: [Iteration 1 complete report]
Claude: [Loop check] → Starting Iteration 2...
...
[Session ends due to time limit]
[3 minutes later, cron auto-triggers]
Claude: [Auto-resumes pipeline from breakpoint]
...
[Budget exhausted or user stops]
Claude: [Pipeline complete report] [CronDelete removes resume task]
```

### Resume Parameters

When the scheduled task or user triggers `/burn-my-tokens-X resume`:
1. The Main Agent reads `.burn_state.json`
2. Restores `estimated_spent`, `current_iteration`, `current_stage`, `stage_budgets`, `stage_spent`, `stage_outputs`, `completed_iterations`, `skipped_stages`, `domain`, `pipeline_name`
3. Continues the Pipeline Burn Loop from the breakpoint
4. No need to re-ask tier, domain, budget, or skipped stages (unless the state file is missing)

---

## Output Structure

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

### PIPELINE_REPORT.md

After the pipeline completes (or is stopped), write a master report:

```markdown
# Pipeline Report: <pipeline_name>

## Metadata
- **Started**: <timestamp>
- **Completed**: <timestamp>
- **Mode**: <10k/100k/1M/10M/burn>
- **Target Budget**: <X> tokens
- **Estimated Spent**: <Y> tokens
- **Iterations Completed**: <N>
- **Stages Run**: <list>
- **Stages Skipped**: <list>

## Iteration Summaries

### Iteration 1
- **Idea**: <best idea>
- **Research**: <key insight>
- **MVP**: <project dir>
- **Data**: <datasets>
- **TestGen**: <coverage delta>
- **Review**: <health score>
- **Refactor**: <smell reduction>

### Iteration 2
...

## Overall Metrics
- **Total ideas generated**: <count>
- **Total research sources**: <count>
- **Total MVP projects**: <count>
- **Total datasets processed**: <count>
- **Total tests added**: <count>
- **Total issues found**: <count>
- **Total smells reduced**: <count>

## Key Deliverables
- <list of most important output files>

## Known Limitations
- <what could not be completed>

## Next Steps
- <recommended actions>
```

---

## Fallback Strategies

### Stage Failure Recovery

If any stage fails catastrophically (all subagents fail, no degradation possible):
1. Log the failure in `.burn_state.json` under `stage_outputs[<stage>].status = "failed"`
2. If the stage is critical (idea, mvp), the pipeline cannot proceed. Offer user: "Stage X failed. Retry stage, skip to next iteration, or stop?" If the user chooses to stop, call `CronDelete` to remove the resume task before exiting.
3. If the stage is non-critical (data, testgen, review, refactor), advance to next stage with a warning note
4. Never terminate the entire pipeline due to a single stage failure

### Budget Exhaustion Mid-Stage

If the total budget is exhausted while a stage is in progress:
1. Complete the current subagent batch (do not abort mid-task)
2. Save state
3. Output: "Total budget exhausted. Pipeline paused at Stage X. Current iteration progress saved."
4. Do NOT start a new stage or iteration

### Resume with Missing State File

If `.burn_state.json` is missing on resume:
1. Attempt to reconstruct state from output directory structure
2. If reconstruction is impossible, treat as fresh start but preserve existing output directories
3. Ask user: "Previous state file missing. Reconstruct from outputs or start fresh?"

### Empty Domain (Autonomous Discovery)

If user provides no domain:
1. Use WebSearch to find 2-3 trending/underserved niches with recent activity
2. Present options to user: "No domain specified. Top opportunities: (1) ..., (2) ..., (3) ... Which should we explore?"
3. If user does not respond (burn mode with no domain), pick the highest-opportunity niche and proceed

### Circular Context Overflow

If `stage_outputs` grows too large across iterations:
1. Summarize older iteration outputs into compact form (keep only key metrics, not full text)
2. Archive full outputs to `stage_summaries/iteration_N_full.json`
3. Keep only the most recent 2 iterations' full context in `.burn_state.json`

---

## Integration with Individual Skills

X is the orchestrator. The 7 individual skills are the implementation references:

| Stage | Skill | Key Output | Propagates To |
|-------|-------|-----------|---------------|
| 1 | burn-my-tokens-idea | TOP_PICKS.md | Stage 2 (concept brief) |
| 2 | burn-my-tokens-research | RESEARCH_REPORT.md | Stage 3 (validated assumptions) |
| 3 | burn-my-tokens-MVP | src/, PRD.md | Stage 4 (data requirements), Stage 5 (project dir), Stage 6 (project dir), Stage 7 (project dir) |
| 4 | burn-my-tokens-data | DATA_BURN_REPORT.md | Stage 5 (dataset paths) |
| 5 | burn-my-tokens-testgen | TEST_BURN_REPORT.md | Stage 6 (coverage baseline) |
| 6 | burn-my-tokens-review | MASTER_REVIEW.md | Stage 7 (top issues, health score) |
| 7 | burn-my-tokens-refactor | REFACTOR_BURN_REPORT.md | Next Iteration Stage 1 (refined project) |

They share:
- The same burn contract philosophy (never stop, degrade on error, time-based resume)
- The same state file pattern (`.burn_state.json`)
- The same CronCreate resume mechanism
- The same output directory convention (under `burn-my-tokens-X_output/`)

They differ in:
- **Scope**: X orchestrates all 7 stages; individual skills focus on one stage
- **Context**: X maintains cross-stage state; individual skills are stage-local
- **Looping**: X loops iterations; individual skills loop within their domain

### Recommended Usage

**Full pipeline** (most common):
```
/burn-my-tokens-X
→ Select 10M
→ Domain: "AI tools for veterinary clinics"
→ Budget: 10000000
→ Stages: all
→ Pipeline runs: idea → research → MVP → data → testgen → review → refactor
```

**Quick pipeline** (time-constrained):
```
/burn-my-tokens-X
→ Select 1M
→ Domain: "autonomous vehicle simulation"
→ Budget: 1000000
→ Stages: all
→ Pipeline runs all 7 stages with reduced depth
```

**Burn mode** (unattended):
```
/burn-my-tokens-X
→ Select burn
→ Domain: "sustainable energy analytics"
→ Budget: burn
→ Stages: all
→ Pipeline loops indefinitely until stopped or API limit reached
```

---

## Quality Assurance Checklist

Before declaring a pipeline iteration complete, verify:
- [ ] `.burn_state.json` is up to date with `current_stage`, `current_iteration`, `estimated_spent`, `stage_outputs`
- [ ] Every completed stage has its burn report written to the stage output directory
- [ ] Stage outputs are properly propagated to the next stage (where applicable)
- [ ] No stage exceeded its allocated budget (check `stage_spent` vs `stage_budgets`)
- [ ] `PIPELINE_REPORT.md` is updated after each iteration
- [ ] Token tracker has been output after each batch
- [ ] Resume cron job is active while `status == "running"`
- [ ] Resume cron job is deleted when pipeline completes or stops
- [ ] If in burn mode, iteration report shows meaningful deltas (not flat iterations)
- [ ] All source code follows `.claude/rules/python-coding-standards.md`
- [ ] No original data files were overwritten (data stage safety)
- [ ] No source files were modified by review stage (review stage read-only safety)
- [ ] Refactor stage only modified files after baseline test run and with rollback capability
