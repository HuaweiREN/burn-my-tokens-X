---
name: burn-my-tokens-review
description: When a user's coding plan quota is about to reset, automatically perform deep code reviews on an existing codebase, generating structured review reports with issue severity ratings, rationale, and concrete fix suggestions — but does NOT automatically apply fixes. Creates a human-in-the-loop safety gate: review first, then decide which fixes to apply. Supports unattended self-iterative burning with pure text interaction. Graceful stop via stop command.
version: 1.1.0
---

## Trigger Conditions

This skill activates when the user inputs any of the following:
- `/burn-my-tokens-review` — Start the review burn flow
- `/burn-my-tokens-review stop` / `停止审查` / `stop review` / `stop burn` — Gracefully end the burn
- Expresses anxiety such as "code review / need review / quality check / audit my code / tokens about to expire / review my code"

## Graceful Stop

When the user inputs a stop command, the Main Agent must:
1. Set `status` to `"stopped"` in `.burn_state.json`
2. Output stop confirmation: `Review burn gracefully stopped. Completed X files, found Y issues, consumed approximately XXk tokens.`
3. Call `CronDelete` to remove the resume cron job (if it exists)
4. Subsequent `CronCreate` triggers reading `status == "stopped"` will be ignored

### MetaBot Queue Compatibility Strategy

Bot frameworks like MetaBot queue new user instructions, so a stop command sent during the current response cannot immediately interrupt execution. To ensure graceful stop works reliably:

**After each review batch (or each module if in 10k mode) completes, the Main Agent must:**
1. Save current progress to `.burn_state.json` (update `estimated_spent`, `files_reviewed`, `issues_found`, `code_health_score`)
2. Output `[Token Tracker]` and a concise progress report
3. **Actively end the current response**

This allows queued stop commands to be processed in the next turn. If the user sends no new instruction, burning auto-resumes via:
- The `CronCreate` scheduled task triggering `/burn-my-tokens-review resume` after 3 minutes
- Or any message from the user in chat triggering resume

**This behavior does not violate the "Never Stop on Your Own" contract** — ending the response is to respond to potential user instructions, not to terminate the burn flow. `status` in `.burn_state.json` remains `"running"`, and `CronCreate` handles recovery.

## Startup Flow (Main Agent)

### Step 0: Tier Selection & Context Gathering

Present tier options to the user and wait for selection:
- `10k` — Quick scan + review 1-2 critical files
- `100k` — Standard review (~1 module deeply reviewed)
- `1M` — Deep review burn (~3-4 modules or full codebase)
- `10M` — Massive review burn (~8-12 modules or enterprise-scale codebase)
- `burn` — Burn until no more issues found or request fails

**Then ask:**
```
Which codebase directory should be reviewed?
(Leave empty to use current working directory)

Examples:
- POC_Cam_Viewer
- src/
- ../another_workspace/some_project
```

**Then ask:**
```
What focus areas? (performance / readability / architecture / security / all)
(Leave empty for "all")
```

**Auto-detect and confirm:**

The Main Agent uses Bash/Glob to attempt detecting:
- Existing test framework: check for `pytest.ini`, `pyproject.toml` with pytest config, or `tests/` directory
- Code structure: find `src/` or top-level Python modules
- Language: check file extensions (`.py`, `.js`, `.ts`, etc.)
- Coding standards: look for `.claude/rules/*.md`

Present detected info:
```
Detected:
- Language: <language>
- Source structure: <src/ or top-level modules: ...>
- Test framework: <pytest/unittest/none>
- Coding standards: <found / not found>
Correct? (Press Enter to confirm, or input corrections)
```

Record confirmed values in `.burn_state.json`.

**Four execution modes:**
- **Mode A** (user provided directory + focus): Analyze codebase → Generate reviews → Consolidate
- **Mode B** (user provided directory, no focus): Analyze codebase → Review all axes
- **Mode C** (no directory, user provided focus): Use current working directory → Apply focus
- **Mode D** (neither provided): Use current working directory + "all" focus

### Step 1: Code Analysis

**Read target files and build structural understanding:**

1. Use Glob to enumerate source files in the target directory
2. For each file, use Read to load content
3. Build internal analysis of:
   - **Complexity hotspots**: Functions with high cyclomatic complexity (many branches/loops), functions >100 lines, nesting depth >4
   - **Coupling analysis**: Which modules import which, circular dependencies
   - **Documentation gaps**: Functions/classes missing docstrings
   - **Type hint gaps**: Functions missing type annotations
   - **Potential bugs**: Unhandled exceptions, bare excepts, mutable default arguments, race condition patterns, resource leaks
   - **Performance issues**: Nested loops, inefficient data structure choices, string concatenation in loops
   - **Security issues**: Hardcoded secrets, SQL injection patterns, unsafe eval/exec, missing input validation

**If static analysis tools are available**, try to run:
- `python -m radon cc --min B <source_dir>` (cyclomatic complexity)
- `python -m radon mi <source_dir>` (maintainability index)
- `ruff check <source_dir>` (linting issues)
- `bandit -r <source_dir>` (security issues, if available)

If tools are missing, attempt `pip install radon ruff bandit`. If installation fails, proceed with AST-based manual analysis using Python's built-in `ast` module or line-based heuristics.

**Output:** `burn-my-tokens-review_output/analysis_report.md`

Format:
```markdown
# Code Analysis Report

## Project: <project_name>
## Date: <timestamp>
## Focus: <focus_areas>

## Structure Overview
- Language: <language>
- Total files: X
- Total functions: Y
- Total classes: Z
- Test framework: <framework>

## Complexity Hotspots
| File | Function | Lines | CC | Nesting Depth |
|------|----------|-------|-----|---------------|

## Coupling Analysis
| File | Imports | Imported By | Coupling Score |
|------|---------|-------------|----------------|

## Documentation Gaps
| File | Functions | With Docstrings | Coverage |
|------|-----------|-----------------|----------|

## Type Hint Gaps
| File | Functions | With Type Hints | Coverage |
|------|-----------|-----------------|----------|

## Potential Bugs (Heuristic)
| File | Line | Pattern | Confidence |
|------|------|---------|------------|

## Performance Issues
| File | Line | Issue | Severity |
|------|------|-------|----------|

## Security Issues
| File | Line | Issue | Severity |
|------|------|-------|----------|
```

### Step 2: Issue Prioritization

Rank all identified issues by the severity classification matrix:

**Severity Matrix:**

| Likelihood \ Impact | High | Medium | Low |
|---------------------|------|--------|-----|
| **High** | **Critical** | **High** | **Medium** |
| **Medium** | **High** | **Medium** | **Low** |
| **Low** | **Medium** | **Low** | **Low** |

**Severity Definitions:**

| Level | Definition | Examples |
|-------|-----------|----------|
| **Critical** | Could cause crashes, data loss, security vulnerabilities, or incorrect behavior in production | SQL injection, unhandled exceptions in critical paths, race conditions, missing input validation on public APIs |
| **High** | Significantly impacts maintainability, performance, or correctness | Deep nesting (>4 levels), functions >100 lines, tight coupling, missing error handling, type hint gaps in public APIs |
| **Medium** | Moderate impact on code quality or developer experience | Missing docstrings, inconsistent naming, magic numbers, moderate complexity (CC 11-20) |
| **Low** | Minor style or cosmetic issues | Trailing whitespace, import ordering, minor naming inconsistencies |

**Impact Axes:**

| Axis | High Impact | Medium Impact | Low Impact |
|------|------------|--------------|-----------|
| **Correctness** | Could produce wrong results or crash | Edge case handling missing | Theoretical issue only |
| **Performance** | O(n^2) where O(n) possible, memory leaks | Inefficient data structures | Micro-optimization |
| **Security** | Injection, XSS, auth bypass | Missing validation, exposed internals | Information disclosure |
| **Maintainability** | Untestable, tightly coupled, no docs | Complex but documented | Minor style issues |
| **Readability** | Incomprehensible without domain knowledge | Unclear variable names | Inconsistent formatting |

Group issues into review batches:
- **Batch 1**: Critical issues across all targeted modules
- **Batch 2**: High severity issues
- **Batch 3**: Medium severity issues
- **Batch 4**: Low severity issues (if budget remains)

**Output:** `burn-my-tokens-review_output/review_plan.md`

Format:
```markdown
# Review Plan

## Batches

### Batch 1: Critical Issues
- `src/module_a.py`: 2 critical issues (unhandled exception, missing validation)
- `src/module_b.py`: 1 critical issue (race condition)
Estimated review cost: ~X tokens

### Batch 2: High Severity
- `src/module_a.py`: 3 high issues (complexity, coupling)
- `src/module_c.py`: 2 high issues (performance)
Estimated review cost: ~X tokens

### Batch 3: Medium Severity
...

### Batch 4: Low Severity
...
```

### Step 3: Generate Reviews (Parallel Subagents)

Launch subagents per module (max 2 parallel).

**Scheduling strategy:**
- First batch: launch up to 2 modules in parallel
- After completion, evaluate remaining budget before next batch
- In `10k` mode: process 1 file at a time, no parallelism

Each subagent's output directory: `burn-my-tokens-review_output/review_reports/`

**Subagent workflow:**

#### Subagent Step 1: Read Source
Read the target module source code. Also read:
- `.claude/rules/python-coding-standards.md` (if exists)
- `burn-my-tokens-review_output/analysis_report.md` (relevant module section)
- `burn-my-tokens-review_output/review_plan.md` (relevant batch section)

#### Subagent Step 2: Deep Review
Generate a detailed review document following this structure:

```markdown
# Review: <module_name>

## File Summary
- Lines of code: X
- Functions: Y
- Classes: Z
- Overall severity: <critical/high/medium/low>
- Module health score: XX/100

## Issues

### Issue #<N>: <title>
- **Location**: `<function_name>` (lines X-Y)
- **Severity**: <critical/high/medium/low>
- **Category**: <correctness/performance/security/maintainability/readability>
- **Rationale**: <why this is a problem — explain the impact>
- **Current code**:
  ```python
  <exact code snippet>
  ```
- **Suggested fix**:
  ```python
  <concrete code example showing the fix>
  ```
- **Reference**: <link to coding standard or best practice>
```

**Review rules:**
- Every issue must have a code snippet proving the issue exists
- Every issue must have a concrete fix suggestion with code example
- Reference project coding standards if available
- If no coding standards available, reference PEP 8, Google Python Style Guide, or general software engineering best practices
- Include line numbers for every issue
- Flag confidence level: `high` (certain), `medium` (likely), `low` (suspicious)

#### Subagent Step 3: Calculate Module Health Score

Calculate module-level health score:
```
module_health_score = 100 - penalty_total

penalty_total = (
    critical_issues * 10 +
    high_issues * 5 +
    medium_issues * 2 +
    low_issues * 0.5 +
    avg_cyclomatic_complexity_penalty +
    docstring_coverage_penalty +
    type_hint_coverage_penalty
)

avg_cyclomatic_complexity_penalty = max(0, (avg_cc - 10) * 2)
docstring_coverage_penalty = (100 - docstring_coverage%) * 0.3
type_hint_coverage_penalty = (100 - type_hint_coverage%) * 0.3
```

Score interpretation:
| Score | Rating |
|-------|--------|
| 90-100 | Excellent |
| 70-89 | Good |
| 50-69 | Fair |
| 30-49 | Poor |
| 0-29 | Critical |

#### Subagent Step 4: Write Review Report
Write the review to `burn-my-tokens-review_output/review_reports/<module>_review.md`

#### Subagent Step 5: Report to Main Agent
Report back:
```
Module: <module_name>
Issues found: X (Critical: A, High: B, Medium: C, Low: D)
Module health score: XX/100
Confidence: high/medium/low
Status: complete / partial / degraded
```

### Step 4: Consolidation

After each batch of subagents completes:

1. Read all generated `review_reports/<module>_review.md` files
2. Merge into a single `MASTER_REVIEW.md`
3. Calculate overall code health score (average of module scores, weighted by lines of code)
4. Identify top 10 issues across all modules

**Output:** `burn-my-tokens-review_output/MASTER_REVIEW.md`

Format:
```markdown
# Master Review Report

## Executive Summary
- Files reviewed: X
- Total issues: Y (Critical: A, High: B, Medium: C, Low: D)
- Overall code health score: XX/100
- Top concern: <one sentence>

## Top 10 Issues
1. [CRITICAL] `module_a.function_b` (line 45) — <brief description>
2. [HIGH] `module_c.function_d` (line 120) — <brief description>
...

## Module Health Scores
| Module | Score | Critical | High | Medium | Low |
|--------|-------|----------|------|--------|-----|

## Recommendations by Priority
### Immediate (Critical)
- Fix issue #1, #2 before any production deployment

### Short-term (High)
- Address issues #3-7 in next sprint

### Medium-term (Medium/Low)
- Schedule refactoring for issues #8+

## Known Limitations
- <what the review could not cover>
```

### Step 5: Report

After all pending modules complete (or budget exhausted), output:

```
Review burn complete!

Project: <project_name>
Directory: <source_dir>
Focus: <focus_areas>

Files reviewed: X / Y total
Issues found: Z (Critical: A, High: B, Medium: C, Low: D)
Overall code health score: XX/100

Module breakdown:
- module_a: score 65/100 (1 critical, 3 high, 5 medium)
- module_b: score 82/100 (0 critical, 1 high, 2 medium)
...

Top 3 issues:
1. [CRITICAL] module_a.function_b (line 45) — <description>
2. [HIGH] module_c.function_d (line 120) — <description>
3. [HIGH] module_e.function_f (line 200) — <description>

Next steps:
- Run `/burn-my-tokens-refactor` to apply selected fixes
- Or manually fix critical issues first

Output directory: burn-my-tokens-review_output/
Reports:
- MASTER_REVIEW.md (consolidated findings)
- analysis_report.md (structural analysis)
- review_plan.md (prioritization)
- review_reports/<module>_review.md (per-module details)
```

Also write `burn-my-tokens-review_output/REVIEW_BURN_REPORT.md` with the same content.

---

## Burn Contract

The Main Agent must adhere to the following hard constraints and must not violate them for any reason:

### Constraint 1: Never Stop on Your Own
- After completing a review batch, immediately check budget and remaining files
- If budget is not exhausted and files remain, immediately begin the next batch
- Prohibited from outputting terminal phrases such as "that's all for now", "let's pause here", "continue next time"
- The only stop conditions are: all targeted files reviewed, reaching the 95% tier threshold, API returning 429/403, or user-triggered graceful stop
- "Ending the response between batches" is not a stop — it is a necessary window to respond to queued user instructions. `.burn_state.json` remains `status == "running"`, and `CronCreate` auto-resumes

### Constraint 2: Degrade on Error, Don't Terminate
- When a subagent fails to review a file (too complex, parsing errors, etc.), the Main Agent must attempt degradation:
  1. First failure: Switch to high-level summary review (file-level only, no per-function analysis)
  2. Second failure: Switch to issue-list only (list known patterns found in the file)
  3. Third failure: Skip file, log reason, move to next
- Only after 3 consecutive degradation failures on the same file is skipping allowed
- When skipping a file, briefly log the reason without extended discussion

### Constraint 3: Read-Only Safety
- This skill must NEVER modify source code files
- Only write to `burn-my-tokens-review_output/` directory
- Before any Write/Edit tool call, verify the target path is within the output directory
- If a subagent attempts to edit source code, abort the subagent and degrade

### Constraint 4: Time-Based Resume
- If the response is perceived to be approaching the time limit (executed >4 minutes), the Main Agent must:
  1. Write current progress to `burn-my-tokens-review_output/.burn_state.json`
  2. Ensure the CronCreate resume task is started
  3. Output: "Time limit triggered resume; scheduled task will auto-restart review burning in 3 minutes"
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

## Core Execution Loop (Review Burn Loop)

The Main Agent's core execution logic is as follows. It must iterate as much as possible within a single response:

```
while estimated_spent < target_budget * 0.95 and files_remain:
    # 1. Check for unfinished subagents
    if pending_agents:
        collect_results()
        update_issue_counts()
    
    # 2. If no batches planned, generate from analysis
    if not pending_batches:
        pending_batches = generate_batches_from_analysis()
    
    # 3. Launch next batch of subagents (max 2 in parallel, 3-4 in 10M mode, 1 in 10k mode)
    batch = pending_batches.pop_next(2)
    for module in batch:
        if estimated_spent + module.estimated_cost > target_budget * 0.95:
            break
        launch_subagent(module)
        estimated_spent += module.estimated_cost
    
    # 4. Consolidate completed reviews
    if completed_modules:
        update_master_review()
        recalculate_code_health_score()
    
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

## Subagent Review Workflow

Each subagent must strictly follow these 5 steps:

### Step 1: Read & Scope (~10-15% budget)
- Read the target module source code
- Read `.claude/rules/python-coding-standards.md` if it exists
- Read relevant sections of `analysis_report.md` and `review_plan.md`
- Use AST extraction to scope context if the file is large (>200 lines)
- Output: internal notes on target functions/lines

### Step 2: Deep Review (~50-60% budget)
- Analyze each function/class for issues across focus areas
- For each issue identified:
  - Record exact line numbers
  - Assign severity using the severity matrix
  - Write rationale explaining why it's a problem
  - Provide concrete fix suggestion with code example
  - Reference coding standards or best practices
- Output: structured issue list

### Step 3: Calculate Health Score (~5% budget)
- Count issues by severity
- Calculate docstring coverage
- Calculate type hint coverage
- Compute module health score using the formula
- Output: score and rating

### Step 4: Write Report (~15-20% budget)
- Write formatted review to `review_reports/<module>_review.md`
- Ensure every issue has: location, severity, category, rationale, current code, suggested fix, reference
- Output: review report file

### Step 5: Report to Main Agent (~5% budget)
- Report: module name, issues found (by severity), health score, confidence, status
- Output: concise status message

---

## Project Specifications

- **Output directory**: `burn-my-tokens-review_output/`
- **Review reports**: `burn-my-tokens-review_output/review_reports/`
- **State file**: `burn-my-tokens-review_output/.burn_state.json`
- **Coding standards reference**: Strictly follow `.claude/rules/python-coding-standards.md` if it exists
- **Supported languages (v1)**: Python (architecture supports extension)

---

## Severity Classification Reference

### Cyclomatic Complexity Thresholds (Radon)
| Score | Rank | Review Action |
|-------|------|---------------|
| 1-5 | A (low) | No issue |
| 6-10 | B (medium) | Note in review |
| 11-20 | C (high) | Flag as high severity |
| 21-50 | D (very high) | Flag as critical severity |
| 50+ | E (extreme) | Flag as critical severity |

### Maintainability Index Thresholds (Radon)
| Score | Meaning | Review Action |
|-------|---------|---------------|
| 0-9 | Extremely difficult to maintain | Flag as critical |
| 10-19 | Difficult to maintain | Flag as high |
| 20-39 | Moderate | Flag as medium |
| 40-69 | Fairly easy to maintain | Note in review |
| 70-100 | Easy to maintain | No issue |

### Function Length Guidelines
| Lines | Review Action |
|-------|---------------|
| 1-30 | No issue |
| 31-50 | Note in review |
| 51-100 | Flag as medium |
| 101-200 | Flag as high |
| 200+ | Flag as critical |

### Nesting Depth Guidelines
| Depth | Review Action |
|-------|---------------|
| 1-2 | No issue |
| 3 | Note in review |
| 4 | Flag as medium |
| 5+ | Flag as high |

---

## Token Tracking Rules

**Important**: Claude Code does not provide a remaining token API; all tracking is estimated.

- Each WebSearch ≈ 500-1500 tokens
- Each file read/write ≈ proportional to file size
- Each Agent call ≈ 2000-5000 tokens overhead
- Each radon/ruff/bandit invocation ≈ 500-1000 tokens equivalent
- Each review report generation ≈ 3000-8000 tokens (depends on module size and issue count)

The Main Agent maintains a running estimate in the conversation, formatted as:
```
[Token Tracker] Estimated consumed: XXk / Target: XXk | Files: Y reviewed / Z total | Issues: W found
```

**Tier mode 95% threshold**:
When estimated consumption reaches 95% of the target tier, the Main Agent proactively pauses and reports:
> "Estimated consumption is approaching the target budget (XXk / XXk). Files reviewed: Y / Z. Continue with the last batch or stop?"
If the user chooses to stop, call `CronDelete` to remove the resume task before exiting.

**Burn-until-stopped mode**:
Do not stop proactively. When a subsequent Claude API call returns any blocking error (429/403/500/timeout), catch the exception, call `CronDelete` to remove the resume task, output the final report, and gracefully exit.

---

## Resume Mechanism (Pure Text Version)

This skill **does not depend on MetaBot and requires no command-line operations**. Resume is implemented via Claude Code's built-in `CronCreate`, fully transparent to the user.

### State File

The Main Agent updates this at the start of each loop (on resume, if missing/corrupt, reconstruct from output dirs or start fresh):
`burn-my-tokens-review_output/.burn_state.json`

Content:
```json
{
  "status": "running",
  "mode": "1M",
  "target_budget": 1000000,
  "estimated_spent": 3500,
  "codebase_directory": "/path/to/codebase",
  "focus_areas": ["performance", "readability"],
  "files_reviewed": ["src/module_a.py", "src/module_b.py"],
  "files_pending": ["src/module_c.py", "src/module_d.py"],
  "issues_found": {
    "critical": 2,
    "high": 5,
    "medium": 8,
    "low": 12
  },
  "code_health_score": 62,
  "last_heartbeat": "2026-05-12T14:30:00",
  "cron_job_id": "burn-review-resume-xxx"
}
```

### Auto-Resume Scheduled Task

**On first startup**, the Main Agent calls `CronCreate` to create a durable scheduled task:

- **cron**: `1-59/3 * * * *` (every 3 minutes, avoiding top of the hour)
- **durable**: `true` (persists across session restarts)
- **prompt**:
  ```
  Read burn-my-tokens-review_output/.burn_state.json.
  If status == "running" and last_heartbeat is more than 10 minutes ago:
    Execute /burn-my-tokens-review resume
  Otherwise: ignore, do nothing.
  ```

Record the returned `job_id` in `.burn_state.json` under the `cron_job_id` field.

**After burning completes**, the Main Agent calls `CronDelete` to remove the scheduled task to avoid useless polling.

### User-Transparent Flow

```
User: /burn-my-tokens-review
Claude: [Select tier] [Confirm directory] [Confirm focus] → Start review burn
Claude: [Internal] CronCreate creates resume task
...
[Session ends due to time limit]
[3 minutes later, cron auto-triggers]
Claude: [Auto-resumes review burning from breakpoint]
...
[Budget exhausted or all files reviewed]
Claude: [Review burn complete report] [CronDelete removes resume task]
```

### Resume Parameters

When the scheduled task or user triggers `/burn-my-tokens-review resume`:
1. The Main Agent reads `.burn_state.json`
2. Restores `estimated_spent`, `files_reviewed`, `files_pending`, `codebase_directory`, `focus_areas`, `code_health_score`, `issues_found`
3. Continues the Review Burn Loop from the breakpoint
4. No need to re-ask tier, directory, or focus (unless the state file is missing)

---

## Fallback Strategies

### No Static Analysis Tools Available
If radon/ruff/bandit cannot be installed:
1. Use Python's built-in `ast` module to compute:
   - Function line counts (flag >100 lines)
   - Nested `if` depth (flag >4 levels)
   - Number of return statements (flag >6)
   - Import organization (detect unused imports via `ast`)
2. Use line-based heuristics for common issues:
   - `except:` or `except Exception:` → flag bare except
   - `def func(arg=[])` → flag mutable default
   - `eval(` or `exec(` → flag unsafe execution
   - `password = ` or `secret = ` → flag potential hardcoded secret
3. Proceed with manual analysis; note limitations in `analysis_report.md`

### Large Files (>500 lines)
If a single file exceeds 500 lines:
1. Scope the subagent to specific functions within the file (AST extraction)
2. Process the file across multiple batches
3. Note in report: "File exceeds 500 lines; review scoped to highest-risk functions"

### Binary or Non-Text Files
If a file is binary or not parseable:
1. Skip the file
2. Log in state: `"skipped_files": ["path/to/file"]`
3. Note in report: "Binary file skipped"

### Missing Coding Standards
If `.claude/rules/python-coding-standards.md` does not exist:
1. Use PEP 8 as baseline
2. Use Google Python Style Guide for docstring and type hint expectations
3. Note in report: "Review used PEP 8 / Google Style Guide as baseline; project-specific standards not found"

---

## Integration with Sibling Skills

This skill is a sibling to `burn-my-tokens` (new project generation), `burn-my-tokens-refactor` (automatic refactoring), and `burn-my-tokens-testgen` (test generation). They share:
- The same burn contract philosophy (never stop, degrade on error, time-based resume)
- The same state file pattern (`.burn_state.json`)
- The same CronCreate resume mechanism
- The same output directory convention (`burn-my-tokens-review_output/` vs `burn-my-tokens_output/`)

They differ in:
- **Goal**: Identify issues vs generate new projects / refactor code / generate tests
- **Validation**: Structured review reports vs test-driven safety / run verification / test coverage
- **Output**: Review reports with severity ratings vs PRDs + source code / diff logs / test files
- **Safety**: Read-only by design vs modifies source code

### Recommended Workflow Sequence
1. `/burn-my-tokens-review` to identify all issues
2. Read `MASTER_REVIEW.md` and decide which issues to fix
3. `/burn-my-tokens-refactor` to apply selected fixes
4. `/burn-my-tokens-testgen` to add missing tests
5. `/burn-my-tokens-review` again to verify improvements and update code health score

---

## Quality Assurance Checklist

Before declaring a review complete, verify:
- [ ] Every issue has exact line numbers
- [ ] Every issue has a severity rating
- [ ] Every issue has rationale explaining why it's a problem
- [ ] Every issue has a concrete fix suggestion with code example
- [ ] No source files were modified
- [ ] All output is in `burn-my-tokens-review_output/`
- [ ] `MASTER_REVIEW.md` includes executive summary and top 10 issues
- [ ] Code health score is calculated and documented
- [ ] State file is updated with `files_reviewed`, `issues_found`, `code_health_score`

---

## Pipeline Integration (burn-my-tokens-X Entry Point)

This skill can be invoked as a stage in the `burn-my-tokens-X` pipeline orchestrator.

### Stage Position
- **Stage**: 6 of 7 (Code Review)
- **Preceded by**: `burn-my-tokens-testgen` (Stage 5)
- **Followed by**: `burn-my-tokens-refactor` (Stage 7)

### Parameters from X
When invoked by `burn-my-tokens-X`, the Main Agent receives:
- `allocated_budget`: Token budget allocated for this stage
- `source_dir`: Path to MVP source code from Stage 3
- `test_outputs`: Paths to test files from Stage 5
- `pipeline_mode`: "sequential" (default) or "parallel"
- `previous_outputs`: Outputs from Stage 5 (testgen)

### Behavior in Pipeline Mode
1. Skip tier selection — use allocated budget directly
2. Use `source_dir` as the codebase directory
3. Default focus: "all" (performance, readability, architecture, security)
4. Adjust scope based on `allocated_budget`:
   - < 5k: Review 1-2 critical files
   - 5k-20k: ~1 module deeply reviewed
   - 20k-80k: ~3-4 modules or full codebase
   - 80k+: ~8-12 modules or enterprise-scale
5. Write outputs to shared pipeline directory
6. Return structured summary to X:
   - `files_reviewed`: List of reviewed files
   - `issues_found`: Breakdown by severity
   - `code_health_score`: Overall score (0-100)
   - `top_concerns`: List of top issues
   - `estimated_spent`: Actual tokens consumed
   - `output_paths`: Generated review reports

### Budget Allocation Guidelines for X
| Pipeline Budget | Suggested Review Stage Allocation | Notes |
|-----------------|----------------------------------|-------|
| 500k total | 50k-100k | Quick scan of critical files |
| 1M total | 100k-150k | Standard module review |
| 5M total | 400k-600k | Deep review across codebase |
| 10M+ total | 800k-1.2M | Comprehensive audit |
