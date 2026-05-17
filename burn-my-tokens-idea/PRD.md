# burn-my-tokens-idea — Product Requirements Document

## 1. Overview

**Product Name**: burn-my-tokens-idea  
**Family**: burn-my-tokens  
**Version**: 1.1.0  
**Status**: L3 MVP  
**Date**: 2026-05-12  

### 1.1 Purpose

burn-my-tokens-idea converts expiring coding plan quotas into durable, high-value structured idea documents. It performs deep structured ideation — generating, evaluating, and refining niche creative ideas and concepts in a user-specified domain. Unlike project-generation skills, it produces idea cards, concept briefs, and feasibility analyses but NEVER generates source code.

### 1.2 Target User

- Solo founders exploring niche opportunities before committing to build
- Product managers seeking differentiated feature concepts
- Content creators looking for underserved content angles
- Intrapreneurs identifying new business lines within constraints
- Anyone with expiring tokens who wants durable creative output

### 1.3 Key Differentiators

| Dimension | burn-my-tokens-idea | burn-my-tokens (project gen) | Generic brainstorm |
|-----------|---------------------|------------------------------|--------------------|
| **Output** | Structured idea documents | PRD + source code | Unstructured list |
| **Depth** | Multi-axis evaluation + refinement | Implementation-focused | Surface-level |
| **Niche focus** | Mandatorily specific | Can be broad | Usually generic |
| **Feasibility** | Scored and validated | Assumed feasible | Ignored |
| **Code** | Never generates | Core deliverable | N/A |
| **Cross-pollination** | Hybrid concept generation | N/A | N/A |

## 2. User Stories

### 2.1 Primary User Stories

**US-1: Quick ideation sprint**  
As a user with limited tokens, I want to run a 10k ideation burn so that I can get 3-5 niche ideas in my domain within minutes.

**US-2: Deep niche exploration**  
As a founder considering a market, I want to run a 100k-1M burn so that I get evaluated, validated ideas with concept briefs I can pitch or research further.

**US-3: Massive industry mapping**  
As a VC or product strategist, I want to run a 10M burn so that I get a full industry map with 80+ evaluated ideas and hybrid concepts.

**US-4: Graceful stop**  
As a user in a chat bot environment, I want to send "stop" and have the burn halt cleanly after the current batch, preserving all progress.

**US-5: Resume after interruption**  
As a user whose session timed out, I want the burn to auto-resume so that no tokens are wasted and progress continues seamlessly.

**US-6: Idea-to-project pipeline**  
As a user who found a winning idea, I want to hand it off to `/burn-my-tokens` so that it becomes a full project with PRD and code.

### 2.2 System User Stories

**SYS-1: Subagent quality gate**  
As the system, I want subagents to reject generic ideas so that only niche, specific concepts enter the evaluation pipeline.

**SYS-2: Competitive validation**  
As the system, I want WebSearch validation of top ideas so that users don't waste time on me-too concepts.

**SYS-3: Cross-pollination**  
As the system, I want to identify hybrid combinations so that novel concepts emerge from idea recombination.

## 3. Functional Requirements

### 3.1 Startup Flow (FR-1)

**FR-1.1**: The system shall present 5 tier options: 10k, 100k, 1M, 10M, burn.  
**FR-1.2**: The system shall collect: domain, constraints, idea type, audience, emphasis preferences.  
**FR-1.3**: The system shall auto-detect if the domain is too broad and require narrowing.  
**FR-1.4**: The system shall record all inputs in `.burn_state.json`.  
**FR-1.5**: The system shall support 4 execution modes (A/B/C/D) based on what the user provided.

### 3.2 Domain Analysis (FR-2)

**FR-2.1**: The system shall use WebSearch to research market landscape, pain points, trends, and competition.  
**FR-2.2**: The system shall output `domain_analysis.md` with: market maturity, pain point clusters, underserved segments, adjacent niches, trend vectors.  
**FR-2.3**: The system shall cite at least 3 research sources with URLs.  
**FR-2.4**: The system shall flag overly broad domains and suggest narrowing directions.

### 3.3 Idea Generation (FR-3)

**FR-3.1**: The system shall launch subagents per niche angle (max 2 parallel, 3-4 in 10M mode, 1 in 10k mode).  
**FR-3.2**: Each subagent shall use structured creativity techniques (constraint-first, trend-riding, edge-exploitation, cross-domain移植, anti-pattern inversion).  
**FR-3.3**: Each idea shall pass the NICHE test before being written.  
**FR-3.4**: Each subagent shall output `raw_ideas/<angle>.md` with structured idea entries.  
**FR-3.5**: Target idea counts: 3-5 (10k), 8-12 (100k), 25-40 (1M), 80+ (10M).

### 3.4 Idea Evaluation (FR-4)

**FR-4.1**: The Main Agent shall read all raw ideas and score each on 5 axes (1-10): feasibility, novelty, market_size, differentiation, personal_fit.  
**FR-4.2**: Overall score shall use weighted formula: feasibility*0.25 + novelty*0.25 + market_size*0.20 + differentiation*0.20 + personal_fit*0.10.  
**FR-4.3**: Top ideas (score >= 7.0) shall undergo WebSearch validation for originality and demand evidence.  
**FR-4.4**: Validation failures shall trigger score downgrades (1-3 points) with documented reasons.  
**FR-4.5**: Output shall be `IDEA_CARDS.md` with ranked table and score distribution.

### 3.5 Idea Refinement (FR-5)

**FR-5.1**: Top ideas shall be refined into structured concept briefs with all required sections.  
**FR-5.2**: Refinement count: top 3 (10k), top 5 (100k), top 10 (1M), top 15 (10M).  
**FR-5.3**: Each concept brief shall include: name, one-liner, problem, solution, target user, differentiation, tech stack suggestion, MVP scope, risks, next steps, validation notes.  
**FR-5.4**: Tech stack suggestions shall remain high-level — no code generation.  
**FR-5.5**: Output shall be `TOP_PICKS.md`.

### 3.6 Cross-Pollination (FR-6)

**FR-6.1**: The system shall analyze top idea pairs for combination potential.  
**FR-6.2**: Each hybrid shall be scored on synergy (1-10), feasibility impact, and novelty boost.  
**FR-6.3**: Promising hybrids (synergy >= 7) shall get mini concept briefs.  
**FR-6.4**: Output shall append to `TOP_PICKS.md`.

### 3.7 Reporting (FR-7)

**FR-7.1**: Final report shall include: domain summary, angles explored, idea counts, top 3 ideas, score distribution, next steps.  
**FR-7.2**: Report shall be output in chat and written to `IDEA_BURN_REPORT.md`.  
**FR-7.3**: Progress reports shall be concise and embedded in execution flow, never blocking.

### 3.8 Graceful Stop (FR-8)

**FR-8.1**: Stop commands shall set `status = "stopped"` in `.burn_state.json`.  
**FR-8.2**: Stop shall trigger `CronDelete` to remove resume jobs.  
**FR-8.3**: Stop confirmation shall summarize progress (angles, ideas, tokens).  
**FR-8.4**: The system shall end response between batches to allow queued stop commands to process.

### 3.9 Resume (FR-9)

**FR-9.1**: On first startup, the system shall create a `CronCreate` resume task (every 3 minutes).  
**FR-9.2**: Resume shall read `.burn_state.json` and restore all context.  
**FR-9.3**: Resume shall continue from the last completed batch.  
**FR-9.4**: On completion, the system shall call `CronDelete` to clean up.

## 4. Non-Functional Requirements

### 4.1 Performance

**NFR-1**: Each subagent ideation batch should complete within reasonable token budget (3k-8k tokens per angle).  
**NFR-2**: Domain analysis should use 2-4 WebSearch calls maximum.  
**NFR-3**: The system should save state and end response if execution exceeds 4 minutes.

### 4.2 Quality

**NFR-4**: At least 70% of generated ideas must be original (no direct competitor with same approach).  
**NFR-5**: All ideas must pass the NICHE test.  
**NFR-6**: Concept briefs must be specific enough that a reader could sketch an MVP plan.

### 4.3 Safety

**NFR-7**: The system must NEVER generate source code, implementation files, or prototypes.  
**NFR-8**: The system must NEVER modify existing files outside the output directory.  
**NFR-9**: WebSearch validation must not leak user-sensitive information in queries.

### 4.4 Usability

**NFR-10**: Output documents must use clear markdown formatting with tables and headers.  
**NFR-11**: Token tracker must be displayed after every batch.  
**NFR-12**: Final report must fit in a concise chat card (Feishu/Telegram compatible).

## 5. System Architecture

### 5.1 Component Diagram

```
+-------------------+         +-------------------+         +-------------------+
|   User Input      |-------->|   Main Agent      |<--------|   .burn_state.json |
|  (/burn-my-tokens-|         |  (orchestrator)   |         |  (state persistence)|
|   idea [stop])    |         +-------------------+         +-------------------+
+-------------------+              |    |    |
                                   |    |    |
                    +--------------+    |    +--------------+
                    |                   |                   |
           +--------v--------+  +-------v-------+  +--------v--------+
           | Domain Analysis |  |  Subagent 1   |  |  Subagent 2     |
           |  (WebSearch)    |  | (ideation)    |  | (ideation)      |
           +--------+--------+  +-------+-------+  +--------+--------+
                    |                   |                   |
                    |            +------v------+     +------v------+
                    |            | raw_ideas/  |     | raw_ideas/  |
                    |            | <angle1>.md |     | <angle2>.md |
                    |            +------+------+     +------+------+
                    |                   |                   |
                    +-------------------+-------------------+
                                        |
                              +---------v---------+
                              |  Idea Evaluation  |
                              |  (Main Agent)     |
                              +---------+---------+
                                        |
                              +---------v---------+
                              |  Idea Refinement  |
                              |  (concept briefs) |
                              +---------+---------+
                                        |
                              +---------v---------+
                              | Cross-Pollination |
                              |  (hybrid ideas)   |
                              +---------+---------+
                                        |
                              +---------v---------+
                              |  Final Report     |
                              |  IDEA_BURN_REPORT |
                              +-------------------+
```

### 5.2 Data Flow

1. User triggers skill → Main Agent collects inputs → Writes `.burn_state.json`
2. Main Agent performs domain analysis → Writes `domain_analysis.md`
3. Main Agent generates angles from domain analysis → Queues subagents
4. Subagents read domain analysis → Research angle → Generate raw ideas → Write `raw_ideas/<angle>.md`
5. Main Agent reads all raw ideas → Scores each → Writes `IDEA_CARDS.md`
6. Main Agent refines top picks → Writes `TOP_PICKS.md`
7. Main Agent runs cross-pollination → Appends hybrids to `TOP_PICKS.md`
8. Main Agent generates final report → Writes `IDEA_BURN_REPORT.md`

## 6. Output Specifications

### 6.1 Directory Structure

```
burn-my-tokens-idea_output/
└── <task_name>/
    ├── .burn_state.json
    ├── domain_analysis.md
    ├── IDEA_CARDS.md
    ├── TOP_PICKS.md
    ├── IDEA_BURN_REPORT.md
    └── raw_ideas/
        ├── <angle1>.md
        ├── <angle2>.md
        └── ...
```

### 6.2 File Formats

All output files are Markdown (.md) with:
- YAML frontmatter optional
- Tables for structured data
- Headers for hierarchy
- Bullet points for lists
- No HTML unless necessary for tables

### 6.3 State File Schema

```json
{
  "status": "running | stopped | completed",
  "mode": "10k | 100k | 1M | 10M | burn",
  "target_budget": 10000,
  "estimated_spent": 0,
  "domain": "string",
  "constraints": "string",
  "idea_type": "product | feature | business | content | all",
  "audience": "string",
  "emphasis": ["string"],
  "completed_angles": ["string"],
  "pending_angles": ["string"],
  "idea_count": 0,
  "evaluated_count": 0,
  "refined_count": 0,
  "top_ideas": [
    {
      "name": "string",
      "overall_score": 0.0,
      "rank": 0,
      "angle": "string"
    }
  ],
  "hybrids_identified": 0,
  "last_heartbeat": "ISO8601",
  "cron_job_id": "string"
}
```

## 7. Error Handling

### 7.1 Degradation Matrix

| Failure | First Attempt | Second Attempt | Third Attempt |
|---------|---------------|----------------|---------------|
| Subagent can't generate ideas | Narrow angle | Switch creativity technique | Skip angle |
| Ideas too generic | Add constraints | Switch to constraint-first | Skip angle |
| WebSearch fails | Retry once | Skip validation, note in report | Continue without validation |
| Domain too broad | Suggest sub-niches | Force user to pick | Abort with guidance |

### 7.2 Abort Conditions

The system shall abort (with explanatory message) only when:
- User provides no domain and refuses to select from suggestions (Mode D failure)
- All angles fail after 3 degradation attempts
- API returns persistent 403/429 errors

## 8. Token Budget Estimates

| Tier | Target Budget | Estimated Output |
|------|---------------|------------------|
| 10k | 10,000 tokens | 3-5 raw ideas, 1 angle, no refinement |
| 100k | 100,000 tokens | 8-12 ideas, 1-2 angles, top 5 refined |
| 1M | 1,000,000 tokens | 25-40 ideas, 3-4 angles, top 10 refined, hybrids |
| 10M | 10,000,000 tokens | 80+ ideas, 8-12 angles, top 15 refined, full hybrids |
| burn | unlimited | Until stopped |

## 9. Success Metrics

### 9.1 Output Quality Metrics

- **Niche pass rate**: % of ideas passing NICHE test (target: 100%)
- **Originality rate**: % of ideas with no direct competitor (target: >= 70%)
- **Score distribution**: At least 10% of ideas should score >= 7.0
- **Concept brief completeness**: % of top picks with all required sections (target: 100%)

### 9.2 Process Metrics

- **Resume success rate**: % of interrupted burns that resume correctly (target: 100%)
- **Graceful stop latency**: Batches between stop command and actual stop (target: <= 1)
- **Token efficiency**: Ideas generated per 10k tokens (target: >= 2 for 100k+ modes)

## 10. Future Enhancements (Post-MVP)

- **FE-1**: User feedback loop — allow user to upvote/downvote ideas during burn to steer subagents
- **FE-2**: Idea lineage tracking — full genealogy of how ideas evolved across iterations
- **FE-3**: Market size quantification — integrate TAM/SAM/SOM estimates with real data
- **FE-4**: Visual idea maps — generate Mermaid or graphviz diagrams of idea relationships
- **FE-5**: Competitive deep dives — dedicated subagent for competitive analysis per top idea
- **FE-6**: Multi-domain burns — compare opportunities across 2-3 domains simultaneously
- **FE-7**: Export formats — PDF pitch decks, Notion imports, Airtable CSVs

## 11. Dependencies

- Claude Code with WebSearch capability
- CronCreate / CronDelete for resume mechanism
- File system write access to `burn-my-tokens-idea_output/`
- No external Python packages required (pure text/skill-based)

## 12. Open Questions

1. Should we integrate with `metamemory` to save top ideas across sessions?
2. Should we add a "user feedback" mechanism mid-burn to steer direction?
3. Should we support image-based outputs (e.g., idea canvas diagrams)?
4. How should we handle ideas that are great but outside user's constraints?

---

**Document Owner**: burn-my-tokens family  
**Last Updated**: 2026-05-12  
**Change Log**: See `PROGRESS.md`
