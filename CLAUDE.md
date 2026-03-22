# CLAUDE.md — CrowdTest Project Guide (for Claude Code)

## Project Overview

**CrowdTest** is an open-source AI tool that generates diverse synthetic user personas to test web products and produce structured UX feedback.

- **Repo**: github.com/kaikezhang/crowd-test
- **Packaging**: Agent Skills 开放标准 (SKILL.md) — 兼容 Claude Code / Cursor / Gemini CLI / Copilot / OpenClaw 等
- **安装**: `npx skills add kaikezhang/crowd-test`
- **Core idea**: `/crowd-test https://myapp.com` → N personas browse → Product Score + "Fix ONE Thing" report
- **KEY INSIGHT**: The product IS the SKILL.md. There is NO application code. Everything is prompt orchestration.
- **Canonical vision**: `docs/product-design-v2.md` — READ THIS FIRST for design philosophy

## Architecture (v2 Pipeline — 8 Phases)

```
Phase 0: Product Analysis  → "What IS this product? Who's it for? What can they DO?"
Phase 1: Scout             → "Is it testable? Build a site map."
Phase 2: Persona Generation → "Who would use this?" (derived FROM the product, not random)
Phase 3: Task Assignment   → "What should each persona TRY to do?" (specific, measurable)
Phase 4: Browser Sessions  → persona browses with task + emotional state tracking
Phase 5: Journey Recon     → action log → narrative arc + confidence curve + key moments
Phase 6: Feedback Synthesis → grounded, tiered, evidence-linked feedback per persona
Phase 7: Aggregation       → Product Score (X/10) + "Fix ONE Thing" + deduplicated issues
Phase 8: Delta Analysis    → compare with previous run if exists
```

Each phase is a section in SKILL.md with clear inputs, steps, and output schemas.

## Tech Stack

- **Browser**: Playwright MCP (`@playwright/mcp`) — accessibility snapshots, NOT screenshots
- **LLM**: Claude Sonnet 4.6 (default), Opus 4.6 (optional for aggregation)
- **Observation**: Accessibility snapshots only (3-5x cheaper than screenshots, deterministic)
- **Execution**: Serial (v1) — one persona at a time
- **Output**: Markdown report with Product Score
- **No app code**: SKILL.md is the entire product

## Key Files

| File | Purpose |
|------|---------|
| `SKILL.md` | **THE PRODUCT** — orchestration prompt for all 8 phases |
| `docs/product-design-v2.md` | Canonical vision (read this for WHY decisions were made) |
| `docs/technical-design.md` | Data structures, prompt templates |
| `docs/PRD.md` | Product requirements |
| `docs/competitive-research.md` | Competitive analysis |
| `docs/sprint-plan.md` | Sprint-level task breakdown |
| `TASK.md` | Current sprint task spec — **READ THIS FOR YOUR ASSIGNMENT** |

## Design Principles

### 1. Product Analysis Before Everything

Phase 0 is the difference between:
- ❌ "AI randomly browses a website" → generic slop
- ✅ "freelancer tries to send an invoice using InvoiceNinja" → actionable feedback

Without understanding the product, personas are random and feedback is generic.

### 2. Grounded Feedback (NOT Generic Slop)

Three-phase pipeline:
1. Browser session produces action_log (ground truth)
2. LLM reviews action_log and writes feedback referencing specific steps
3. Structured extraction with evidence pointers

**Every issue MUST cite a specific action log step.** "The UI could be improved" = failure.

### 3. Behavioral Rules > Personality Descriptions

```
BAD:  "You are impatient and easily frustrated"
GOOD: "After 2 failed clicks, express frustration. After 5 total failures, abandon."
```

LLMs can't "act dumb." Use executable IF-THEN constraints.

### 4. Personas Derived From Product

Don't generate random combinations from a dimension matrix. Analyze the product → identify archetype users → generate personas that test specific aspects:
- The First-Timer (tests onboarding)
- The Switcher (tests vs competitor expectations)
- The Power User (tests efficiency)
- The Skeptic (tests trust signals)
- The Edge Case Finder (tests robustness)

### 5. Task Completion Is The North Star

A single number — "3/10 personas completed the core task" — is more compelling than 50 issue descriptions. Every persona gets a specific, measurable task with clear success criteria.

### 6. Product Score Creates Retention

Score (X/10) with 6 dimensions → founders obsess over improving it → re-run after fixes → see improvement → share on Twitter. This is the viral loop.

### 7. Canary Self-Validation

After generating feedback, check: "Is this specific and grounded, or generic slop?" Flag and regenerate if needed.

## Constraints

| Constraint | Value | Reason |
|-----------|-------|--------|
| Max actions per persona | 20 | Circuit breaker |
| Max time per persona | 3 minutes | Real users form opinions fast |
| Observation mode | Accessibility snapshots | 3-5x cheaper, deterministic |
| Max retries (page load) | 2 | Don't waste time on broken pages |
| Max consecutive failures | 5 | Force stop unstable sessions |
| Default persona count | 10 | Sweet spot: diverse + fast + cheap |
| Default model | Sonnet 4.6 | Best cost/quality ratio |

## Development Rules

### What You're Building
You are writing/improving **SKILL.md** — a prompt that an LLM follows to orchestrate browser tools. You are NOT writing application code (Python, JS, etc.).

### Branching
- Work on feature branches: `feat/module-name`
- **⚠️ CREATE PR AND STOP. DO NOT MERGE.** 晚晚 merges after review.

### Quality
- JSON output schemas defined inline in SKILL.md
- Every phase has clear error handling instructions
- Instructions must be unambiguous — walk through them mentally as an LLM

### Commit Messages
- Conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`
- Reference phase: `feat(phase-0): product analysis prompt`

## Role Clarification

You are the **implementer**. 晚晚 (the orchestrator) makes architectural decisions, reviews PRs, and merges. You implement per TASK.md spec, create PRs, and stop.

**Read TASK.md for your current assignment.** If TASK.md doesn't exist or is empty, ask 晚晚 for instructions.
