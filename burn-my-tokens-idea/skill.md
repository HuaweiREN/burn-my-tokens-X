---
name: burn-my-tokens-idea
description: When a user's coding plan quota is about to reset, automatically perform deep structured ideation — generating, evaluating, and refining niche creative ideas and concepts in a user-specified domain. Burns tokens by deeply brainstorming, evaluating, and refining novel product/feature/business/content ideas. Produces structured idea documents (idea cards, concept briefs, feasibility analyses) but NEVER generates source code. Supports unattended self-iterative burning with pure text interaction. Graceful stop via stop command. Now includes image generation prompts for top picks and pipeline integration for burn-my-tokens-X orchestration.
version: 1.1.0
---

## Trigger Conditions

This skill activates when the user inputs any of the following:
- `/burn-my-tokens-idea` — Start the ideation burn flow
- `/burn-my-tokens-idea stop` / `停止创意燃烧` / `stop idea burn` / `stop burn` — Gracefully end the burn
- Expresses anxiety such as "need ideas / ideation / brainstorm / tokens about to expire / burn tokens on ideas / niche opportunities / creative concepts / product ideas / startup ideas / feature ideas"

## Graceful Stop

When the user inputs a stop command, the Main Agent must:
1. Set `status` to `"stopped"` in `.burn_state.json`
2. Output stop confirmation: `Idea burn gracefully stopped. Completed X angles, generated Y ideas, refined Z concepts, consumed approximately XXk tokens.`
3. Call `CronDelete` to remove the resume cron job (if it exists)
4. Subsequent `CronCreate` triggers reading `status == "stopped"` will be ignored

### MetaBot Queue Compatibility Strategy

Bot frameworks like MetaBot queue new user instructions, so a stop command sent during the current response cannot immediately interrupt execution. To ensure graceful stop works reliably:

**After each ideation batch (or each angle if in 10k mode) completes, the Main Agent must:**
1. Save current progress to `.burn_state.json` (update `estimated_spent`, `completed_angles`, `pending_angles`, `idea_count`, `top_ideas`)
2. Output `[Token Tracker]` and a concise progress report
3. **Actively end the current response**

This allows queued stop commands to be processed in the next turn. If the user sends no new instruction, burning auto-resumes via:
- The `CronCreate` scheduled task triggering `/burn-my-tokens-idea resume` after 3 minutes
- Or any message from the user in chat triggering resume

**This behavior does not violate the "Never Stop on Your Own" contract** — ending the response is to respond to potential user instructions, not to terminate the burn flow. `status` in `.burn_state.json` remains `"running"`, and `CronCreate` handles recovery.

## Startup Flow (Main Agent)

### Step 0: Tier Selection & Context Gathering

Present tier options to the user and wait for selection:
- `10k` — Quick ideation sprint (~3-5 raw ideas, 1 angle, no parallelism)
- `100k` — Standard ideation (~1 niche deeply explored, 8-12 refined ideas)
- `1M` — Deep ideation (~3-4 adjacent niches, 25-40 refined ideas with feasibility analysis)
- `10M` — Massive ideation (~8-12 niches or full industry map, 80+ ideas with full evaluation)
- `burn` — Burn until stopped

**Then ask:**
```
Target domain or field?
(Be as specific as possible — "AI tools for dentists" beats "healthcare AI")

Examples:
- B2B SaaS for veterinary clinics
- Consumer hardware for urban apartment dwellers
- AI-powered creative tools for indie game developers
- Sustainable packaging for e-commerce subscription boxes
- Niche content platforms for underwater photographers
```

**Then ask:**
```
Constraints? (budget / tech / time / regulatory / other)
(Leave empty for "no hard constraints")

Examples:
- Budget: <$50k MVP
- Tech: must use off-the-shelf AI APIs, no custom ML training
- Time: 3-month runway to first revenue
- Regulatory: must be HIPAA-compliant
```

**Then ask:**
```
Desired idea type? (product / feature / business / content / all)
(Leave empty for "all")

Examples:
- product: physical or digital product
- feature: feature addition to existing product line
- business: business model or venture concept
- content: content series, media property, or educational program
```

**Then ask:**
```
Target audience?
(Leave empty for "to be discovered during ideation")

Examples:
- Solo practitioners in regulated industries
- Gen Z hobbyists with disposable income
- Enterprise IT managers at mid-market companies
- Remote workers in developing economies
```

**Then ask (optional angles):**
```
Emphasis preferences? (Select any)
[ ] Competitive differentiation — ideas with strong moats
[ ] Tech novelty — ideas leveraging cutting-edge or emerging tech
[ ] Business model innovation — novel revenue or distribution models
[ ] Content angle — narrative, educational, or media-driven concepts
[ ] All of the above
```

**Auto-detect and confirm:**

If the user has a `.burn_state.json` from a previous burn with the same domain, offer to resume or start fresh.

If the user's domain is extremely broad (e.g., "AI", "healthcare", "education"), the Main Agent must:
1. Flag it immediately
2. Suggest 2-3 narrowing directions based on WebSearch of current trends
3. Ask the user to pick one before proceeding

Record confirmed values in `.burn_state.json` under `domain`, `constraints`, `idea_type`, `audience`, `emphasis`.

**Four execution modes:**
- **Mode A** (user provided domain + type + audience): Domain analysis → Generate ideas for specified audience
- **Mode B** (user provided domain, no type/audience): Domain analysis → Auto-discover audience segments → Generate ideas
- **Mode C** (no domain, user provided type/audience): Broad trend scan → Suggest domains → User picks → Proceed as Mode A
- **Mode D** (neither provided): Full open ideation → Use WebSearch to find trending underserved niches → Present top 3 domains → User picks → Proceed as Mode A

## Step 1: Domain Analysis

**Research the domain to identify gaps, pain points, underserved segments, and emerging trends.**

1. Use WebSearch to find:
   - Recent market reports and trend analyses in the target domain
   - Common complaints and unmet needs (search "<domain> pain points", "<domain> problems", "why <domain> sucks")
   - Emerging technologies or regulatory changes creating new opportunities
   - Underserved sub-segments (e.g., "<domain> for <niche audience>")
   - Recent startup funding in adjacent spaces (signals of market interest)

2. Use WebSearch to validate competitive landscape:
   - Search for existing solutions in each potential niche angle
   - Identify market leaders and their weaknesses
   - Look for "<niche> alternative" or "<niche> vs" comparisons
   - Check for discontinued products (signals of failed approaches to avoid or gaps left behind)

3. Build a domain map:
   - **Primary niche**: The user's specified domain
   - **Adjacent niches**: 3-5 related areas that share audience or technology
   - **Trend vectors**: Emerging forces (tech, regulatory, social) that could create new opportunities
   - **Pain point clusters**: Grouped unmet needs by frequency and severity
   - **Underserved segments**: Specific audience slices with money + need + low competition

**Output:** `burn-my-tokens-idea_output/<task_name>/domain_analysis.md`

Format:
```markdown
# Domain Analysis: <domain_name>

Generated: <timestamp>
Domain: <user's domain>
Constraints: <budget/tech/time constraints>
Idea Type: <product/feature/business/content/all>
Target Audience: <audience or "TBD">

## Market Landscape
- Market maturity: <emerging/growing/mature/saturated>
- Key players: <3-5 major incumbents>
- Recent funding activity: <trends from WebSearch>
- Regulatory environment: <stable/shifting/heavily regulated>

## Pain Point Clusters
| Cluster | Frequency | Severity | Current Solutions | Gap Size |
|---------|-----------|----------|-------------------|----------|
| <pain point> | high/medium/low | high/medium/low | <what exists> | large/medium/small |

## Underserved Segments
| Segment | Size Estimate | Willingness to Pay | Competition Level | Opportunity Rating |
|---------|---------------|--------------------|-------------------|--------------------|
| <segment> | <estimate> | high/medium/low | high/medium/low | A/B/C/D |

## Adjacent Niches
| Niche | Relevance | Trend Direction | Crossover Potential |
|-------|-----------|---------------|---------------------|
| <niche> | high/medium/low | growing/stable/declining | high/medium/low |

## Trend Vectors
| Trend | Impact Timeline | Opportunity Type | Confidence |
|-------|-----------------|--------------------|------------|
| <trend> | <0-12mo / 1-3yr / 3-5yr> | <enabler/disruptor/constraint> | high/medium/low |

## Research Sources
- <source 1>
- <source 2>
- <source 3>
```

**If the domain is too broad**, apply the narrowing strategy:
1. Identify the 3 most promising sub-niches from the domain analysis
2. Present to user: "Your domain is quite broad. Based on analysis, the highest-opportunity sub-niches are: (1) ..., (2) ..., (3) ... Which would you like to focus on?"
3. Update `.burn_state.json` with the narrowed domain

## Step 2: Idea Generation (Parallel Subagents)

Launch subagents per niche angle (max 2 parallel, 3-4 in 10M mode).

**Angle assignment strategy:**
- From `domain_analysis.md`, extract 1 angle per underserved segment or trend vector
- In `10k` mode: 1 angle only, no parallelism
- In `100k` mode: 1-2 angles
- In `1M` mode: 3-4 angles
- In `10M` mode: 8-12 angles, can run 3-4 in parallel
- In `burn` mode: unlimited angles until stopped

**Each subagent's output directory:** `burn-my-tokens-idea_output/<task_name>/raw_ideas/`

**Subagent workflow:** (see dedicated section below)

**Quality gate for raw ideas:**
Each idea must pass the NICHE test before being written:
- **N**arrow: Targets a specific audience or use case, not "everyone"
- **I**nnovative: Has a novel angle, combination, or approach
- **C**oncrete: Can be described in one sentence with clear value proposition
- **H**eavy: Has real substance — not a feature masquerading as a product
- **E**dge: Exploits an edge — tech shift, regulatory gap, behavioral change, or underserved need

Ideas that fail the NICHE test must be rejected by the subagent and replaced.

**Output:** `burn-my-tokens-idea_output/<task_name>/raw_ideas/<angle>.md`

Format:
```markdown
# Raw Ideas: <angle_name>

Angle: <description of the niche angle>
Trend Vector: <which trend this angle exploits>
Target Segment: <specific audience>

## Ideas

### Idea 1: <name>
- **One-liner**: <single sentence value prop>
- **Problem**: <specific pain point>
- **Solution**: <how it solves the problem>
- **Why now**: <trend or shift enabling this>
- **Known competitors**: <from WebSearch validation>
- **Differentiation angle**: <how it's different>
- **Niche score**: <self-assessment 1-10 on how niche/specific this is>

### Idea 2: ...
```

## Step 3: Idea Evaluation

**Main Agent reads all raw ideas from `raw_ideas/<angle>.md` and evaluates each.**

For each idea, score on 5 axes (1-10 scale):

| Axis | Definition | High Score (8-10) | Low Score (1-3) |
|------|-----------|-------------------|-----------------|
| **Feasibility** | Can this be built/prototyped with current tech and constraints? | Clear path to MVP in <6 months | Requires breakthrough tech or $10M+ |
| **Novelty** | How unique is this idea vs. known solutions? | No direct competitor; genuinely new category | Me-too product with minor twist |
| **Market Size** | Addressable market potential | Large TAM with clear SAM/SOM path | Micro-market with no expansion path |
| **Differentiation** | Competitive moat and defensibility | Network effects, proprietary data, or deep domain expertise required | Easily copyable by any competent team |
| **Personal Fit** | Alignment with user's skills/resources (if known) | Directly leverages user's expertise | Completely outside user's domain |

**Overall Score Calculation:**
```
overall_score = (
    feasibility * 0.25 +
    novelty * 0.25 +
    market_size * 0.20 +
    differentiation * 0.20 +
    personal_fit * 0.10
)
```

**Validation via WebSearch:**
For each top-scoring idea (score >= 7.0), perform a WebSearch to verify:
1. No exact product already exists with the same approach
2. The claimed pain point is real (forum discussions, Reddit threads, reviews)
3. The tech enabler is actually available (not vaporware)
4. The target audience is reachable (has communities, channels, or publications)

If validation fails, downgrade the score by 1-3 points and note the reason.

**Output:** `burn-my-tokens-idea_output/<task_name>/IDEA_CARDS.md`

Format:
```markdown
# Idea Cards: <domain_name>

Generated: <timestamp>
Total Ideas Generated: <count>
Total Ideas Evaluated: <count>

## All Ideas (Ranked by Overall Score)

| Rank | Idea Name | Feasibility | Novelty | Market | Diff | Fit | Overall | Validation |
|------|-----------|-------------|---------|--------|------|-----|---------|------------|
| 1 | <name> | 8 | 9 | 7 | 8 | 6 | 7.75 | passed |
| 2 | <name> | 7 | 8 | 8 | 7 | 7 | 7.45 | passed |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |

## Score Distribution
- 8.0+: <count> (exceptional)
- 7.0-7.9: <count> (strong)
- 6.0-6.9: <count> (viable)
- 5.0-5.9: <count> (marginal)
- <5.0: <count> (weak — not refined further)
```

## Step 4: Idea Refinement

**Top-scoring ideas get refined into structured concept briefs.**

Select top ideas for refinement:
- `10k` mode: top 3 ideas
- `100k` mode: top 5 ideas
- `1M` mode: top 10 ideas
- `10M` mode: top 15 ideas
- `burn` mode: top 10 ideas per batch

For each selected idea, produce a structured concept brief:

**Output:** `burn-my-tokens-idea_output/<task_name>/TOP_PICKS.md`

Format per idea:
```markdown
## Concept Brief #<N>: <Idea Name>

**One-liner**: <single compelling sentence>

**Overall Score**: <X.XX>/10 | **Rank**: <N>

### Problem
<2-3 sentences describing the specific pain point. Include evidence from domain analysis or WebSearch validation.>

### Solution
<3-5 sentences describing the proposed solution. Be specific about mechanism, not vague.>

### Target User
- **Primary**: <specific persona>
- **Secondary**: <adjacent persona if applicable>
- **User journey**: <how they discover, adopt, and benefit>

### Differentiation
<What makes this defensible? Why can't a competitor copy this in 3 months?>

### Tech Stack Suggestion
<High-level technology components. This is NOT code — just architectural notes.>
- Core platform: <e.g., mobile app, web SaaS, hardware + cloud>
- Key technologies: <e.g., LLM APIs, computer vision, IoT sensors>
- Data sources: <what data it needs and how it's obtained>

### MVP Scope
<What is the smallest version that validates the core hypothesis?>
- Must-have features (3-5)
- Timeline estimate: <weeks/months>
- Budget estimate: <$X>

### Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| <risk> | high/medium/low | high/medium/low | <strategy> |

### Next Steps
1. <actionable step>
2. <actionable step>
3. <actionable step>

### Validation Notes
<WebSearch findings: what exists, what doesn't, evidence of demand>
```

### Image Generation Prompts

For each refined idea in `TOP_PICKS.md`, the subagent must ALSO generate an **image generation prompt** and write it to `image_prompts/<idea_name>_prompt.md`.

The prompt must:
- Be written in English (suitable for DALL-E, Midjourney, Stable Diffusion)
- Be 2-4 sentences long
- Describe a visual concept that captures the essence of the idea
- Include style hints (e.g., "minimalist illustration", "3D render", "tech infographic")
- Be concrete and specific enough to produce a usable image

Example:
```markdown
# Image Prompt: AI Soil Microbiome Analyzer

A futuristic vertical farm tower with glowing roots and holographic bacteria visualization. Microscopic organisms float in beams of light showing data readouts. Clean sci-fi aesthetic, soft blue and green palette, 3D render, isometric view.
```

## Step 5: Cross-Pollination

**Identify idea combinations that create hybrid concepts.**

After refining top picks, analyze pairs for combination potential:

1. For each pair of top ideas ( Idea A + Idea B ):
   - Can their technologies combine? (e.g., AI soil analysis + vertical farm automation)
   - Can their audiences cross-over? (e.g., dentists + veterinary clinics = specialist SaaS platform)
   - Can their business models hybridize? (e.g., subscription + marketplace)

2. Score each hybrid on:
   - **Synergy** (1-10): Is the combination greater than the sum of parts?
   - **Feasibility impact** (+/-): Does combining make it harder or easier?
   - **Novelty boost** (+/-): Does combining increase or decrease uniqueness?

3. Promising hybrids (synergy >= 7) get a mini concept brief:
   - Name, one-liner, parent ideas, why the combination works, risks of combining

**Output:** Append to `burn-my-tokens-idea_output/<task_name>/TOP_PICKS.md` under section `## Hybrid Concepts`

Format:
```markdown
## Hybrid Concepts

### Hybrid #1: <Name> (Idea X + Idea Y)
- **Parent Ideas**: <Idea A name> + <Idea B name>
- **One-liner**: <combined value prop>
- **Why it works**: <explanation of synergy>
- **New risks**: <risks introduced by combining>
- **Synergy score**: <X>/10
```

## Step 6: Report

After all pending angles complete (or budget exhausted), output:

```
Idea burn complete!

Domain: <domain_name>
Constraints: <constraints>
Idea Type: <type>
Target Audience: <audience>

Angles explored: X / Y total
Ideas generated: Z
Ideas evaluated: W
Top picks refined: V
Hybrids identified: U

Top 3 ideas:
1. [SCORE: X.XX] <Idea Name> — <one-liner>
2. [SCORE: X.XX] <Idea Name> — <one-liner>
3. [SCORE: X.XX] <Idea Name> — <one-liner>

Score distribution:
- 8.0+: A ideas (exceptional)
- 7.0-7.9: B ideas (strong)
- 6.0-6.9: C ideas (viable)
- <6.0: D ideas (not refined)

Next steps:
- Run `/burn-my-tokens` to turn a top pick into a full project with PRD and code
- Run `/burn-my-tokens-idea` again with a narrowed angle for deeper exploration
- Run `/burn-my-tokens-data` to research market data for a specific idea

Output directory: burn-my-tokens-idea_output/<task_name>/
Reports:
- domain_analysis.md (market research and gap analysis)
- raw_ideas/<angle>.md (per-angle raw ideation)
- IDEA_CARDS.md (all ideas with scores)
- TOP_PICKS.md (top concepts with full briefs + hybrids)
- IDEA_BURN_REPORT.md (consolidated summary)
```

Also write `burn-my-tokens-idea_output/<task_name>/IDEA_BURN_REPORT.md` with the same content.

---

## Burn Contract

The Main Agent must adhere to the following hard constraints and must not violate them for any reason:

### Constraint 1: Never Stop on Your Own
- After completing an ideation batch, immediately check budget and remaining angles
- If budget is not exhausted and angles remain, immediately begin the next batch
- Prohibited from outputting terminal phrases such as "that's all for now", "let's pause here", "continue next time"
- The only stop conditions are: all planned angles completed, reaching the 95% tier threshold, API returning 429/403, or user-triggered graceful stop
- "Ending the response between batches" is not a stop — it is a necessary window to respond to potential user instructions. `.burn_state.json` remains `status == "running"`, and `CronCreate` auto-resumes

### Constraint 2: Degrade on Error, Don't Terminate
- When a subagent fails to generate ideas for an angle (too vague, too constrained, etc.), the Main Agent must attempt degradation:
  1. First failure: Narrow the angle further (e.g., "AI for dentists" → "AI for orthodontists")
  2. Second failure: Switch to constraint-first ideation (start with user's constraints, find what fits)
  3. Third failure: Skip angle, log reason, move to next angle
- Only after 3 consecutive degradation failures on the same angle is skipping allowed
- When skipping an angle, briefly log the reason without extended discussion
- If ideas are too generic (fail NICHE test), add constraints: budget cap, tech limitation, or audience restriction

### Constraint 3: No Code Generation
- This skill must NEVER generate source code, implementation files, or technical prototypes
- Only write structured idea documents, concept briefs, and analyses
- Tech stack suggestions in concept briefs must remain high-level architectural notes
- If a subagent attempts to write code, abort the subagent and degrade

### Constraint 4: Time-Based Resume
- If the response is perceived to be approaching the time limit (executed >4 minutes), the Main Agent must:
  1. Write current progress to `burn-my-tokens-idea_output/<task_name>/.burn_state.json`
  2. Ensure the CronCreate resume task is started
  3. Output: "Time limit triggered resume; scheduled task will auto-restart idea burning in 3 minutes"
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

## Core Execution Loop (Idea Burn Loop)

The Main Agent's core execution logic is as follows. It must iterate as much as possible within a single response:

```
while estimated_spent < target_budget * 0.95 and angles_remain:
    # 1. Check for unfinished subagents
    if pending_agents:
        collect_results()
        update_idea_counts()
    
    # 2. If no angles planned, generate from domain analysis
    if not pending_angles:
        pending_angles = generate_angles_from_domain_analysis()
    
    # 3. Launch next batch of subagents (max 2 in parallel, 3-4 in 10M mode, 1 in 10k mode)
    batch = pending_angles.pop_next(2)
    for angle in batch:
        if estimated_spent + angle.estimated_cost > target_budget * 0.95:
            break
        launch_subagent(angle)
        estimated_spent += angle.estimated_cost
    
    # 4. Evaluate and refine completed angles
    if completed_angles:
        read_raw_ideas()
        evaluate_ideas()
        update_idea_cards()
        if top_ideas_ready:
            refine_top_picks()
            update_top_picks_md()
    
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

# After all angles complete
if all_angles_complete:
    run_cross_pollination()
    generate_final_report()
    write_idea_burn_report()
    CronDelete(resume_job_id)
```

**Note**: The pseudocode above is for illustrative purposes only. The actual execution is driven by natural-language instructions in this skill.md.

---

## Subagent Ideation Workflow

Each subagent must strictly follow these 6 steps:

### Step 1: Read Domain Analysis (~10-15% budget)
- Read `domain_analysis.md`
- Focus on the assigned angle's underserved segment, trend vector, and pain point cluster
- Read any previously generated `raw_ideas/` files for the same domain (to avoid duplication)
- Output: internal notes on angle scope and constraints

### Step 2: Research Angle Context (~15-20% budget)
- Use WebSearch to validate the angle:
  - Search for existing solutions in this specific niche
  - Search for forum discussions, Reddit threads, or reviews mentioning the pain point
  - Search for recent news or product launches in this micro-niche
- Note any direct competitors or near-competitors found
- Output: competitive landscape notes for this angle

### Step 3: Generate Raw Ideas (~40-50% budget)
- Brainstorm ideas within the assigned angle using structured creativity techniques:
  - **Constraint-first**: Start with user's hard constraints, find what fits
  - **Trend-riding**: Combine the angle's trend vector with underserved need
  - **Edge-exploitation**: Find regulatory gaps, tech shifts, or behavioral changes
  - **Cross-domain移植**: Borrow solutions from adjacent industries
  - **Anti-pattern inversion**: Take common failed approaches and invert one assumption
- Each idea must pass the NICHE test (Narrow, Innovative, Concrete, Heavy, Edge)
- Target: 5-8 raw ideas per angle (3-5 in 10k mode, 10-15 in 10M mode)
- Output: structured raw idea list

### Step 4: Validate Originality (~10-15% budget)
- For each idea, perform a quick WebSearch to check if an identical or near-identical product exists
- If a direct competitor exists, either:
  - Pivot the idea to differentiate (change audience, tech, or business model)
  - Reject and replace with a new idea
- Ensure at least 70% of ideas are original (no direct competitor with same approach)
- Output: validation notes per idea

### Step 5: Write Raw Ideas File (~10-15% budget)
- Write formatted raw ideas to `raw_ideas/<angle>.md`
- Ensure every idea has: name, one-liner, problem, solution, why now, known competitors, differentiation angle, niche score
- Output: raw ideas file

### Step 6: Report to Main Agent (~5% budget)
- Report: angle name, ideas generated, ideas rejected for being unoriginal, niche score average, status
- Output: concise status message

---

## Output Specifications

- **Output directory**: `burn-my-tokens-idea_output/<task_name>/`
- **Domain analysis**: `burn-my-tokens-idea_output/<task_name>/domain_analysis.md`
- **Raw ideas**: `burn-my-tokens-idea_output/<task_name>/raw_ideas/<angle>.md`
- **Idea cards**: `burn-my-tokens-idea_output/<task_name>/IDEA_CARDS.md`
- **Top picks**: `burn-my-tokens-idea_output/<task_name>/TOP_PICKS.md`
- **Image prompts**: `burn-my-tokens-idea_output/<task_name>/image_prompts/<idea_name>_prompt.md`
- **Final report**: `burn-my-tokens-idea_output/<task_name>/IDEA_BURN_REPORT.md`
- **State file**: `burn-my-tokens-idea_output/<task_name>/.burn_state.json`
- **Output type**: Markdown documents only — NO code files, NO prototypes, NO implementation

---

## Token Tracking Rules

**Important**: Claude Code does not provide a remaining token API; all tracking is estimated.

- Each WebSearch ≈ 500-1500 tokens
- Each file read/write ≈ proportional to file size
- Each Agent call ≈ 2000-5000 tokens overhead
- Each domain analysis (WebSearch + synthesis) ≈ 2000-4000 tokens
- Each ideation subagent ≈ 3000-8000 tokens (depends on angle complexity and idea count)
- Each idea evaluation (read + score + validation search) ≈ 500-1500 tokens per idea
- Each concept brief refinement ≈ 1000-3000 tokens per idea
- Each image generation prompt ≈ 200-500 tokens per idea
- Cross-pollination analysis ≈ 1000-2000 tokens

The Main Agent maintains a running estimate in the conversation, formatted as:
```
[Token Tracker] Estimated consumed: XXk / Target: XXk | Angles: X completed / Y pending | Ideas: Z generated / W evaluated / V refined
```

**Tier mode 95% threshold**:
When estimated consumption reaches 95% of the target tier, the Main Agent proactively pauses and reports:
> "Estimated consumption is approaching the target budget (XXk / XXk). Angles completed: X. Ideas generated: Z. Continue with the last angle or stop?"
If the user chooses to stop, call `CronDelete` to remove the resume task before exiting.

**Burn-until-stopped mode**:
Do not stop proactively. When a subsequent Claude API call returns any blocking error (429/403/500/timeout), catch the exception, call `CronDelete` to remove the resume task, output the final summary report with all ideas generated so far, and gracefully exit.

---

## Resume Mechanism (Pure Text Version)

This skill **does not depend on MetaBot and requires no command-line operations**. Resume is implemented via Claude Code's built-in `CronCreate`, fully transparent to the user.

### State File

The Main Agent updates this at the start of each loop (on resume, if missing/corrupt, reconstruct from output dirs or start fresh):
`burn-my-tokens-idea_output/<task_name>/.burn_state.json`

Content:
```json
{
  "status": "running",
  "mode": "1M",
  "target_budget": 1000000,
  "estimated_spent": 3500,
  "domain": "AI tools for veterinary clinics",
  "constraints": "budget <$50k, 3-month runway",
  "idea_type": "product",
  "audience": "veterinary clinic owners",
  "emphasis": ["competitive differentiation", "tech novelty"],
  "completed_angles": ["diagnostic AI for small animals", "client communication automation"],
  "pending_angles": ["inventory management for exotic pets", "telehealth for rural vets"],
  "idea_count": 18,
  "evaluated_count": 12,
  "refined_count": 5,
  "top_ideas": [
    {
      "name": "AI-Powered Feline Renal Risk Predictor",
      "overall_score": 8.15,
      "rank": 1,
      "angle": "diagnostic AI for small animals"
    },
    {
      "name": "VetClient — Automated Post-Surgery Pet Owner Updates",
      "overall_score": 7.60,
      "rank": 2,
      "angle": "client communication automation"
    }
  ],
  "hybrids_identified": 1,
  "last_heartbeat": "2026-05-12T14:30:00",
  "cron_job_id": "burn-idea-resume-xxx"
}
```

### Auto-Resume Scheduled Task

**On first startup**, the Main Agent calls `CronCreate` to create a durable scheduled task:

- **cron**: `1-59/3 * * * *` (every 3 minutes, avoiding top of the hour)
- **durable**: `true` (persists across session restarts)
- **prompt**:
  ```
  Read burn-my-tokens-idea_output/<task_name>/.burn_state.json.
  If status == "running" and last_heartbeat is more than 10 minutes ago:
    Execute /burn-my-tokens-idea resume
  Otherwise: ignore, do nothing.
  ```

Record the returned `job_id` in `.burn_state.json` under the `cron_job_id` field.

**After burning completes**, the Main Agent calls `CronDelete` to remove the scheduled task to avoid useless polling.

### User-Transparent Flow

```
User: /burn-my-tokens-idea
Claude: [Select tier] [Confirm domain] [Confirm constraints] [Confirm type/audience] → Start idea burn
Claude: [Internal] CronCreate creates resume task
...
[Session ends due to time limit]
[3 minutes later, cron auto-triggers]
Claude: [Auto-resumes idea burning from breakpoint]
...
[Budget exhausted or all angles explored]
Claude: [Idea burn complete report] [CronDelete removes resume task]
```

### Resume Parameters

When the scheduled task or user triggers `/burn-my-tokens-idea resume`:
1. The Main Agent reads `.burn_state.json`
2. Restores `estimated_spent`, `completed_angles`, `pending_angles`, `domain`, `constraints`, `idea_type`, `audience`, `top_ideas`
3. Continues the Idea Burn Loop from the breakpoint
4. No need to re-ask tier, domain, or constraints (unless the state file is missing)

---

## Fallback Strategies

### Domain Too Broad
If the user's domain is too broad (e.g., "AI", "healthcare", "education"):
1. Flag immediately and do not proceed with ideation
2. Use WebSearch to find 2-3 high-opportunity sub-niches with recent activity
3. Present narrowing options to user with brief rationale
4. Update `.burn_state.json` with narrowed domain before continuing

### Ideas Too Generic
If subagent-generated ideas fail the NICHE test (too generic):
1. First attempt: Add a constraint (budget cap, tech limitation, or audience restriction)
2. Second attempt: Switch creativity technique (e.g., from "trend-riding" to "anti-pattern inversion")
3. Third attempt: Narrow the angle further and retry
4. If still failing after 3 attempts: Skip angle, log reason, move to next

### Evaluation Deadlock
If multiple ideas cluster at similar scores and ranking is unclear:
1. Apply tie-breaker rules in order:
   - Higher feasibility wins (can actually be built)
   - Higher differentiation wins (harder to copy)
   - Higher novelty wins (more unique)
2. If still tied, flag as "co-ranked" and refine both
3. Never get stuck in endless re-evaluation — make a decision and move on

### WebSearch Validation Failure
If WebSearch reveals a direct competitor for a top-scoring idea:
1. Downgrade the idea's score by 1-3 points
2. Note the competitor in validation notes
3. If the competitor is dominant (>$100M revenue, 5+ years old), consider rejecting the idea entirely
4. If the competitor is new or weak, keep the idea but emphasize differentiation in the concept brief

### No Angles Generate Viable Ideas
If all angles produce weak ideas (all scores < 5.0):
1. Re-read `domain_analysis.md` — the domain may be saturated or constraints too tight
2. Suggest to user: loosen constraints, shift to adjacent niche, or abandon domain
3. Do not generate fake high scores to please the user

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
- The same output directory convention (`burn-my-tokens-idea_output/`)

They differ in:
- **Goal**: Idea generation vs code generation vs refactoring vs testing vs review vs data engineering
- **Validation**: WebSearch competitive validation vs test pass/fail vs smell reduction vs data quality metrics
- **Output**: Structured idea documents vs PRDs + source code vs diff logs vs test files vs review reports vs datasets
- **Safety**: No code generation by design vs modifies source code

### Recommended Workflow Sequence
1. `/burn-my-tokens-idea` to explore niche opportunities and identify the best concept
2. Read `TOP_PICKS.md` and select the most promising idea
3. `/burn-my-tokens` to turn the selected idea into a full project with PRD and code
4. `/burn-my-tokens-data` to research market data and validate assumptions
5. `/burn-my-tokens-testgen` to add test coverage
6. `/burn-my-tokens-review` to audit the implementation

### Idea-to-Project Pipeline
When transitioning from `burn-my-tokens-idea` to `burn-my-tokens`:
- The concept brief in `TOP_PICKS.md` serves as the seed for the project PRD
- The problem statement becomes the project motivation
- The MVP scope becomes the v1 feature set
- The tech stack suggestion informs architecture decisions
- The risks become the project's known limitations section

---

## Quality Assurance Checklist

Before declaring an idea burn complete, verify:
- [ ] Domain analysis includes at least 3 research sources with URLs
- [ ] Every idea passes the NICHE test (Narrow, Innovative, Concrete, Heavy, Edge)
- [ ] Every evaluated idea has scores for all 5 axes (feasibility, novelty, market_size, differentiation, personal_fit)
- [ ] Top picks have full concept briefs with all required sections
- [ ] Every top pick has WebSearch validation notes
- [ ] Every top pick has an image generation prompt in `image_prompts/`
- [ ] No source code or implementation files were generated
- [ ] All output is in `burn-my-tokens-idea_output/<task_name>/`
- [ ] `IDEA_CARDS.md` includes score distribution summary
- [ ] `TOP_PICKS.md` includes hybrid concepts section
- [ ] `IDEA_BURN_REPORT.md` includes top 3 ideas and score distribution
- [ ] State file is updated with `completed_angles`, `pending_angles`, `idea_count`, `top_ideas`

---

## Pipeline Integration (burn-my-tokens-X Entry Point)

This skill can be invoked as a stage in the `burn-my-tokens-X` pipeline orchestrator.

### Stage Position
- **Stage**: 1 of 7 (Idea Generation)
- **Preceded by**: None (pipeline entry point)
- **Followed by**: `burn-my-tokens-research` (Stage 2)

### Parameters from X
When invoked by `burn-my-tokens-X`, the Main Agent receives:
- `allocated_budget`: Token budget allocated for this stage
- `domain`: Pre-validated domain string
- `constraints`: Pre-collected constraints
- `pipeline_mode`: "sequential" (default) or "parallel"
- `previous_outputs`: None (first stage)

### Behavior in Pipeline Mode
1. Skip the tier selection dialog — use the allocated budget directly
2. Skip the domain/constraints collection — use provided values
3. Adjust output targets based on `allocated_budget`:
   - < 5k: 3-5 raw ideas, top 1 refined, no image prompts
   - 5k-15k: 8-12 ideas, top 3 refined, image prompts for top 3
   - 15k-50k: 25-40 ideas, top 5 refined, image prompts for top 5
   - 50k+: 80+ ideas, top 10 refined, image prompts for top 10
4. Write outputs to the shared pipeline output directory (if specified by X)
5. Return a structured summary to X containing:
   - `top_idea_names`: List of top idea names
   - `top_idea_scores`: Their overall scores
   - `recommended_next_stage`: Whether to proceed to research or skip
   - `estimated_spent`: Actual tokens consumed
   - `output_paths`: List of generated file paths

### Budget Allocation Guidelines for X
| Pipeline Budget | Suggested Idea Stage Allocation | Notes |
|-----------------|--------------------------------|-------|
| 500k total | 80k-120k | Quick ideation + top 3 refinement |
| 1M total | 150k-250k | Standard ideation + top 5 refinement |
| 5M total | 600k-1M | Deep ideation + top 10 + full hybrids |
| 10M+ total | 1.2M-2M | Massive ideation + full refinement |
