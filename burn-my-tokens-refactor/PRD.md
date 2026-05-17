# PRD: burn-my-tokens-refactor

**Version:** 1.1.0  
**Date:** 2026-05-12  
**Status:** L3 MVP

## 1. Problem Statement

Users who rapidly generate MVP projects accumulate technical debt: duplicated logic, overly complex functions, inconsistent naming, unused imports, and poor module organization. When their Claude Code token quota nears reset, they have two bad options:
1. Let tokens expire unused
2. Generate yet another new project, adding to the debt pile

**burn-my-tokens-refactor** solves this by converting remaining tokens into codebase quality improvements. It autonomously detects code smells, prioritizes them, executes safe refactors with test validation, and reports measurable improvements.

## 2. Solution Overview

A Claude Code skill that:
- Detects code smells using static analysis (radon, jscpd, pylint)
- Prioritizes smells by impact x ease of fix
- Launches parallel subagents to refactor modules
- Validates every change against existing tests
- Rolls back on failure, degrades strategy on repeated failure
- Never stops until budget exhausted or no more smells found
- Saves state for time-based resume

## 3. Target Users

- Developers with 5-20+ MVP projects in a workspace
- Users who generate code fast and clean up later (or never)
- Token-quota-conscious users who want to maximize value per token

## 4. User Stories

1. As a user with messy MVP projects, I want to run `/burn-my-tokens-refactor` so that my existing code gets cleaner without manual effort.
2. As a user nearing token reset, I want to burn remaining tokens on refactoring so that I don't waste quota.
3. As a user, I want the skill to preserve my tests so that refactoring doesn't break working code.
4. As a user, I want a before/after report so that I can see what improved.
5. As a user with limited time, I want the skill to auto-resume if interrupted so that I don't lose progress.

## 5. Functional Requirements

### FR1: Trigger & Tier Selection
- Command: `/burn-my-tokens-refactor`
- Tiers: `10k`, `100k`, `1M`, `10M`, `burn`
- Context gathering: target directory + focus area (complexity / duplication / naming / all)

### FR2: Auto-Detection
- Detect code structure (find `src/` or top-level `.py` modules)
- Detect test suite (pytest, unittest)
- Detect language (Python primary; degrade gracefully for others)

### FR3: Smell Detection
- Run radon for complexity and maintainability index
- Run jscpd for duplication
- Run pylint for semantic smells
- Output: `smell_report.md` with ranked list

### FR4: Prioritization
- Rank smells by: severity × ease_of_fix × module_importance
- Group related smells into refactor batches
- Output: `refactor_plan.md`

### FR5: Refactoring Execution
- Launch subagents per module (max 2 parallel)
- Each subagent: read code → plan changes → apply → run tests → log diff
- If tests fail: rollback → conservative retry (max 3 attempts)
- If 3 rollbacks: skip module, log reason

### FR6: Validation
- Run full test suite after each batch
- Re-run smell detection after each batch
- If smells reduced: continue
- If smells stalled: switch strategy (structural vs local)

### FR7: Reporting
- Before/after smell counts and severity
- Files refactored
- Lines changed
- Known limitations

### FR8: State & Resume
- Save `.burn_state.json` after each batch
- CronCreate every 3 minutes for auto-resume
- Resume reads state, restores context, continues loop

### FR9: Graceful Stop
- `/burn-my-tokens-refactor stop` stops after current batch
- Updates state to `"stopped"`
- Removes cron job

## 6. Non-Functional Requirements

### NFR1: Safety
- Never refactor without running tests first
- Always preserve test pass status
- Rollback on any test failure

### NFR2: Reliability
- Degrade on error, don't terminate
- Skip modules that can't be safely refactored
- Handle missing tools gracefully (fallback to AST-based detection)

### NFR3: Transparency
- Progress reports embedded in execution flow
- Token tracker after each batch
- Diff logs for every module

### NFR4: Efficiency
- Parallel subagents (max 2)
- Context scoping to reduce tokens
- Conservative retry instead of repeated failure

## 7. Architecture

```
User Input
    |
    v
[Main Agent] -- Tier Selection + Context Gathering
    |
    v
[Step 1: Smell Detection] -- radon + jscpd + pylint
    |
    v
[Step 2: Prioritization] -- Rank + Batch
    |
    v
[Step 3: Refactor Loop] -- Launch subagents (max 2 parallel)
    |                    |
    |                    v
    |            [Subagent A] -- Read → Plan → Apply → Test → Diff
    |                    |
    |            [Subagent B] -- Read → Plan → Apply → Test → Diff
    |
    v
[Step 4: Validation] -- Full tests + smell re-detection
    |
    v
[Step 5: Report] -- Before/after metrics
    |
    v
[State Save] -- .burn_state.json + CronCreate
```

## 8. Data Models

### Smell Report Entry
```json
{
  "file": "src/collectors/web_collector.py",
  "function": "fetch_with_retry",
  "smell_type": "high_complexity",
  "severity": "critical",
  "metric": "cyclomatic_complexity",
  "value": 24,
  "threshold": 10,
  "line_start": 45,
  "line_end": 120
}
```

### Refactor Plan Entry
```json
{
  "batch_id": 1,
  "modules": ["src/collectors/web_collector.py", "src/collectors/api_collector.py"],
  "strategy": "extract_functions",
  "estimated_smell_reduction": 8,
  "risk_level": "medium"
}
```

### Burn State
```json
{
  "status": "running",
  "mode": "100k",
  "target_budget": 100000,
  "estimated_spent": 0,
  "smells_before": 25,
  "smells_after": 0,
  "completed_modules": [],
  "pending_modules": ["module_a", "module_b"],
  "last_heartbeat": "2026-05-12T01:30:00",
  "cron_job_id": null
}
```

## 9. Tooling Stack

| Purpose | Tool | Fallback |
|---------|------|----------|
| Complexity metrics | radon | AST-based manual calculation |
| Duplication detection | jscpd | Pylint duplicate-code |
| Semantic smells | pylint | ruff check |
| Auto-fix low-hanging | ruff check --fix | Manual edit |
| Formatting | ruff format | black |
| Test runner | pytest | python -m unittest |
| Git isolation | git worktree | git stash + branch |

## 10. Error Handling Strategy

| Error | Response |
|-------|----------|
| Tool not installed | Install via pip, or fallback to AST-based detection |
| No tests found | Refactor with extra caution; flag for manual review |
| Tests fail after refactor | Rollback, try conservative approach (max 3) |
| 3 rollbacks on same module | Skip module, log reason, move to next |
| All modules skipped | Switch strategy or stop with report |
| API quota exhausted | Save state, output final report, graceful exit |

## 11. Success Metrics

- Smell count reduction (target: 30-50% for 100k tier, 60-80% for 1M tier)
- Test pass rate (must remain 100%)
- Maintainability Index improvement (target: +10 points average)
- Modules successfully refactored / total attempted

## 12. Out of Scope

- Multi-language support (v1 is Python-only)
- Automatic dependency upgrades
- Architectural rewrites (e.g., monolith → microservices)
- Security vulnerability patching (Bandit findings are reported but not auto-fixed)
- Database migration refactoring

## 13. Acceptance Criteria

1. `/burn-my-tokens-refactor` starts the flow and prompts for directory + focus
2. Smell detection runs and produces `smell_report.md`
3. Refactor plan groups smells into batches
4. Subagents refactor modules and preserve test passes
5. Failed refactors are rolled back and retried up to 3 times
6. State is saved to `.burn_state.json` after each batch
7. CronCreate resume works after session interruption
8. Final report shows before/after metrics
9. Stop command gracefully halts after current batch
10. No module is left in a broken state

## 14. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Refactor breaks hidden behavior | Comprehensive test suite requirement; rollback on failure |
| Over-zealous refactoring | Conservative retry strategy; skip if 3 failures |
| Token budget exceeded mid-refactor | State save + resume mechanism |
| False positives in smell detection | Human review of smell report; subagent validates before applying |
| Large codebase overwhelms context | AST-based context scoping; module-level subagents |

## 15. Future Enhancements

- TypeScript/JavaScript support
- Integration with GitHub PR workflow
- Automatic PR creation with diff
- Learning from user feedback on refactors
- Cross-project duplication detection
- Performance regression testing (benchmark before/after)
