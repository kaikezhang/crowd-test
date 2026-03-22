# 🧪 CrowdTest

> AI-powered synthetic user testing. One command. Zero users needed.

```bash
crowdtest https://myapp.com
```

10 AI personas browse your product → find UX issues you can't see → deliver a prioritized report.

---

## Why CrowdTest?

You just rebuilt your onboarding flow. It's 11pm. You need fresh eyes but have nobody to ask.

| Alternative | Cost | Speed | CrowdTest |
|------------|------|-------|-----------|
| 5 friends | Free (+ social debt) | Days | **Instant, diverse, private** |
| UserTesting.com | $245 (5 users) | Hours | **~$1, minutes** |
| Hotjar/FullStory | Free tier | Needs traffic | **Works with zero users** |
| UX consultant | $150-300/hr | Days to book | **Instant sanity check** |

**CrowdTest is not a replacement for real user testing.** It's the only option when you have no users, no budget, and no time. Think of it as a fast, cheap UX sanity check — 60-70% of real user feedback quality at 1% of the cost.

---

## How It Works

```
┌──────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────┐     ┌────────┐
│  Scout   │ ──► │  Persona     │ ──► │  Browser     │ ──► │ Feedback │ ──► │ Report │
│  (check  │     │  Generator   │     │  Sessions    │     │ Aggreg.  │     │        │
│   page)  │     │  (N users)   │     │  (per user)  │     │ (dedup)  │     │        │
└──────────┘     └──────────────┘     └─────────────┘     └──────────┘     └────────┘
```

1. **Scout** checks your page loads correctly, detects auth walls and CAPTCHAs
2. **Persona Generator** creates N diverse users (tech novice → developer, ages 18-65, different goals)
3. **Browser Sessions** — each persona browses your product using Playwright, logging every action
4. **Feedback** — each persona reviews their action log and writes grounded, specific feedback
5. **Report** — issues deduplicated across personas, ranked by severity and consensus

### What makes the feedback useful?

Every piece of feedback is **grounded in action logs** — not opinions pulled from thin air:

```
❌ "The UI could be improved" (generic slop)
✅ "When I clicked 'Add to Cart', the button didn't change to confirm
    the action. I clicked 3 times thinking it was broken. Cart now
    shows 3 items." (grounded in action log step 7-8)
```

---

## Example Output

```
╔══════════════════════════════════════════════════╗
║  CrowdTest Report — myapp.com                    ║
║  10 personas · 89 pages visited · 18 issues      ║
║  Avg NPS: 5.8  ·  Would pay: 35%                ║
╠══════════════════════════════════════════════════╣
║                                                  ║
║  🔴 CRITICAL (2)                                  ║
║  ┌─────────────────────────────────────────────┐ ║
║  │ Signup form rejects valid .co email TLDs    │ ║
║  │ 7/10 personas affected · Evidence: steps    │ ║
║  │ 12-14 across 7 sessions                     │ ║
║  └─────────────────────────────────────────────┘ ║
║  ┌─────────────────────────────────────────────┐ ║
║  │ "Start Free" button leads to payment page   │ ║
║  │ 6/10 personas felt misled                   │ ║
║  └─────────────────────────────────────────────┘ ║
║                                                  ║
║  🟡 HIGH (4)  ·  🟠 MEDIUM (6)  ·  🟢 LOW (6)    ║
║                                                  ║
║  📊 Segment Insights                              ║
║  • Novice users: couldn't find pricing page     ║
║  • Mobile users: CTA hidden below fold          ║
║  • Power users: wanted keyboard shortcuts       ║
╚══════════════════════════════════════════════════╝
```

---

## Installation

CrowdTest follows the [Agent Skills](https://agentskills.io) open standard — works with **15+ AI tools** out of the box.

### One-line install (any supported agent)

```bash
npx skills add kaikezhang/crowd-test
```

Works with: Claude Code, Cursor, Gemini CLI, VS Code Copilot, OpenCode, Goose, Amp, Junie, and more.

### OpenClaw Skill

```bash
clawhub install crowd-test
```

### Requirements

- Any AI coding agent with [Playwright MCP](https://playwright.dev/docs/test-agents) support
- Anthropic API key (Claude Sonnet 4.6+)

---

## Usage

### Basic (zero config)

```bash
crowdtest https://myapp.com
```

### With options

```bash
crowdtest https://myapp.com \
  --personas 20 \
  --model opus \
  --audience "e-commerce shoppers" \
  --output report.md
```

### Testing auth-protected sites

```bash
# Export cookies from your browser
crowdtest https://app.mysite.com --storage-state cookies.json
```

---

## Persona System

CrowdTest doesn't generate random users. It builds a **dimension matrix** to ensure coverage:

| Dimension | Values |
|-----------|--------|
| Tech level | Novice → Developer |
| Purpose | Inferred from your product |
| Age | 18-65+ |
| Patience | Low → High |
| Exploration | Focused → Curious |
| Device | Desktop / Mobile / Tablet |

Each persona gets **behavioral rules** — not personality adjectives:

```
✅ "After 3 failed attempts, express frustration and try search"
✅ "Skip text sections longer than 3 lines"
✅ "Only use visible buttons and links (no keyboard shortcuts)"

❌ "You are impatient" (LLMs can't act dumb on command)
```

---

## Limitations (Honest)

| What CrowdTest Can't Do | Why |
|-------------------------|-----|
| Replace real user testing | LLMs are rational actors, real users aren't |
| Evaluate visual design | Sees DOM structure, not colors/layout/whitespace |
| Test behind CAPTCHAs | No bypass capability |
| Simulate accessibility | Can't run a screen reader |
| Test performance/speed | Reports slow loads but can't measure Core Web Vitals |
| Handle complex SPAs reliably | 50-70% success on SPAs vs 90%+ on static sites |

**Position it right**: CrowdTest produces "AI-generated hypotheses about UX issues," not "synthetic user research."

---

## Cost

| Personas | Model | Cost | Time |
|----------|-------|------|------|
| 5 | Sonnet | ~$0.30 | ~5 min |
| 10 | Sonnet | ~$0.60 | ~10 min |
| 20 | Sonnet | ~$1.20 | ~20 min |
| 10 | Opus | ~$3.00 | ~10 min |

For reference: UserTesting.com charges $49 per real human tester.

---

## Architecture

See `docs/` for detailed documentation:

- [`docs/PRD.md`](docs/PRD.md) — Product requirements
- [`docs/technical-design.md`](docs/technical-design.md) — Architecture & data structures
- [`docs/competitive-research.md`](docs/competitive-research.md) — Competitive analysis
- [`docs/ceo-review.md`](docs/ceo-review.md) — Product strategy review
- [`docs/eng-review.md`](docs/eng-review.md) — Engineering feasibility
- [`docs/roadmap.md`](docs/roadmap.md) — Version milestones
- [`docs/sprint-plan.md`](docs/sprint-plan.md) — Sprint breakdown

---

## Roadmap

- [x] v0.1 — Single persona proof of concept
- [ ] v0.2 — Scout + Persona Generator + Single-persona loop
- [ ] v0.3 — Multi-persona + Aggregation + Report
- [ ] v0.4 — Skill packaging + Self-validation
- [ ] v1.0 — Public release

---

## Contributing

CrowdTest is open source. PRs welcome.

```bash
git clone https://github.com/kaikezhang/crowd-test.git
cd crowd-test
# Read CLAUDE.md (for Claude Code) or AGENTS.md (for Codex)
# Check TASK.md for current sprint
```

---

## License

MIT

---

*Built by [Kaike](https://github.com/kaikezhang) & [晚晚](https://github.com/kaikezhang/crowd-test). Ship fast, test faster.* 🧪
