# 🧪 CrowdTest

**It's 11pm. You just shipped a new feature. You need fresh eyes — but everyone's asleep.**

CrowdTest generates AI personas, gives each one a real browser, and lets them use your product the way actual humans would. Click by click. Page by page. Then it tells you what's broken, what's confusing, and what one thing you should fix first.

```
/crowd-test https://your-app.com --personas 5
```

5 synthetic users. Real browsers. Structured feedback. One report.

---

## How It Actually Works

Most "AI testing" tools are one LLM pretending to be 10 people in a single conversation. CrowdTest is different.

```
    You                    CrowdTest                     Your Product
     │                        │                              │
     │  "Test my app"         │                              │
     ├───────────────────────►│                              │
     │                        │  Scout: is the site up?      │
     │                        ├─────────────────────────────►│
     │                        │  Analyze: what IS this?      │
     │                        │  Generate 5 personas         │
     │                        │                              │
     │                        │  Persona 1: open browser     │
     │                        │  ├─ click, type, scroll      │
     │                        │  ├─ get confused at step 4   │
     │                        │  └─ write feedback           │
     │                        │                              │
     │                        │  Persona 2: open browser     │
     │                        │  ├─ different device          │
     │                        │  ├─ different goal            │
     │                        │  └─ find different bugs       │
     │                        │                              │
     │                        │  ... (repeat for each)       │
     │                        │                              │
     │  Report: 7.2/10        │                              │
     │  Fix: signup form      │                              │
     │◄────────────────────────                              │
```

Each persona is a **separate agent session** with its own browser, its own worktree, its own viewport size. They don't share context. They don't know about each other. They interact with your product the way a human would — clicking buttons, typing in fields, scrolling, dragging.

If a button can't be clicked through the UI, that's a finding. Not a workaround.

---

## What You Get Back

### Per persona
- **Action log** — every click, every confusion, every "where the hell is the button"
- **Journey narrative** — first-person story of their experience
- **Structured feedback** — issues ranked by severity, scores across 8 dimensions, verdict

### Aggregated report
```
Score: 7.2/10 — Decent with notable issues

🎯 If You Fix One Thing:
   "The signup form rejects valid .co email TLDs"
   — 4/5 personas hit this. Blocking.

🔴 Critical (1)  🟡 High (2)  🟠 Medium (3)  🟢 Low (1)
```

Every issue is **grounded in action logs** — not vibes, not generic advice.

```
❌  "The UI could be improved"
✅  "At step 7, I clicked '+' expecting 'New Invoice' but got Settings.
     Tried 3 more buttons. Gave up after 14 actions."
```

---

## Personas Are Not Random

CrowdTest derives personas from your product. An invoicing tool gets freelancers and accountants, not teenagers.

Each persona has:
- **A real device** — iPhone SE (375×667), Pixel 7 (412×915), laptop (1366×768), 4K monitor...
- **Behavioral rules** — not "you are impatient" (LLMs can't act dumb), but "skip text longer than 2 lines" and "if stuck for 3 clicks, try search"
- **A specific task** — "create an invoice for $500 to Acme Corp", not "explore the app"
- **Emotional tracking** — confident → confused → frustrated → abandoned

Five archetypes, mixed across runs:

| Type | What it tests |
|------|--------------|
| **First-Timer** | Can a new user figure this out? |
| **Switcher** | How does it compare to [competitor]? |
| **Power User** | Can I do this fast? |
| **Skeptic** | Should I trust this enough to sign up? |
| **Edge Case** | What breaks on a small phone? With weird input? |

---

## Focus Mode

Don't want a full product audit? Just test one thing.

```
/crowd-test https://your-app.com --focus "export flow"
```

CrowdTest will speed-run through setup to reach the export feature, then spend all its attention there. The pages it passes through on the way are not evaluated — they're just prerequisites.

---

## Installation

CrowdTest is an [Agent Skill](https://agentskills.io) — a prompt architecture, not a binary.

```bash
# For Claude Code, Cursor, Gemini CLI, Copilot, etc.
npx skills add kaikezhang/crowd-test

# For OpenClaw
clawhub install crowd-test
```

### Requirements
- An AI agent with browser tools (Playwright MCP or equivalent)
- Sub-agent dispatch capability (OpenClaw `sessions_spawn`, or `codex`/`claude` CLI)

### Headless / no-GPU environments
If your server has no GPU, add these Chrome flags for WebGL support:
```
--enable-unsafe-swiftshader
--use-angle=swiftshader
--ignore-gpu-blocklist
```

---

## Limitations (Honest)

| What CrowdTest can't do | Why |
|--------------------------|-----|
| Replace real user testing | LLMs are rational; real users aren't |
| Judge visual design quality | It reads DOM structure, not aesthetics |
| Bypass CAPTCHAs | No workaround exists |
| Test native mobile apps | Browser-only |
| Guarantee SPA reliability | ~60-70% success on complex SPAs |

CrowdTest produces **AI-generated hypotheses about UX issues**, not user research. Think of it as: 5 junior PMs spending 2 minutes each on your site. Fast, cheap, surprisingly useful.

---

## Cost

| Personas | Estimated cost |
|----------|---------------|
| 5 | $1–5 |
| 10 | $2–10 |
| 20 | $4–20 |

Range depends on sub-agent model (Codex is cheaper, Claude Code costs more).
For reference: a single UserTesting.com session costs $49.

---

## Project Structure

```
SKILL.md              ← The skill itself (the whole product)
references/
  ui-review-prompt.md ← Strict UI evaluation reference
docs/                 ← Design docs and research
README.md             ← You are here
```

CrowdTest is a **skill + prompt architecture** project. The product is `SKILL.md`.

---

## License

MIT

---

*Built by [Kaike](https://github.com/kaikezhang). Ship fast, test faster.* 🧪
