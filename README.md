# CrowdTest

> Generate synthetic users, let them operate your product like real people, and get brutally honest feedback.

CrowdTest is an **Agent Skill** for running **multi-agent product testing**.

A main agent:
1. understands the product,
2. creates personas,
3. assigns focused tasks,
4. dispatches one sub-agent per persona,
5. lets each sub-agent use a real browser in an isolated worktree,
6. aggregates the findings into a report.

This is not “one LLM pretending to be 10 users in one chat.”
This is **orchestrated, persona-specific, browser-based testing**.

---

## What CrowdTest Does

CrowdTest is built for situations like:
- “Test my onboarding flow with 5 different user types.”
- “Only test the export flow on mobile.”
- “Act like a skeptical first-time user and see where you get stuck.”
- “Simulate desktop vs mobile users with different screen sizes.”
- “Get a harsh UI/UX review, not polite fluff.”

The skill is optimized for:
- web apps
- landing pages
- editors and dashboards
- multi-step user flows
- focused feature validation

---

## Core Model

CrowdTest runs in **orchestrator mode**.

### Main agent responsibilities
The main agent handles:
- target analysis
- product understanding
- persona generation
- task assignment
- deciding device + viewport per persona
- deciding complexity budget per task
- collecting results
- final aggregation and reporting

### Sub-agent responsibilities
Each sub-agent handles:
- one persona only
- one isolated worktree
- one browser session
- real user-like interactions (`click`, `type`, `drag`, `scroll`)
- writing structured outputs:
  - `action-log.json`
  - `journey.json`
  - `feedback.json`

---

## Key Design Principles

### 1. Real interaction, not DOM cheating
CrowdTest prefers human-like actions:
- click
- type
- drag
- scroll

Sub-agents are instructed **not** to use JavaScript injection to click hidden elements or manipulate the DOM directly.
If a real user cannot reach something through the visible UI, that is a finding.

### 2. Persona-specific environments
Each persona includes:
- device type
- exact viewport
- context / motivation
- behavioral rules
- task focus

Examples:
- desktop laptop `1366×768`
- desktop 1080p `1920×1080`
- mobile iPhone `390×844`
- mobile small phone `375×667`
- Android `412×915`
- tablet `820×1180`

The browser is resized to match the persona before testing begins.

### 3. Focus Mode
If the caller specifies a feature to test, CrowdTest switches into **Focus Mode**.

That means:
- use the shortest path to reach the target feature
- skip unrelated pages and flows
- treat setup steps as prerequisites, not test targets
- spend the action budget on the requested feature only

Example:
- `--focus "export flow"`
- `--focus "mobile nav, checkout"`

### 4. Dynamic budgets by task complexity
Task limits are chosen by the main agent based on complexity:

| Complexity | Actions | Time |
|------------|---------|------|
| Light | 10–20 | 10 min |
| Standard | 20–40 | 20 min |
| Deep | 40–60 | 40 min |

This prevents simple tasks from wandering forever while still allowing complex flows enough room to finish.

### 5. Ruthless feedback quality
CrowdTest is designed to be:
- direct
- structured
- specific
- actionable
- intolerant of weak UX

The main agent can reference `references/ui-review-prompt.md` to generate harsher, more systematic UI critiques.

---

## Output Format

Each persona produces three artifacts.

### `action-log.json`
A step-by-step record of what the user actually did.

Includes things like:
- action type
- target element
- expected outcome
- actual outcome
- success/failure
- emotional state
- progress

### `journey.json`
A compressed narrative of the session.

Includes:
- confidence curve
- key moments
- short first-person story
- one-line journey summary

### `feedback.json`
Structured persona feedback.

Includes:
- task completion result
- prioritized issues
- positives
- scoring
- verdict

Typical scoring dimensions include:
- first impression
- task completion
- navigation
- trust
- error handling
- visual quality
- device fit
- nps

---

## Typical Workflow

### 1. Product understanding
CrowdTest first figures out:
- what the product does
- who it is for
- what core tasks matter
- what pages and entry points exist

### 2. Persona generation
It creates personas based on:
- intended user type
- task relevance
- device + viewport
- emotional context
- competitor familiarity
- usage environment

### 3. Task assignment
Each persona gets:
- a primary task
- success criteria
- optional secondary tasks
- focus features (if applicable)
- time/action budget based on complexity

### 4. Sub-agent execution
For each persona:
- create isolated worktree
- write `PERSONA-TASK.md`
- dispatch sub-agent
- run real browser session
- collect output files

### 5. Aggregation
Main agent deduplicates issues and produces a final report.

---

## Browser / WebGL Notes

CrowdTest often runs in headless or GPU-less environments.
For browser-heavy apps that require WebGL, CrowdTest works best when Chromium is launched with software rendering enabled.

In this repo’s development/testing environment, this was solved via persistent browser flags in OpenClaw config:
- `--enable-unsafe-swiftshader`
- `--use-angle=swiftshader`
- `--ignore-gpu-blocklist`

This allows WebGL-dependent apps (like map editors) to be tested without a physical GPU.

---

## Repository Layout

```text
SKILL.md                     # Main CrowdTest skill definition
README.md                    # This file
references/
  ui-review-prompt.md        # Strict UI review reference prompt
docs/
  PRD.md
  technical-design.md
  roadmap.md
  ...
```

This repo is primarily a **skill + prompt architecture** project, not a traditional application codebase.

---

## Current Status

CrowdTest has evolved from an early single-agent concept into an **orchestrator-first testing skill** with:
- isolated sub-agent execution
- worktree-based persona separation
- real browser interactions
- speed/focus guardrails
- focus-mode testing
- per-persona device simulation
- stricter structured feedback

---

## What This Repo Is Not

CrowdTest is **not**:
- a replacement for real user testing
- a visual diff regression tool
- a generic screenshot linter
- a static heuristic checklist only

It is best understood as:
> a multi-agent synthetic user testing system that creates realistic browser-based task runs and turns them into structured product feedback.

---

## Usage Notes

CrowdTest is intended to be invoked through the host agent system that loads `SKILL.md`.
The exact command format depends on the host environment (OpenClaw, Claude Code skills, etc.).

Conceptually, inputs look like:
- target URL
- number of personas
- optional focus area(s)
- optional product context
- optional known competitors
- optional auth/storage state

---

## Development Notes

This repo contains planning and design docs used during development.
Runtime artifacts, temporary reports, VibeFlow state, and exploratory files should generally not be committed unless they are intentionally part of the product.

---

## License

MIT
