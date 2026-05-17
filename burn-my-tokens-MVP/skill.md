---
name: burn-my-tokens-MVP
description: When a user's coding plan quota is about to reset, automatically explore niche directions based on existing projects and generate new L3 MVP projects, ensuring remaining tokens are converted into valuable personal output. Supports unattended self-iterative burning with pure text interaction. Graceful stop via stop command.
version: 1.1.0
---

## Trigger Conditions

This skill activates when the user inputs any of the following:
- `/burn-my-tokens-MVP` — Start the burn flow
- `/burn-my-tokens-MVP stop` / `停止燃烧` / `stop burn` — Gracefully end the burn
- Expresses anxiety such as "tokens are about to expire / can't use them all / burn tokens"

## Graceful Stop

When the user inputs a stop command, the Main Agent must:
1. Set `status` to `"stopped"` in `.burn_state.json`
2. Output stop confirmation: `Burn gracefully stopped. Completed X projects, consumed approximately XXk tokens.`
3. Call `CronDelete` to remove the resume cron job (if it exists)
4. Subsequent `CronCreate` triggers reading `status == "stopped"` will be ignored

### MetaBot Queue Compatibility Strategy

Bot frameworks like MetaBot queue new user instructions, so a stop command sent during the current response cannot immediately interrupt execution. To ensure graceful stop works reliably:

**After each project (or each batch of parallel subagents) completes, the Main Agent must:**
1. Save current progress to `.burn_state.json` (update `estimated_spent`, `completed_projects`)
2. Output `[Token Tracker]` and a concise progress report
3. **Actively end the current response**

This allows queued stop commands to be processed in the next turn. If the user sends no new instruction, burning auto-resumes via:
- The `CronCreate` scheduled task triggering `/burn-my-tokens-MVP resume` after 3 minutes
- Or any message from the user in chat triggering resume

**This behavior does not violate the "Never Stop on Your Own" contract** — ending the response is to respond to potential user instructions, not to terminate the burn flow. `status` in `.burn_state.json` remains `"running"`, and `CronCreate` handles recovery.

## Startup Flow (Main Agent)

### Step 0: Tier Selection & Context Gathering

Present tier options to the user and wait for selection:
- `10k` — Lightweight exploration (~1 L1 research+PRD or attempt L2)
- `100k` — Standard exploration (~1 L3 MVP)
- `1M` — Deep exploration (~3-4 L3 MVPs)
- `10M` — Massive exploration (~8-12 L3 MVPs or full product line)
- `burn` — Burn until stopped (continuously generate until request fails)

If the user selects `10k`, explicitly inform: "10k is insufficient for an L3 MVP; output will be downgraded to L1/L2. Continue?"

**Then ask:**
```
📂 Which project directories should be read to analyze extension directions?
(Leave empty to explore directly based on your domain interests without analyzing existing projects)

Examples:
- misc/splatcraft
- misc/wm_atlas
- ../another_workspace/some_project
```

Also ask:
```
🎯 What technical domain or desired direction are you currently focused on?
(Leave empty for the skill to discover autonomously)
```

**Simultaneously auto-detect and confirm the memory path:**

The Main Agent uses Bash/Glob to attempt locating the Claude memory directory on the current device:
- Linux/macOS: `~/.claude/projects/*/memory/`
- Windows: `%USERPROFILE%/.claude/projects/*/memory/`

Present the detected path to the user:
```
🧠 Detected memory directory: <detected_path>
Correct? (Press Enter to confirm, or input the correct memory path)
```

Record the user-confirmed path in `.burn_state.json` under the `memory_path` field for future session reuse.

**Four execution modes:**
- **Mode A** (user provided directories + domain): Analyze existing projects → Combine with domain interests → Generate directions
- **Mode B** (user provided directories, no domain): Analyze existing projects → Autonomously discover extension directions
- **Mode C** (no directories, user provided domain): Skip existing project analysis → Directly research niche directions based on domain interests
- **Mode D** (neither provided): Ask the user to provide at least one; otherwise cannot proceed

### Step 1: Analyze Existing Projects (Modes A/B only)

Read all directories confirmed by the user in Step 0.
For each directory, look for:
- `*/PRD.md`
- `*/PROGRESS.md`
- `*/README.md`

Extract tech stack, problems solved, and uncovered adjacent domains.

If a directory does not exist or is empty, log and skip it without terminating with an error.

Also read the following locations for user background:
- `burn-my-tokens-MVP_output/burn_memory/*.md`
- `<user_memory_path>/*.md` (path confirmed in Step 0)

If memory path detection fails and the user did not manually specify one, skip memory reading without affecting the main flow.

### Step 2: Generate Direction List

Based on analysis results (Modes A/B) or domain interests (Mode C), use WebSearch to research 2-3 potential niche directions and generate a candidate project list (3-5 items).
Each candidate includes:
- Project name (short, lowercase, no underscores)
- One-sentence value description
- Technical connection to existing projects (or domain connection)
- Estimated token cost (L3: 20k-50k, L2: 5k-15k, L1: 2k-5k)

### Step 3: Self-Critique (Red Team)

The Main Agent reviews the list from a "critic" perspective and eliminates:
- Items with >50% overlap with existing projects
- Pure concepts with no technical feasibility
- Dependencies on environments the user clearly lacks (e.g., CUDA) that cannot be downgraded

### Step 4: Critique-Agent Review

Launch a critique-agent with:
- Existing project summary (or user's domain interests)
- Candidate list
- User memories (memory files from the Step 0 confirmed path)

critique-agent outputs:
- Score for each candidate (1-10)
- Eliminate / keep recommendations
- Priority ranking of kept projects

### Step 5: Present & Confirm

Output the final list to the user:
```
🔥 burn-my-tokens candidate directions (sorted by priority):

1. <name>: <description> (~XXk tokens)
2. <name>: <description> (~XXk tokens)
3. <name>: <description> (~XXk tokens)

Mode: [Semi-auto] Please reply with project numbers to execute (e.g., 1,3 or all)
      [Full-auto] All directions auto-selected, starting now
```

If the user selected "Full-auto" mode in Step 0, skip confirmation and start immediately.

### Step 6: Allocate Subagents

Launch subagents based on user selection.

**Scheduling strategy (hybrid mode):**
- First 2 projects launched in parallel (2 Agents executing simultaneously)
- After completion, evaluate remaining budget before deciding whether to continue the next batch

Each subagent's directory: `burn-my-tokens-MVP_output/<project_name>/`

### Step 7: Monitor & Report

Subagents report progress via message after each Step (research / PRD / dev / test / verification).
The Main Agent aggregates and reports to the user:
```
🔥 burn-my-tokens progress update:

✅ <project_a>: Step 3/6 in development (approx. XXk tokens remaining)
⏳ <project_b>: Step 5/6 in verification

Remaining budget: approx. XXk / XXk
```

The user can input "progress" or "status" at any time for the Main Agent to output the current state.

### Step 8: Loop Check

After all selected projects complete:

**Tier modes (10k/100k/1M/10M):**
- The Main Agent estimates cumulative consumption. If it reaches 95% of the tier, pause and output a summary report.
- If budget remains, return to Step 2 to generate new directions.

**Burn-until-stopped mode:**
- Return to Step 2 to continuously generate new projects.
- When a subsequent request fails due to any blocking error, catch the error, call `CronDelete` to remove the resume task, output the final summary report, and gracefully exit.

### Step 9: Final Report

Output:
```
🔥 burn-my-tokens burn complete!

Projects produced:
- burn-my-tokens-MVP_output/<project_a>/ (L3/L2/L1)
- burn-my-tokens-MVP_output/<project_b>/ (L3/L2/L1)

Total estimated consumption: XXk tokens
See PROGRESS.md in each directory for project details
```

---

## Burn Contract

The Main Agent must adhere to the following hard constraints and must not violate them for any reason:

### Constraint 1: Never Stop on Your Own
- After completing a project, immediately check the budget
- If budget is not exhausted, immediately begin the next project (return to Step 2 or launch a new Subagent)
- Prohibited from outputting terminal phrases such as "that's all for now", "let's pause here", "continue next time"
- The only stop conditions are: reaching the 95% tier threshold, API returning 429/403, or user-triggered graceful stop
- "Ending the response between projects" is not a stop — it is a necessary window to respond to queued user instructions. `.burn_state.json` remains `status == "running"`, and `CronCreate` auto-resumes

### Constraint 2: Degrade on Error, Don't Terminate
- When a Subagent fails, the Main Agent must attempt degradation (L3→L2→L1), not terminate the entire flow
- Only after 3 consecutive degradation failures is skipping the project and continuing to the next allowed
- When skipping a project, briefly log the reason without extended discussion

### Constraint 3: Time-Based Resume
- If the response is perceived to be approaching the time limit (executed >4 minutes), the Main Agent must:
  1. Write current progress to `burn-my-tokens-MVP_output/.burn_state.json`
  2. Ensure the CronCreate resume task is started
  3. Output: "Time limit triggered resume; scheduled task will auto-restart burning in 3 minutes"
- Upon resume, read `.burn_state.json` to restore context

### Constraint 4: Progress Reports Must Not Block Execution
- Progress reports must be embedded in the execution flow in the most concise format possible
- Prohibited from interrupting the execution chain for a "complete report"
- Detailed reports only when the user queries progress

### Constraint 5: Budget Visualization Obligation
- After each project completes, update and display `[Token Tracker]`
- This serves as both a report and a self-restraint trigger for the Main Agent

### Constraint 6: Subagent Timeout
- If a subagent produces no output within reasonable time, treat as failure and degrade.

---

## Core Execution Loop (Burn Loop)

The Main Agent's core execution logic is as follows. It must iterate as much as possible within a single response:

```
while estimated_spent < target_budget * 0.95:
    # 1. Check for unfinished subagents
    if pending_agents:
        collect_results()
    
    # 2. If direction list is empty, regenerate
    if not candidate_list:
        candidate_list = generate_directions()
        candidate_list = self_critique(candidate_list)
        candidate_list = critique_agent_review(candidate_list)
    
    # 3. Launch next batch of subagents (max 2 in parallel)
    batch = candidate_list.pop_next(2)
    for project in batch:
        if estimated_spent + project.estimated_cost > target_budget * 0.95:
            break
        launch_subagent(project)
        estimated_spent += project.estimated_cost
    
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

## Subagent L3 MVP Workflow

Each subagent must strictly follow these 6 steps:

### Step 1: Web Research (~15-20% budget)
- Use WebSearch to research technical solutions and feasibility
- Output: `research.md`

### Step 2: Write PRD (~15-20% budget)
- Format: Problem Statement, Solution, User Stories, Implementation Decisions, Testing Decisions, Out of Scope, Acceptance Criteria
- Reference AutoPRD output style
- Output: `PRD.md`

### Step 3: Core Development (~40-50% budget)
- Implement MVP code according to PRD
- Follow Python Coding Standards (`.claude/rules/python-coding-standards.md`)
- Must include `requirements.txt` and a runnable `main.py`
- Output: `src/` directory

### Step 4: Unit Tests (~10-15% budget)
- pytest, covering core functionality
- Tests must pass
- Output: `tests/` directory

### Step 5: Verification Run (~5-10% budget)
- Actually run `main.py` or the core entry point
- Record run results
- If run fails: **auto-degrade to L2**, keep code skeleton and PRD, document failure reason in `PROGRESS.md`
- Output: run log summary

### Step 6: Write PROGRESS.md (~5% budget)
- Record milestones, known issues, how to run
- Output: `PROGRESS.md`

---

## Project Specifications

- **Directory**: `burn-my-tokens-MVP_output/<project_name>/`
- **Naming**: Short, lowercase, no underscores (e.g., `gsplatmobile`, `tracedash`)
- **Coding**: Strictly follow `.claude/rules/python-coding-standards.md`
- **Dependencies**: Python 3.10+, prefer pure Python or pip-installable libraries

---

## Memory Updates

After each project completes, the Main Agent automatically writes:
`burn-my-tokens-MVP_output/burn_memory/<project_name>.md`

Format:
```markdown
---
name: <project_name>
description: <one-sentence description>
type: project
date: <YYYY-MM-DD>
depth: <L1/L2/L3>
status: <completed/demoted>
---

- Problem solved:
- Tech stack:
- Relationship to existing projects:
- Known limitations:
- Can serve as a base for future extensions:
```

---

## Token Tracking Rules

**Important**: Claude Code does not provide a remaining token API; all tracking is estimated.

- Each WebSearch ≈ 500-1500 tokens
- Each file read/write ≈ proportional to file size
- Each Agent call ≈ 2000-5000 tokens overhead

The Main Agent maintains a running estimate in the conversation, formatted as:
```
[Token Tracker] Estimated consumed: XXk / Target: XXk
```

**Tier mode 95% threshold**:
When estimated consumption reaches 95% of the target tier, the Main Agent proactively pauses and reports:
> "Estimated consumption is approaching the target budget (XXk / XXk). Continue with the last project or stop?"
If the user chooses to stop, call `CronDelete` to remove the resume task before exiting.

**Burn-until-stopped mode**:
Do not stop proactively. When a subsequent Claude API call returns any blocking error (429/403/500/timeout), catch the exception, call `CronDelete` to remove the resume task, output the final report, and gracefully exit.

---

## Resume Mechanism (Pure Text Version)

This skill **does not depend on MetaBot and requires no command-line operations**. Resume is implemented via Claude Code's built-in `CronCreate`, fully transparent to the user.

### State File

The Main Agent updates this at the start of each loop (on resume, if missing/corrupt, reconstruct from output dirs or start fresh):
`burn-my-tokens-MVP_output/.burn_state.json`

Content:
```json
{
  "status": "running",
  "mode": "1M",
  "target_budget": 1000000,
  "estimated_spent": 3500,
  "completed_projects": 1,
  "pending_projects": ["project_b", "project_c"],
  "last_heartbeat": "2026-05-11T14:30:00",
  "cron_job_id": "burn-resume-xxx",
  "memory_path": "~/.claude/projects/xxx/memory"
}
```

### Auto-Resume Scheduled Task

**On first startup**, the Main Agent calls `CronCreate` to create a durable scheduled task:

- **cron**: `1-59/3 * * * *` (every 3 minutes, avoiding top of the hour)
- **durable**: `true` (persists across session restarts)
- **prompt**:
  ```
  Read burn-my-tokens-MVP_output/.burn_state.json.
  If status == "running" and last_heartbeat is more than 10 minutes ago:
    Execute /burn-my-tokens-MVP resume
  Otherwise: ignore, do nothing.
  ```

Record the returned `job_id` in `.burn_state.json` under the `cron_job_id` field.

**After burning completes**, the Main Agent calls `CronDelete` to remove the scheduled task to avoid useless polling.

### User-Transparent Flow

```
User: /burn-my-tokens-MVP
Claude: [Select tier] [Confirm directories] [Confirm memory path] [Full/Semi-auto project selection] → Start burning
Claude: [Internal] CronCreate creates resume task
...
[Session ends due to time limit]
[3 minutes later, cron auto-triggers]
Claude: [Auto-resumes burning from breakpoint]
...
[Budget exhausted]
Claude: [Burn complete report] [CronDelete removes resume task]
```

### Resume Parameters

When the scheduled task or user triggers `/burn-my-tokens-MVP resume`:
1. The Main Agent reads `.burn_state.json`
2. Restores `estimated_spent`, `completed_projects`, `pending_projects`, `memory_path`
3. Continues the Burn Loop from the breakpoint
4. No need to re-ask tier, directories, or memory path (unless the state file is missing)
