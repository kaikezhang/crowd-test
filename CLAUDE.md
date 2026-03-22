# CLAUDE.md — CrowdTest Project Guide (for Claude Code)

## Project Overview

**CrowdTest** is an open-source AI tool that generates diverse synthetic user personas to test web products and produce structured UX feedback.

- **Repo**: github.com/kaikezhang/crowd-test
- **Packaging**: Agent Skills 开放标准 (SKILL.md) — 兼容 Claude Code / Cursor / Gemini CLI / Copilot / OpenClaw 等 15+ 工具
- **安装**: `npx skills add kaikezhang/crowd-test`
- **Core idea**: `/crowd-test https://myapp.com` → N personas browse → grounded feedback → prioritized report

## Architecture

```
Scout → Persona Generator → Browser Sessions → Feedback Writer → Aggregator → Report
```

5 modules, each a distinct LLM orchestration step. See `docs/technical-design.md` for full data flow and state machines.

## Tech Stack

- **Browser automation**: Playwright MCP (`@playwright/mcp`) — accessibility snapshots, NOT screenshots
- **LLM**: Claude Sonnet 4.6 (default), Opus 4.6 (optional for aggregation)
- **Observation**: Accessibility snapshots only (token efficient, deterministic)
- **Execution**: Serial (v1) — one persona at a time
- **Output**: Markdown report (v1), HTML (v2)
- **Packaging**: Agent Skills 标准 (SKILL.md) — 兼容 Claude Code / Cursor / OpenClaw 等

## Key Files

| File | Purpose |
|------|---------|
| `SKILL.md` | The skill definition — orchestration prompt |
| `docs/PRD.md` | Product requirements |
| `docs/technical-design.md` | Architecture, data structures, prompt designs |
| `docs/competitive-research.md` | Competitive analysis |
| `docs/ceo-review.md` | Product strategy review |
| `docs/eng-review.md` | Engineering feasibility review |
| `docs/roadmap.md` | Version milestones |
| `docs/sprint-plan.md` | Sprint-level task breakdown |
| `TASK.md` | Current sprint task spec |

## Core Data Structures

Defined in `docs/technical-design.md` section 2:
- `SiteMap` — Scout output
- `Persona` — Generated user profile with behavioral rules
- `ActionLog` — Browser session recording
- `PersonaFeedback` — Grounded feedback per persona
- `AggregatedReport` — Cross-persona deduplicated report

## Design Principles

### 1. Grounded Feedback (NOT Generic Slop)

The #1 quality requirement. Three-phase feedback pipeline:
1. **Phase 1**: Browser session produces action_log (ground truth)
2. **Phase 2**: LLM reviews action_log and writes feedback referencing specific steps
3. **Phase 3**: Structured extraction with evidence pointers

**Skip Phase 1 = worthless output.** Every issue MUST reference an action log step.

### 2. Behavioral Rules > Personality Descriptions

```
BAD:  "You are impatient and easily frustrated"
GOOD: "After 2 failed clicks, express frustration. After 5 total failures, abandon."
```

LLMs are too smart to "act dumb." Use executable constraints, not personality adjectives.

### 3. Scout Is Table Stakes

Before ANY persona runs, the Scout phase MUST:
- Verify page loads (>5 interactive elements)
- Detect auth walls → prompt user for `--storage-state`
- Detect CAPTCHAs → abort with clear message
- Dismiss cookie banners
- Build site_map for persona generator

Without Scout, 30%+ of runs produce garbage feedback about loading screens.

### 4. Cumulative Diversity

Each persona receives "already covered issues" from previous personas and is instructed to find DIFFERENT problems. Without this, 10 personas give 10 variations of the same 3 observations.

### 5. Canary Self-Validation

After generating feedback, an LLM check asks: "Is this specific or generic?" Flag and optionally regenerate generic feedback.

## Constraints

| Constraint | Value | Reason |
|-----------|-------|--------|
| Max actions per persona | 20 | Circuit breaker |
| Max time per persona | 3 minutes | Real users form opinions fast |
| Observation mode | Accessibility snapshots | 3-5x cheaper than screenshots |
| Max retries (page load) | 2 | Don't waste time on broken pages |
| Max consecutive failures | 5 | Force stop unstable sessions |
| Default persona count | 10 | Sweet spot: diverse + fast + cheap |
| Default model | Sonnet 4.6 | Best cost/quality ratio |

## Development Rules

### Branching
- Work on feature branches: `feat/module-name`
- **Create PR and stop. Do NOT merge.**晚晚 merges after cross-review.

### Code Quality
- Type annotations on all data structures
- JSON schema validation on LLM outputs (with retry on parse failure)
- Every module must handle its failure modes gracefully (see tech design section 6)

### Testing
- Test fixtures with planted bugs for validation
- Canary self-check on feedback quality
- `≥75%` known-issue discovery rate on test fixtures

### Commit Messages
- Conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`
- Reference module: `feat(scout): add cookie banner dismissal`

## Role Clarification

You are the **implementer**. 晚晚 (the orchestrator) makes architectural decisions, reviews PRs, and merges. You implement per TASK.md spec, create PRs, and stop.

**Read TASK.md for your current assignment.** If TASK.md doesn't exist or is empty, ask 晚晚 for instructions.
