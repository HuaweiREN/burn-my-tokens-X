# PRD: burn-my-tokens-research

## 1. Overview

**Product Name**: `burn-my-tokens-research`
**Family**: burn-my-tokens
**Version**: 1.1.0 (L3 MVP)
**Date**: 2026-05-12

### 1.1 Purpose
`burn-my-tokens-research` converts expiring coding plan quotas into durable, structured research reports. Unlike sibling skills that generate code, refactor, or test, this skill produces exclusively markdown research documents — comprehensive, citation-backed analyses of user-specified domains.

### 1.2 Target User
- Developers, product managers, or founders with expiring token quotas
- Users who need rapid but rigorous domain research without manual search
- Anyone who wants to turn idle compute into actionable intelligence

### 1.3 Success Criteria
- Research reports are generated with >=80% of factual claims carrying citations
- Reports follow a standardized template (Executive Summary, Key Findings, Market/Industry Analysis, Technology Landscape, Competitive Analysis, Trends & Forecasts, Gaps & Opportunities, Methodology, Sources)
- The skill supports unattended burning with auto-resume via CronCreate
- Graceful stop works reliably in MetaBot queue environments

---

## 2. User Stories

| ID | Story | Priority |
|----|-------|----------|
| US-01 | As a user with tokens about to expire, I want to type `/burn-my-tokens-research` and get a structured research report so that my quota is not wasted. | P0 |
| US-02 | As a researcher, I want to select a tier (10k/100k/1M/10M/burn) so that I control the depth and cost of the research. | P0 |
| US-03 | As a user, I want every factual claim in the report to have a source citation so that I can verify and trust the findings. | P0 |
| US-04 | As a user in a MetaBot queue environment, I want to send "stop" and have the burn halt gracefully so that I retain control. | P0 |
| US-05 | As a user whose session times out, I want the research to auto-resume so that I don't lose progress. | P0 |
| US-06 | As a strategist, I want the report to include market analysis, competitive landscape, and trend forecasts so that I can make decisions. | P1 |
| US-07 | As a user, I want the skill to never generate code — only research documents — so that output is focused and safe. | P0 |
| US-08 | As a user, I want contradictions and gaps to be documented, not hidden, so that I understand the limits of the research. | P1 |

---

## 3. Functional Requirements

### 3.1 Trigger & Startup

**FR-01**: The skill MUST activate on `/burn-my-tokens-research` and variants.
**FR-02**: The skill MUST present tier selection (10k, 100k, 1M, 10M, burn) and wait for user input.
**FR-03**: The skill MUST gather: target domain, specific questions, output depth, and requested analysis sections.
**FR-04**: The skill MUST record all gathered context in `.burn_state.json`.
**FR-05**: The skill MUST NOT proceed without at least a target domain (Mode D must re-prompt).

### 3.2 Research Scoping

**FR-06**: The skill MUST decompose the domain into subtopics based on tier:
- 10k: 1 subtopic
- 100k: 1 domain, 3-4 subtopics
- 1M: 3-4 related domains, 8-12 subtopics
- 10M: 8-12 domains or full industry landscape, 16-24+ subtopics
- burn: unlimited until stopped

**FR-07**: The skill MUST generate a `research_outline.md` with core research questions, subtopic definitions, and research strategy.

### 3.3 Information Gathering

**FR-08**: The skill MUST launch subagents per subtopic, max 2 parallel (3-4 in 10M mode, 1 in 10k mode).
**FR-09**: Each subagent MUST use `WebSearch` as its primary research tool.
**FR-10**: Each subagent MUST cite sources with URLs for every factual claim.
**FR-11**: Each subagent MUST write findings to `findings/<subtopic>.md`.
**FR-12**: Subagents MUST assess source reliability (Tier 1/2/3) and document it.

### 3.4 Synthesis & Analysis

**FR-13**: The Main Agent MUST read all findings and produce `synthesis_notes.md` covering patterns, gaps, contradictions, and implications.
**FR-14**: The Main Agent MUST update the research outline if new subtopics are discovered and budget allows.

### 3.5 Report Generation

**FR-15**: The skill MUST generate `RESEARCH_REPORT.md` as the master report with all standard sections.
**FR-16**: The skill MUST generate per-subtopic reports in `findings/`.
**FR-17**: The report template MUST include: Executive Summary, Key Findings, Market/Industry Analysis, Technology Landscape, Competitive Analysis, Trends & Forecasts, Gaps & Opportunities, Methodology, Sources.
**FR-18**: The skill MUST generate `RESEARCH_BURN_REPORT.md` as a summary of the burn session.

### 3.6 Validation

**FR-19**: The skill MUST produce `validation_report.md` checking coverage, citations, source diversity, and confidence calibration.
**FR-20**: If validation reveals significant gaps and budget remains, the skill MUST launch gap-filling subagents.

### 3.7 Graceful Stop & Resume

**FR-21**: On stop command, the skill MUST set `status == "stopped"`, confirm stop, and call `CronDelete`.
**FR-22**: The skill MUST end the response between batches to allow queued stop commands to process.
**FR-23**: The skill MUST use `CronCreate` (every 3 minutes) for auto-resume.
**FR-24**: The state file MUST include research-specific fields: `research_outline`, `completed_subtopics`, `pending_subtopics`, `sources_count`.

---

## 4. Non-Functional Requirements

### 4.1 Output Constraints
**NFR-01**: The skill MUST NOT generate source code, scripts, or executable files.
**NFR-02**: All output MUST be markdown (.md) documents.
**NFR-03**: Output MUST be written to `burn-my-tokens-research_output/<task_name>/`.

### 4.2 Performance
**NFR-04**: The skill SHOULD complete 10k tier research in a single response.
**NFR-05**: The skill SHOULD process at least 2 subtopics per response in 100k+ tiers.

### 4.3 Reliability
**NFR-06**: The skill MUST degrade on error (broader queries → WebFetch on known URLs → "insufficient public data"), never terminate.
**NFR-07**: The skill MUST handle paywalled sources, search failures, and conflicting information gracefully.

### 4.4 Quality
**NFR-08**: >=80% of factual claims MUST have inline citations.
**NFR-09**: Sources MUST span at least 3 different domains.
**NFR-10**: Contradictions MUST be documented, not resolved by fiat.

---

## 5. Architecture

### 5.1 Components

```
Main Agent
├── Startup Flow (Tier Selection, Context Gathering)
├── Step 1: Research Scoping → research_outline.md
├── Step 2: Information Gathering → launch Subagents
│   └── Subagent (per subtopic)
│       ├── Plan Search Strategy
│       ├── Execute WebSearch + WebFetch
│       ├── Cross-Reference & Validate
│       ├── Synthesize Findings
│       └── Write findings/<subtopic>.md
├── Step 3: Synthesis & Analysis → synthesis_notes.md
├── Step 4: Report Generation → RESEARCH_REPORT.md
├── Step 5: Validation → validation_report.md
└── Burn Report → RESEARCH_BURN_REPORT.md
```

### 5.2 State Management

**State file**: `.burn_state.json`
- `status`: running | stopped
- `mode`: 10k | 100k | 1M | 10M | burn
- `target_budget`: integer
- `estimated_spent`: integer
- `domain`: string
- `research_questions`: string[]
- `output_depth`: brief | standard | comprehensive
- `sections_requested`: string[]
- `research_outline`: path to outline file
- `completed_subtopics`: string[]
- `pending_subtopics`: string[]
- `sources_count`: integer
- `sources_by_tier`: { tier_1, tier_2, tier_3 }
- `last_heartbeat`: ISO timestamp
- `cron_job_id`: string

### 5.3 Resume Mechanism

- **CronCreate**: `1-59/3 * * * *`, durable=true
- **Prompt**: Read state file → if running and heartbeat >10min old → `/burn-my-tokens-research resume`
- **Cleanup**: `CronDelete` on burn completion or graceful stop

---

## 6. Data Model

### 6.1 research_outline.md

```markdown
# Research Outline: <domain>
## Date, Tier, Output Depth, Sections
## Core Research Questions (3-7)
## Subtopics (with scope, entities, info types, priority, estimated search rounds)
## Research Strategy
```

### 6.2 findings/<subtopic>.md

```markdown
# Findings: <subtopic>
## Summary
## Key Findings (with inline citations)
## Data Points (table)
## Source Reliability Assessment (table)
## Gaps & Uncertainties
## Citations
```

### 6.3 RESEARCH_REPORT.md

```markdown
# Research Report: <domain>
## Metadata
## Executive Summary
## Key Findings
## Market / Industry Analysis
## Technology Landscape
## Competitive Analysis
## Trends & Forecasts
## Gaps & Opportunities
## Methodology
## Sources
```

### 6.4 validation_report.md

```markdown
# Validation Report
## Coverage (table per RQ)
## Citation Audit (table per section)
## Source Diversity
## Gaps Documented
## Overall Quality Rating
```

---

## 7. UI / Interaction Design

### 7.1 Startup Conversation

```
User: /burn-my-tokens-research
Claude: 🔥 Research Burn Mode

Select your tier:
- 10k — Quick snapshot (~1 topic, 1-2 searches)
- 100k — Standard research (~1 domain, 5-8 searches)
- 1M — Deep research (~3-4 domains, 15-25 searches)
- 10M — Massive research (~8-12 domains, 50+ searches)
- burn — Burn until stopped

What is your target domain or field?
> autonomous vehicle LiDAR market

What specific questions do you want answered?
> Who are the top 5 competitors and their pricing?

Output depth? (brief / standard / comprehensive)
> standard

Which analysis sections? (market / tech / competitive / trends / all)
> all

Claude: [Auto-detects nothing — research is domain-driven]
Claude: Starting research burn...
```

### 7.2 Progress Reporting

Between batches:
```
[Token Tracker] Estimated consumed: 4.2k / Target: 10k | Subtopics: 2 completed / 2 pending | Sources: 18 collected
- Completed: market_sizing, key_players
- Pending: technology_trends, regulatory_landscape
```

### 7.3 Final Report

```
Research burn complete!

Domain: autonomous vehicle LiDAR market
Tier: 10k
Output depth: standard

Subtopics researched: 4 / 4 planned
Total sources: 31 unique URLs

Reports generated:
- RESEARCH_REPORT.md
- findings/market_sizing.md
- findings/key_players.md
- findings/technology_trends.md
- findings/regulatory_landscape.md
- research_outline.md
- synthesis_notes.md
- validation_report.md

Output directory: burn-my-tokens-research_output/lidar_research_20260512/
```

---

## 8. Error Handling & Edge Cases

| Scenario | Handling |
|----------|----------|
| WebSearch returns no results | Reformulate queries → broader terms → adjacent topics → "insufficient public data" |
| Paywalled source | Capture headline/abstract → search for open alternative → cite as paywalled if no alternative |
| Conflicting sources | Document both → assess credibility → present with confidence levels |
| WebFetch fails | Retry once → use snippet → lower confidence → search for alternative |
| Domain has sparse public data | Document opacity → search proxy signals (patents, funding) → note limitation |
| User sends stop mid-batch | Finish current subagent → save state → end response → next turn reads stop |
| Session times out | Save state → CronCreate resumes in 3 minutes |
| API 429/403 | Catch exception → output final summary → graceful exit |
| User asks for code during burn | Politely redirect to `/burn-my-tokens` |

---

## 9. Open Questions

1. Should we support multi-language research (e.g., searching Chinese sources for China-specific domains)?
2. Should we integrate with `metamemory` to save research findings for future sessions?
3. Should we support PDF or HTML output in addition to markdown?
4. Should we add a "research delta" feature to update an existing report with new information?

---

## 10. Changelog

| Date | Version | Change | Author |
|------|---------|--------|--------|
| 2026-05-12 | 1.0.0 | Initial L3 MVP PRD | Claude |
