---
name: burn-my-tokens-research
description: When a user's coding plan quota is about to reset, automatically perform deep, iterative research on a user-specified domain and produce comprehensive, citation-backed research documents — converting remaining tokens into durable, structured research reports. Supports unattended self-iterative burning with pure text interaction. Graceful stop via stop command. Never generates code; output is exclusively markdown research documents.
version: 1.1.0
---

## Trigger Conditions

This skill activates when the user inputs any of the following:
- `/burn-my-tokens-research` — Start the research burn flow
- `/burn-my-tokens-research stop` / `停止研究` / `stop research` / `stop burn` — Gracefully end the burn
- Expresses anxiety such as "research / need research / market analysis / competitive landscape / trend report / tokens about to expire / burn tokens on research"

## Graceful Stop

When the user inputs a stop command, the Main Agent must:
1. Set `status` to `"stopped"` in `.burn_state.json`
2. Output stop confirmation: `Research burn gracefully stopped. Completed X subtopics, produced Y findings files, consumed approximately XXk tokens.`
3. Call `CronDelete` to remove the resume cron job (if it exists)
4. Subsequent `CronCreate` triggers reading `status == "stopped"` will be ignored

### MetaBot Queue Compatibility Strategy

Bot frameworks like MetaBot queue new user instructions, so a stop command sent during the current response cannot immediately interrupt execution. To ensure graceful stop works reliably:

**After each research batch (or each subtopic if in 10k mode) completes, the Main Agent must:**
1. Save current progress to `.burn_state.json` (update `estimated_spent`, `completed_subtopics`, `pending_subtopics`, `sources_count`)
2. Output `[Token Tracker]` and a concise progress report
3. **Actively end the current response**

This allows queued stop commands to be processed in the next turn. If the user sends no new instruction, burning auto-resumes via:
- The `CronCreate` scheduled task triggering `/burn-my-tokens-research resume` after 3 minutes
- Or any message from the user in chat triggering resume

**This behavior does not violate the "Never Stop on Your Own" contract** — ending the response is to respond to potential user instructions, not to terminate the burn flow. `status` in `.burn_state.json` remains `"running"`, and `CronCreate` handles recovery.

## Startup Flow (Main Agent)

### Step 0: Tier Selection & Context Gathering

Present tier options to the user and wait for selection:
- `10k` — Quick research snapshot (~1 topic, 1-2 search rounds)
- `100k` — Standard research (~1 domain deeply researched, 5-8 search rounds)
- `1M` — Deep research (~3-4 related domains, 15-25 search rounds)
- `10M` — Massive research (~8-12 domains or full industry landscape, 50+ search rounds)
- `burn` — Burn until stopped

If the user selects `10k`, explicitly inform: "10k is insufficient for a full research report; output will be limited to a quick snapshot with minimal citations. Continue?"

**Then ask:**
```
What is your target domain or field?
(Examples: "autonomous vehicle LiDAR market", "LLM agent frameworks 2026", "solid-state battery startups")
```

**Then ask:**
```
What specific questions do you want answered?
(Leave empty for broad landscape research)

Examples:
- "Who are the top 5 competitors and their pricing?"
- "What are the key technical bottlenecks?"
- "What regulatory changes are expected in the next 2 years?"
```

**Then ask:**
```
Output depth? (brief / standard / comprehensive)
(Leave empty for "standard")

- brief: Executive summary + key findings only
- standard: Full report with analysis and citations
- comprehensive: Multi-section deep dive with tables, timelines, and forecasts
```

**Then ask:**
```
Which analysis sections do you want included?
(Leave empty for "all")

- market analysis
- technology landscape
- competitive analysis
- trend forecast
- all
```

Record confirmed values in `.burn_state.json`.

**Four execution modes:**
- **Mode A** (user provided domain + questions): Research domain → Answer specific questions → Synthesize
- **Mode B** (user provided domain, no questions): Research domain → Broad landscape analysis
- **Mode C** (no domain, user provided questions): Infer domain from questions → Research → Answer
- **Mode D** (neither provided): Prompt user again; do not proceed without at least a domain

## Step 1: Research Scoping

**Define research questions, identify key sub-topics, create research outline:**

1. Based on the user's domain and questions, formulate 3-7 core research questions (RQs)
2. Decompose the domain into sub-topics (1 subtopic for 10k, 3-4 for 100k/1M, 8-12 for 10M)
3. For each subtopic, define:
   - Key entities to investigate (companies, technologies, metrics)
   - Information types needed (market size, technical specs, timelines, pricing)
   - Priority: `critical` (must have) / `important` (should have) / `nice-to-have`

**Output:** `burn-my-tokens-research_output/<task_name>/research_outline.md`

Format:
```markdown
# Research Outline: <domain>

## Date: <timestamp>
## Tier: <10k/100k/1M/10M/burn>
## Output Depth: <brief/standard/comprehensive>
## Sections Requested: <market/tech/competitive/trends>

## Core Research Questions
1. <RQ1>
2. <RQ2>
3. <RQ3>

## Subtopics

### Subtopic 1: <name> (Priority: critical)
- **Scope**: <what to investigate>
- **Key entities**: <companies, technologies, metrics>
- **Information types**: <market size, specs, timelines>
- **Estimated search rounds**: X

### Subtopic 2: <name> (Priority: important)
...

## Research Strategy
- <overall approach: e.g., "Start with market sizing, then drill into competitors">
- <source preferences: e.g., "Prioritize primary sources (company blogs, SEC filings) over secondary>
- <citation density target: e.g., "At least 2 citations per factual claim">
```

## Step 2: Information Gathering (Parallel Subagents)

Launch subagents per sub-topic (max 2 parallel, 3-4 in 10M mode).

**Scheduling strategy:**
- First batch: launch up to 2 subtopics in parallel
- After completion, evaluate remaining budget before next batch
- In `10k` mode: process 1 subtopic at a time, no parallelism

Each subagent's output directory: `burn-my-tokens-research_output/<task_name>/findings/`

**Subagent workflow:**

#### Subagent Step 1: Read Research Outline
Read `burn-my-tokens-research_output/<task_name>/research_outline.md` and extract the assigned subtopic's scope, key entities, and information types.

#### Subagent Step 2: Plan Search Strategy (~10-15% budget)
- Formulate 3-5 search queries targeting the subtopic
- Prioritize diverse source types: industry reports, academic papers, news articles, company blogs, SEC filings, patent databases
- Note any known authoritative sources for this subtopic
- Output: internal search plan with query list

#### Subagent Step 3: Execute WebSearch (~50-60% budget)
- Execute search queries using `WebSearch`
- For each result, use `WebFetch` to extract detailed content when the snippet is insufficient
- **Every factual claim must be traced to a source URL**
- Record findings in a structured format:
  ```markdown
  ## Finding: <title>
  - **Source**: <URL>
  - **Source type**: <industry report / news / academic / company blog / government>
  - **Date published**: <YYYY-MM-DD or "unknown">
  - **Claim**: <verbatim or paraphrased factual statement>
  - **Confidence**: <high / medium / low>
  - **Relevance to RQ**: <which research question this answers>
  ```
- If search results are sparse, reformulate queries and search again
- If paywalled: note the paywall, capture the headline/abstract, and search for open-access alternatives

#### Subagent Step 4: Cross-Reference & Validate (~10-15% budget)
- For critical claims, search for corroborating sources
- Flag contradictions between sources
- Assess source reliability (tier 1: primary/authoritative; tier 2: reputable media; tier 3: blog/opinion)
- Output: validation notes with confidence adjustments

#### Subagent Step 5: Synthesize Subtopic Findings (~15-20% budget)
- Organize raw findings into a coherent narrative
- Group by theme or research question
- Highlight gaps where no public data was found
- Output: structured markdown document

#### Subagent Step 6: Write Findings File (~5-10% budget)
Write the subtopic findings to `burn-my-tokens-research_output/<task_name>/findings/<subtopic_slug>.md`

Format:
```markdown
# Findings: <subtopic_name>

## Summary
<2-3 sentence overview of what was found>

## Key Findings

### Finding 1: <title>
<Detailed description with inline citations like [1], [2]>

### Finding 2: <title>
...

## Data Points
| Metric | Value | Source | Date |
|--------|-------|--------|------|
| <metric> | <value> | [1] | <date> |

## Source Reliability Assessment
| Source | URL | Type | Reliability | Notes |
|--------|-----|------|-------------|-------|
| [1] | <URL> | <type> | <high/medium/low> | <notes> |

## Gaps & Uncertainties
- <what could not be found>
- <areas where sources conflict>

## Citations
[1] <URL> — <title or description>
[2] <URL> — <title or description>
```

#### Subagent Step 7: Report to Main Agent
Report back:
```
Subtopic: <subtopic_name>
Findings: X key findings
Sources: Y unique URLs (Tier 1: A, Tier 2: B, Tier 3: C)
Confidence: high/medium/low
Gaps: <brief description of information gaps>
Status: complete / partial / degraded
```

## Step 3: Synthesis & Analysis

After each batch of subagents completes (or all subtopics finished):

1. Read all generated `findings/<subtopic>.md` files
2. Synthesize into a coherent narrative:
   - **Patterns**: What trends appear across multiple subtopics?
   - **Gaps**: What questions remain unanswered?
   - **Contradictions**: Where do sources disagree? Which side is more credible?
   - **Implications**: What do the findings mean for the user's domain?
3. Update `research_outline.md` with any new subtopics discovered during research (if budget allows)

**Output:** `burn-my-tokens-research_output/<task_name>/synthesis_notes.md`

Format:
```markdown
# Synthesis Notes: <domain>

## Cross-Cutting Patterns
1. <pattern description with supporting citations>
2. ...

## Identified Gaps
1. <gap description>
2. ...

## Source Contradictions
| Claim A | Source(s) | Claim B | Source(s) | Assessment |
|---------|-----------|---------|-----------|------------|
| <claim> | [1], [2] | <contradictory claim> | [3] | <which is more credible and why> |

## Implications
- <strategic implication 1>
- <strategic implication 2>
```

## Step 4: Report Generation

Generate structured research report(s):

### Master Report: `RESEARCH_REPORT.md`

**Read all findings and synthesis notes, then write:**

```markdown
# Research Report: <domain>

## Metadata
- **Date**: <timestamp>
- **Tier**: <10k/100k/1M/10M/burn>
- **Output Depth**: <brief/standard/comprehensive>
- **Subtopics Researched**: X / Y planned
- **Total Sources**: Z unique URLs
- **Research Hours Equivalent**: <estimated>

---

## Executive Summary
<3-5 paragraph summary of the entire research domain. Cover: what was researched, why it matters, the top 3-5 findings, and the bottom-line implication for decision-makers.>

---

## Key Findings

### Finding 1: <title>
<Detailed description. Every factual claim must have an inline citation [N].>

### Finding 2: <title>
...

---

## Market / Industry Analysis
<If requested. Include: TAM/SAM/SOM estimates, growth rates, key segments, geographic breakdown. All figures cited.>

| Segment | Market Size | CAGR | Source |
|---------|-------------|------|--------|
| <segment> | $<value> | <X%> | [N] |

---

## Technology Landscape
<If requested. Include: key technologies, maturity levels, technical differentiators, bottlenecks.>

| Technology | Maturity | Key Players | Bottleneck | Source |
|------------|----------|-------------|------------|--------|
| <tech> | <early/mid/late> | <companies> | <bottleneck> | [N] |

---

## Competitive Analysis
<If requested. Include: competitor profiles, market positioning, strengths/weaknesses, pricing where available.>

| Competitor | Positioning | Strengths | Weaknesses | Source |
|------------|-------------|-----------|------------|--------|
| <company> | <positioning> | <strengths> | <weaknesses> | [N] |

---

## Trends & Forecasts
<If requested. Include: 1-year, 3-year, 5-year outlook; emerging trends; disruptive risks.>

| Trend | Timeframe | Impact | Confidence | Source |
|-------|-----------|--------|------------|--------|
| <trend> | <1yr/3yr/5yr> | <high/medium/low> | <high/medium/low> | [N] |

---

## Gaps & Opportunities
<What remains unknown or under-researched, and what opportunities this creates.>

---

## Methodology
- **Search rounds**: X
- **Sources consulted**: Y
- **Source tier distribution**: Tier 1: A, Tier 2: B, Tier 3: C
- **Limitations**: <e.g., "Some primary sources were paywalled", "Limited public data on private companies">

---

## Sources
[1] <URL> — <title/description> (Tier 1/2/3)
[2] <URL> — <title/description> (Tier 1/2/3)
...
```

### Per-Subtopic Reports
Already written to `findings/<subtopic>.md` during Step 2. Ensure they are referenced in the master report.

## Step 5: Validation

Check report completeness against research outline:

1. **Coverage check**: Does `RESEARCH_REPORT.md` address every core research question in `research_outline.md`?
2. **Citation check**: Does every factual claim have an inline citation? Flag any uncited claims.
3. **Source diversity check**: Are sources from at least 3 different domains (e.g., not all from one news site)?
4. **Confidence calibration**: Are high-confidence claims well-supported by multiple tier-1 sources?
5. **Gap documentation**: Are all identified gaps documented in the report?

**Output:** `burn-my-tokens-research_output/<task_name>/validation_report.md`

Format:
```markdown
# Validation Report

## Coverage
| RQ | Addressed | Location in Report | Confidence |
|----|-----------|-------------------|------------|
| RQ1 | Yes / Partial / No | Section X | High / Medium / Low |

## Citation Audit
| Section | Claims | Cited | Uncited | Action |
|---------|--------|-------|---------|--------|
| Executive Summary | X | Y | Z | <action> |

## Source Diversity
- Tier 1 sources: X
- Tier 2 sources: Y
- Tier 3 sources: Z
- Domain diversity: <list of source domains>

## Gaps Documented
- <gap 1>
- <gap 2>

## Overall Quality Rating
<Pass / Pass with Notes / Needs Revision>
```

If validation reveals significant gaps and budget remains, launch additional subagents to fill gaps before finalizing.

### Final Report Output

After all pending subtopics complete (or budget exhausted), output:

```
Research burn complete!

Domain: <domain>
Tier: <tier>
Output depth: <depth>

Subtopics researched: X / Y planned
Total sources: Z unique URLs
  - Tier 1 (primary/authoritative): A
  - Tier 2 (reputable media): B
  - Tier 3 (blog/opinion): C

Reports generated:
- RESEARCH_REPORT.md (master report with executive summary)
- findings/<subtopic>.md (X per-subtopic reports)
- research_outline.md (research plan)
- synthesis_notes.md (cross-cutting analysis)
- validation_report.md (quality audit)

Key findings (top 3):
1. <finding 1 with citation>
2. <finding 2 with citation>
3. <finding 3 with citation>

Known limitations:
- <limitation 1>
- <limitation 2>

Output directory: burn-my-tokens-research_output/<task_name>/
```

Also write `burn-my-tokens-research_output/<task_name>/RESEARCH_BURN_REPORT.md` with the same content.

---

## Burn Contract

The Main Agent must adhere to the following hard constraints and must not violate them for any reason:

### Constraint 1: Never Stop on Your Own
- After completing a research batch, immediately check budget and remaining subtopics
- If budget is not exhausted and subtopics remain, immediately begin the next batch
- Prohibited from outputting terminal phrases such as "that's all for now", "let's pause here", "continue next time"
- The only stop conditions are: all planned subtopics researched, reaching the 95% tier threshold, API returning 429/403, or user-triggered graceful stop
- "Ending the response between batches" is not a stop — it is a necessary window to respond to queued user instructions. `.burn_state.json` remains `status == "running"`, and `CronCreate` auto-resumes

### Constraint 2: Degrade on Error, Don't Terminate
- When a subagent fails to complete research on a subtopic (search fails, no results, parsing errors), the Main Agent must attempt degradation:
  1. First failure: Switch to broader search queries (less specific, more general terms)
  2. Second failure: Try `WebFetch` on known authoritative URLs for the domain
  3. Third failure: Note the subtopic as "insufficient public data" in findings, write a brief gap note, and move to the next subtopic
- Only after 3 consecutive degradation failures on the same subtopic is skipping allowed
- When skipping a subtopic, briefly log the reason without extended discussion

### Constraint 3: No Code Generation
- This skill must NEVER generate source code, scripts, or executable programs
- Output is exclusively markdown research documents
- If the user asks for code during a research burn, politely redirect: "This skill produces research reports only. For code generation, use `/burn-my-tokens`."

### Constraint 4: Time-Based Resume
- If the response is perceived to be approaching the time limit (executed >4 minutes), the Main Agent must:
  1. Write current progress to `burn-my-tokens-research_output/<task_name>/.burn_state.json`
  2. Ensure the CronCreate resume task is started
  3. Output: "Time limit triggered resume; scheduled task will auto-restart research burning in 3 minutes"
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

## Core Execution Loop (Research Burn Loop)

The Main Agent's core execution logic is as follows. It must iterate as much as possible within a single response:

```
while estimated_spent < target_budget * 0.95 and subtopics_remain:
    # 1. Check for unfinished subagents
    if pending_agents:
        collect_results()
        update_source_counts()
    
    # 2. If no batches planned, generate from research outline
    if not pending_batches:
        pending_batches = generate_batches_from_outline()
    
    # 3. Launch next batch of subagents (max 2 in parallel, 3-4 in 10M mode, 1 in 10k mode)
    batch = pending_batches.pop_next(2)
    for subtopic in batch:
        if estimated_spent + subtopic.estimated_cost > target_budget * 0.95:
            break
        launch_subagent(subtopic)
        estimated_spent += subtopic.estimated_cost
    
    # 4. Synthesize completed findings
    if completed_subtopics:
        update_synthesis_notes()
        update_research_report_draft()
    
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

## Subagent Research Workflow

Each subagent must strictly follow these 6 steps:

### Step 1: Read & Scope (~10-15% budget)
- Read `research_outline.md`
- Extract assigned subtopic scope, key entities, information types
- Formulate 3-5 search queries
- Output: internal search plan

### Step 2: Execute WebSearch (~50-60% budget)
- Execute search queries using `WebSearch`
- Use `WebFetch` for detailed extraction when snippets are insufficient
- Record every finding with source URL, source type, date, claim, confidence, and relevance to RQ
- If results are sparse, reformulate and re-search
- Output: structured raw findings list

### Step 3: Cross-Reference & Validate (~10-15% budget)
- Search for corroborating sources on critical claims
- Flag contradictions
- Assess source reliability (tier 1/2/3)
- Adjust confidence levels
- Output: validation notes

### Step 4: Synthesize Findings (~15-20% budget)
- Organize raw findings into coherent narrative
- Group by theme or research question
- Highlight gaps
- Output: structured markdown draft

### Step 5: Write Findings File (~5-10% budget)
- Write to `findings/<subtopic_slug>.md`
- Ensure every factual claim has inline citation
- Include data tables, source reliability assessment, and gaps
- Output: findings file

### Step 6: Report to Main Agent (~5% budget)
- Report: subtopic name, findings count, sources count (by tier), confidence, gaps, status
- Output: concise status message

---

## Output Specifications

- **Directory**: `burn-my-tokens-research_output/<task_name>/`
- **Master report**: `burn-my-tokens-research_output/<task_name>/RESEARCH_REPORT.md`
- **Per-subtopic findings**: `burn-my-tokens-research_output/<task_name>/findings/<subtopic_slug>.md`
- **Research outline**: `burn-my-tokens-research_output/<task_name>/research_outline.md`
- **Synthesis notes**: `burn-my-tokens-research_output/<task_name>/synthesis_notes.md`
- **Validation report**: `burn-my-tokens-research_output/<task_name>/validation_report.md`
- **Burn report**: `burn-my-tokens-research_output/<task_name>/RESEARCH_BURN_REPORT.md`
- **State file**: `burn-my-tokens-research_output/<task_name>/.burn_state.json`
- **Format**: All outputs are markdown (.md) documents
- **No code files**: This skill does not generate .py, .js, or any executable files

---

## Source Reliability Tiers

| Tier | Definition | Examples |
|------|-----------|----------|
| **Tier 1** | Primary sources, authoritative institutions, peer-reviewed research | SEC filings, company investor relations, Nature/Science papers, government statistics, patent databases |
| **Tier 2** | Reputable media, established analyst firms, known industry publications | Reuters, Bloomberg, McKinsey reports, Gartner, IEEE Spectrum, TechCrunch (for funding news) |
| **Tier 3** | Blogs, opinion pieces, forums, social media, unverified claims | Personal blogs, Reddit threads, Twitter/X posts, Medium articles without clear authorship |

**Citation rules:**
- Prefer Tier 1 for quantitative claims (market size, revenue, technical specs)
- Tier 2 is acceptable for qualitative analysis and trend identification
- Tier 3 must be flagged as such and used only for sentiment or early signals, never as sole source for quantitative claims
- Every factual claim must have at least one citation; high-confidence claims should have 2+ corroborating sources

---

## Token Tracking Rules

**Important**: Claude Code does not provide a remaining token API; all tracking is estimated.

- Each WebSearch ≈ 500-1500 tokens
- Each WebFetch ≈ 500-1500 tokens
- Each file read/write ≈ proportional to file size
- Each Agent call ≈ 2000-5000 tokens overhead
- Each findings file generation ≈ 2000-6000 tokens (depends on subtopic depth and source count)
- Each synthesis pass ≈ 1500-4000 tokens
- Master report generation ≈ 3000-8000 tokens

The Main Agent maintains a running estimate in the conversation, formatted as:
```
[Token Tracker] Estimated consumed: XXk / Target: XXk | Subtopics: X completed / Y pending | Sources: Z collected
```

**Tier mode 95% threshold:**
When estimated consumption reaches 95% of the target tier, the Main Agent proactively pauses and reports:
> "Estimated consumption is approaching the target budget (XXk / XXk). Subtopics completed: X / Y. Continue with the last subtopic or stop?"
If the user chooses to stop, call `CronDelete` to remove the resume task before exiting.

**Burn-until-stopped mode:**
Do not stop proactively. When a subsequent Claude API call returns any blocking error (429/403/500/timeout), catch the exception, call `CronDelete` to remove the resume task, output the final research summary, and gracefully exit.

---

## Resume Mechanism (Pure Text Version)

This skill **does not depend on MetaBot and requires no command-line operations**. Resume is implemented via Claude Code's built-in `CronCreate`, fully transparent to the user.

### State File

The Main Agent updates this at the start of each loop (on resume, if missing/corrupt, reconstruct from output dirs or start fresh):
`burn-my-tokens-research_output/<task_name>/.burn_state.json`

Content:
```json
{
  "status": "running",
  "mode": "1M",
  "target_budget": 1000000,
  "estimated_spent": 3500,
  "domain": "autonomous vehicle LiDAR market",
  "research_questions": ["RQ1", "RQ2", "RQ3"],
  "output_depth": "standard",
  "sections_requested": ["market", "competitive"],
  "research_outline": "burn-my-tokens-research_output/<task_name>/research_outline.md",
  "completed_subtopics": ["market_sizing", "key_players"],
  "pending_subtopics": ["technology_trends", "regulatory_landscape"],
  "sources_count": 24,
  "sources_by_tier": {"tier_1": 8, "tier_2": 12, "tier_3": 4},
  "last_heartbeat": "2026-05-12T14:30:00",
  "cron_job_id": "burn-research-resume-xxx"
}
```

### Auto-Resume Scheduled Task

**On first startup**, the Main Agent calls `CronCreate` to create a durable scheduled task:

- **cron**: `1-59/3 * * * *` (every 3 minutes, avoiding top of the hour)
- **durable**: `true` (persists across session restarts)
- **prompt**:
  ```
  Read burn-my-tokens-research_output/<task_name>/.burn_state.json.
  If status == "running" and last_heartbeat is more than 10 minutes ago:
    Execute /burn-my-tokens-research resume
  Otherwise: ignore, do nothing.
  ```

Record the returned `job_id` in `.burn_state.json` under the `cron_job_id` field.

**After burning completes**, the Main Agent calls `CronDelete` to remove the scheduled task to avoid useless polling.

### User-Transparent Flow

```
User: /burn-my-tokens-research
Claude: [Select tier] [Confirm domain] [Confirm questions] [Confirm depth] → Start research burn
Claude: [Internal] CronCreate creates resume task
...
[Session ends due to time limit]
[3 minutes later, cron auto-triggers]
Claude: [Auto-resumes research burning from breakpoint]
...
[Budget exhausted or all subtopics researched]
Claude: [Research burn complete report] [CronDelete removes resume task]
```

### Resume Parameters

When the scheduled task or user triggers `/burn-my-tokens-research resume`:
1. The Main Agent reads `.burn_state.json`
2. Restores `estimated_spent`, `completed_subtopics`, `pending_subtopics`, `domain`, `research_questions`, `output_depth`, `sections_requested`, `sources_count`
3. Continues the Research Burn Loop from the breakpoint
4. No need to re-ask tier, domain, questions, or depth (unless the state file is missing)

---

## Fallback Strategies

### Search Fails or Returns No Results
If `WebSearch` returns no relevant results for a query:
1. Reformulate with broader keywords (e.g., "LiDAR market size 2026" → "automotive sensor market trends")
2. Try searching in a different language if the domain is region-specific
3. Search for adjacent topics that may contain proxy data
4. If still no results: note "insufficient public data" and move on

### Paywalled Sources
If a critical source is behind a paywall:
1. Capture the headline, abstract, and publication date from search snippets
2. Search for open-access summaries, press releases, or analyst quotes referencing the same data
3. If no alternative found: cite as paywalled with URL, note limitation in methodology section

### Conflicting Information
If sources provide contradictory claims:
1. Document both claims with their respective sources
2. Assess credibility: which source is more authoritative? Which is more recent?
3. If unclear: present both sides with confidence levels and explain the contradiction in the synthesis
4. Never resolve contradictions by fiat — document the uncertainty

### WebFetch Fails
If `WebFetch` fails on a URL:
1. Retry once
2. If still failing: use the search snippet as the best available information
3. Lower the confidence level for that finding
4. Search for alternative sources covering the same claim

### Domain Has Very Sparse Public Data
If an entire domain lacks public information (e.g., stealth startups, classified military tech):
1. Document the opacity as a finding itself
2. Search for proxy indicators (patent filings, job postings, funding rounds, conference presentations)
3. Note in report: "Limited public data available; findings based on indirect signals"

---

## Integration with burn-my-tokens Family

This skill is a sibling to:
- `burn-my-tokens` (new project generation)
- `burn-my-tokens-refactor` (code refactoring)
- `burn-my-tokens-testgen` (test generation)
- `burn-my-tokens-review` (code review)
- `burn-my-tokens-data` (data engineering)

They share:
- The same burn contract philosophy (never stop, degrade on error, time-based resume)
- The same state file pattern (`.burn_state.json`)
- The same CronCreate resume mechanism
- The same output directory convention (`burn-my-tokens-research_output/`)

They differ in:
- **Goal**: Research reports vs code generation vs refactoring vs testing vs review vs data pipelines
- **Validation**: Citation completeness vs test pass/fail vs smell reduction vs quality metrics
- **Output**: Markdown research documents with citations vs PRDs + source code vs diff logs vs test files vs review reports vs datasets + plots
- **Safety**: Read-only by design (no code generation) vs modifies source code

### Recommended Workflow Sequence
1. `/burn-my-tokens-research` to deeply research a domain, market, or technology
2. Read `RESEARCH_REPORT.md` and decide on strategic direction
3. `/burn-my-tokens` to generate an MVP based on research insights
4. `/burn-my-tokens-data` to collect and analyze datasets supporting the research
5. `/burn-my-tokens-review` to audit the resulting codebase

---

## Quality Assurance Checklist

Before declaring research complete, verify:
- [ ] Every factual claim in `RESEARCH_REPORT.md` has an inline citation
- [ ] At least 3 source domains are represented (no over-reliance on one site)
- [ ] All core research questions from `research_outline.md` are addressed
- [ ] Contradictions are documented, not resolved by fiat
- [ ] Gaps are explicitly noted in the report
- [ ] Source reliability tiers are documented
- [ ] No source code or executable files were generated
- [ ] All output is in `burn-my-tokens-research_output/<task_name>/`
- [ ] `RESEARCH_REPORT.md` includes executive summary and methodology
- [ ] `validation_report.md` is complete
- [ ] State file is updated with `completed_subtopics`, `pending_subtopics`, `sources_count`

---

## Pipeline Integration (burn-my-tokens-X Entry Point)

This skill can be invoked as a stage in the `burn-my-tokens-X` pipeline orchestrator.

### Stage Position
- **Stage**: 2 of 7 (Research)
- **Preceded by**: `burn-my-tokens-idea` (Stage 1)
- **Followed by**: `burn-my-tokens-MVP` (Stage 3)

### Parameters from X
When invoked by `burn-my-tokens-X`, the Main Agent receives:
- `allocated_budget`: Token budget allocated for this stage
- `domain`: Domain from Stage 1 or user input
- `top_idea`: Best idea name and concept brief from Stage 1
- `pipeline_mode`: "sequential" (default) or "parallel"
- `previous_outputs`: Outputs from Stage 1 (idea)

### Behavior in Pipeline Mode
1. Skip tier selection — use allocated budget directly
2. Skip domain collection — use `domain` and `top_idea` from X
3. Research focus: validate the top idea's assumptions
4. Adjust scope based on `allocated_budget`:
   - < 5k: 1 topic, 1-2 search rounds
   - 5k-20k: 1 domain deeply researched, 5-8 search rounds
   - 20k-80k: 3-4 related domains, 15-25 search rounds
   - 80k+: 8-12 domains or full industry landscape, 50+ search rounds
5. Write outputs to shared pipeline directory
6. Return structured summary to X:
   - `key_findings`: Top 5 findings
   - `market_validation`: Whether the idea is supported by research
   - `risks_identified`: List of risks from research
   - `recommendation`: "proceed" / "caution" / "pivot"
   - `estimated_spent`: Actual tokens consumed
   - `output_paths`: Generated research reports

### Budget Allocation Guidelines for X
| Pipeline Budget | Suggested Research Stage Allocation | Notes |
|-----------------|------------------------------------|-------|
| 500k total | 50k-80k | Quick validation of top idea |
| 1M total | 100k-150k | Standard domain research |
| 5M total | 400k-600k | Deep multi-domain research |
| 10M+ total | 800k-1.2M | Comprehensive industry analysis |
