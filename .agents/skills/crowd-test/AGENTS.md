# AGENTS.md — CrowdTest Project Guide (for Codex CLI)

## Project Overview

**CrowdTest** — open-source AI tool: generate diverse synthetic user personas → browse web products → produce structured UX feedback.

- **Repo**: github.com/kaikezhang/crowd-test
- **Packaging**: Agent Skills 标准 (SKILL.md) — 兼容 Claude Code / Cursor / Gemini CLI / OpenClaw 等 15+ 工具
- **安装**: `npx skills add kaikezhang/crowd-test`
- **One command**: `/crowd-test https://myapp.com` → N personas → grounded feedback → prioritized report

## Architecture

```
Scout → Persona Generator → Browser Sessions → Feedback Writer → Aggregator → Report
```

Full details: `docs/technical-design.md`

## Tech Stack

- Browser: Playwright MCP (`@playwright/mcp`) — accessibility snapshots only
- LLM: Claude Sonnet 4.6 (default)
- Execution: Serial (v1)
- Output: Markdown → HTML (v2)
- Format: OpenClaw SKILL.md

## Key Design Rules

1. **Three-phase feedback** — action log → grounded reflection → structured extraction. Every issue MUST reference an action log step.
2. **Behavioral rules > personality** — "After 3 failures, abandon" not "you're impatient"
3. **Scout first** — verify page loads, detect auth/CAPTCHA, dismiss banners BEFORE any persona
4. **Cumulative diversity** — each persona gets "already covered issues" list, must find NEW problems
5. **Canary check** — LLM self-validates: is feedback specific or generic?

## Constraints

- Max 20 actions per persona, max 3 min per session
- Accessibility snapshots (not screenshots) — cheaper, deterministic
- Default 10 personas on Sonnet 4.6
- Serial execution in v1

## Data Structures

See `docs/technical-design.md` section 2:
- `SiteMap`, `Persona`, `ActionLog`, `PersonaFeedback`, `AggregatedReport`

## Development Rules

- Feature branches: `feat/module-name`
- **⚠️ CREATE PR AND STOP. DO NOT MERGE.** 晚晚 merges after review.
- Conventional commits: `feat(scout):`, `fix(persona):`, etc.
- Type annotations on all structures
- JSON schema validation on LLM outputs with retry
- Handle failures gracefully (retries, circuit breakers)

## Current Task

**Read TASK.md for your current assignment.** Implement per spec, create PR, stop.

## Role

You are the **implementer**. 晚晚 makes architecture decisions and merges. You code per TASK.md spec.
