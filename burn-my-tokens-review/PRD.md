# PRD: burn-my-tokens-review

## Product Requirements Document

**Version:** 1.1.0
**Date:** 2026-05-12
**Status:** L3 MVP

---

## 1. Problem Statement

Users with expiring coding plan quotas need a way to convert remaining tokens into durable, high-value output. The existing burn-my-tokens family covers:
- **burn-my-tokens**: Generates new L3 MVP projects from scratch
- **burn-my-tokens-refactor**: Automatically refactors and simplifies existing code
- **burn-my-tokens-testgen**: Automatically generates test coverage for existing code

**Missing piece**: A skill that performs **deep code reviews** on existing codebases, generating structured review reports with issue severity ratings, rationale, and concrete fix suggestions — but does NOT automatically apply fixes. This creates a **human-in-the-loop safety gate**: review first, then decide which fixes to apply.

### Why Review-First Matters
- **Safety**: Auto-refactoring can break working code; review gives the user visibility before changes
- **Learning**: Review reports teach developers about patterns and anti-patterns
- **Auditability**: Structured reports serve as documentation of code quality decisions
- **Prioritization**: Severity ratings help users focus on critical issues first

---

## 2. Solution Overview

**burn-my-tokens-review** is a Claude Code skill that:
1. Analyzes an existing codebase for quality issues across multiple axes (performance, readability, architecture, security)
2. Generates structured per-module review reports with severity ratings
3. Consolidates findings into a master report with executive summary and code health score
4. Does NOT modify source code — outputs reports only
5. Burns remaining tokens continuously until stopped or exhausted

### Key Differentiators
- **Read-only by design**: Never modifies source files
- **Structured severity system**: Critical / High / Medium / Low with likelihood x impact matrix
- **Concrete fix suggestions**: Every issue includes a code example of the fix
- **Code health scoring**: Quantitative before/after scores for tracking improvement
- **Parallel subagent review**: Multiple modules reviewed concurrently for speed

---

## 3. Target Users

- Developers with expiring coding plan quotas who want to audit code quality
- Teams preparing for code review sprints or technical debt reduction
- Solo developers who want an external-quality review of their codebase
- Users who want to identify issues before deciding which to fix (via refactor skill)

---

## 4. User Stories

### US-1: Quick Quality Check
As a developer with 10k tokens remaining, I want a fast scan of 1-2 critical files so I can see the most important issues without spending much budget.

### US-2: Module Deep Dive
As a developer with 100k tokens remaining, I want one module deeply reviewed so I can understand all quality issues and plan fixes.

### US-3: Full Codebase Audit
As a developer with 1M tokens remaining, I want 3-4 modules or my full codebase reviewed so I have a comprehensive quality report.

### US-4: Burn Until Clean
As a developer in burn mode, I want continuous review of my entire codebase until no more issues are found or my quota expires.

### US-5: Review Before Refactor
As a developer, I want to run review first, then selectively apply fixes via burn-my-tokens-refactor so I have control over what changes.

### US-6: Track Code Health Over Time
As a developer, I want code health scores before and after fixes so I can measure improvement.

---

## 5. Functional Requirements

### FR-1: Trigger Conditions
The skill must activate on:
- `/burn-my-tokens-review` command
- `/burn-my-tokens-review stop` / `停止审查` / `stop review` / `stop burn`
- User expressions of anxiety: "code review / need review / quality check / tokens about to expire"

### FR-2: Tier Selection
Present five tiers:
- `10k` — Quick scan + review 1-2 critical files
- `100k` — Standard review (~1 module deeply reviewed)
- `1M` — Deep review burn (~3-4 modules or full codebase)
- `10M` — Massive review burn (~8-12 modules or full codebase + cross-module analysis)
- `burn` — Burn until no more issues found or request fails

### FR-3: Context Gathering
Ask user:
- Which codebase directory should be reviewed?
- What focus areas? (performance / readability / architecture / security / all)

Auto-detect:
- Code structure (src/, top-level modules)
- Test framework (pytest, unittest, none)
- Language (Python, JavaScript, etc.)
- Existing coding standards (`.claude/rules/*.md`)

### FR-4: Code Analysis (Step 1)
- Read target files and build AST/structural understanding
- Identify: complexity hotspots, long functions, deep nesting, tight coupling, missing docstrings, type hint gaps, potential bugs, performance issues
- Output: `analysis_report.md`

### FR-5: Issue Prioritization (Step 2)
- Rank issues by severity (critical / high / medium / low) and impact
- Group into review batches
- Output: `review_plan.md`

### FR-6: Generate Reviews (Step 3)
- Launch subagents per module (max 2 parallel)
- Each subagent generates a detailed review document with:
  - File-level summary
  - Per-function issues with line numbers
  - Severity rating
  - Rationale (why this is a problem)
  - Concrete fix suggestion (code example)
  - Reference to best practices or project coding standards
- Write review to `review_reports/<module>_review.md`

### FR-7: Consolidation (Step 4)
- Merge all per-module reviews into `MASTER_REVIEW.md`
- Add executive summary with top 10 issues and overall code health score

### FR-8: Final Report (Step 5)
- Files reviewed
- Issues found by severity
- Code health score before/after (if re-reviewed)
- Known limitations

### FR-9: State Management
- Maintain `.burn_state.json` with standard burn state fields plus:
  - `focus_areas`
  - `files_reviewed`
  - `issues_found`
  - `code_health_score`

### FR-10: Resume Mechanism
- Use `CronCreate` for time-based resume
- Save state after each batch
- Restore context on resume

---

## 6. Non-Functional Requirements

### NFR-1: Never Stop on Your Own
- After completing a review batch, immediately check budget
- If budget not exhausted, begin next batch
- Only stop at: 95% tier threshold, API 429/403, user stop, or no more files to review

### NFR-2: Degrade on Error
- If a file is too complex to review, switch to high-level summary
- If AST parsing fails, fall back to line-based heuristics
- Never terminate the entire flow due to single file failure

### NFR-3: Progress Reports Must Not Block
- Embed progress in execution flow concisely
- Do not interrupt for "complete report"
- Detailed reports only on user query

### NFR-4: Budget Visualization
- Display `[Token Tracker]` after each batch
- Format: `Estimated consumed: XXk / Target: XXk | Issues: Y found / Z reviewed`

### NFR-5: Read-Only Safety
- Skill must NEVER write to source files
- Only write to `burn-my-tokens-review_output/` directory
- Explicit guard: check tool calls to prevent accidental source modification

---

## 7. Severity Classification System

### Severity Matrix

| Likelihood \ Impact | High | Medium | Low |
|---------------------|------|--------|-----|
| **High** | **Critical** | **High** | **Medium** |
| **Medium** | **High** | **Medium** | **Low** |
| **Low** | **Medium** | **Low** | **Low** |

### Severity Definitions

| Level | Definition | Examples |
|-------|-----------|----------|
| **Critical** | Could cause crashes, data loss, security vulnerabilities, or incorrect behavior in production | SQL injection, unhandled exceptions in critical paths, race conditions, missing input validation on public APIs |
| **High** | Significantly impacts maintainability, performance, or correctness | Deep nesting (>4 levels), functions >100 lines, tight coupling, missing error handling, type hint gaps in public APIs |
| **Medium** | Moderate impact on code quality or developer experience | Missing docstrings, inconsistent naming, magic numbers, moderate complexity (CC 11-20) |
| **Low** | Minor style or cosmetic issues | Trailing whitespace, import ordering, minor naming inconsistencies |

### Impact Axes

| Axis | High Impact | Medium Impact | Low Impact |
|------|------------|--------------|-----------|
| **Correctness** | Could produce wrong results or crash | Edge case handling missing | Theoretical issue only |
| **Performance** | O(n^2) where O(n) possible, memory leaks | Inefficient data structures | Micro-optimization |
| **Security** | Injection, XSS, auth bypass | Missing validation, exposed internals | Information disclosure |
| **Maintainability** | Untestable, tightly coupled, no docs | Complex but documented | Minor style issues |
| **Readability** | Incomprehensible without domain knowledge | Unclear variable names | Inconsistent formatting |

---

## 8. Code Health Score Algorithm

### Scoring Formula

```
code_health_score = 100 - penalty_total

penalty_total = (
    critical_issues * 10 +
    high_issues * 5 +
    medium_issues * 2 +
    low_issues * 0.5 +
    avg_cyclomatic_complexity_penalty +
    docstring_coverage_penalty +
    type_hint_coverage_penalty +
    test_coverage_penalty
)
```

### Component Penalties

| Metric | Penalty Calculation |
|--------|-------------------|
| Avg CC penalty | `max(0, (avg_cc - 10) * 2)` |
| Docstring coverage penalty | `(100 - docstring_coverage%) * 0.3` |
| Type hint coverage penalty | `(100 - type_hint_coverage%) * 0.3` |
| Test coverage penalty | `(100 - test_coverage%) * 0.2` |

### Score Interpretation

| Score | Rating | Description |
|-------|--------|-------------|
| 90-100 | Excellent | Production-ready, minimal issues |
| 70-89 | Good | Solid codebase, some improvements possible |
| 50-69 | Fair | Noticeable technical debt, needs attention |
| 30-49 | Poor | Significant issues, refactoring recommended |
| 0-29 | Critical | Major rework needed |

---

## 9. Output Specifications

### Directory Structure
```
burn-my-tokens-review_output/
├── .burn_state.json
├── analysis_report.md
├── review_plan.md
├── MASTER_REVIEW.md
└── review_reports/
    ├── module_a_review.md
    ├── module_b_review.md
    └── ...
```

### analysis_report.md Format
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
```

### Per-Module Review Format
```markdown
# Review: <module_name>

## File Summary
- Lines of code: X
- Functions: Y
- Classes: Z
- Overall severity: <critical/high/medium/low>
- Module health score: XX/100

## Issues

### Issue #1: <title>
- **Location**: `<function_name>` (lines X-Y)
- **Severity**: <critical/high/medium/low>
- **Category**: <correctness/performance/security/maintainability/readability>
- **Rationale**: <why this is a problem>
- **Current code**:
  ```python
  <snippet>
  ```
- **Suggested fix**:
  ```python
  <snippet>
  ```
- **Reference**: <link to coding standard or best practice>
```

### MASTER_REVIEW.md Format
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

---

## 10. Integration with Sibling Skills

### With burn-my-tokens-refactor
1. Run `/burn-my-tokens-review` first to identify issues
2. Review MASTER_REVIEW.md and decide which issues to fix
3. Run `/burn-my-tokens-refactor` with focus on specific modules
4. Re-run `/burn-my-tokens-review` to verify improvements

### With burn-my-tokens-testgen
1. Review may identify missing test coverage as an issue
2. Run `/burn-my-tokens-testgen` to fill gaps
3. Re-run review to update code health score

### With burn-my-tokens (new project generation)
1. Review existing projects for quality patterns
2. Use findings to inform new project structure
3. New projects start with higher baseline quality

---

## 11. Success Metrics

| Metric | Target |
|--------|--------|
| Review completion rate | >95% of targeted files reviewed |
| False positive rate | <20% (reported issues that are not real issues) |
| Report generation speed | <5 minutes per module in 10k tier |
| Code health score accuracy | Correlation >0.7 with human expert ratings |
| User satisfaction | Users report actionable findings in >80% of reviews |

---

## 12. Out of Scope

- **Automatic fix application**: This is intentionally out of scope; use burn-my-tokens-refactor
- **Multi-language support (v1)**: Python only for MVP; architecture supports extension
- **Real-time review**: Batch review only, not IDE-integrated
- **Security vulnerability scanning**: Heuristic only, not replacement for SAST tools
- **Performance profiling**: Static analysis only, not runtime profiling

---

## 13. Acceptance Criteria

- [ ] Skill activates on `/burn-my-tokens-review` command
- [ ] Tier selection works for all five tiers (10k, 100k, 1M, 10M, burn)
- [ ] Context gathering asks for directory and focus areas
- [ ] Auto-detection identifies code structure, test framework, and language
- [ ] Analysis report is generated with complexity hotspots and documentation gaps
- [ ] Review plan prioritizes issues by severity
- [ ] Subagents generate per-module reviews with line numbers, severity, rationale, and fix suggestions
- [ ] MASTER_REVIEW.md consolidates findings with executive summary and top 10 issues
- [ ] Code health score is calculated and displayed
- [ ] State file tracks progress with focus_areas, files_reviewed, issues_found
- [ ] Resume mechanism works via CronCreate
- [ ] Skill never modifies source files
- [ ] Graceful stop works on user command
- [ ] Degradation on error (complex files get high-level summary)
- [ ] Progress reports do not block execution
- [ ] Token tracker displays after each batch

---

## 14. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| LLM hallucinates issues that don't exist | Medium | Require code evidence for each issue; confidence scoring |
| Large files exceed token budget | Medium | Scope to functions; degrade to summary |
| Review reports are too verbose | Low | Structured templates; severity filtering |
| User expects auto-fixes | Low | Clear messaging: "review-only, use refactor for fixes" |
| Cron resume fails | Low | State file + user can trigger manual resume |

---

## 15. Future Enhancements

- Multi-language support (JavaScript, TypeScript, Go, Rust)
- Integration with GitHub PRs for automated review comments
- Diff-based review (review only changed files)
- Custom review rule configuration
- Historical trend tracking (code health over time)
- Integration with SAST tools (Bandit, Semgrep) for security findings
