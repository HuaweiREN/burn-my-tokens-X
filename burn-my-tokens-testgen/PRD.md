# burn-my-tokens-testgen — Product Requirements Document

**Version:** 1.1.0  
**Date:** 2026-05-12  
**Status:** L3 MVP

## Problem Statement

Developers accumulate technical debt in the form of missing test coverage. When token quotas near reset, this debt represents a wasted opportunity: remaining tokens could be converted into durable, high-value test coverage instead of expiring unused.

Existing test generation tools are either:
- **Too narrow**: Focus on single-file coverage without project-wide strategy
- **Too manual**: Require extensive human prompting per test file
- **Too fragile**: Generate tests that break on first run and require manual fixing

## Solution

**burn-my-tokens-testgen** is a Claude Code skill that autonomously burns remaining token quota to generate comprehensive test coverage for an existing codebase. It follows a systematic, tiered approach: analyze gaps, prioritize by risk, generate tests via parallel subagents, validate, and iterate until the budget is exhausted or coverage targets are met.

## User Stories

- As a developer with expiring tokens, I want to convert them into test coverage so my codebase quality improves permanently.
- As a project maintainer, I want an autonomous agent to identify uncovered critical paths and generate tests without my manual intervention.
- As a team lead, I want a coverage report showing before/after metrics and known limitations.

## Target Users

- Solo developers with Claude Code coding plans nearing reset
- Teams with legacy codebases lacking comprehensive tests
- Anyone who wants to improve code quality without manual test writing

## Core Features

### 1. Tiered Burn Modes

| Tier | Budget | Output |
|------|--------|--------|
| `10k` | ~10,000 tokens | Quick audit + 1-2 critical path tests |
| `100k` | ~100,000 tokens | Standard coverage push (~1 module fully covered) |
| `1M` | ~1,000,000 tokens | Deep coverage burn (~3-4 modules or full codebase) |
| `10M` | ~10,000,000 tokens | Massive coverage burn (~8-12 modules or enterprise codebase) |
| `burn` | Unlimited | Burn until coverage hits target or request fails |

### 2. Context Auto-Detection

- Detect existing test framework (pytest/unittest)
- Run `pytest --cov` to establish baseline coverage
- Discover code structure (`src/`, top-level modules)
- Read `.claude/rules/python-coding-standards.md` for style compliance

### 3. Coverage Gap Analysis

- Run coverage analysis to find uncovered lines/functions
- Identify critical paths: public APIs, complex functions, edge cases
- Output: `coverage_gap.md`

### 4. Risk-Based Test Strategy

- Prioritize uncovered paths by risk tier:
  1. Public APIs and entry points (highest)
  2. Complex functions (high cyclomatic complexity)
  3. Private helpers used by public APIs
  4. Edge cases and error handling
- Decide test types per gap: unit, boundary, property-based (hypothesis), integration
- Output: `test_strategy.md`

### 5. Parallel Test Generation via Subagents

- Launch subagents per module (max 2 parallel)
- Each subagent:
  - Reads module source code
  - Generates pytest tests with type hints and docstrings
  - Follows `.claude/rules/python-coding-standards.md`
  - Writes tests to `tests/test_<module>.py`
  - Runs tests; if fail, auto-fixes or degrades to simpler test

### 6. Validation & Iteration

- Run full test suite after each batch
- Run coverage report
- If coverage increased: continue to next gap
- If coverage stalled (no improvement after 2 rounds): switch strategy (e.g., integration tests instead of unit tests)

### 7. Burn Contract

1. **Never stop on your own** — keep generating tests until budget exhausted or coverage target met
2. **Degrade on error** — if a generated test fails and can't be fixed in 2 attempts, skip that gap and move to next
3. **Time-based resume** — save state to `.burn_state.json` and use CronCreate if response approaches time limit
4. **Budget visualization** — show `[Token Tracker]` after each module batch
5. **Progress reports must not block execution** — embed concise updates in execution flow

### 8. Resume Mechanism

- Save state to `.burn_state.json` after each module batch
- CronCreate every 3 minutes to auto-resume if heartbeat > 10 min old
- On resume: read state, restore context, continue from pending modules

## Out of Scope

- Non-Python languages (v1.0 is Python-only)
- GUI/frontend testing (Selenium, Playwright)
- Performance/load testing
- Security penetration testing
- Modifying source code to make it more testable (tests only, no source changes)

## Acceptance Criteria

- [ ] Skill activates on `/burn-my-tokens-testgen` command
- [ ] Auto-detects pytest/unittest and current coverage
- [ ] Generates `coverage_gap.md` and `test_strategy.md`
- [ ] Launches parallel subagents (max 2) per module
- [ ] Generated tests follow `.claude/rules/python-coding-standards.md`
- [ ] Tests pass before being declared complete
- [ ] Coverage report shows before/after metrics
- [ ] State file `.burn_state.json` is maintained throughout
- [ ] CronCreate resume works after session interruption
- [ ] Graceful stop on `/burn-my-tokens-testgen stop`

## Technical Architecture

```
Main Agent
├── Step 0: Context Gathering (user input + auto-detect)
├── Step 1: Coverage Analysis → coverage_gap.md
├── Step 2: Test Strategy → test_strategy.md
├── Step 3: Launch Subagents (max 2 parallel)
│   └── Subagent per module:
│       ├── Read source code
│       ├── Generate tests
│       ├── Write tests/test_<module>.py
│       ├── Run tests → fix or degrade
│       └── Report results
├── Step 4: Validation (full suite + coverage)
├── Step 5: Report (before/after, limitations)
└── Loop: Continue until budget exhausted or target met
```

## State File Format

```json
{
  "status": "running",
  "mode": "100k",
  "target_budget": 100000,
  "estimated_spent": 0,
  "target_coverage": 80,
  "current_coverage": 45,
  "completed_modules": [],
  "pending_modules": ["module_a", "module_b"],
  "last_heartbeat": "2026-05-12T01:30:00",
  "cron_job_id": null
}
```

## Metrics & Success Criteria

- Coverage improvement: target +20-50% depending on tier
- Test pass rate: 100% of generated tests must pass
- Degradation rate: <20% of gaps skipped due to unfixable failures
- Resume reliability: 100% successful resume from `.burn_state.json`
