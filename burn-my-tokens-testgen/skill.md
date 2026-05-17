---
name: burn-my-tokens-testgen
description: When a user's coding plan quota is about to reset, automatically analyze an existing codebase for test coverage gaps and generate comprehensive pytest tests, converting remaining tokens into durable test coverage. Supports unattended self-iterative burning with pure text interaction. Graceful stop via stop command.
version: 1.1.0
---

## Trigger Conditions

This skill activates when the user inputs any of the following:
- `/burn-my-tokens-testgen` — Start the test generation burn flow
- `/burn-my-tokens-testgen stop` / `停止测试燃烧` / `stop test burn` — Gracefully end the burn
- Expresses anxiety such as "low coverage / missing tests / need more tests / tokens about to expire / burn tokens on tests"

## Graceful Stop

When the user inputs a stop command, the Main Agent must:
1. Set `status` to `"stopped"` in `.burn_state.json`
2. Output stop confirmation: `Test burn gracefully stopped. Completed X modules, coverage moved from Y% to Z%, consumed approximately XXk tokens.`
3. Call `CronDelete` to remove the resume cron job (if it exists)
4. Subsequent `CronCreate` triggers reading `status == "stopped"` will be ignored

### MetaBot Queue Compatibility Strategy

Bot frameworks like MetaBot queue new user instructions, so a stop command sent during the current response cannot immediately interrupt execution. To ensure graceful stop works reliably:

**After each module batch completes, the Main Agent must:**
1. Save current progress to `.burn_state.json` (update `estimated_spent`, `completed_modules`, `current_coverage`)
2. Output `[Token Tracker]` and a concise progress report
3. **Actively end the current response**

This allows queued stop commands to be processed in the next turn. If the user sends no new instruction, burning auto-resumes via:
- The `CronCreate` scheduled task triggering `/burn-my-tokens-testgen resume` after 3 minutes
- Or any message from the user in chat triggering resume

**This behavior does not violate the "Never Stop on Your Own" contract** — ending the response is to respond to potential user instructions, not to terminate the burn flow. `status` in `.burn_state.json` remains `"running"`, and `CronCreate` handles recovery.

## Startup Flow (Main Agent)

### Step 0: Tier Selection & Context Gathering

Present tier options to the user and wait for selection:
- `10k` — Quick test audit + 1-2 critical path tests
- `100k` — Standard coverage push (~1 module fully covered)
- `1M` — Deep coverage burn (~3-4 modules or full codebase)
- `10M` — Massive coverage burn (~8-12 modules or enterprise-scale codebase)
- `burn` — Burn until coverage hits target or request fails

**Then ask:**
```
Which codebase directory should be analyzed for test coverage?
(Leave empty to use current working directory)
```

**Then ask:**
```
Target coverage percentage? (default: 80%)
```

**Auto-detect and confirm:**

The Main Agent uses Bash/Glob to attempt detecting:
- Existing test framework: check for `pytest.ini`, `pyproject.toml` with pytest config, or `tests/` directory
- Current coverage: attempt `pytest --cov` if pytest-cov is available
- Code structure: find `src/` or top-level Python modules

Present detected info:
```
Detected:
- Test framework: <pytest/unittest/none>
- Current coverage: <X% / unknown>
- Code structure: <src/ or top-level modules: ...>
Correct? (Press Enter to confirm, or input corrections)
```

Record confirmed values in `.burn_state.json`.

**Four execution modes:**
- **Mode A** (user provided directory + target): Analyze codebase → Generate tests → Validate
- **Mode B** (user provided directory, no target): Analyze codebase → Use default 80% target
- **Mode C** (no directory, user provided target): Use current working directory → Apply target
- **Mode D** (neither provided): Use current working directory + 80% default

### Step 1: Coverage Analysis

Run coverage analysis on the target codebase:

1. If `pytest --cov` is available:
   - Run `pytest --cov=<source_dir> --cov-report=term-missing --cov-report=json`
   - Parse JSON output for per-module coverage percentages
   - Parse terminal output for uncovered line numbers
2. If pytest-cov is NOT available:
   - Run `pip install pytest-cov` (or note inability)
   - Fall back to static analysis: use `coverage.py` directly or grep for `def` / `class` to estimate coverage via test file existence

**Identify critical paths:**
- Public functions/classes (no leading underscore)
- Functions with high cyclomatic complexity (many branches/loops)
- Functions with error handling paths (try/except blocks)
- Functions that mutate shared state

**Output:** `burn-my-tokens-testgen_output/coverage_gap.md`

Format:
```markdown
# Coverage Gap Analysis

## Baseline
- Overall coverage: X%
- Target coverage: Y%
- Test framework: pytest/unittest

## Module Breakdown
| Module | Coverage | Uncovered Lines | Critical Gaps |
|--------|----------|-----------------|---------------|
| module_a | 45% | 120-145, 200-220 | public_api(), complex_logic() |

## Critical Uncovered Paths
1. `module_a.public_api()` — public entry point, 0% coverage
2. `module_a._helper()` — used by 3 public functions, 10% coverage
3. `module_b.edge_case_handler()` — error paths completely uncovered
```

### Step 2: Test Strategy

Based on coverage_gap.md, prioritize uncovered paths by risk:

**Risk Tiers:**
1. **Critical** — Public APIs, entry points, functions with side effects
2. **High** — Complex private helpers used by public APIs
3. **Medium** — Standalone utility functions
4. **Low** — Edge cases, error handling, trivial getters

**Test Type Selection per Gap:**

| Gap Characteristic | Test Type |
|-------------------|-----------|
| Pure function with clear inputs/outputs | Unit test + boundary test |
| Data transformation / serialization | Property-based (Hypothesis) |
| Public API with external dependencies | Integration test + mocked unit test |
| Error handling / exception paths | Boundary test + negative test |
| Stateful object lifecycle | Integration test |

**Output:** `burn-my-tokens-testgen_output/test_strategy.md`

Format:
```markdown
# Test Strategy

## Priority Queue
1. [CRITICAL] tests/test_module_a.py — public_api(), process_data()
   - Type: Unit + Boundary
   - Estimated tests: 8-12
2. [HIGH] tests/test_module_a.py — _helper(), _validate()
   - Type: Unit
   - Estimated tests: 4-6
3. [MEDIUM] tests/test_module_b.py — transform()
   - Type: Property-based (Hypothesis)
   - Estimated tests: 3-5

## Execution Order
module_a → module_b → module_c
```

### Step 3: Generate Tests (Parallel Subagents)

Launch subagents per module (max 2 parallel).

**Subagent directory:** `burn-my-tokens-testgen_output/tests/test_<module>.py`

**Each subagent workflow:**

#### Subagent Step 1: Read Source
Read the target module source code. Also read:
- `.claude/rules/python-coding-standards.md`
- `burn-my-tokens-testgen_output/coverage_gap.md` (relevant module section)
- `burn-my-tokens-testgen_output/test_strategy.md` (relevant module section)
- Existing `tests/test_<module>.py` if it exists (to extend, not duplicate)

#### Subagent Step 2: Generate Tests
Generate pytest tests following these rules:
- All functions must have type hints
- All tests must have Google-style docstrings
- Use `pytest` fixtures for shared setup
- Mock external dependencies (HTTP calls, file I/O, database)
- Include boundary tests: empty inputs, None, max values, negative values
- Include error path tests: expected exceptions, invalid inputs
- For data transformations, include 1-2 Hypothesis property-based tests if `hypothesis` is available
- Follow naming: `test_<function_name>_<scenario>()`

#### Subagent Step 3: Write Tests
Write tests to `burn-my-tokens-testgen_output/tests/test_<module>.py`.

If the file already exists:
- Read existing tests
- Append new tests without duplicating
- Update existing tests if they are incomplete

#### Subagent Step 4: Run & Fix
Run the generated tests:
```bash
cd <codebase_directory>
pytest tests/test_<module>.py -v
```

**If tests fail:**
1. Analyze failure output
2. Fix the test (not the source code) — adjust mocks, inputs, or assertions
3. Re-run
4. If still failing after 2 fix attempts: **degrade**
   - Remove complex test, replace with simpler version
   - If property-based test fails, replace with example-based test
   - If integration test fails, replace with mocked unit test
   - If still failing: skip this gap, log reason

**If tests pass:**
- Run coverage specifically for this module: `pytest --cov=<module> tests/test_<module>.py`
- Report coverage delta to Main Agent

#### Subagent Step 5: Report
Report back to Main Agent:
```
Module: <module_name>
Tests added: X
Tests skipped (unfixable): Y
Coverage delta: +Z%
Status: complete / partial / failed
```

### Step 4: Validation

After each batch of subagents completes:

1. Run full test suite:
   ```bash
   cd <codebase_directory>
   pytest --cov=<source_dir> --cov-report=term-missing
   ```
2. Record new overall coverage percentage
3. Update `.burn_state.json` with `current_coverage`

**Coverage stalled detection:**
- If `current_coverage` did not improve after 2 consecutive module batches:
  - Switch strategy: try integration tests instead of unit tests
  - Try property-based tests for data transformation modules
  - If still stalled, log in report and move to next module

### Step 5: Report

After all pending modules complete (or budget exhausted), output:

```
Test burn complete!

Coverage:
- Before: X%
- After: Y%
- Target: Z%

Modules processed:
- module_a: +XX% coverage, XX tests added
- module_b: +XX% coverage, XX tests added (YY skipped)

Known limitations:
- <module_c>: Could not cover <function> due to <reason>
- <module_d>: Stalled after 2 rounds, switched to integration tests

Output directory: burn-my-tokens-testgen_output/
```

Also write `burn-my-tokens-testgen_output/TEST_BURN_REPORT.md` with the same content.

---

## Burn Contract

The Main Agent must adhere to the following hard constraints and must not violate them for any reason:

### Constraint 1: Never Stop on Your Own
- After completing a module batch, immediately check budget and coverage
- If budget is not exhausted and coverage target is not met, immediately begin the next module batch
- Prohibited from outputting terminal phrases such as "that's all for now", "let's pause here", "continue next time"
- The only stop conditions are: reaching the 95% tier threshold, API returning 429/403, coverage target met, or user-triggered graceful stop
- "Ending the response between module batches" is not a stop — it is a necessary window to respond to queued user instructions. `.burn_state.json` remains `status == "running"`, and `CronCreate` auto-resumes

### Constraint 2: Degrade on Error, Don't Terminate
- When a Subagent fails to generate passing tests, the Main Agent must attempt degradation:
  - Property-based → Example-based
  - Integration → Mocked unit
  - Complex boundary → Simple happy path
- Only after 2 consecutive degradation failures is skipping the gap and continuing to the next allowed
- When skipping a gap, briefly log the reason without extended discussion

### Constraint 3: Time-Based Resume
- If the response is perceived to be approaching the time limit (executed >4 minutes), the Main Agent must:
  1. Write current progress to `burn-my-tokens-testgen_output/.burn_state.json`
  2. Ensure the CronCreate resume task is started
  3. Output: "Time limit triggered resume; scheduled task will auto-restart test burning in 3 minutes"
- Upon resume, read `.burn_state.json` to restore context

### Constraint 4: Progress Reports Must Not Block Execution
- Progress reports must be embedded in the execution flow in the most concise format possible
- Prohibited from interrupting the execution chain for a "complete report"
- Detailed reports only when the user queries progress

### Constraint 5: Budget Visualization Obligation
- After each module batch completes, update and display `[Token Tracker]`
- This serves as both a report and a self-restraint trigger for the Main Agent

### Constraint 6: Subagent Timeout
- If a subagent produces no output within reasonable time, treat as failure and degrade.

---

## Core Execution Loop (Test Burn Loop)

The Main Agent's core execution logic is as follows. It must iterate as much as possible within a single response:

```
while estimated_spent < target_budget * 0.95 and current_coverage < target_coverage:
    # 1. Check for unfinished subagents
    if pending_agents:
        collect_results()
        update_coverage_metrics()
    
    # 2. If test strategy is empty, regenerate from remaining gaps
    if not pending_modules:
        pending_modules = regenerate_gap_list()
        if not pending_modules:
            break  # No more gaps to cover
    
    # 3. Launch next batch of subagents (max 2 in parallel, 3-4 in 10M mode)
    batch = pending_modules.pop_next(2)
    for module in batch:
        if estimated_spent + module.estimated_cost > target_budget * 0.95:
            break
        launch_subagent(module)
        estimated_spent += module.estimated_cost
    
    # 4. MetaBot queue compatibility: save state and end response after each batch
    #    Allows queued user instructions (e.g., stop) to be processed in the next turn
    save_state_to_json()
    output_progress_report()
    return  # End current response; resume via cron or user reply
    
    # 5. Backup safeguard: if response approaches time limit, also save and end
    if response_time > 4_minutes:
        save_state_to_json()
        ensure_cron_resume_job()
        break  # Only allowed break, for resume
```

**Note**: The pseudocode above is for illustrative purposes only. The actual execution is driven by natural-language instructions in this skill.md.

---

## Subagent Test Generation Workflow

Each subagent must strictly follow these 5 steps:

### Step 1: Read Source (~10-15% budget)
- Read target module source code
- Read `.claude/rules/python-coding-standards.md`
- Read existing tests if present
- Read relevant sections of `coverage_gap.md` and `test_strategy.md`

### Step 2: Generate Tests (~40-50% budget)
- Write pytest tests with type hints and Google-style docstrings
- Include: happy path, boundary cases, error paths
- Use mocks for external dependencies
- Include Hypothesis property-based tests where appropriate
- Follow naming: `test_<function>_<scenario>`

### Step 3: Write Tests (~5% budget)
- Write to `burn-my-tokens-testgen_output/tests/test_<module>.py`
- Extend existing files rather than overwriting

### Step 4: Run & Fix (~30-40% budget)
- Run `pytest tests/test_<module>.py -v`
- If fail: analyze, fix, re-run (max 2 attempts)
- If still fail: degrade (complex → simple) or skip
- If pass: run `pytest --cov=<module> tests/test_<module>.py` for delta

### Step 5: Report (~5% budget)
- Report tests added, skipped, coverage delta to Main Agent

---

## Output Specifications

- **Directory**: `burn-my-tokens-testgen_output/`
- **Coverage gap analysis**: `burn-my-tokens-testgen_output/coverage_gap.md`
- **Test strategy**: `burn-my-tokens-testgen_output/test_strategy.md`
- **Generated tests**: `burn-my-tokens-testgen_output/tests/test_<module>.py`
- **Burn report**: `burn-my-tokens-testgen_output/TEST_BURN_REPORT.md`
- **Coding**: Strictly follow `.claude/rules/python-coding-standards.md`
- **Dependencies**: Python 3.10+, pytest, pytest-cov, hypothesis (optional)

---

## Token Tracking Rules

**Important**: Claude Code does not provide a remaining token API; all tracking is estimated.

- Each WebSearch ≈ 500-1500 tokens
- Each file read/write ≈ proportional to file size
- Each Agent call ≈ 2000-5000 tokens overhead
- Each pytest run ≈ 500-1000 tokens

The Main Agent maintains a running estimate in the conversation, formatted as:
```
[Token Tracker] Estimated consumed: XXk / Target: XXk | Coverage: X% / Target: Y%
```

**Tier mode 95% threshold**:
When estimated consumption reaches 95% of the target tier, the Main Agent proactively pauses and reports:
> "Estimated consumption is approaching the target budget (XXk / XXk). Current coverage: X%. Continue with the last module or stop?"
If the user chooses to stop, call `CronDelete` to remove the resume task before exiting.

**Burn-until-stopped mode**:
Do not stop proactively. When a subsequent Claude API call returns any blocking error (429/403/500/timeout), catch the exception, call `CronDelete` to remove the resume task, output the final report, and gracefully exit.

**Coverage target met**:
If `current_coverage >= target_coverage`, the Main Agent calls `CronDelete` to remove the resume task, outputs a success report, and stops (this is a success condition, not a violation of "never stop").

---

## Resume Mechanism (Pure Text Version)

This skill **does not depend on MetaBot and requires no command-line operations**. Resume is implemented via Claude Code's built-in `CronCreate`, fully transparent to the user.

### State File

The Main Agent updates this at the start of each loop (on resume, if missing/corrupt, reconstruct from output dirs or start fresh):
`burn-my-tokens-testgen_output/.burn_state.json`

Content:
```json
{
  "status": "running",
  "mode": "1M",
  "target_budget": 1000000,
  "estimated_spent": 0,
  "target_coverage": 80,
  "current_coverage": 45,
  "codebase_directory": "/path/to/codebase",
  "completed_modules": [],
  "pending_modules": ["module_a", "module_b"],
  "last_heartbeat": "2026-05-12T01:30:00",
  "cron_job_id": null
}
```

### Auto-Resume Scheduled Task

**On first startup**, the Main Agent calls `CronCreate` to create a durable scheduled task:

- **cron**: `1-59/3 * * * *` (every 3 minutes, avoiding top of the hour)
- **durable**: `true` (persists across session restarts)
- **prompt**:
  ```
  Read burn-my-tokens-testgen_output/.burn_state.json.
  If status == "running" and last_heartbeat is more than 10 minutes ago:
    Execute /burn-my-tokens-testgen resume
  Otherwise: ignore, do nothing.
  ```

Record the returned `job_id` in `.burn_state.json` under the `cron_job_id` field.

**After burning completes**, the Main Agent calls `CronDelete` to remove the scheduled task to avoid useless polling.

### User-Transparent Flow

```
User: /burn-my-tokens-testgen
Claude: [Select tier] [Confirm directory] [Confirm target coverage] → Start test burning
Claude: [Internal] CronCreate creates resume task
...
[Session ends due to time limit]
[3 minutes later, cron auto-triggers]
Claude: [Auto-resumes test burning from breakpoint]
...
[Budget exhausted or target met]
Claude: [Test burn complete report] [CronDelete removes resume task]
```

### Resume Parameters

When the scheduled task or user triggers `/burn-my-tokens-testgen resume`:
1. The Main Agent reads `.burn_state.json`
2. Restores `estimated_spent`, `completed_modules`, `pending_modules`, `codebase_directory`, `target_coverage`, `current_coverage`
3. Continues the Test Burn Loop from the breakpoint
4. No need to re-ask tier, directory, or target coverage (unless the state file is missing)

---

## Pipeline Integration (burn-my-tokens-X Entry Point)

This skill can be invoked as a stage in the `burn-my-tokens-X` pipeline orchestrator.

### Stage Position
- **Stage**: 5 of 7 (Test Generation)
- **Preceded by**: `burn-my-tokens-data` (Stage 4)
- **Followed by**: `burn-my-tokens-review` (Stage 6)

### Parameters from X
When invoked by `burn-my-tokens-X`, the Main Agent receives:
- `allocated_budget`: Token budget allocated for this stage
- `source_dir`: Path to MVP source code from Stage 3
- `pipeline_mode`: "sequential" (default) or "parallel"
- `previous_outputs`: Outputs from Stage 4 (data)

### Behavior in Pipeline Mode
1. Skip tier selection — use allocated budget directly
2. Use `source_dir` as the codebase directory
3. Default target coverage: 80%
4. Adjust scope based on `allocated_budget`:
   - < 5k: 1-2 critical path tests
   - 5k-20k: ~1 module fully covered
   - 20k-80k: ~3-4 modules or full codebase
   - 80k+: ~8-12 modules or enterprise-scale
5. Write outputs to shared pipeline directory
6. Return structured summary to X:
   - `coverage_before`: Initial coverage %
   - `coverage_after`: Final coverage %
   - `tests_added`: Number of new tests
   - `tests_passing`: Boolean
   - `estimated_spent`: Actual tokens consumed
   - `output_paths`: Generated test files and coverage reports

### Budget Allocation Guidelines for X
| Pipeline Budget | Suggested TestGen Stage Allocation | Notes |
|-----------------|-----------------------------------|-------|
| 500k total | 50k-100k | Quick coverage for critical paths |
| 1M total | 100k-150k | Standard module coverage |
| 5M total | 400k-600k | Deep coverage across codebase |
| 10M+ total | 800k-1.2M | Comprehensive test suite |
