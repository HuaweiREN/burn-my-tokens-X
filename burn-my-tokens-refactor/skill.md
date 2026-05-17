---
name: burn-my-tokens-refactor
description: When a user's coding plan quota is about to reset, automatically refactor, simplify, and clean up an existing codebase by detecting code smells, prioritizing them, and executing safe refactors with test validation. Burns remaining tokens into measurable code quality improvements instead of letting them expire. Supports unattended self-iterative burning with pure text interaction. Graceful stop via stop command.
version: 1.1.0
---

## Trigger Conditions

This skill activates when the user inputs any of the following:
- `/burn-my-tokens-refactor` — Start the refactor burn flow
- `/burn-my-tokens-refactor stop` / `停止重构` / `stop refactor` / `stop burn` — Gracefully end the burn
- Expresses anxiety such as "code is messy / tech debt / need cleanup / tokens about to expire / refactor my code"

## Graceful Stop

When the user inputs a stop command, the Main Agent must:
1. Set `status` to `"stopped"` in `.burn_state.json`
2. Output stop confirmation: `Refactor burn gracefully stopped. Completed X modules, reduced Y smells, consumed approximately XXk tokens.`
3. Call `CronDelete` to remove the resume cron job (if it exists)
4. Subsequent `CronCreate` triggers reading `status == "stopped"` will be ignored

### MetaBot Queue Compatibility Strategy

Bot frameworks like MetaBot queue new user instructions, so a stop command sent during the current response cannot immediately interrupt execution. To ensure graceful stop works reliably:

**After each refactor batch (or each module if in 10k mode) completes, the Main Agent must:**
1. Save current progress to `.burn_state.json` (update `estimated_spent`, `completed_modules`, `smells_after`)
2. Output `[Token Tracker]` and a concise progress report
3. **Actively end the current response**

This allows queued stop commands to be processed in the next turn. If the user sends no new instruction, burning auto-resumes via:
- The `CronCreate` scheduled task triggering `/burn-my-tokens-refactor resume` after 3 minutes
- Or any message from the user in chat triggering resume

**This behavior does not violate the "Never Stop on Your Own" contract** — ending the response is to respond to potential user instructions, not to terminate the burn flow. `status` in `.burn_state.json` remains `"running"`, and `CronCreate` handles recovery.

## Startup Flow (Main Agent)

### Step 0: Tier Selection & Context Gathering

Present tier options to the user and wait for selection:
- `10k` — Quick smell scan + 1-2 file simplifications
- `100k` — Standard refactor push (~1 module deeply cleaned)
- `1M` — Deep refactor burn (~3-4 modules or full codebase)
- `10M` — Massive refactor burn (~8-12 modules or enterprise-scale codebase)
- `burn` — Burn until no more improvements found or request fails

**CRITICAL — Backup Warning:**
Before proceeding, the Main Agent must output:
```
⚠️  IMPORTANT: burn-my-tokens-refactor will MODIFY your source code.
   Please ensure you have a backup of your project (e.g., git commit or copy).
   Continue? (yes / no)
```
If the user answers "no" or does not confirm, stop immediately and remind them to back up before retrying.

**Then ask:**
```
Which codebase directory should be refactored?
(Leave empty to use current working directory)

Examples:
- POC_Cam_Viewer
- src/
- ../another_workspace/some_project
```

**Also ask:**
```
Focus area? (complexity / duplication / naming / all)
(Leave empty for "all")
```

**Auto-detect and confirm:**

The Main Agent uses Bash/Glob to attempt locating the target directory structure:
- Look for `src/` subdirectory
- Look for top-level `.py` files
- Look for `tests/` or `test_*.py` files
- Look for `pytest.ini`, `setup.py`, `pyproject.toml` (to confirm Python project)

Present detected structure to the user:
```
Detected project structure:
  Source: src/ (X modules, Y files)
  Tests: tests/ (Z test files)
  Language: Python
Correct? (Press Enter to confirm, or input the correct source/test directories)
```

If no Python project is detected, inform the user: "No Python project detected in the target directory. burn-my-tokens-refactor currently supports Python only. Please specify a Python project directory or cancel."

Record the user-confirmed paths in `.burn_state.json` under `source_dir` and `test_dir` fields for future session reuse.

### Step 1: Smell Detection

**Prerequisite check**: Ensure tools are available. Try to run:
- `python -m radon cc --min B <source_dir>`
- `python -m radon mi <source_dir>`
- `python -m radon raw <source_dir>`
- `jscpd <source_dir> --reporters console,json --output .jscpd.json` (if available)
- `pylint --disable=all --enable=R,C,W <source_dir>` (fallback)
- `ruff check <source_dir>` (fallback / auto-fix)

If tools are missing, attempt `pip install radon jscpd pylint ruff` in the project environment. If installation fails, fall back to AST-based manual smell detection using Python's built-in `ast` module.

**Run detection** and produce `smell_report.md` in the output directory (`burn-my-tokens_output/refactor_<project_name>/`):

```markdown
# Smell Report for <project_name>

Generated: <timestamp>
Focus: <complexity|duplication|naming|all>

## Summary
- Total files scanned: X
- Total functions/classes scanned: Y
- Critical smells: A
- High smells: B
- Medium smells: C
- Low smells: D
- Maintainability Index < 20: E files

## Top Smelliest Files (by combined severity score)
1. `src/module_a.py` — Score: 85 (CC max: 24, MI: 15, Duplication: 2 blocks)
2. `src/module_b.py` — Score: 62 (CC max: 18, MI: 28, Duplication: 1 block)
...

## Detailed Smells

### src/module_a.py
#### fetch_with_retry (lines 45-120)
- **Type**: high_complexity
- **Severity**: critical
- **Metric**: cyclomatic_complexity = 24 (threshold: 10)
- **Suggestion**: Extract retry logic, extract error handling, split into smaller functions

#### process_response (lines 150-280)
- **Type**: high_complexity
- **Severity**: high
- **Metric**: cyclomatic_complexity = 16 (threshold: 10)
- **Suggestion**: Extract validation logic, use early returns

### Duplication Blocks
#### Block A (jscpd similarity: 85%)
- File 1: `src/module_a.py` lines 45-80
- File 2: `src/module_b.py` lines 30-65
- **Suggestion**: Extract shared utility function
```

**Smell scoring formula** (simplified):
```
severity_score = (
    (cc_over_threshold * 3) +
    (duplication_blocks * 5) +
    (mi_under_20 * 4) +
    (function_length_over_100 * 2) +
    (naming_violations * 1)
)
```

### Step 2: Prioritization

Rank smells by `impact × ease_of_fix` and group into refactor batches.

**Impact factors:**
- Critical severity = 4x
- High severity = 3x
- Medium severity = 2x
- Low severity = 1x

**Ease of fix factors:**
- Extract function (local, no API change) = 3x
- Rename variable/function = 3x
- Remove duplication (may need new module) = 2x
- Simplify conditionals = 2x
- Reorganize imports = 3x
- Structural change (class hierarchy) = 1x

Produce `refactor_plan.md`:

```markdown
# Refactor Plan for <project_name>

## Batch 1: Quick Wins (Low Risk, High Impact)
- `src/module_c.py`: Reorganize imports, rename ambiguous variables
- `src/module_d.py`: Extract 2 helper functions from `main()`
Estimated smell reduction: 6
Risk: low

## Batch 2: Complexity Reduction
- `src/module_a.py`: Extract retry logic and error handling from `fetch_with_retry`
- `src/module_b.py`: Split `process_data` into pipeline stages
Estimated smell reduction: 10
Risk: medium

## Batch 3: Duplication Elimination
- Extract shared utility from `src/module_a.py` and `src/module_b.py`
- Create `src/utils/shared_helpers.py`
Estimated smell reduction: 8
Risk: medium

## Batch 4: Deep Cleanup (if budget remains)
- `src/module_e.py`: Simplify nested conditionals, introduce guard clauses
- `src/module_f.py`: Reduce class responsibilities (extract class)
Estimated smell reduction: 12
Risk: high
```

### Step 3: Baseline Test Run

**Before any refactoring**, run the existing test suite:
```bash
cd <project_dir> && python -m pytest --tb=short -q
```

Record the result in `.burn_state.json`:
```json
{
  "baseline_tests": {
    "command": "python -m pytest --tb=short -q",
    "passed": true,
    "tests_count": 42,
    "failures": 0,
    "timestamp": "2026-05-11T14:30:00"
  }
}
```

If tests fail at baseline, warn the user: "Baseline tests are failing (X failures). Refactoring will proceed but broken tests may mask regressions. Continue?"

If no tests are found, warn the user: "No test suite detected. Refactoring will proceed with extra caution. Consider adding tests first for safer refactoring."

### Step 4: Execute Refactoring (Parallel Subagents)

Launch subagents per batch (max 2 parallel).

**Scheduling strategy:**
- First batch: launch up to 2 modules in parallel
- After completion, evaluate remaining budget and smell reduction before next batch
- In `10k` mode: process 1 file at a time, no parallelism

Each subagent's workspace: `burn-my-tokens_output/refactor_<project_name>/refactor_logs/`

### Step 5: Monitor & Report

Subagents report progress after each phase (read → plan → apply → test → diff).
The Main Agent aggregates and reports to the user:
```
Refactor burn progress update:

✅ src/module_a.py: Extracted 3 functions, reduced CC from 24 to 8, tests pass
⏳ src/module_b.py: In testing phase (conservative retry 1/3)

Smells remaining: 18 / 25
Remaining budget: approx. XXk / XXk
```

The user can input "progress" or "status" at any time for the Main Agent to output the current state.

### Step 6: Validation Loop

After each batch completes:

1. **Run full test suite**: `python -m pytest --tb=short -q`
2. **Re-run smell detection** on refactored modules
3. **Compare metrics**:
   - If `smells_after < smells_before`: continue to next batch
   - If `smells_after == smells_before` for 2 consecutive batches: switch strategy (e.g., from local extraction to structural change)
   - If all batches complete and smells reduced: generate final report

### Step 7: Loop Check

**Tier modes (10k/100k/1M/10M):**
- The Main Agent estimates cumulative consumption. If it reaches 95% of the tier, pause and output a summary report.
- If budget remains and smells remain, return to Step 2 to generate next batch plan.

**Burn-until-stopped mode:**
- Continue processing batches until no smells remain or no further reduction is possible
- When a subsequent request fails due to any blocking error, catch the error, call `CronDelete` to remove the resume task, output the final summary report, and gracefully exit
- If all modules are clean, call `CronDelete` to remove the resume task, then output: "All detectable smells resolved. Burn complete."

### Step 8: Final Report

Output:
```
Refactor burn complete!

Project: <project_name>
Directory: <source_dir>

Smells Before: X (Critical: A, High: B, Medium: C, Low: D)
Smells After: Y (Critical: E, High: F, Medium: G, Low: H)
Reduction: Z%

Files Refactored: N
Lines Changed: +AAA / -BBB
Maintainability Index Delta: +XX points average

Test Status: PASS (42/42)

Refactor Logs:
- burn-my-tokens_output/refactor_<project_name>/refactor_logs/module_a.diff
- burn-my-tokens_output/refactor_<project_name>/refactor_logs/module_b.diff

Known Limitations:
- <module_x>: Could not reduce complexity below threshold due to external API constraints
- <module_y>: 3 rollback attempts exceeded; skipped to preserve test safety

See refactor_plan.md and smell_report.md for full details.
```

---

## Burn Contract

The Main Agent must adhere to the following hard constraints and must not violate them for any reason:

### Constraint 1: Never Stop on Your Own
- After completing a batch, immediately check remaining smells and budget
- If smells remain and budget is not exhausted, immediately begin the next batch
- Prohibited from outputting terminal phrases such as "that's all for now", "let's pause here", "continue next time"
- The only stop conditions are: all smells resolved, reaching the 95% tier threshold, API returning 429/403, or user-triggered graceful stop
- "Ending the response between batches" is not a stop — it is a necessary window to respond to queued user instructions. `.burn_state.json` remains `status == "running"`, and `CronCreate` auto-resumes

### Constraint 2: Degrade on Error, Don't Terminate
- When a subagent's refactor breaks tests, the Main Agent must attempt a more conservative approach, not terminate the entire flow
- Conservative retry sequence:
  1. First failure: Rollback, try extracting smaller functions only
  2. Second failure: Rollback, try renaming and import cleanup only
  3. Third failure: Rollback, skip module, log reason, move to next
- Only after 3 consecutive conservative failures on the same module is skipping allowed
- When skipping a module, briefly log the reason without extended discussion

### Constraint 3: Safety First
- Never apply refactoring without first recording the baseline test state
- Always run tests before declaring a module "done"
- If tests were passing at baseline, they must pass after refactor
- If no tests exist, apply only the most conservative refactorings (renaming, import cleanup, dead code removal)

### Constraint 4: Time-Based Resume
- If the response is perceived to be approaching the time limit (executed >4 minutes), the Main Agent must:
  1. Write current progress to `burn-my-tokens_output/refactor_<project_name>/.burn_state.json`
  2. Ensure the CronCreate resume task is started
  3. Output: "Time limit triggered resume; scheduled task will auto-restart refactoring in 3 minutes"
- Upon resume, read `.burn_state.json` to restore context

### Constraint 5: Progress Reports Must Not Block Execution
- Progress reports must be embedded in the execution flow in the most concise format possible
- Prohibited from interrupting the execution chain for a "complete report"
- Detailed reports only when the user queries progress

### Constraint 6: Budget Visualization Obligation
- After each batch completes, update and display `[Token Tracker]`
- This serves as both a report and a self-restraint trigger for the Main Agent

### Constraint 7: Subagent Timeout
- If a subagent produces no output within reasonable time, treat as failure and degrade.

---

## Core Execution Loop (Refactor Burn Loop)

The Main Agent's core execution logic is as follows. It must iterate as much as possible within a single response:

```
while estimated_spent < target_budget * 0.95 and smells_remaining > 0:
    # 1. Check for unfinished subagents
    if pending_agents:
        collect_results()
    
    # 2. If no batches planned, generate from smell report
    if not pending_batches:
        pending_batches = generate_batches_from_smell_report()
    
    # 3. Launch next batch of subagents (max 2 in parallel, 3-4 in 10M mode, 1 in 10k mode)
    batch = pending_batches.pop_next(2)
    for module in batch:
        if estimated_spent + module.estimated_cost > target_budget * 0.95:
            break
        launch_subagent(module)
        estimated_spent += module.estimated_cost
    
    # 4. Validate batch results
    run_full_test_suite()
    rerun_smell_detection_on_batch()
    
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

## Subagent Refactor Workflow

Each subagent must strictly follow these 6 phases:

### Phase 1: Read & Scope (~10% budget)
- Read the target module source code
- Use AST extraction to scope context if the file is large (>200 lines)
- Identify the specific smells to address from the refactor plan
- Output: internal notes on target functions/lines

### Phase 2: Plan Changes (~15% budget)
- Generate a specific, concrete refactoring plan (not vague advice)
- For each change, specify:
  - Exact function name and line range
  - What to extract / rename / simplify / remove
  - Expected impact on cyclomatic complexity or duplication
- Output: `refactor_logs/<module>_plan.md`

### Phase 3: Apply Changes (~40% budget)
- Apply the planned changes using Edit/Write tools
- Changes may include:
  - Extract functions/methods
  - Rename variables/functions/classes for clarity
  - Remove duplicated code (extract to shared utility)
  - Simplify nested conditionals (guard clauses, early returns)
  - Reorganize imports
  - Remove dead code
- Follow `.claude/rules/python-coding-standards.md` if it exists
- Output: modified source file

### Phase 4: Run Tests (~20% budget)
- Run the module's relevant tests: `python -m pytest <test_dir> -k <module_name> --tb=short -q`
- If no module-specific tests, run full suite
- Record result

### Phase 5: Handle Failures
- **If tests pass**: Proceed to Phase 6
- **If tests fail**:
  1. Rollback all changes to the module (restore from git or original read)
  2. Analyze failure reason
  3. Generate a more conservative plan (fewer changes, smaller scope)
  4. Retry from Phase 3
  5. If 3 rollbacks occur, abort module and report: `SKIP: <module> — 3 conservative attempts failed, preserving original state`

### Phase 6: Write Diff Log (~15% budget)
- Generate a before/after diff of the module
- Write to `refactor_logs/<module>.diff`
- Include summary comment at top:
```diff
# Refactor: src/module_a.py
# Changes: Extracted 3 functions, renamed 5 variables, simplified 2 conditionals
# CC before: 24 → CC after: 8
# MI before: 15 → MI after: 42
# Tests: PASS
# Rollbacks: 0
```

---

## Project Specifications

- **Output directory**: `burn-my-tokens_output/refactor_<project_name>/`
- **Refactor logs**: `burn-my-tokens_output/refactor_<project_name>/refactor_logs/`
- **State file**: `burn-my-tokens_output/refactor_<project_name>/.burn_state.json`
- **Coding standards**: Strictly follow `.claude/rules/python-coding-standards.md` if it exists
- **Dependencies**: Python 3.10+, radon, jscpd (optional), pylint, ruff, pytest

---

## Smell Detection Reference

### Cyclomatic Complexity Thresholds (Radon)
| Score | Rank | Action |
|-------|------|--------|
| 1-5 | A (low) | No action |
| 6-10 | B (medium) | Monitor |
| 11-20 | C (high) | Refactor |
| 21-50 | D (very high) | Prioritize |
| 50+ | E (extreme) | Critical |

### Maintainability Index Thresholds (Radon)
| Score | Meaning |
|-------|---------|
| 0-9 | Extremely difficult to maintain |
| 10-19 | Difficult to maintain |
| 20-39 | Moderate |
| 40-69 | Fairly easy to maintain |
| 70-100 | Easy to maintain |

### Duplication Thresholds (jscpd)
| Similarity | Action |
|------------|--------|
| >90% | Critical — extract immediately |
| 70-90% | High — extract if in same domain |
| 50-70% | Medium — review for abstraction opportunity |

---

## Token Tracking Rules

**Important**: Claude Code does not provide a remaining token API; all tracking is estimated.

- Each WebSearch ≈ 500-1500 tokens
- Each file read/write ≈ proportional to file size
- Each Agent call ≈ 2000-5000 tokens overhead
- Each radon/jscpd invocation ≈ 500-1000 tokens equivalent
- Each test run ≈ 1000-3000 tokens equivalent

The Main Agent maintains a running estimate in the conversation, formatted as:
```
[Token Tracker] Estimated consumed: XXk / Target: XXk | Smells: Y remaining / Z initial
```

**Tier mode 95% threshold**:
When estimated consumption reaches 95% of the target tier, the Main Agent proactively pauses and reports:
> "Estimated consumption is approaching the target budget (XXk / XXk). Continue with the last batch or stop?"
If the user chooses to stop, call `CronDelete` to remove the resume task before exiting.

**Burn-until-stopped mode**:
Do not stop proactively. When a subsequent Claude API call returns any blocking error (429/403/500/timeout), catch the exception, call `CronDelete` to remove the resume task, output the final report, and gracefully exit.

---

## Resume Mechanism (Pure Text Version)

This skill **does not depend on MetaBot and requires no command-line operations**. Resume is implemented via Claude Code's built-in `CronCreate`, fully transparent to the user.

### State File

The Main Agent updates this at the start of each loop (on resume, if missing/corrupt, reconstruct from output dirs or start fresh):
`burn-my-tokens_output/refactor_<project_name>/.burn_state.json`

Content:
```json
{
  "status": "running",
  "mode": "1M",
  "target_budget": 1000000,
  "estimated_spent": 3500,
  "source_dir": "POC_Cam_Viewer/src",
  "test_dir": "POC_Cam_Viewer/tests",
  "focus": "all",
  "smells_before": 25,
  "smells_after": 18,
  "baseline_tests": {
    "command": "python -m pytest --tb=short -q",
    "passed": true,
    "tests_count": 42,
    "failures": 0,
    "timestamp": "2026-05-11T14:30:00"
  },
  "completed_modules": ["src/module_c.py", "src/module_d.py"],
  "pending_modules": ["src/module_a.py", "src/module_b.py"],
  "skipped_modules": [],
  "last_heartbeat": "2026-05-11T14:30:00",
  "cron_job_id": "burn-refactor-resume-xxx"
}
```

### Auto-Resume Scheduled Task

**On first startup**, the Main Agent calls `CronCreate` to create a durable scheduled task:

- **cron**: `1-59/3 * * * *` (every 3 minutes, avoiding top of the hour)
- **durable**: `true` (persists across session restarts)
- **prompt**:
  ```
  Read burn-my-tokens_output/refactor_<project_name>/.burn_state.json.
  If status == "running" and last_heartbeat is more than 10 minutes ago:
    Execute /burn-my-tokens-refactor resume
  Otherwise: ignore, do nothing.
  ```

Record the returned `job_id` in `.burn_state.json` under the `cron_job_id` field.

**After burning completes**, the Main Agent calls `CronDelete` to remove the scheduled task to avoid useless polling.

### User-Transparent Flow

```
User: /burn-my-tokens-refactor
Claude: [Select tier] [Confirm directory] [Confirm focus] → Start refactor burn
Claude: [Internal] CronCreate creates resume task
...
[Session ends due to time limit]
[3 minutes later, cron auto-triggers]
Claude: [Auto-resumes refactoring from breakpoint]
...
[Budget exhausted or all smells resolved]
Claude: [Refactor burn complete report] [CronDelete removes resume task]
```

### Resume Parameters

When the scheduled task or user triggers `/burn-my-tokens-refactor resume`:
1. The Main Agent reads `.burn_state.json`
2. Restores `estimated_spent`, `completed_modules`, `pending_modules`, `source_dir`, `test_dir`, `focus`, `baseline_tests`
3. Re-runs smell detection on pending modules to refresh metrics
4. Continues the Refactor Burn Loop from the breakpoint
5. No need to re-ask tier, directory, or focus (unless the state file is missing)

---

## Fallback Strategies

### No Static Analysis Tools Available
If radon/jscpd/pylint cannot be installed:
1. Use Python's built-in `ast` module to compute:
   - Function line counts (flag >100 lines)
   - Nested `if` depth (flag >4 levels)
   - Number of return statements (flag >6)
   - Import organization (detect unused imports via `ast`)
2. Use simple text comparison for near-duplicate blocks (>80% line similarity)
3. Proceed with manual smell detection; note limitations in smell_report.md

### No Tests Available
If no test suite is found:
1. Restrict refactoring to:
   - Import reorganization
   - Variable/function renaming
   - Dead code removal (unused functions/classes identified by ast)
   - Comment/docstring improvements
2. Do NOT extract functions or change control flow without tests
3. Flag in report: "Limited refactoring due to missing tests"

### Large Files (>500 lines)
If a single file exceeds 500 lines:
1. Scope the subagent to specific functions within the file (AST extraction)
2. Process the file across multiple batches
3. Consider recommending file splitting in refactor_plan.md

---

## Integration with burn-my-tokens

This skill is a sibling to `burn-my-tokens` (new project generation). They share:
- The same burn contract philosophy (never stop, degrade on error, time-based resume)
- The same state file pattern (`.burn_state.json`)
- The same CronCreate resume mechanism
- The same output directory convention (`burn-my-tokens_output/`)

They differ in:
- **Goal**: Improve existing code vs generate new projects
- **Validation**: Test-driven safety vs run verification
- **Output**: Diff logs + smell reports vs PRDs + source code

Both skills can be used in sequence:
1. `/burn-my-tokens` to generate a new MVP
2. `/burn-my-tokens-refactor` to clean it up

---

## Pipeline Integration (burn-my-tokens-X Entry Point)

This skill can be invoked as a stage in the `burn-my-tokens-X` pipeline orchestrator.

### Stage Position
- **Stage**: 7 of 7 (Code Refactoring)
- **Preceded by**: `burn-my-tokens-review` (Stage 6)
- **Followed by**: `burn-my-tokens-idea` (Stage 1, next iteration)

### Parameters from X
When invoked by `burn-my-tokens-X`, the Main Agent receives:
- `allocated_budget`: Token budget allocated for this stage
- `source_dir`: Path to MVP source code from Stage 3
- `review_outputs`: Paths to review reports from Stage 6
- `pipeline_mode`: "sequential" (default) or "parallel"
- `previous_outputs`: Outputs from Stage 6 (review)

### Behavior in Pipeline Mode
1. Skip tier selection — use allocated budget directly
2. Skip project structure detection — use `source_dir` from X
3. Use `review_outputs` to prioritize smells (if available)
4. Adjust scope based on `allocated_budget`:
   - < 5k: 1-2 files, conservative refactoring only
   - 5k-20k: 1 module deeply cleaned
   - 20k-80k: 3-4 modules or full codebase
   - 80k+: 8-12 modules or enterprise-scale
5. Write outputs to shared pipeline directory
6. Return structured summary to X:
   - `files_refactored`: List of modified files
   - `smells_before`: Initial smell count
   - `smells_after`: Final smell count
   - `tests_passing`: Boolean
   - `estimated_spent`: Actual tokens consumed
   - `output_paths`: Generated diff logs and reports

### Budget Allocation Guidelines for X
| Pipeline Budget | Suggested Refactor Stage Allocation | Notes |
|-----------------|------------------------------------|-------|
| 500k total | 50k-100k | Conservative cleanup only |
| 1M total | 120k-200k | Standard refactor push |
| 5M total | 600k-1M | Deep refactor across codebase |
| 10M+ total | 1.2M-2M | Enterprise-scale refactoring |
