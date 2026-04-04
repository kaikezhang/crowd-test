# CrowdTest v2 — Multi-Agent E2E Testing Design Spec

> Date: 2026-04-04
> Status: Approved
> Budget: MVP

## 1. Goal

Upgrade CrowdTest from "one agent pretending to be N users" to "N independent sub-agents, each performing real E2E browser testing as a unique persona." The upgrade must:

1. Keep backward compatibility — single-agent environments (Claude Code, Cursor, etc.) still work with the existing serial flow
2. Auto-detect orchestrator capabilities and switch to multi-agent mode when available
3. Emphasize **real browser E2E testing** via Playwright for web projects
4. Produce the same structured report format as v1

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│ SKILL.md (single file, Agent Skill standard)            │
│                                                         │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Environment Detection (top of SKILL.md)             │ │
│ │                                                     │ │
│ │  Can spawn sub-agents?  ──yes──► Orchestrator Mode  │ │
│ │       │                                             │ │
│ │       no                                            │ │
│ │       │                                             │ │
│ │       ▼                                             │ │
│ │  Legacy Mode (current serial flow, unchanged)       │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                         │
│ Phase 0-3: Unchanged (main agent executes)              │
│   Scout → Product Analysis → Persona Gen → Tasks        │
│                                                         │
│ Phase 4-6: Multi-Agent (Orchestrator) or Serial (Legacy)│
│   Orchestrator: dispatch sub-agent per persona          │
│   Legacy: single agent runs all personas inline         │
│                                                         │
│ Phase 7-8: Unchanged (main agent aggregates)            │
│   Aggregation → Score → Report → Delta                  │
└─────────────────────────────────────────────────────────┘
```

## 3. Environment Detection

Add a new section near the top of SKILL.md, after Prerequisites:

```markdown
## Execution Mode Detection

Before starting the pipeline, determine which mode to use:

**Orchestrator Mode** — if ANY of these are available:
- `sessions_spawn` tool (OpenClaw)
- Ability to exec background processes with `codex` or `claude` CLI
- Any mechanism to dispatch independent agent sessions

**Legacy Mode** — if none of the above:
- Fall back to current serial execution (one agent, all personas inline)
- No changes to existing Phase 4-6 logic

The rest of this document uses [ORCHESTRATOR] and [LEGACY] tags
to mark mode-specific instructions.
```

## 4. Orchestrator Mode — Phase 4-6 Redesign

### 4.1 Serial Execution (One Persona at a Time)

```
For persona_index = 1 to N:
  1. Main agent prepares a "Persona Test Package" (see §4.2)
  2. Main agent dispatches ONE sub-agent with the package
  3. Wait for sub-agent to complete (poll or webhook)
  4. Main agent reads sub-agent output files from worktree
  5. Extract covered_issues from this persona's feedback
  6. Append covered_issues to cumulative list
  7. Report progress: "✅ Persona {i}/{N}: {name} — {result}"
  8. If sub-agent failed: log failure, skip to next persona
  9. Clean up worktree
```

### 4.2 Persona Test Package

Each sub-agent receives a self-contained instruction set. The main agent writes this to a file (`PERSONA-TASK.md`) in the sub-agent's worktree:

```markdown
# Persona Test Instructions

## Your Identity
{persona profile: name, age, tech_level, archetype, device, behavioral_rules[]}

## Your Task
{task description from Phase 3}

## Target Product
- URL: {url}
- SiteMap: {JSON from Phase 1 Scout}
- Product Context: {from Phase 0}

## Already Known Issues (avoid repeating these)
{covered_issues[] from previous personas}

## How to Test

### Browser Setup
1. Launch Playwright with a fresh browser context
2. Set viewport to {device_viewport} based on your device type
3. Navigate to {url}

### Testing Loop (ORIENT → ACT → OBSERVE → DECIDE)
For each action:
1. ORIENT: Take accessibility snapshot, understand current page
2. ACT: Perform one action based on your persona's behavior rules
3. OBSERVE: Take new snapshot, compare with expectation
4. DECIDE: Continue exploring, or move to next task step

### Behavioral Rules
- Follow your persona's behavioral rules strictly
- If confused for >3 actions, express frustration and try alternative path
- Maximum 30 actions per session
- Track emotional state: confident / neutral / confused / frustrated / ready_to_leave

### Recording Format
Log every action to `action-log.json`:
{action log JSON schema}

### When Done
1. Write `action-log.json` — raw action log
2. Write `journey.json` — confidence curve + key moments + narrative
3. Write `feedback.json` — structured feedback with scores
4. Exit
```

### 4.3 Sub-Agent Output Schema

Each sub-agent produces 3 files in its worktree:

**action-log.json:**
```json
{
  "persona_id": "persona_001",
  "url": "https://example.com",
  "device": "mobile",
  "viewport": {"width": 375, "height": 812},
  "started_at": "ISO-timestamp",
  "completed_at": "ISO-timestamp",
  "status": "completed|failed|abandoned",
  "failure_reason": null,
  "actions": [
    {
      "step": 1,
      "phase": "ORIENT|ACT|OBSERVE|DECIDE",
      "action_type": "navigate|click|type|scroll|snapshot|wait",
      "target": "element description or selector",
      "expected": "what persona expected to happen",
      "actual": "what actually happened",
      "emotional_state": "confident|neutral|confused|frustrated|ready_to_leave",
      "timestamp": "ISO-timestamp",
      "snapshot_summary": "brief accessibility tree summary"
    }
  ]
}
```

**journey.json:**
```json
{
  "persona_id": "persona_001",
  "confidence_curve": [8, 8, 6, 4, 2, 2, 4, 6, 8],
  "key_moments": [
    {"type": "positive|confusion|recovery|abandonment|delight", "step": 1, "description": "..."}
  ],
  "narrative": "First-person journey story (3-5 sentences)",
  "journey_summary": "One-line arc description"
}
```

**feedback.json:**
```json
{
  "persona_id": "persona_001",
  "task_result": "COMPLETED|FAILED|PARTIAL|ABANDONED",
  "scores": {
    "first_impression": 7,
    "navigation": 6,
    "trust": 5,
    "error_handling": 4,
    "nps": 5
  },
  "issues": [
    {
      "id": "issue_001",
      "title": "Short descriptive title",
      "severity": "critical|high|medium|low",
      "description": "Detailed grounded description",
      "evidence_steps": [7, 8, 9],
      "suggested_fix": "Actionable suggestion",
      "persona_quote": "First-person quote from the persona"
    }
  ],
  "positives": [
    {
      "feature": "What worked well",
      "evidence_steps": [1, 2],
      "quote": "First-person positive comment"
    }
  ],
  "competitor_comparison": "If persona has competitor experience, comparison notes"
}
```

### 4.4 Worktree Management

Each sub-agent gets a temporary worktree:

```bash
# Before dispatching persona N:
git worktree add -b crowdtest/persona-N /tmp/crowdtest-persona-N main

# After collecting results:
git worktree remove /tmp/crowdtest-persona-N --force
git branch -D crowdtest/persona-N
```

The worktree contains:
- `PERSONA-TASK.md` — the test instructions (written by main agent)
- `action-log.json` — output (written by sub-agent)
- `journey.json` — output (written by sub-agent)
- `feedback.json` — output (written by sub-agent)

### 4.5 Sub-Agent Dispatch

The main agent dispatches sub-agents using available mechanisms:

**OpenClaw (sessions_spawn):**
```
sessions_spawn({
  task: "Read PERSONA-TASK.md and execute the persona test. Write results to action-log.json, journey.json, feedback.json.",
  cwd: "/tmp/crowdtest-persona-N",
  mode: "run",
  runtime: "subagent"
})
```

**CLI (exec fire-and-forget):**
```bash
cd /tmp/crowdtest-persona-N && \
codex --dangerously-bypass-approvals-and-sandbox exec \
  "Read PERSONA-TASK.md and execute the test instructions. Write output files."
```

Or with Claude Code:
```bash
cd /tmp/crowdtest-persona-N && \
claude -p "Read PERSONA-TASK.md and execute the test instructions. Write output files." \
  --dangerously-skip-permissions
```

### 4.6 Error Handling

- Sub-agent timeout (default 5 min per persona): kill, log failure, continue
- Sub-agent crash: log error, mark persona as failed, continue
- Missing output files: treat as failed persona
- Partial output (e.g., action-log exists but no feedback): use what's available
- All personas failed: report error to user with diagnostics

### 4.7 Progress Reporting

After each persona completes:
```
Persona 1/10: Maria Chen (first_timer, mobile)
  ├── Task: "Sign up and create first project" → ✅ COMPLETED
  ├── Issues found: 3 (1 critical)
  └── Emotional arc: confident → confused → frustrated → recovered

Persona 2/10: James Wright (power_user, desktop)
  ├── Task: "Import CSV data and generate report" → ❌ FAILED (timeout)
  ├── Issues found: 0
  └── Skipped — will note in final report
```

## 5. E2E Browser Testing Requirements

### 5.1 Playwright Integration

The sub-agent MUST use real browser automation, not simulated interactions:

- Use Playwright MCP (`@playwright/mcp`) if available as a tool
- Otherwise use Playwright Node.js API directly via exec
- Each persona gets a fresh `BrowserContext` (clean cookies, storage, cache)
- Viewport set according to persona's device type:
  - Desktop: 1920×1080
  - Tablet: 768×1024  
  - Mobile: 375×812

### 5.2 Accessibility Snapshots over Screenshots

- Primary observation method: `browser_snapshot` (accessibility tree)
- Token-efficient (~200-500 tokens vs 1500+ for screenshots)
- Deterministic and grounded — references actual DOM elements
- Screenshots only as supplementary evidence for specific visual issues

### 5.3 Non-Web Projects

For non-web products (CLI tools, APIs, desktop apps):
- Skip Playwright, use appropriate testing method
- CLI: exec commands and observe output
- API: curl/httpie requests
- The SKILL.md should detect project type from Scout phase and adapt

## 6. Backward Compatibility

### 6.1 Legacy Mode (No Changes)

When orchestrator capabilities are not detected:
- Phase 4-6 run exactly as they do today
- Single agent, single browser, serial persona loop
- All existing behavior preserved

### 6.2 Detection Logic

```
IF tool "sessions_spawn" is available:
  → Orchestrator Mode (OpenClaw native)
ELSE IF command "codex" exists on PATH:
  → Orchestrator Mode (Codex CLI dispatch)
ELSE IF command "claude" exists on PATH:
  → Orchestrator Mode (Claude Code CLI dispatch)
ELSE:
  → Legacy Mode
```

### 6.3 User Override

Allow explicit mode selection via flags:
```
/crowd-test https://myapp.com --mode orchestrator
/crowd-test https://myapp.com --mode legacy
```

## 7. Changes to Existing Phases

### Phase 0-3: No changes
Scout, Product Analysis, Persona Generation, Task Assignment — all stay the same.

### Phase 4: Browser Sessions
- [LEGACY] Unchanged
- [ORCHESTRATOR] Replaced by sub-agent dispatch (§4)

### Phase 5: Journey Reconstruction  
- [LEGACY] Main agent processes action log inline
- [ORCHESTRATOR] Sub-agent writes journey.json; main agent reads it

### Phase 6: Feedback Synthesis
- [LEGACY] Main agent generates feedback inline
- [ORCHESTRATOR] Sub-agent writes feedback.json; main agent reads it

### Phase 7-8: No changes
Aggregation, scoring, report generation, delta analysis — all stay the same.
Main agent reads all persona outputs (from worktree files or inline) and aggregates.

## 8. SKILL.md Structure (New Sections)

The updated SKILL.md will have these new/modified sections:

1. **Execution Mode Detection** (new, after Prerequisites)
2. **Orchestrator Mode Overview** (new, before Phase 4)
3. **Phase 4-6 [ORCHESTRATOR]** (new alternate path)
4. **Sub-Agent Test Package Template** (new, as appendix or inline)
5. **Output File Schemas** (new, reference for sub-agents)
6. Phase 4-6 [LEGACY] tags on existing content (minimal changes)

## 9. Success Criteria

- [ ] SKILL.md auto-detects orchestrator vs legacy mode
- [ ] Orchestrator mode dispatches one sub-agent per persona, serially
- [ ] Each sub-agent performs real Playwright E2E browser testing
- [ ] Each sub-agent works in an isolated worktree
- [ ] Sub-agent outputs (action-log, journey, feedback) follow defined JSON schemas
- [ ] Covered issues accumulate across personas (no duplicate reporting)
- [ ] Failed personas don't abort the run
- [ ] Progress reported after each persona
- [ ] Final report (Phase 7-8) works with both modes' output
- [ ] Legacy mode is completely unchanged
- [ ] `--mode` flag allows explicit override

## 10. Non-Goals (v2)

- No parallel execution (serial only for now)
- No custom browser profiles per persona (clean context only)
- No video recording of browser sessions
- No visual regression testing (accessibility snapshots only)
- No persistent persona memory across runs
