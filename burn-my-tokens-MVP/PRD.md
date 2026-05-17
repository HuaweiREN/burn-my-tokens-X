# burn-my-tokens-MVP — Product Requirements Document

## 1. Overview

**Product Name**: burn-my-tokens-MVP
**Family**: burn-my-tokens
**Version**: 1.1.0
**Status**: L3 MVP
**Date**: 2026-05-12

### 1.1 Purpose

burn-my-tokens-MVP converts expiring coding plan quotas into runnable L3 MVP projects. It analyzes the user's existing projects and domain interests, discovers niche extension directions via WebSearch, and launches subagents to produce complete projects with PRD, source code, tests, and verification runs.

### 1.2 Target User

- Developers with existing projects seeking adjacent product ideas
- Solo builders who want to rapidly prototype new concepts before token quotas reset
- Technical founders exploring multiple MVP directions in parallel
- Anyone with expiring tokens who wants durable, runnable code output

### 1.3 Key Differentiators

| Dimension | burn-my-tokens-MVP | burn-my-tokens-idea | Generic code generator |
|-----------|-------------------|---------------------|------------------------|
| **Output** | PRD + src/ + tests/ + verification | Structured idea documents only | Code only |
| **Context-aware** | Reads existing projects & memories | Domain-only | None |
| **Auto-degradation** | L3 → L2 → L1 on failure | N/A (no code) | No degradation |
| **Memory** | Writes burn_memory/*.md per project | N/A | None |
| **Niche discovery** | WebSearch + self-critique + critique-agent | WebSearch + evaluation | Usually direct prompt |

## 2. User Stories

### 2.1 Primary User Stories

**US-1: Explore from existing projects**
As a developer with existing codebases, I want to provide my project directories so that the skill discovers logical extension directions I haven't thought of.

**US-2: Domain-driven greenfield**
As a founder with a target domain, I want to skip existing project analysis and directly research niche directions so that I get fresh ideas unrelated to my current work.

**US-3: Generate runnable MVPs**
As a user with 10k+ tokens remaining, I want to produce 1-4 complete L3 MVP projects so that I have tangible codebases to iterate on later.

**US-4: Parallel exploration**
As a user with ample tokens, I want 2 projects generated in parallel so that I maximize output before quota resets.

**US-5: Graceful stop**
As a user in a chat bot environment, I want to send "stop" and have the burn halt cleanly after the current project, preserving all completed work.

**US-6: Resume after interruption**
As a user whose session timed out, I want the burn to auto-resume so that no tokens are wasted and the next project starts seamlessly.

### 2.2 System User Stories

**SYS-1: Self-critique filtering**
As the system, I want to eliminate candidates with >50% overlap with existing projects so that outputs are genuinely novel.

**SYS-2: Critique-agent validation**
As the system, I want an independent agent to score and rank candidate directions so that low-quality ideas are filtered before subagent allocation.

**SYS-3: Automatic degradation**
As the system, I want subagent failures to trigger L3→L2→L1 degradation so that the burn flow never terminates prematurely.

**SYS-4: Memory accumulation**
As the system, I want each completed project recorded in burn_memory so that future burns can build on past outputs.

## 3. Functional Requirements

### 3.1 Startup Flow (FR-1)

**FR-1.1**: The system shall present 5 tier options: 10k, 100k, 1M, 10M, burn.  
**FR-1.2**: The system shall collect: existing project directories (optional), technical domain / desired direction (optional).  
**FR-1.3**: The system shall auto-detect and confirm the user's Claude memory directory path.  
**FR-1.4**: The system shall support 4 execution modes (A/B/C/D) based on what the user provided.  
**FR-1.5**: The system shall record all inputs in `.burn_state.json` under `memory_path` for session reuse.

### 3.2 Existing Project Analysis (FR-2)

**FR-2.1**: The system shall read user-confirmed directories and look for `*/PRD.md`, `*/PROGRESS.md`, `*/README.md`.  
**FR-2.2**: The system shall extract tech stack, problems solved, and uncovered adjacent domains per project.  
**FR-2.3**: The system shall read `burn-my-tokens-MVP_output/burn_memory/*.md` and `<memory_path>/*.md` for user background.  
**FR-2.4**: Missing or empty directories shall be logged and skipped without terminating.

### 3.3 Direction Generation (FR-3)

**FR-3.1**: The system shall use WebSearch to research 2-3 potential niche directions.  
**FR-3.2**: The system shall generate 3-5 candidate projects, each with name, one-sentence description, technical connection, and estimated token cost.  
**FR-3.3**: The Main Agent shall self-critique to eliminate >50% overlaps, infeasible concepts, and unmeetable dependencies.  
**FR-3.4**: A critique-agent shall score each candidate (1-10), recommend eliminate/keep, and produce priority ranking.

### 3.4 Project Selection (FR-4)

**FR-4.1**: The system shall present the final candidate list sorted by priority.  
**FR-4.2**: Semi-auto mode: the user selects project numbers to execute.  
**FR-4.3**: Full-auto mode: all directions are auto-selected and execution begins immediately.

### 3.5 Subagent Allocation (FR-5)

**FR-5.1**: The system shall launch up to 2 subagents in parallel for the first batch.  
**FR-5.2**: After batch completion, the system shall evaluate remaining budget before launching the next batch.  
**FR-5.3**: Each subagent workspace shall be `burn-my-tokens-MVP_output/<project_name>/`.

### 3.6 Subagent L3 MVP Workflow (FR-6)

**FR-6.1**: Step 1 — Web Research (~15-20% budget): research technical solutions, output `research.md`.  
**FR-6.2**: Step 2 — Write PRD (~15-20% budget): write `PRD.md` with Problem Statement, Solution, User Stories, Implementation Decisions, Testing Decisions, Out of Scope, Acceptance Criteria.  
**FR-6.3**: Step 3 — Core Development (~40-50% budget): implement MVP code, output `src/`, `requirements.txt`, runnable `main.py`.  
**FR-6.4**: Step 4 — Unit Tests (~10-15% budget): pytest covering core functionality, output `tests/`.  
**FR-6.5**: Step 5 — Verification Run (~5-10% budget): run `main.py`, record results. If run fails, auto-degrade to L2.  
**FR-6.6**: Step 6 — Write PROGRESS.md (~5% budget): record milestones, known issues, how to run.

### 3.7 Monitoring & Reporting (FR-7)

**FR-7.1**: Subagents shall report progress after each step.  
**FR-7.2**: The Main Agent shall aggregate progress and display token tracker after each project.  
**FR-7.3**: The user shall be able to query "progress" or "status" at any time.

### 3.8 Loop Check (FR-8)

**FR-8.1**: Tier modes: at 95% budget, pause and output summary; if budget remains, return to Step 2.  
**FR-8.2**: Burn mode: continuously return to Step 2 until API 429/403 or user stop.

### 3.9 Memory Updates (FR-9)

**FR-9.1**: After each project completes, write `burn-my-tokens-MVP_output/burn_memory/<project_name>.md` with metadata, problem solved, tech stack, relationship to existing projects, known limitations, and extension potential.

### 3.10 Graceful Stop (FR-10)

**FR-10.1**: Stop commands set `status = "stopped"` in `.burn_state.json`.  
**FR-10.2**: Stop triggers `CronDelete` and outputs confirmation with project count and estimated tokens.  
**FR-10.3**: The system ends response between projects to allow queued stop commands to process.

### 3.11 Resume (FR-11)

**FR-11.1**: On first startup, create a `CronCreate` resume task (every 3 minutes).  
**FR-11.2**: Resume reads `.burn_state.json` and restores `estimated_spent`, `completed_projects`, `pending_projects`, `memory_path`.  
**FR-11.3**: On completion, call `CronDelete` to clean up.

## 4. Non-Functional Requirements

### 4.1 Performance

**NFR-1**: Each subagent L3 MVP should fit within 20k-50k tokens; L2 within 5k-15k; L1 within 2k-5k.  
**NFR-2**: The Main Agent should save state and end response if execution exceeds 4 minutes.  
**NFR-3**: Each WebSearch call should target 500-1500 tokens.

### 4.2 Quality

**NFR-4**: Generated code must follow `.claude/rules/python-coding-standards.md`.  
**NFR-5**: All L3 projects must include `requirements.txt` and a runnable `main.py`.  
**NFR-6**: Unit tests must pass before a project is declared complete.

### 4.3 Safety

**NFR-7**: The system must never modify files outside `burn-my-tokens-MVP_output/`.  
**NFR-8**: On subagent failure, degradation must be attempted before skip.  
**NFR-9**: Verification run failures must be documented in `PROGRESS.md` with failure reason.

### 4.4 Usability

**NFR-10**: Output project names must be short, lowercase, no underscores.  
**NFR-11**: Token tracker must be displayed after every project.  
**NFR-12**: Final report must fit in a concise chat card.

## 5. System Architecture

### 5.1 Component Diagram

```
+-------------------+         +-------------------+         +-------------------+
|   User Input      |-------->|   Main Agent      |<--------|   .burn_state.json |
|  (/burn-my-tokens-|         |  (orchestrator)   |         |  (state persistence)|
|   MVP [stop])     |         +-------------------+         +-------------------+
+-------------------+              |    |    |
                                   |    |    |
                        +----------+    |    +--------------+
                        |               |                   |
               +--------v--------+  +---v-----------+  +----v----------+
               | Project Analysis|  |  Subagent 1   |  |  Subagent 2   |
               |  (PRD/README)   |  | (L3 MVP gen)  |  | (L3 MVP gen)  |
               +--------+--------+  +-------+-------+  +-------+-------+
                        |                  |                   |
               +--------v--------+         |                   |
               | Direction List  |         |                   |
               |  (WebSearch)    |         |                   |
               +--------+--------+         |                   |
                        |                  |                   |
               +--------v--------+         |                   |
               | Self-Critique   |         |                   |
               | + Critique-Agent|         |                   |
               +--------+--------+         |                   |
                        |                  |                   |
               +--------v--------+         |                   |
               | Present & Confirm|        |                   |
               +--------+--------+         |                   |
                        |                  |                   |
                        +------------------+-------------------+
                                           |
                                 +---------v---------+
                                 | burn_memory/*.md  |
                                 | (project records) |
                                 +-------------------+
```

### 5.2 Data Flow

1. User triggers skill → Main Agent collects inputs → Writes `.burn_state.json`
2. Main Agent reads existing projects + memories → Extracts tech stack and gaps
3. Main Agent researches niche directions via WebSearch → Generates candidate list
4. Main Agent self-critiques + critique-agent reviews → Produces prioritized list
5. User confirms (or full-auto) → Main Agent launches subagents (max 2 parallel)
6. Each subagent executes: Research → PRD → Dev → Tests → Verification → PROGRESS
7. Main Agent monitors, reports progress, updates token tracker
8. On project completion: write burn_memory, check budget, loop or stop

## 6. Output Specifications

### 6.1 Directory Structure

```
burn-my-tokens-MVP_output/
├── .burn_state.json
├── burn_memory/
│   ├── <project_a>.md
│   └── <project_b>.md
└── <project_name>/
    ├── research.md
    ├── PRD.md
    ├── src/
    ├── tests/
    ├── requirements.txt
    ├── main.py
    └── PROGRESS.md
```

### 6.2 File Formats

- All markdown files use standard markdown with tables and headers
- Source code follows Python 3.10+ with type hints and Google-style docstrings
- `requirements.txt` lists all pip-installable dependencies

### 6.3 State File Schema

```json
{
  "status": "running | stopped | completed",
  "mode": "10k | 100k | 1M | 10M | burn",
  "target_budget": 10000,
  "estimated_spent": 0,
  "completed_projects": ["project_a"],
  "pending_projects": ["project_b", "project_c"],
  "memory_path": "~/.claude/projects/xxx/memory",
  "last_heartbeat": "ISO8601",
  "cron_job_id": "string"
}
```

## 7. Error Handling

### 7.1 Degradation Matrix

| Failure | First Attempt | Second Attempt | Third Attempt |
|---------|---------------|----------------|---------------|
| Subagent L3 fails | Degrade to L2 (code skeleton + PRD) | Degrade to L1 (PRD only) | Skip project |
| Verification run fails | Document in PROGRESS.md, keep as L2 | N/A | N/A |
| WebSearch fails | Retry once | Skip research, use known patterns | Continue with limited research |
| Existing project directory missing | Log and skip | N/A | N/A |

### 7.2 Abort Conditions

The system shall abort (with explanatory message) only when:
- User provides neither directories nor domain (Mode D failure)
- All candidates eliminated after critique
- API returns persistent 403/429 errors
- User triggers graceful stop

## 8. Token Budget Estimates

| Tier | Target Budget | Estimated Output |
|------|---------------|------------------|
| 10k | 10,000 tokens | ~1 L1 research+PRD or L2 attempt |
| 100k | 100,000 tokens | ~1 L3 MVP |
| 1M | 1,000,000 tokens | ~3-4 L3 MVPs |
| 10M | 10,000,000 tokens | ~8-12 L3 MVPs or full product line |
| burn | unlimited | Until stopped |

## 9. Success Metrics

### 9.1 Output Quality Metrics

- **L3 delivery rate**: % of subagents that produce runnable main.py (target: >= 70%)
- **Test pass rate**: % of projects with all tests passing (target: 100% for L3)
- **Novelty rate**: % of projects with <50% overlap with existing work (target: 100%)

### 9.2 Process Metrics

- **Resume success rate**: % of interrupted burns that resume correctly (target: 100%)
- **Graceful stop latency**: Projects between stop command and actual stop (target: <= 1)
- **Token efficiency**: Projects delivered per 10k tokens (target: >= 1 for 10k+ modes)

## 10. Future Enhancements (Post-MVP)

- **FE-1**: Multi-language support (JavaScript/TypeScript, Go, Rust MVPs)
- **FE-2**: Integration with external repos (GitHub clone + analyze)
- **FE-3**: Custom project templates (user-defined scaffolding)
- **FE-4**: Cross-project dependency detection (avoid generating overlapping projects across burns)
- **FE-5**: Auto-deploy (generate Dockerfile / Vercel config)

## 11. Dependencies

- Claude Code with WebSearch and Agent capabilities
- CronCreate / CronDelete for resume mechanism
- File system write access to `burn-my-tokens-MVP_output/`
- Python 3.10+ environment for generated projects

## 12. Open Questions

1. Should we support non-Python project generation (e.g., web apps with React + FastAPI)?
2. Should we integrate with `metamemory` to persist burn_memory across long-term sessions?
3. How should we handle users with no existing projects and no domain preference?
4. Should L2/L1 projects also generate basic code skeletons, or just documents?

---

**Document Owner**: burn-my-tokens family
**Last Updated**: 2026-05-12
**Change Log**: See skill version history
