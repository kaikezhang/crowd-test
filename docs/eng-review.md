# CrowdTest — Engineering Review

> VP Engineering / Tech Lead Review
> Date: 2026-03-22
> Status: Pre-implementation technical feasibility assessment
> Verdict: **Conditionally viable** — the idea is sound, but execution has hard constraints that must be designed around, not ignored.

---

## 1. Browser Automation Reliability

**The honest assessment: 70-80% reliable for happy paths, drops to 40-50% for complex SPAs.**

### What works

Playwright MCP with accessibility snapshots is the right primitive. The LLM sees structured DOM semantics, not pixels:

```
- textbox "Email address"
- button "Sign in"
- heading "Dashboard" [level=1]
```

This is deterministic, token-efficient (~200-500 tokens per snapshot vs 1500+ for a screenshot description), and robust against CSS changes. For static marketing sites, forms, and simple CRUD apps, reliability is high.

### What breaks

| Failure Mode | Frequency | Impact |
|---|---|---|
| LLM clicks wrong element (ambiguous labels) | ~15% of complex pages | Session derails, persona gives feedback on wrong screen |
| Dynamic content not loaded (SPA hydration race) | ~10-20% | Snapshot shows skeleton/loader, LLM acts on stale state |
| Infinite scroll / lazy load | ~30% of modern sites | LLM doesn't know to scroll; sees partial content |
| Modals / overlays / cookie banners | ~60% of sites | Blocks interaction until dismissed; LLM may not recognize the blocker |
| Canvas / WebGL / complex visualizations | ~100% failure | Accessibility snapshot is empty; LLM is blind |
| Client-side routing (React Router, etc.) | ~10% | `browser_navigate` may not trigger expected view |

### Mitigation strategy

```
Before persona interaction:
  1. Navigate to URL
  2. Wait for network idle + DOM stable (browser_wait_for)
  3. Dismiss cookie banners (heuristic: look for "Accept" / "Close" buttons)
  4. Take baseline snapshot
  5. If snapshot has <5 interactive elements → flag as "page may not have loaded"
  6. Retry once with 3s delay

During interaction:
  7. After each action, wait 500ms + re-snapshot
  8. If snapshot unchanged after click → action may have failed, try alternative
  9. Cap at 20 actions per session (circuit breaker)
  10. Log every action + snapshot pair for debugging
```

**Recommendation**: Build a "page readiness" heuristic as the FIRST thing. Without it, 30%+ of test runs will produce garbage feedback about loading screens. This is a blocking prerequisite.

---

## 2. Persona Fidelity

**The honest assessment: LLMs can approximate persona PREFERENCES but not persona BEHAVIOR.**

### What the Stanford paper actually says

The 2024 Stanford study ("Generative Agent Simulations of 1,000 People") showed:
- 85% accuracy matching **survey responses** (attitudes, preferences, opinions)
- Tested on General Social Survey and Big Five personality
- Key caveat: these are **stated preferences**, not **observed behavior**

The 15% gap matters. The failures cluster in:
- **Extreme opinions**: LLMs regress to the mean. A persona described as "hates technology" will still navigate competently because the LLM is competent.
- **Emotional reactions**: Real frustrated users close the tab. LLM personas politely describe their frustration.
- **Cognitive limitations**: A "non-technical 55-year-old nurse" described to the LLM will still understand developer jargon because the LLM understands it. You can't make Claude dumb.
- **Temporal behavior**: Real users get bored, distracted, check their phone. LLM personas are relentlessly focused.

### The fundamental problem

```
What you want:              What you get:
───────────────             ──────────────
"Confused grandma who       "Articulate AI that says
can't find the button"      'a less tech-savvy user
                             might struggle to find
                             this button'"
```

The LLM will DESCRIBE what a persona would experience rather than EXPERIENCING it. This is the meta-cognition trap: the model is too smart to authentically simulate someone who is not smart.

### How to partially solve it

1. **Constrain the action space, not just the prompt**: Instead of telling the LLM "you are a novice", limit what tools it can use. A "novice" persona:
   - Cannot use browser search (Ctrl+F)
   - Must navigate only via visible links and buttons
   - Ignores elements with technical labels (API, webhook, etc.)
   - Gives up after 3 failed attempts at any task

2. **Behavioral rules > personality descriptions**:
   ```
   BAD:  "You are impatient and easily frustrated"
   GOOD: "After 2 clicks without finding what you want, express frustration
          and try a different approach. After 5 total failed attempts,
          abandon the task and write a negative review."
   ```

3. **Temperature and sampling**: Use temperature 0.9-1.0 for persona actions to increase behavioral variance. Use temperature 0.3 for feedback synthesis to keep it coherent.

4. **Accept the limitation**: Frame CrowdTest feedback as "AI-generated hypotheses about user experience" not "synthetic user research". The value is in coverage and speed, not in perfect human fidelity.

**Estimated fidelity by dimension**:

| Dimension | Fidelity | Notes |
|---|---|---|
| Feature preferences | High (80%+) | "Would a trader want price charts?" — yes, LLMs get this right |
| Navigation patterns | Medium (50-60%) | Exploration paths are LLM-idiosyncratic, not human-like |
| Frustration thresholds | Low (30-40%) | Must be hard-coded via behavioral rules |
| Accessibility issues | Low (20-30%) | LLM can't simulate screen reader experience from accessibility snapshot |
| First impressions | Medium (60%) | LLMs are too thorough; real users form snap judgments |

---

## 3. Feedback Quality

**The core question: Is this useful or is it generic slop?**

### The slop spectrum

```
USELESS (generic)                                         USEFUL (specific)
◄─────────────────────────────────────────────────────────────────────────►
"The UI is clean       "The sign-up form    "When I click 'Add to Cart'
and intuitive"         could be improved"    on the /products page, the
                                             button text doesn't change
                                             to confirm the action was
                                             registered. I clicked it
                                             3 times thinking it was
                                             broken. The cart shows
                                             3 items now."
```

### Why LLMs default to slop

1. **RLHF politeness bias**: Models are trained to be helpful and positive. "The UI is clean" is the default mode.
2. **No real experience**: The LLM hasn't actually felt confusion. It's pattern-matching on what "confused user feedback" looks like.
3. **Lack of grounding**: Without a specific UI element or interaction to reference, feedback is necessarily abstract.

### How to force specificity

**The key insight: ground feedback in ACTIONS, not opinions.**

```
Phase 1: Persona browses the product (actions logged)
  → Action log: clicked "Pricing" → saw pricing table → scrolled down →
    clicked "Start Free Trial" → saw signup form → typed email →
    clicked "Submit" → got error "Invalid email" → typed again → success

Phase 2: Persona reviews their own action log and writes feedback
  → "The email validation rejected 'user@company.co' — I had to try
     my gmail. Also, the pricing page doesn't explain what 'Pro' gets
     me that 'Free' doesn't. I almost left."

Phase 3: Structured extraction
  → {
       "page": "/pricing",
       "issue": "Email validation rejects valid TLDs",
       "severity": "high",
       "evidence": "Action log step 7-8: two attempts needed",
       "persona_impact": "Almost abandoned signup"
     }
```

This three-phase approach is critical. Phase 1 creates ground truth. Phase 2 reflects on it. Phase 3 structures it. Skip Phase 1 and you get slop.

### Diversity problem

Without intervention, 10 personas will give you 10 variations of the same 3 observations. The LLM has systematic biases about what "matters" in UX.

**Mitigation**:
- After each persona's feedback, feed it as "already covered" to subsequent personas
- Prompt: "The following issues have already been reported by other testers. Find DIFFERENT issues: [list]"
- This is the same accumulative diversity pattern from Channel-Labs, applied to feedback instead of personas
- Still expect diminishing returns after ~15-20 personas per product

### Estimated feedback utility

| Feedback Type | Usefulness | Why |
|---|---|---|
| Navigation confusion | High | Grounded in actual click paths |
| Missing features | High | LLMs are good at "what's missing" |
| Copy/labeling issues | High | LLMs are excellent at language clarity |
| Visual design | Low | LLMs don't see design; they see DOM structure |
| Performance | Medium | Can report slow loads but can't feel "sluggish" |
| Emotional response | Low | See persona fidelity section |
| Accessibility | Medium | Can flag missing alt text, but can't simulate screen readers |

---

## 4. Architecture Proposal

### 4.1 End-to-End Data Flow

```
USER INPUT                    CROWDTEST PIPELINE                           OUTPUT
─────────                     ──────────────────                           ──────

                         ┌─────────────────────────┐
  Product URL ──────────►│  1. SCOUT               │
  Product description    │     Navigate URL         │
  Target audience hint   │     Take snapshot        │
                         │     Identify: pages,     │
                         │       features, flows    │
                         │     Output: site_map{}   │
                         └──────────┬──────────────┘
                                    │
                                    ▼
                         ┌─────────────────────────┐
                         │  2. PERSONA GENERATOR    │
                         │     Input: site_map +    │
                         │       audience hint      │
                         │     Dimension matrix     │
                         │     Diversity check      │
                         │     Output: personas[]   │
                         └──────────┬──────────────┘
                                    │
                         ┌──────────┴──────────────┐
                         │  For each persona (parallel where possible):
                         │                         │
                         │  ┌─────────────────────┐│
                         │  │ 3. BROWSER SESSION   ││
                         │  │    Init context      ││
                         │  │    (viewport, device) ││
                         │  │    Navigate product   ││
                         │  │    Explore as persona ││         ┌──────────────┐
                         │  │    Log all actions    ││────────►│  action_log  │
                         │  │    Cap: 20 actions    ││         │  + snapshots │
                         │  │    Cap: 5 min         ││         └──────┬───────┘
                         │  └─────────────────────┘│                 │
                         │                         │                 ▼
                         │  ┌─────────────────────┐│         ┌──────────────┐
                         │  │ 4. FEEDBACK WRITER   ││────────►│  feedback{}  │
                         │  │    Review action log ││         │  per persona │
                         │  │    Write grounded    ││         └──────┬───────┘
                         │  │    feedback          ││                │
                         │  └─────────────────────┘│                │
                         └─────────────────────────┘                │
                                                                    ▼
                                                         ┌─────────────────────┐
                                                         │ 5. AGGREGATOR       │
                                                         │    Dedup issues     │
                                                         │    Score severity   │
                                                         │    Cluster by theme │
                                                         │    Cross-persona    │──► Report
                                                         │    consensus check  │   (Markdown
                                                         │    Generate report  │    or HTML)
                                                         └─────────────────────┘
```

### 4.2 State Machine (Per-Persona Session)

```
                    ┌──────────┐
                    │  INIT    │
                    │  Set up  │
                    │  context │
                    └────┬─────┘
                         │
                         ▼
                    ┌──────────┐     page not loaded
                    │ NAVIGATE │────────────────────► RETRY (max 2)
                    │ to URL   │                          │
                    └────┬─────┘◄─────────────────────────┘
                         │ success
                         ▼
                    ┌──────────┐
                    │ ORIENT   │  Take snapshot, identify
                    │          │  interactive elements,
                    │          │  plan next action
                    └────┬─────┘
                         │
                         ▼
               ┌─────────────────┐
               │    ACT          │◄──────────────┐
               │  Click/type/    │               │
               │  scroll/nav     │               │
               └────┬───────┬────┘               │
                    │       │                    │
              success│    failure               │
                    │       │                    │
                    ▼       ▼                    │
              ┌──────┐  ┌──────────┐            │
              │OBSERVE│  │RECOVER   │            │
              │ New   │  │ Try alt  │            │
              │ state │  │ approach │            │
              └──┬───┘  └────┬─────┘            │
                 │           │                   │
                 └─────┬─────┘                   │
                       │                         │
                       ▼                         │
                 ┌───────────┐    actions < 20   │
                 │  DECIDE   │───────────────────┘
                 │  Continue │
                 │  or stop? │
                 └─────┬─────┘
                       │ done (goal met / stuck / limit hit)
                       ▼
                 ┌───────────┐
                 │  REFLECT  │  Review action log,
                 │           │  write feedback
                 └─────┬─────┘
                       │
                       ▼
                 ┌───────────┐
                 │   DONE    │
                 └───────────┘
```

### 4.3 Core Data Structures

```
Persona {
  id: string
  name: string
  tech_level: "novice" | "intermediate" | "advanced" | "developer"
  purpose: string              // why they'd use this product
  age: number
  industry: string
  personality: {
    patience: 0.0-1.0
    exploration_tendency: 0.0-1.0
    attention_to_detail: 0.0-1.0
  }
  behavioral_rules: string[]   // hard constraints on actions
  narrative: string            // full prompt paragraph
  device: "desktop" | "mobile" | "tablet"
}

ActionLog {
  persona_id: string
  actions: [{
    step: number
    type: "navigate" | "click" | "type" | "scroll" | "wait"
    target: string             // element description
    snapshot_before: string    // accessibility snapshot
    snapshot_after: string
    success: boolean
    notes: string              // LLM's in-character observation
  }]
  duration_seconds: number
  pages_visited: string[]
}

PersonaFeedback {
  persona_id: string
  first_impression: string
  journey_narrative: string
  issues: [{
    page: string
    element: string
    issue: string
    severity: "critical" | "high" | "medium" | "low"
    evidence: string           // reference to action log step
  }]
  positives: [{ feature: string, why: string }]
  nps_score: 1-10
  would_pay: boolean
  would_return: boolean
  one_line_verdict: string
}

AggregatedReport {
  product_url: string
  personas_count: number
  avg_nps: number
  pay_rate: number
  unique_issues: [{
    id: string
    description: string
    severity: string
    personas_affected: number  // out of N
    evidence: string[]
  }]
  consensus_positives: [{ feature: string, mention_count: number }]
  segment_insights: [{        // grouped by persona dimension
    segment: string
    key_finding: string
  }]
}
```

---

## 5. LLM Cost Analysis

### Per-action token budget

| Operation | Input tokens | Output tokens | Model | Cost per call |
|---|---|---|---|---|
| Accessibility snapshot (average page) | ~800 | — | — | — |
| Persona system prompt | ~500 | — | — | — |
| Action decision (snapshot + history) | ~2,000 | ~100 | Sonnet | $0.006 |
| Feedback writing (full action log) | ~3,000 | ~800 | Sonnet | $0.015 |
| Persona generation (1 persona) | ~1,500 | ~300 | Sonnet | $0.006 |
| Issue dedup + aggregation | ~5,000 | ~1,500 | Sonnet | $0.025 |

Using Claude Sonnet 4.6 pricing: $3/1M input, $15/1M output.

### Per-persona cost

```
1 persona session:
  - Persona generation:      1 call  × $0.006  = $0.006
  - Scout/orient:            2 calls × $0.006  = $0.012
  - Actions (avg 12):       12 calls × $0.006  = $0.072
  - Feedback writing:        1 call  × $0.015  = $0.015
                                        ──────────────────
  Total per persona:                    ~$0.105
```

### Per-run cost (N personas)

| Personas | LLM Cost | Aggregation | Total LLM | Browser time* |
|---|---|---|---|---|
| 5 | $0.53 | $0.025 | **$0.55** | ~5 min |
| 10 | $1.05 | $0.035 | **$1.09** | ~10 min |
| 20 | $2.10 | $0.050 | **$2.15** | ~20 min |
| 50 | $5.25 | $0.080 | **$5.33** | ~50 min (serial) |
| 100 | $10.50 | $0.120 | **$10.62** | ~100 min (serial) |

*Browser time assumes serial execution, ~1 min per persona session.

### Using Opus vs Sonnet

If using Claude Opus 4.6 ($15/1M input, $75/1M output) for higher quality:
- Per persona: ~$0.53 (5x Sonnet)
- 20 personas: ~$10.85
- 100 personas: ~$53.50

### Cost verdict

**$1-5 per run with 10-20 personas on Sonnet is very reasonable.** This is cheaper than a single hour of human user testing. Even 100 personas on Opus at ~$50 is competitive vs. recruiting 100 real testers.

The cost bottleneck is NOT LLM tokens — it's browser execution time (see Scalability section).

---

## 6. Scalability

### Bottleneck analysis

```
Bottleneck hierarchy (worst → least bad):

1. BROWSER INSTANCES           ← Primary bottleneck
   Each persona needs a browser.
   Each Chromium instance: ~150-300MB RAM.
   10 parallel browsers: ~2-3GB RAM.
   50 parallel browsers: ~10-15GB RAM. Starts swapping on <16GB machines.
   100 parallel: Not feasible on a single machine.

2. LLM RATE LIMITS             ← Secondary bottleneck
   Anthropic rate limits (standard tier):
     - Sonnet: ~4,000 RPM, ~400K input TPM
     - With 10 parallel personas × ~1 req/5sec = 120 RPM → fine
     - With 50 parallel personas × ~1 req/5sec = 600 RPM → fine
     - With 100 parallel → 1200 RPM → getting close

3. NETWORK                     ← Tertiary
   Each browser makes real HTTP requests to the target.
   100 parallel browsers hitting the same site = accidental DDoS.
   Could trigger rate limiting or WAF blocks on the TARGET site.

4. CONTEXT WINDOW              ← Manageable
   Accumulative diversity prompt grows with persona count.
   At 50+ personas, need to switch to embedding-based dedup
   instead of dumping all prior personas into prompt.
```

### Practical limits within Claude Code skill

A Claude Code skill runs in the user's terminal. Realistically:

| Scenario | Personas | Parallelism | Time | RAM |
|---|---|---|---|---|
| Quick test | 5 | Serial | ~5 min | ~500MB |
| Standard run | 10-15 | 2-3 parallel | ~7-10 min | ~1-2GB |
| Deep test | 20-30 | 3-5 parallel | ~10-15 min | ~2-3GB |
| Max practical | 50 | 5 parallel | ~15-20 min | ~3GB |

**100 parallel personas is not realistic within a skill context.** It would need a cloud orchestration layer (not in v1 scope).

### Recommendation

Target 10-20 personas for v1. This is the sweet spot:
- Enough diversity to surface real issues
- Completes in <15 minutes
- Costs <$3
- Fits in a single machine's resources

---

## 7. The "Garbage In" Problem

**This is the #1 risk to user satisfaction.** If the URL doesn't work, the whole run is wasted, and the user blames CrowdTest.

### Failure modes by site type

| Site Type | Likelihood of Success | Issues |
|---|---|---|
| Static marketing site | 90%+ | Works well. Minimal JS, clear structure. |
| Server-rendered app (Rails, Django) | 80%+ | Usually works. May need auth. |
| SPA (React, Vue, Angular) | 50-70% | Hydration delays, client routing, dynamic content. |
| Behind auth wall | 0% without setup | Need cookies/tokens. User MUST provide auth state. |
| Localhost / dev server | 70%+ | Works if server is running. Port conflicts possible. |
| Mobile-only PWA | 40-60% | May have mobile-only features, responsive breakpoints. |
| Sites with CAPTCHA | 0% | Game over. No workaround. |
| Sites behind VPN/firewall | 0% | Browser can't reach it. |

### Is "just give me a URL" realistic?

**For public sites without auth: mostly yes.**
**For anything else: no. Setup is required.**

### Required pre-flight checks (the Scout phase)

```
Scout phase (before any persona runs):
  1. Navigate to URL
  2. Check HTTP status (200? redirect? error?)
  3. Wait for page load (network idle)
  4. Take accessibility snapshot
  5. Check: >5 interactive elements? (if not → page didn't load properly)
  6. Check: login/auth wall detected? (look for "sign in", "log in" patterns)
  7. Check: CAPTCHA detected?
  8. Check: cookie consent banner? (dismiss it)
  9. Report findings to user BEFORE running personas

If auth detected:
  → "This site requires login. Please provide:
      a) A storage-state JSON file with cookies, OR
      b) Test credentials I can use to log in"

If CAPTCHA detected:
  → "This site has CAPTCHA protection. CrowdTest cannot bypass CAPTCHAs.
      Options: test on staging without CAPTCHA, or provide pre-auth cookies."

If page didn't load:
  → "The page appears to not have loaded correctly. Is this a SPA?
      Is the dev server running? Here's what I see: [snapshot]"
```

**This Scout phase is table stakes. Ship it in v0.1.** Without it, every failed run feels like CrowdTest is broken.

### Auth handling

Playwright's `--storage-state` flag accepts a JSON file with cookies and localStorage. This is the cleanest approach:

```json
{
  "cookies": [
    { "name": "session", "value": "abc123", "domain": ".example.com", "path": "/" }
  ],
  "origins": [
    { "origin": "https://example.com", "localStorage": [
      { "name": "token", "value": "xyz789" }
    ]}
  ]
}
```

The user exports this once. CrowdTest uses it for all personas.

For the gstack `/browse` path within Claude Code, the `/setup-browser-cookies` skill can import cookies from the user's real browser — this is the most ergonomic approach for the skill context.

---

## 8. Testing the Tester

**The meta-problem: how do you know CrowdTest's feedback is good?**

### Validation strategy

#### Tier 1 — Known-issue validation (do this first)

Take a product with KNOWN usability issues. Run CrowdTest. Check if it finds them.

```
Test case: TodoMVC (intentionally broken version)
  Known issues planted:
    1. "Add" button is invisible (white on white)         → Severity: critical
    2. Delete icon only appears on hover (not obvious)    → Severity: medium
    3. "All / Active / Completed" filters have no labels  → Severity: low
    4. No confirmation on "Clear completed"               → Severity: medium

  Run CrowdTest with 10 personas.

  Grading:
    - Found 4/4 → Excellent
    - Found 3/4 → Good
    - Found 2/4 → Acceptable
    - Found 1/4 → Needs work
    - Found 0/4 → Broken
```

Build 3-5 such test fixtures with planted issues of varying severity. This becomes CrowdTest's own test suite.

#### Tier 2 — Comparison with real user feedback

If you have access to a product with existing user feedback (support tickets, NPS surveys, UserTesting.com sessions), run CrowdTest on the same product and compare:

```
Metrics:
  - Issue overlap: what % of real user issues did CrowdTest find?
  - False positive rate: what % of CrowdTest issues are non-issues?
  - Severity alignment: does CrowdTest rank issues similarly to real users?
```

Event Radar is the perfect first candidate since you have real user context.

#### Tier 3 — A/B with real humans (future)

Run CrowdTest + real user testing on the same product. Compare findings. This is the gold standard but expensive and slow.

### Continuous calibration

Every CrowdTest run should include at least one "canary" check:

```
Internal validation: After generating feedback, ask the LLM:
  "Review this feedback. Is it specific and grounded in the action log,
   or is it generic and could apply to any product?"

Flag feedback that scores as "generic" and either regenerate or discard.
```

---

## 9. Agent Skills Constraints

**This is the hardest architectural constraint.** A Claude Code skill (SKILL.md) is fundamentally just a prompt file. It cannot:

- Bundle dependencies (no `package.json`, no `requirements.txt`)
- Run a background server
- Install packages (no `pip install`, no `npm install`)
- Persist state between invocations
- Manage its own processes
- Have its own file structure

### What the skill actually does vs. what the LLM figures out

```
SKILL.md provides:
  ├── System prompt (persona generation instructions)
  ├── Structured output schemas (JSON formats)
  ├── Behavioral rules (how to browse, how to give feedback)
  ├── Phase orchestration (scout → generate → browse → feedback → aggregate)
  └── Report template

The LLM figures out:
  ├── How to use Playwright MCP tools (already in its tool set)
  ├── How to navigate specific sites (from accessibility snapshots)
  ├── What feedback to give (from persona + observations)
  ├── How to aggregate across personas (synthesis task)
  └── How to write the final report (from template)
```

### The serial execution problem

In Claude Code, the skill runs in a single conversation. The LLM processes one persona at a time. For 20 personas, that's 20 sequential browser sessions. This is slow but reliable.

**Parallelism options within Claude Code**:
1. **Agent tool**: Spawn sub-agents for each persona. Each gets its own browser. But: sub-agents can't share state easily, and spawning 20 agents may hit rate limits or overwhelm the system.
2. **Batch then aggregate**: Generate all personas first (1 LLM call), write them to a JSON file, then process sequentially. The sequential processing is the bottleneck, not generation.
3. **Accept serial**: For v1, just run 5-10 personas serially. A 10-minute skill run is acceptable.

### What MUST be in SKILL.md vs. what can be external

| Component | Location | Why |
|---|---|---|
| Orchestration prompt | SKILL.md | Core skill logic |
| Persona generation prompt | SKILL.md | Templated in the skill |
| Feedback schemas | SKILL.md | Structured output definitions |
| Browser interaction rules | SKILL.md | Behavioral constraints |
| Report template | SKILL.md or generated | LLM can generate markdown |
| Playwright MCP | Pre-installed | Required dependency — document in README |
| HTML report generator | Generated at runtime | LLM writes the HTML file |

### The dependency problem

CrowdTest assumes `@playwright/mcp` is available. In OpenClaw, this may be bundled. In Claude Code, the user must have it configured.

**Recommendation**: SKILL.md should start with a dependency check:

```
Step 0: Verify prerequisites
  - Check if Playwright MCP is available (try browser_navigate)
  - If not → print setup instructions and exit
  - Check if target URL is reachable
```

### Realistic skill scope for v1

The skill should be a well-structured prompt that orchestrates existing tools. It should NOT try to be a framework. Think of it as a 500-line prompt, not a 5000-line codebase.

---

## 10. Build vs. Buy Components

| Component | Build | Buy/Reuse | Recommendation |
|---|---|---|---|
| **Browser automation** | — | Playwright MCP | **Reuse**. This is solved. Don't reinvent it. |
| **Persona generation** | Custom dimension matrix + prompts | PersonaHub dataset for bootstrapping | **Build** the generator, optionally **seed** from PersonaHub |
| **Diversity enforcement** | Accumulative prompt (<20), embedding check (>20) | Channel-Labs pattern | **Build**, borrowing the pattern |
| **Feedback extraction** | Custom structured prompts | — | **Build**. This is the core IP. No existing tool does this. |
| **Issue deduplication** | LLM-based dedup | ArkSim pattern | **Borrow** the pattern, adapt for product issues |
| **Report generation** | LLM generates markdown/HTML | ArkSim HTML template | **Build** simple markdown for v1, **borrow** HTML pattern for v2 |
| **Evaluation metrics** | Custom (usability, clarity, NPS) | ArkSim metric framework | **Build** custom metrics, **borrow** the ABC pattern |
| **Cookie/auth handling** | — | Playwright `--storage-state`, gstack `/setup-browser-cookies` | **Reuse** |
| **Page readiness detection** | Custom heuristic | — | **Build**. Critical for reliability. |
| **LLM orchestration** | SKILL.md prompt | — | **Build**. This is the skill. |

### What NOT to build

- **Don't build a custom browser automation layer.** Playwright MCP exists and is maintained by Microsoft.
- **Don't build a persona database.** Generate on-the-fly per product. Persona reuse across products is a v2+ feature.
- **Don't build a web server or dashboard.** Output is a markdown/HTML file. Keep it simple.
- **Don't build an eval framework from scratch.** Adapt ArkSim's patterns.
- **Don't build multi-machine orchestration.** 10-20 personas on a single machine is enough for v1.

---

## 11. Technical Recommendations

### Priority order for implementation

```
P0 (v0.2 — blocks everything):
  ├── Scout phase (page readiness + auth detection)
  ├── Persona generator (dimension matrix → N personas)
  ├── Single-persona browser session (ORIENT → ACT → OBSERVE → REFLECT loop)
  └── Grounded feedback extraction (action log → structured feedback)

P1 (v0.3 — makes it useful):
  ├── Multi-persona sequential execution
  ├── Accumulative feedback diversity ("find different issues")
  ├── Issue deduplication + aggregation
  └── Markdown report generation

P2 (v0.4 — makes it shippable):
  ├── SKILL.md packaging
  ├── Dependency check + setup instructions
  ├── Error handling + graceful degradation
  └── 3-5 test fixtures for self-validation

P3 (v1.0+ — makes it great):
  ├── HTML interactive report
  ├── Parallel persona execution via sub-agents
  ├── Embedding-based persona diversity (>20 personas)
  ├── Return-user simulation (multi-session)
  └── A/B variant testing (compare two URLs)
```

### Key technical decisions

| Decision | Recommendation | Rationale |
|---|---|---|
| LLM model for persona actions | Sonnet 4.6 | Good enough quality, 5x cheaper than Opus, fast |
| LLM model for feedback synthesis | Sonnet 4.6 | Quality matters here but Sonnet is sufficient |
| LLM model for aggregation | Opus 4.6 | Synthesis across 20 personas benefits from stronger reasoning |
| Browser observation mode | Accessibility snapshots (not vision) | 3-5x cheaper, more deterministic, faster |
| Persona count default | 10 | Sweet spot: diverse enough, completes in <10 min, costs ~$1 |
| Max actions per persona | 20 | Circuit breaker. Prevents runaway sessions. |
| Max session time per persona | 3 minutes | Real users form opinions fast. |
| Feedback format | Structured JSON + narrative | JSON for aggregation, narrative for readability |
| Report format (v1) | Markdown | Simple, works everywhere, LLM generates it naturally |
| Execution model (v1) | Serial | Reliable, simple, sufficient for 10 personas |

### Risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Browser sessions fail on complex SPAs | High | High | Scout phase + retry logic + graceful skip |
| Feedback is generic slop | Medium | Critical | Action-grounded feedback + self-validation check |
| All personas give same feedback | Medium | High | Accumulative diversity prompt |
| Auth walls block testing | High | High | Clear auth setup flow + pre-flight detection |
| LLM rate limits during parallel execution | Low (v1) | Medium | Serial execution in v1; backoff in v2 |
| Cost exceeds expectations | Low | Medium | Default to Sonnet; show cost estimate before run |
| Persona fidelity questioned by users | Medium | Medium | Frame as "AI-generated hypotheses", not "real user data" |

---

## 12. Final Verdict

### What CrowdTest CAN be (realistically)

A fast, cheap way to get **structured hypotheses about UX issues** from diverse AI-generated perspectives. Think of it as "20 junior product reviewers who each spend 2 minutes on your site" — not "20 real users." The value is in:

1. **Coverage**: Finding issues a single tester would miss (different navigation paths, different mental models)
2. **Speed**: 10 minutes vs. days of user recruitment
3. **Cost**: $1-3 vs. $500+ for real user testing
4. **Structure**: Machine-readable issues with severity and evidence, not raw video recordings

### What CrowdTest CANNOT be (and shouldn't claim to be)

- A replacement for real user testing (the 15% fidelity gap is real)
- An accessibility audit tool (it can't simulate assistive technology)
- A visual design reviewer (it can't see the design, only the DOM)
- A performance testing tool (it can flag slow loads but not measure Core Web Vitals)
- A security testing tool (that's a completely different problem)

### Go/No-Go

**Go**, with these conditions:
1. Scout phase ships in v0.2 (the garbage-in problem is existential)
2. Feedback is grounded in action logs (no slop)
3. Marketing positions it as "AI product review" not "synthetic user testing" (manages expectations)
4. Self-validation test suite exists before public launch (test the tester)
5. Default to 10 personas on Sonnet (cost-effective, sufficient diversity)

The competitive landscape confirms the opportunity: no one has connected persona generation + browser automation + structured product feedback. The technical risk is real but manageable. Ship the Scout phase and single-persona loop first, validate feedback quality, then scale up.
