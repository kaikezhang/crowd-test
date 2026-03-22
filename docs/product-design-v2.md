# CrowdTest — Product Design v2: The Complete Rethink

> Date: 2026-03-22
> Author: Product Design Deep Dive
> Status: Design proposal — supersedes previous docs as the canonical product vision

---

## 1. First Principles — What Is This Product Really?

### Strip everything away. What's left?

CrowdTest is not "AI user testing." That's builder jargon. From the user's perspective:

**CrowdTest answers the question: "What's wrong with my product that I can't see?"**

That's it. The curse of knowledge is the atomic problem. You built the thing. You can't unknow how it works. You navigate perfectly because you DESIGNED the navigation. You understand the copy because you WROTE the copy. You find the CTA because you PUT it there.

CrowdTest breaks the curse by generating **alien perspectives** — minds that encounter your product with zero context, different assumptions, different patience thresholds, different goals. The value isn't in the AI. The value is in the **fresh eyes**.

### The atomic unit of value

It's not the report. It's not the persona. It's the **single insight you didn't know**.

> "The signup form rejects .co email addresses — 7 out of 10 personas couldn't register."

That one sentence justifies the entire run. Everything else (the report, the personas, the scores) is packaging around that moment.

### What makes someone come back?

Two things:

1. **They found something they didn't know** — and it was right. They fix it, and the fix matters.
2. **They can see change** — they run CrowdTest again after the fix, and the issue is gone. The score goes up. Progress is visible.

Without #1, the product is useless. Without #2, it's a one-time novelty.

### The real competitor isn't UserTesting.com

It's **"doing nothing."** 90% of solo founders ship without ANY external feedback. They don't compare CrowdTest to UserTesting ($245). They compare it to **zero**. The bar is: "Is this better than nothing?" The answer should be overwhelmingly yes.

---

## 2. The Complete Pipeline — What's Actually Missing?

The current pipeline is: **Scout → Persona Generator → Browser Sessions → Feedback → Report.**

This is a testing pipeline. It's missing everything that makes the testing INTELLIGENT. Here's the complete picture:

### The Missing Phase 0: Product Analysis

This is the gap the founder called out. Before you generate personas, you need to UNDERSTAND THE PRODUCT. The current Scout phase checks if the page loads. That's plumbing. Product Analysis asks:

**"What IS this thing, who is it for, and what are they supposed to DO with it?"**

#### What Product Analysis should extract:

```
1. Product Category — What type of product is this?
   - SaaS tool, marketplace, content site, e-commerce, social app, dev tool...
   - This determines EVERYTHING about what "good UX" means

2. Value Proposition — What does the product promise?
   - Parse the headline, subhead, and CTA text
   - "Send invoices in 30 seconds" → the persona should TRY to send an invoice
   - If a persona can't figure out what the product DOES in 30 seconds, that's issue #1

3. Core User Tasks — What can someone actually DO here?
   - Identify the primary actions: sign up, create, browse, buy, configure...
   - These become persona GOALS, not generic "explore the site"

4. Competitor Inference — What does this product compete with?
   - A project management tool competes with Notion, Asana, Linear
   - Personas should bring EXPECTATIONS from competitors
   - "I expected drag-and-drop like Trello, but this doesn't have it"

5. Target Audience Signals — Who is this built for?
   - Pricing signals: $9/mo = indie, $99/seat = enterprise
   - Copy signals: "for developers" vs "for teams" vs "for creators"
   - Feature signals: API docs = developers, analytics dashboard = managers

6. Maturity Signals — How polished is this product?
   - Landing page only? Early beta? Full product?
   - This calibrates feedback severity — don't roast an MVP for missing features
```

#### Why this matters:

Without Product Analysis, you get this:
```
Persona goal: "Explore the website"
Persona feedback: "The website has a clean design" ← USELESS
```

With Product Analysis, you get this:
```
Product: Invoice management SaaS
Core task: Create and send an invoice
Persona goal: "Send an invoice to a client named Acme Corp for $500"
Persona feedback: "I couldn't figure out how to add line items.
  The '+' button looks like it should add an item but it opens
  a settings panel instead." ← ACTIONABLE
```

The difference between generic and specific feedback is NOT in the feedback phase. It's in how well you understood the product BEFORE generating personas.

### The Missing Phase 0.5: User Context Intake

The user (founder) KNOWS things about their product that the LLM can't infer:

```
- "The main flow is: sign up → create workspace → invite team → start project"
- "Most users come from Notion and expect similar UX"
- "We just redesigned the pricing page, that's what I want tested"
- "The onboarding has a known issue with mobile — focus there"
- "Users need to connect their Stripe account before they can do anything"
```

Right now, CrowdTest asks for a URL and maybe an `--audience` hint. That's leaving 80% of the founder's context on the table.

#### Proposed context intake (optional, but high-value):

```
crowdtest https://myapp.com \
  --focus "onboarding flow" \
  --context "Invoice management tool for freelancers" \
  --competitors "FreshBooks, Wave, Zoho Invoice" \
  --known-issues "Mobile signup is broken, ignore it" \
  --key-flow "signup → create invoice → send to client"
```

Or conversationally:
```
/crowd-test https://myapp.com

Before I start, a few quick questions:
1. What does your product do in one sentence?
2. What's the main thing a user should be able to accomplish?
3. Anything specific you want me to focus on?
4. Any known issues I should ignore?
```

### The Missing Phase Between Browser and Feedback: Journey Reconstruction

The current pipeline goes: browser session → action log → feedback. But action logs are flat sequences. They miss the STORY.

What's needed is **Journey Reconstruction** — turning a flat action log into a narrative arc:

```
Journey Arc for Persona "Maria Chen":
┌─────────────────────────────────────────────────────────┐
│ Confidence                                              │
│ ██████████                                              │
│ ████████████████                                        │
│ ████████████████████                                    │
│ ████████████████████████                                │
│ ██████████████████████████████                          │
│ ████████████████████████████                            │
│ ████████████████████                                    │
│ ██████████████                                          │
│ ██████████       ← confusion spike (pricing page)       │
│ ████████████████████████                                │
│ ████████████████████████████████████                    │
│                                                         │
│ Landing ──► Features ──► Pricing ──► Signup ──► Dashboard│
│    ✓           ✓           ⚠️          ✓          ✓      │
└─────────────────────────────────────────────────────────┘
```

This is vastly more useful than a list of actions. It shows WHERE in the journey problems occur, and the emotional trajectory tells a story that resonates with founders.

### The Missing Post-Report Phase: Delta Tracking

After the report, what? Currently: nothing. The report is a dead artifact.

What's needed:

```
Run 1 (March 15): Score 5.8/10, 18 issues (2 critical)
   ↓ (founder fixes critical issues)
Run 2 (March 22): Score 7.2/10, 12 issues (0 critical)
   ↓ Delta: +1.4 score, -6 issues, critical issues resolved ✅

"Your product improved 24% since last test.
 Critical issue 'signup rejects .co emails' is now fixed.
 New issue found: checkout flow added unnecessary step."
```

This creates a **feedback loop** — the most powerful retention mechanism possible. The founder doesn't just get a report; they get a SCORE THEY CAN IMPROVE. That's what makes them come back.

### The Complete Redesigned Pipeline

```
Phase 0: PRODUCT ANALYSIS ← NEW
  Input: URL + optional user context
  Output: ProductProfile (category, value prop, core tasks, competitors, audience)

Phase 0.5: USER CONTEXT INTAKE ← NEW (optional)
  Input: Founder's knowledge (focus areas, known issues, key flows)
  Output: TestingBrief (merged ProductProfile + user context)

Phase 1: SCOUT (enhanced)
  Input: URL
  Output: SiteMap + TechProfile + PageInventory
  (Same as before but feeds into Product Analysis)

Phase 2: PERSONA GENERATION (enhanced)
  Input: TestingBrief + SiteMap
  Output: PersonaSet (derived from product, not random)

Phase 3: TASK ASSIGNMENT ← NEW
  Input: PersonaSet + TestingBrief
  Output: PersonaTaskMatrix (specific tasks per persona based on their profile)

Phase 4: BROWSER SESSIONS (enhanced)
  Input: Persona + Tasks + SiteMap
  Output: ActionLog + EmotionalArc + TaskCompletionRecord

Phase 5: JOURNEY RECONSTRUCTION ← NEW
  Input: ActionLog per persona
  Output: JourneyNarrative (story arc, confidence curve, friction points)

Phase 6: FEEDBACK SYNTHESIS (enhanced)
  Input: JourneyNarrative + ActionLog
  Output: PersonaFeedback (grounded, comparative, with expectations)

Phase 7: AGGREGATION & SCORING (enhanced)
  Input: All PersonaFeedback
  Output: AggregatedReport + ProductScore + SegmentAnalysis

Phase 8: DELTA ANALYSIS ← NEW (if previous run exists)
  Input: Current report + previous report
  Output: DeltaReport (what improved, what regressed, new issues)
```

---

## 3. Persona Design — Go Deeper

### The fundamental problem with current persona generation

The current approach treats persona generation as a COMBINATORIAL problem: pick from dimension matrix (tech_level × age × purpose × personality × device), ensure diversity, output N personas.

This produces PLAUSIBLE personas but not USEFUL ones. A 65-year-old novice tablet user with low patience is a valid combination, but is she a realistic user of YOUR product? If you're building a developer tool, she's noise.

### Personas should be DERIVED from the product

The Product Analysis phase should drive persona generation, not a random matrix:

```
Product: "InvoiceNinja — invoice management for freelancers"

Derived persona archetypes:
1. The New Freelancer — just got their first client, never sent a professional invoice
2. The Switcher — using FreshBooks, considering switching, has expectations
3. The Power User — sends 50+ invoices/month, needs efficiency features
4. The Reluctant Invoicer — hates admin work, wants it over with fast
5. The International Freelancer — needs multi-currency, tax compliance
6. The Mobile-First Worker — does everything from their phone between meetings
7. The Accessibility User — uses keyboard navigation, large fonts
8. The Skeptic — doesn't trust new tools with financial data, needs reassurance
9. The Window Shopper — evaluating 3 tools, will spend exactly 2 minutes
10. The Boss's Requester — their manager told them to "check out InvoiceNinja"
```

These personas are derived from WHO WOULD ACTUALLY USE THIS PRODUCT. Each one tests a different aspect of the product's fitness.

### What makes a persona SET useful vs individual personas?

A persona set should function like a **test matrix** — each persona covers a different failure mode:

| Persona Type | What They Test |
|---|---|
| The First-Timer | Onboarding clarity, information scent, time-to-value |
| The Switcher | Feature parity, migration friction, learning curve |
| The Power User | Efficiency, shortcuts, advanced features, scalability |
| The Impatient One | Speed to core task, friction points, abandonment triggers |
| The Confused One | Error handling, help systems, recovery paths |
| The Mobile User | Responsive design, touch targets, content priority |
| The Skeptic | Trust signals, security indicators, social proof |
| The International User | Localization, currency, date formats, RTL |
| The Accessibility User | Keyboard nav, screen reader compat, contrast |
| The Edge Case Finder | Empty states, boundary values, weird inputs |

A useful set has **COVERAGE** — if you removed any persona, you'd lose visibility into a category of issues.

### Ensuring behavioral DIVERGENCE

The core problem: LLMs are RLHF-trained to be helpful, polite, and thorough. Given 10 different persona prompts, they produce 10 slightly different versions of "a competent person politely navigating the site."

The current approach (behavioral rules) is a good start but doesn't go far enough. Here's what's needed:

#### 1. Action Space Constraints (not just prompt instructions)

```
Persona: "The Scanner"
Action constraints:
  - NEVER read any text longer than 1 line
  - ONLY interact with elements above the fold
  - Make a decision about the product within 5 actions
  - If you can't figure out what the product does in 3 clicks, leave

Persona: "The Thorough Reader"
Action constraints:
  - Read ALL visible text before taking any action
  - Scroll to bottom of every page before interacting
  - Never skip a form field, even optional ones
  - Take at least 15 actions before forming an opinion
```

These aren't personality descriptions. They're **hard constraints on the action space** that force genuinely different browsing patterns.

#### 2. Entry Point Variation

Not all users enter through the homepage:

```
Persona A: Arrives at homepage (organic search)
Persona B: Arrives at /pricing (comparison shopping)
Persona C: Arrives at /blog/how-to-send-invoices (content marketing)
Persona D: Arrives at /signup (direct referral link)
Persona E: Arrives at /features (evaluating)
```

Different entry points create fundamentally different user experiences. A user who starts at /pricing has completely different expectations than one who starts at the homepage.

#### 3. Adversarial Personas

These are deliberately hostile or chaotic:

```
"The Breaker"
  - Try to submit forms with empty fields
  - Enter absurdly long text in input fields
  - Click the back button mid-flow
  - Open the same page in multiple tabs
  - Try to do things in the wrong order
  - Look for ways to get error messages

"The Skeptic"
  - Look for trust signals (HTTPS, privacy policy, testimonials)
  - Check if prices include tax
  - Look for fine print on "free trial"
  - Search for reviews or complaints
  - Test if you can actually cancel/delete your account

"The Speed Runner"
  - Try to complete the core task in minimum clicks
  - Skip everything non-essential
  - Ignore all marketing content
  - If stuck for more than 10 seconds, leave
```

#### 4. Emotional State Presets

Real users don't browse in a vacuum. They have context:

```
"The Stressed User"
  - You're on a deadline. Your client needs this invoice in 10 minutes.
  - Behavioral: Skip tutorials, skip setup wizards, go directly to core action
  - If anything takes more than 30 seconds, express frustration
  - Typos in form fields (you're rushing)

"The Distracted User"
  - You're browsing on your phone while commuting
  - Behavioral: Only spend 2-3 seconds on any screen before deciding
  - Lose your place after every 3 actions (scroll to top, re-orient)
  - Abandon if asked to type more than 20 characters

"The Comparison Shopper"
  - You're evaluating this against 2 other products
  - Behavioral: Go directly to pricing and features
  - Note anything that's missing compared to competitors
  - Decide within 5 minutes: is this worth a deeper look?
```

### Persona Memory and Continuity

For v2+, personas should be able to do multi-session testing:

```
Session 1: First visit — sign up, explore
Session 2: Return visit (next day) — can they find where they left off?
Session 3: Power use — can they accomplish their goal efficiently now?
```

This tests onboarding-to-retention, not just first impressions.

---

## 4. The Feedback Problem — Solve It For Real

### The taxonomy of valuable feedback

Not all feedback is equal. Here's a hierarchy from most to least valuable:

```
TIER 1: BLOCKING ISSUES (Gold)
"I literally could not complete [task] because [specific reason]"
- Task completion failures with clear evidence
- These are bugs that lose users. Highest signal.

TIER 2: CONFUSION SIGNALS (Silver)
"I didn't understand [element/copy/flow] and had to [workaround]"
- Moments where the user's mental model breaks
- Often invisible to builders because THEY understand the model

TIER 3: EXPECTATION MISMATCHES (Bronze)
"I expected [X] but got [Y]" or "I expected [X] but it wasn't there"
- Gap between what the product promises and what it delivers
- Often the source of "this feels off" sentiment

TIER 4: FRICTION POINTS (Copper)
"I could do it, but it took [N extra steps / was annoying because]"
- Not blocking, but creates negative sentiment over time
- Death by a thousand papercuts

TIER 5: COMPARATIVE GAPS
"[Competitor] does this with [method], this product [doesn't / does it worse]"
- Only valuable if the competitor comparison is relevant
- Requires Product Analysis to know what competitors matter

TIER 6: DELIGHT MOMENTS (Important but different)
"[Feature/interaction] was surprisingly good because [reason]"
- What to KEEP, not what to fix
- Often overlooked in issue-focused testing
```

### How to make the LLM NOTICE things

The LLM's default mode is to DESCRIBE what it sees. It doesn't REACT to it. Real users react. They get confused, surprised, delighted, frustrated. The LLM needs to be prompted to detect these specific signals:

#### Confusion Detection Prompt

```
After each action, ask yourself:
1. Did the result match what I expected? If not, what did I expect?
2. Do I know what to do next? If not, what's confusing?
3. Can I tell if my action worked? If not, what feedback is missing?
4. Did the page change in a way I didn't anticipate?

If ANY answer is "no" → log a CONFUSION_EVENT with:
  - What you expected
  - What actually happened
  - Why the gap exists
```

#### Absence Detection Prompt

The hardest thing for LLMs to notice is what ISN'T there:

```
At the end of each page visit, ask:
1. Is there a search function? (If no: is the site complex enough to need one?)
2. Is there a way to go back / undo? (If no: is this a destructive action?)
3. Are there error states? (Submit invalid data to check)
4. Is there help / documentation accessible? (If no: is this feature self-explanatory?)
5. Can I tell what this product COSTS? (If pricing isn't clear, flag it)
6. Is there social proof? (If no: would I trust this product without it?)
7. Is there a mobile experience? (If mobile, are touch targets adequate?)
```

#### Comparative Frame Prompt

```
You know that products like {competitors} exist.
As you use {product}, note any moment where you think:
- "[Competitor] does this better because..."
- "[Competitor] has [feature] that this product lacks"
- "This does [feature] better than [competitor] because..."

These comparative observations are EXTREMELY valuable feedback.
Only compare to competitors that serve the same use case.
```

#### The "First 30 Seconds" Test

The most underrated UX metric: **Can a new user explain what the product does after 30 seconds?**

```
After your first 30 seconds on the site (approximately 3-5 actions), stop and answer:
1. What does this product do? (one sentence)
2. Who is it for?
3. What would I do next to get value from it?
4. Would I continue using this, or leave?

If you can't answer questions 1-2 clearly, that's a CRITICAL issue.
If you can't answer question 3, that's a HIGH issue.
```

### Task Completion Rate — The Missing Metric

The single most important UX metric that the current design completely ignores:

```
For each persona:
  Task: {derived from product analysis}
  Result: COMPLETED / PARTIALLY_COMPLETED / FAILED / ABANDONED
  Steps to complete: {N} (fewer = better)
  Time to complete: {T}
  Frustration events: {count}

Aggregate across personas:
  Task Completion Rate: {X}% of personas completed the core task
  Average Steps: {N} (benchmark against optimal path)
  Efficiency Score: optimal_steps / actual_steps
```

A product where 3/10 personas can complete the core task has a MASSIVE UX problem. That single number is more valuable than 50 individual issue reports.

### The Canary Problem — Go Deeper

Current canary: "Is this feedback specific or generic?" That's necessary but not sufficient.

#### Enhanced Canary Battery:

```
Canary 1 — Specificity: "Could this feedback apply to ANY product, or only this one?"
Canary 2 — Groundedness: "Does every claim reference a specific action/element/page?"
Canary 3 — Actionability: "Could a developer read this and know exactly what to fix?"
Canary 4 — Novelty: "Is this feedback different from what the previous personas said?"
Canary 5 — Persona Fidelity: "Does this feedback reflect the persona's specific
            characteristics (tech level, purpose, patience), or is it generic?"
```

Any feedback that fails ≥2 canaries gets flagged and regenerated.

---

## 5. The Report — What Would Make Someone Screenshot This?

### Current report: good but forgettable

The current report is a severity-ranked issue list with segment insights. It's useful but it looks like every other automated tool output. Nobody screenshots a list.

### What makes a founder go "holy shit"

Three things:

#### 1. The Product Scorecard — A Number They Can Obsess Over

```
╔══════════════════════════════════════════════════════════════╗
║                                                              ║
║   myapp.com — CrowdTest Score: 6.2 / 10                     ║
║                                                              ║
║   ┌────────────────────────────────────────────────────────┐ ║
║   │ First Impression    ████████░░  8.0  "Clear value prop"│ ║
║   │ Task Completion     ████░░░░░░  4.0  "3/10 succeeded"  │ ║
║   │ Navigation          ██████░░░░  6.0  "Some confusion"  │ ║
║   │ Trust & Credibility ███████░░░  7.0  "Good signals"    │ ║
║   │ Error Handling      ███░░░░░░░  3.0  "Poor recovery"   │ ║
║   │ Mobile Experience   █████░░░░░  5.0  "Needs work"      │ ║
║   └────────────────────────────────────────────────────────┘ ║
║                                                              ║
║   🎯 Core Task: "Send an invoice"                            ║
║   📊 Task Completion Rate: 30% (3/10 personas)               ║
║   ⏱️  Avg Time to Complete: 4m 12s (optimal: ~1m 30s)        ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

This is screenshotable. The 6.2/10 creates an EMOTIONAL RESPONSE. "I thought my product was good, but it scored 6.2?" That drives action. And when they fix issues and rescore at 7.8, they'll share the improvement.

The subscores tell them WHERE to focus. A product with 8.0 first impression but 4.0 task completion has a completely different problem than one with 4.0 first impression but 8.0 task completion.

#### 2. The Persona Journey Map — Stories, Not Bullet Points

```
╔════════════════════════════════════════════════════════════════╗
║ 📖 Maria Chen — Freelance Nurse, Mobile, First-Time User      ║
╠════════════════════════════════════════════════════════════════╣
║                                                                ║
║ Confidence: ▁▃▅▇██▇▅▃▁▃▅▇                                    ║
║             Landing → Features → Pricing → Signup → Dashboard  ║
║                                          ↑                     ║
║                                    "Where do I enter           ║
║                                     my client's info?"         ║
║                                                                ║
║ 💬 "I liked the landing page — clear what this does. But when  ║
║    I got to the dashboard, I couldn't figure out how to create ║
║    my first invoice. There's no 'New Invoice' button anywhere  ║
║    obvious. I eventually found it under a '+ ' menu in the    ║
║    top right, but that symbol usually means 'add account'      ║
║    not 'create invoice' in my experience."                     ║
║                                                                ║
║ ⏱️ Time: 3m 42s │ Actions: 14 │ Task: ❌ Not completed         ║
║ 📱 Device: iPhone │ NPS: 4 │ Would return: Maybe               ║
╚════════════════════════════════════════════════════════════════╝
```

This is a STORY. Founders empathize with stories. "Maria Chen couldn't find the New Invoice button" hits different than "Issue #7: New Invoice CTA has low discoverability (severity: high)."

#### 3. The "If You Fix ONE Thing" Callout

```
╔══════════════════════════════════════════════════════════════╗
║ 🎯 IF YOU FIX ONE THING                                      ║
║                                                              ║
║ Make the core action ("Create Invoice") visible on the       ║
║ dashboard without requiring any clicks.                      ║
║                                                              ║
║ WHY: 7/10 personas couldn't find it. This single issue       ║
║ is responsible for 70% of task failures. Every persona who    ║
║ failed mentioned searching for a "New" or "Create" button.   ║
║                                                              ║
║ EVIDENCE:                                                    ║
║ • Maria (mobile): "Looked for 45 seconds, gave up"          ║
║ • James (desktop): "Found it under '+' but expected 'New'"   ║
║ • Priya (novice): "I thought the app was broken"             ║
║                                                              ║
║ IMPACT: Fixing this alone would likely improve task           ║
║ completion from 30% → 70%+                                   ║
╚══════════════════════════════════════════════════════════════╝
```

THIS is what gets shared on Twitter. Not "we used AI to test our product." But "AI tested our product and the #1 finding was that nobody could find the main button."

### Report Structure (Redesigned)

```markdown
# CrowdTest Report — {product_url}
# Score: {score}/10 — {one_line_assessment}

## 🎯 If You Fix One Thing
{highest_impact_single_recommendation}

## 📊 Scorecard
{dimension_scores_with_bars}

## 📈 Task Completion
{task_completion_rate, avg_time, optimal_vs_actual}

## 🔴 Critical Issues ({count})
{issues with multi-persona evidence and impact estimate}

## 🟡 High Issues ({count})
## 🟠 Medium Issues ({count})
## 🟢 Low Issues ({count})

## 📖 Persona Journeys
{expandable journey narratives with confidence curves}

## ✅ What's Working Well
{things to keep — with evidence}

## 📊 Segment Analysis
{by tech level, device, purpose — what different user types experience}

## 📈 Delta from Previous Run (if applicable)
{what improved, what regressed, new issues}

## 🔧 Recommended Fix Priority
{ordered list of fixes with estimated impact}
```

---

## 6. Distribution & Virality — Be Specific

### The sharing mechanism

The report itself is the marketing. Specifically:

#### The "CrowdTest Score" Badge

```
[![CrowdTest Score: 7.8/10](https://crowdtest.dev/badge/myapp.com)](https://crowdtest.dev/report/abc123)
```

Like a build-passing badge for UX. Founders who score well will DISPLAY it. Founders who see it will want their own score.

#### The Twitter/X Moment

The output should generate a natural "look at this" moment:

```
"Just ran @crowdtest on my app. Score: 6.2/10 😬
Biggest finding: 7/10 AI users couldn't find the main CTA.
I was so blind to it. Fixing now.

[screenshot of scorecard]"
```

This post writes itself. The score creates tension. The finding creates narrative. The "I was blind to it" creates relatability. Other founders reading this think: "I wonder what MY score is."

#### The Before/After

```
"Update: Fixed the CTA visibility issue @crowdtest flagged.
Re-tested: Score went from 6.2 → 8.1 🎉
Task completion: 30% → 85%

[side-by-side scorecard screenshots]"
```

This is the RETENTION loop and the VIRAL loop in one. The founder comes back to re-test (retention) and shares the improvement (virality).

### The 30-second hook

The first-run experience must be:

```
1. User types: /crowd-test https://myapp.com
2. Within 10 seconds: "Analyzing myapp.com... detected: SaaS invoicing tool"
3. Within 30 seconds: "Generating 10 diverse personas based on your product..."
4. At 60 seconds: "Persona 1 (Maria, mobile, novice) is browsing your product..."
5. Live progress: real-time status of each persona as they browse
6. At ~8 minutes: Full report with score
```

The key: **make it feel alive while it runs.** Show persona names, show what they're doing, show when they get confused. This turns waiting into watching.

#### Live Progress Output

```
🧪 CrowdTest — myapp.com
━━━━━━━━━━━━━━━━━━━━━━━

📋 Product: Invoice management SaaS for freelancers
🎯 Core task: Create and send an invoice

Persona 1/10: Maria Chen (mobile, novice)
  ├── ✅ Landing page — understood value prop
  ├── ✅ Clicked "Get Started"
  ├── ⚠️ Confused at dashboard — looking for "New Invoice"
  ├── ❌ Couldn't find create button after 45 seconds
  └── 🏁 Abandoned (task not completed)

Persona 2/10: James Wright (desktop, power user)
  ├── ✅ Skipped landing, went to /features
  ├── ✅ Signed up in 40 seconds
  ├── ⚠️ Found '+' menu but expected "New Invoice" button
  ├── ✅ Created invoice (4m 10s — slow)
  └── 🏁 Completed (with difficulty)

Persona 3/10: Priya Sharma (desktop, switcher from FreshBooks)
  ├── Browsing... (step 4/20)
  ...
```

### What the output looks like when shared

For maximum shareability, CrowdTest should optionally generate:

1. **Markdown report** — for embedding in GitHub PRs, Notion, etc.
2. **Standalone HTML** — for sharing as a link (host on user's own domain or CrowdTest CDN)
3. **Terminal summary** — for the immediate "wow" moment
4. **JSON** — for programmatic consumption (CI/CD, dashboards)
5. **Social image** — auto-generated scorecard image for sharing on Twitter/X

The HTML report should have a shareable URL pattern:
```
crowdtest-report-myapp-2026-03-22.html
```

Self-contained, no server needed, opens in any browser.

---

## 7. Edge Cases & Hard Problems

### Multi-language sites

```
Problem: Product is in German, but personas are English-speakers by default.

Solution:
- Product Analysis detects primary language via <html lang=""> and content analysis
- Generate personas appropriate to that language
- Include 1-2 "non-native speaker" personas to test i18n friendliness
- Flag if the product has a language switcher and test both
- Config option: --locale "de-DE" to force language context
```

### Single-page apps with complex state

```
Problem: React/Vue/Angular apps where the URL doesn't change but the view does.
         Accessibility snapshots may show loading states or stale components.

Solution:
- Extended wait strategy: after each action, wait for BOTH network idle AND
  DOM stability (no mutations for 500ms)
- State checkpointing: hash the accessibility snapshot; if unchanged after
  action, wait 1s more, then flag as "potential rendering issue"
- SPA detection in Scout: if URL doesn't change on navigation, flag as SPA
  and increase wait timeouts
- Limit: Accept that some complex SPAs will fail. Report it clearly:
  "This appears to be a complex SPA. CrowdTest had difficulty with
   [specific component]. Results may be incomplete."
```

### Logged-in vs logged-out

```
Problem: The product looks completely different when authenticated.
         Testing only the public face misses the actual product.

Solution:
- Scout phase should detect auth walls early
- Two-mode testing:
  Mode A: Anonymous browsing (landing page, marketing, signup flow)
  Mode B: Authenticated browsing (requires --storage-state)
- IDEAL: Both modes in one run — 5 personas test the public face,
  5 personas test the authenticated experience
- Auto-detect: If persona encounters login wall during browsing,
  log it as a finding ("5/10 pages require authentication")
```

### Products that require domain knowledge

```
Problem: Medical, legal, or financial products have specialized terminology.
         A generic persona won't know how to use a radiology imaging tool.

Solution:
- Product Analysis should detect specialized domains
- Persona generation includes domain-appropriate expertise:
  "You are a radiologist who has been reading X-rays for 15 years"
- BUT: still include 1-2 non-domain personas to test if the product
  is accidentally excluding potential users
- Config option: --domain "healthcare" --expertise-level "practitioner"
```

### Products with no clear "task" (social media, content sites)

```
Problem: What's the "task" on Instagram? Reddit? A blog?
         There's no "create invoice" equivalent.

Solution:
- Product Analysis identifies content/social products
- Switch from task-completion testing to ENGAGEMENT testing:
  - Can the persona find content they're interested in?
  - Does the feed/recommendation surface relevant items?
  - Is the content consumption experience smooth?
  - Can they interact (comment, like, share)?
  - How long before they get bored? (engagement duration)
- Different metrics: Engagement Time, Content Relevance Score,
  Interaction Ease instead of Task Completion Rate
```

### Mobile-only responsive designs

```
Problem: Some products only work on mobile. Desktop testing is pointless.

Solution:
- Scout phase checks responsive breakpoints
- If product has mobile-specific behaviors (swipe, touch gestures),
  note limitation: "CrowdTest uses Playwright which can emulate
  mobile viewports but not native touch gestures"
- Default device split should adapt: if product is mobile-first,
  weight 70% mobile / 20% desktop / 10% tablet instead of the
  current 60/30/10
- Auto-detect mobile-only: if desktop viewport shows "Download our app"
  or "Mobile only", flag it
```

### Sites behind Cloudflare or other WAFs

```
Problem: Cloudflare bot protection, rate limiting, CAPTCHA challenges.

Solution:
- This is a HARD limit. Browser automation triggers bot detection.
- Mitigations (partial):
  - Use realistic browser fingerprints (Playwright does this by default)
  - Rate-limit actions (already 500ms wait between actions)
  - Rotate user agents between personas
  - Use residential-looking viewport sizes
- If blocked: Clear error message — "This site has bot protection
  that blocked CrowdTest. Options: test on staging (without WAF),
  whitelist the testing IP, or provide pre-authenticated cookies."
- Future: Integration with tools like undetected-playwright
- Honest: This will always be an issue. Don't promise to bypass WAFs.
```

### The "empty state" problem

```
Problem: New product with no content. A project management tool with no projects.
         A social app with no posts. The personas have nothing to interact with.

Solution:
- Scout detects empty states: if pages have placeholder/empty content,
  flag it
- Two approaches:
  1. Test the empty state itself: "Can a new user figure out how to
     create their first [item]? Is the empty state helpful?"
  2. Provide seed data: --seed-data "Create 3 sample projects before testing"
     (persona #0 creates seed content, then real personas test)
- Empty state testing is actually VALUABLE: "Maria saw an empty dashboard
  and had no idea what to do first. The product needs an onboarding wizard."
```

### Rate limiting and accidental DDoS

```
Problem: 10 personas × 12 actions each = 120 page loads in 10 minutes.
         Some sites treat this as a DDoS attack.

Solution:
- Serial execution (v1) naturally throttles — only 1 browser at a time
- Add configurable delay between personas: --delay 5 (seconds between persona starts)
- Monitor for 429/503 responses; if detected, increase backoff
- For parallel execution (v2+): max 3 concurrent browsers, stagger starts
- Warn user: "CrowdTest will make approximately 120 requests to your site
  over 10 minutes. If your site has rate limiting, consider increasing
  limits for the test duration."
```

---

## 8. The SKILL.md Redesign — Complete Pipeline

### Phase 0: Product Analysis

```
TRIGGER: User provides URL (and optional context)

PROMPT GOAL: Understand the product before generating any personas.

STEPS:
1. Navigate to the URL
2. Read the landing page content (headline, subhead, CTA, nav items)
3. Answer these questions:
   a. What does this product DO? (one sentence)
   b. What CATEGORY is it? (SaaS, e-commerce, content, social, dev tool, etc.)
   c. Who is the INTENDED USER? (signals from pricing, copy, features)
   d. What is the CORE TASK a user should accomplish?
   e. What COMPETITORS does this resemble?
   f. What MATURITY stage is this? (landing page only, MVP, full product)
   g. Is this a FREE or PAID product? What's the pricing model?

OUTPUT: ProductProfile
{
  "url": "...",
  "name": "...",
  "category": "...",
  "value_proposition": "...",
  "intended_user": "...",
  "core_tasks": ["..."],
  "competitors": ["..."],
  "maturity": "...",
  "pricing_model": "...",
  "language": "...",
  "tech_signals": ["..."]
}

WHY THIS MATTERS: Every subsequent phase uses ProductProfile to generate
relevant personas, appropriate tasks, and grounded feedback.
Without this, you get "generic user explores a website."
With this, you get "freelancer tries to send an invoice using InvoiceNinja."
```

### Phase 1: Scout (Enhanced)

```
TRIGGER: ProductProfile exists

PROMPT GOAL: Verify the site is testable and build a map of what's there.

STEPS: (Same as current Scout, plus)
1. Navigate URL, check health
2. Detect auth/CAPTCHA/cookie banners
3. Multi-page exploration: follow ALL top-level nav links
4. For each page: classify type, count interactive elements, note key features
5. Identify the CRITICAL PATH: what sequence of pages leads to the core task?
6. Check responsive behavior: does the nav change at mobile viewport?

OUTPUT: SiteMap (enhanced)
{
  "url": "...",
  "status": "ready|auth_required|captcha|failed",
  "pages": [...],
  "critical_path": ["/", "/signup", "/dashboard", "/new-invoice"],
  "nav_structure": {...},
  "responsive_issues": [...],
  "empty_states_detected": [...]
}
```

### Phase 2: Persona Generation (Redesigned)

```
TRIGGER: ProductProfile + SiteMap exist

PROMPT GOAL: Generate personas DERIVED from the product, not random combinations.

STEPS:
1. From ProductProfile, identify 5 persona ARCHETYPES relevant to this product:
   - The First-Timer (tests onboarding)
   - The Switcher (tests against competitor expectations)
   - The Power User (tests efficiency)
   - The Skeptic (tests trust)
   - The Edge Case (tests robustness)

2. For each archetype, generate 2 specific personas (total: 10)
   Each persona gets:
   - Demographics tied to the product's audience
   - A SPECIFIC task derived from ProductProfile.core_tasks
   - 3-5 behavioral rules as hard IF-THEN constraints
   - An entry point (not always homepage)
   - An emotional state/context (why they're here NOW)

3. Diversity validation:
   - No two personas share (tech_level + purpose + device)
   - At least 1 adversarial persona (the Breaker or Skeptic)
   - At least 1 mobile persona
   - At least 1 non-default entry point

OUTPUT: PersonaSet
[{
  "id": "persona_001",
  "name": "Maria Chen",
  "archetype": "first_timer",
  "tech_level": "intermediate",
  "purpose": "Send first invoice to client",
  "age": 34,
  "context": "Just landed first freelance client, needs to invoice
              them professionally. Has 30 minutes before a meeting.",
  "entry_point": "/",
  "device": "mobile",
  "competitors_used": ["Wave"],
  "behavioral_rules": [
    "Skip any text longer than 2 sentences",
    "If you can't find what you need in 3 clicks, use search",
    "After 2 failures, express frustration out loud",
    "Ignore anything that says 'enterprise' or 'team plan'",
    "You're in a slight hurry — don't explore, go straight to your goal"
  ],
  "emotional_state": "mildly_stressed",
  "success_criteria": "Successfully created and 'sent' an invoice"
}]
```

### Phase 3: Task Assignment

```
TRIGGER: PersonaSet + ProductProfile exist

PROMPT GOAL: Give each persona a SPECIFIC, measurable task — not "explore the site."

STEPS:
1. For each persona, define:
   - PRIMARY task: the main thing they're trying to do
   - SECONDARY tasks: things they might try along the way
   - SUCCESS criteria: how to determine if the task was completed

2. Task difficulty should match persona:
   - First-Timer: "Sign up and create your first [item]"
   - Power User: "Find the keyboard shortcuts and complete [task] in <5 steps"
   - Skeptic: "Find the privacy policy, cancellation terms, and a customer review"

OUTPUT: PersonaTaskMatrix
[{
  "persona_id": "persona_001",
  "primary_task": "Create an invoice for $500 to 'Acme Corp' and send it",
  "secondary_tasks": [
    "Find the pricing page and understand what's free vs paid",
    "Check if invoices support multiple currencies"
  ],
  "success_criteria": "Invoice created with correct amount and recipient,
                        and 'send' action completed",
  "max_actions": 20,
  "max_time": "3 minutes"
}]
```

### Phase 4: Browser Sessions (Enhanced)

```
TRIGGER: PersonaTaskMatrix exists

PROMPT GOAL: Execute browsing session as persona, with task awareness.

STATE MACHINE: (Same as current, plus emotional tracking)

INIT → NAVIGATE → ORIENT → ACT ⟷ OBSERVE → DECIDE → DONE

ENHANCED ACTION PROMPT:
For each action, the LLM receives:
- Persona narrative + behavioral rules
- Current accessibility snapshot
- Last 5 actions summary
- Their PRIMARY TASK and current progress toward it
- Issues found by previous personas (cumulative diversity)
- Emotional state tracker: "How are you feeling about this product right now?
  (confident / neutral / confused / frustrated / ready_to_leave)"

AFTER EACH ACTION, LOG:
{
  "step": 7,
  "type": "click",
  "target": "'+' button in top-right",
  "expected": "New invoice creation form",
  "actual": "Settings dropdown appeared",
  "success": false,
  "emotional_state": "confused",
  "confusion_event": {
    "expected": "Create new invoice",
    "got": "Account settings dropdown",
    "gap": "'+' icon is ambiguous — used for creation in most apps
            but means settings here"
  },
  "task_progress": "0% — still haven't found invoice creation"
}

OUTPUT: EnhancedActionLog
{
  "persona_id": "persona_001",
  "actions": [...],
  "emotional_arc": ["confident", "confident", "neutral", "confused",
                     "confused", "frustrated", "frustrated"],
  "task_result": "FAILED",
  "task_progress": "20% — found dashboard but never created invoice",
  "abandon_reason": "Could not find invoice creation after 14 actions",
  "confusion_events": [{...}],
  "pages_visited": ["/", "/signup", "/dashboard"],
  "time_seconds": 180,
  "first_30s_impression": "Clean landing page, clear what it does"
}
```

### Phase 5: Journey Reconstruction

```
TRIGGER: EnhancedActionLog exists for a persona

PROMPT GOAL: Turn flat action log into a narrative journey with emotional arc.

STEPS:
1. Identify the KEY MOMENTS in the journey:
   - First impression (actions 1-3)
   - Confusion points (where emotional_state changes to confused/frustrated)
   - Recovery moments (confusion → back to confident)
   - Abandonment point (if applicable)
   - Delight moments (something worked surprisingly well)

2. Construct the CONFIDENCE CURVE:
   - Map emotional_state to a 0-10 confidence score over time
   - Identify the peaks and valleys

3. Write the JOURNEY NARRATIVE:
   - In first person, as the persona
   - Reference specific actions and pages
   - Include the "I expected X but got Y" moments
   - Include comparative observations (if persona has competitor experience)

OUTPUT: JourneyNarrative
{
  "persona_id": "persona_001",
  "confidence_curve": [8, 8, 7, 4, 3, 2, 2, 3, 5, 6, 7, 7, 5, 3],
  "key_moments": [
    { "type": "positive", "step": 1, "description": "Clear value prop on landing" },
    { "type": "confusion", "step": 7, "description": "'+' button opened settings, not new invoice" },
    { "type": "frustration", "step": 12, "description": "Still can't find invoice creation" }
  ],
  "narrative": "I opened the site on my phone and immediately understood
    what it does — 'Send professional invoices in 30 seconds.' Great.
    I signed up (easy, just email + password). But then I hit the dashboard
    and... where do I create an invoice? There's no 'New Invoice' button.
    I see a '+' icon but that opened account settings. I tried the hamburger
    menu — nothing. On Wave, there's a big green 'Create Invoice' button
    right on the dashboard. Here, I genuinely cannot figure out how to start.",
  "comparative_observations": [
    "Wave has a prominent 'Create Invoice' CTA on the dashboard"
  ],
  "overall_verdict": "Good product hidden behind a discoverability problem"
}
```

### Phase 6: Feedback Synthesis (Enhanced)

```
TRIGGER: JourneyNarrative + EnhancedActionLog exist

PROMPT GOAL: Extract structured, grounded, tiered feedback.

ENHANCED FEEDBACK STRUCTURE:
{
  "persona_id": "persona_001",
  "first_impression": "...",
  "task_completion": {
    "task": "Create and send an invoice",
    "result": "FAILED",
    "steps_taken": 14,
    "optimal_steps": 5,
    "efficiency": 0.0,
    "failure_reason": "Could not find invoice creation entry point"
  },
  "issues": [{
    "tier": 1,  // BLOCKING
    "type": "discoverability",
    "page": "/dashboard",
    "element": "Invoice creation entry point",
    "issue": "No visible 'Create Invoice' action on dashboard",
    "evidence": "Steps 7-14: exhaustive search of dashboard UI",
    "expected": "Prominent CTA like competitor Wave",
    "impact": "Prevents core task completion entirely",
    "suggested_fix": "Add a primary 'Create Invoice' button to dashboard"
  }],
  "absence_observations": [
    "No onboarding wizard for first-time users",
    "No tooltip or guided tour",
    "No keyboard shortcuts visible"
  ],
  "comparative_feedback": [
    "Wave makes invoice creation the most prominent dashboard action"
  ],
  "positives": [{
    "feature": "Signup flow",
    "evidence": "Steps 2-4: completed in 40 seconds",
    "why": "Minimal fields, clear labels, no unnecessary steps"
  }],
  "scores": {
    "first_impression": 8,
    "task_completion": 2,
    "navigation": 4,
    "trust": 7,
    "error_handling": 3,
    "overall_nps": 4,
    "would_pay": false,
    "would_return": "maybe"
  },
  "one_line_verdict": "Great product I couldn't use because I couldn't
                        find the main feature"
}
```

### Phase 7: Aggregation & Scoring (Enhanced)

```
TRIGGER: All PersonaFeedback collected

PROMPT GOAL: Synthesize cross-persona insights into a scored, prioritized report.

STEPS:
1. DEDUPLICATION:
   - Group semantically identical issues
   - Preserve the best evidence from each persona

2. PRODUCT SCORE CALCULATION:
   Dimension scores (1-10, averaged across personas):
   - First Impression: avg(persona.scores.first_impression)
   - Task Completion: task_completion_rate × 10
   - Navigation: avg(persona.scores.navigation)
   - Trust & Credibility: avg(persona.scores.trust)
   - Error Handling: avg(persona.scores.error_handling)
   - Mobile Experience: avg(mobile_personas.scores.overall_nps)

   Overall Score = weighted_avg(dimensions)
   Weights: Task Completion (0.3), Navigation (0.2), First Impression (0.15),
            Trust (0.15), Error Handling (0.1), Mobile (0.1)

3. "IF YOU FIX ONE THING" ANALYSIS:
   - Identify the single issue that:
     a. Affects the most personas
     b. Has the highest severity
     c. Would most improve the product score if fixed
   - Write a clear, actionable recommendation

4. SEGMENT ANALYSIS:
   - By tech level: Do novices struggle more than experts?
   - By device: Are mobile users blocked?
   - By purpose: Do different goals lead to different experiences?
   - By archetype: Which persona type has the worst experience?

5. COMPARISON WITH PREVIOUS RUN (if available):
   - Score delta
   - Issues resolved
   - New issues introduced
   - Regression detection

OUTPUT: ComprehensiveReport (see Section 5 for format)
```

### Phase 8: Delta Analysis (Conditional)

```
TRIGGER: Previous CrowdTest report exists for same URL

PROMPT GOAL: Show what changed between runs.

STEPS:
1. Compare product scores: overall and per-dimension
2. Match issues: which old issues are resolved? which persist? which are new?
3. Task completion rate change
4. NPS trend

OUTPUT: DeltaReport
{
  "previous_score": 6.2,
  "current_score": 7.8,
  "delta": "+1.6",
  "resolved_issues": ["Invoice creation CTA visibility"],
  "persistent_issues": ["Mobile nav menu overlap"],
  "new_issues": ["Checkout adds unexpected step"],
  "task_completion_delta": "+40% (30% → 70%)",
  "trend": "improving"
}
```

### Complete Data Flow

```
URL + Context
    │
    ▼
┌──────────────────┐
│ PRODUCT ANALYSIS │ → ProductProfile
│ "What is this?"  │   {category, core_tasks, competitors, audience}
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ SCOUT            │ → SiteMap
│ "Is it testable?"│   {pages, critical_path, responsive_issues}
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ PERSONA GEN      │ → PersonaSet
│ "Who uses this?" │   [{archetype, task, rules, entry_point, context}]
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ TASK ASSIGNMENT   │ → PersonaTaskMatrix
│ "What should they│   [{persona, primary_task, success_criteria}]
│  try to do?"     │
└────────┬─────────┘
         │
    ┌────┴────┐ (for each persona)
    │         │
    ▼         │
┌──────────┐  │
│ BROWSER  │──┤→ EnhancedActionLog
│ SESSION  │  │  {actions, emotional_arc, task_result, confusion_events}
└────┬─────┘  │
     │        │
     ▼        │
┌──────────┐  │
│ JOURNEY  │──┤→ JourneyNarrative
│ RECON    │  │  {confidence_curve, key_moments, narrative}
└────┬─────┘  │
     │        │
     ▼        │
┌──────────┐  │
│ FEEDBACK │──┘→ PersonaFeedback
│ SYNTHESIS│     {tiered_issues, scores, comparisons, absences}
└────┬─────┘
     │
     ▼ (all personas done)
┌──────────────────┐
│ AGGREGATION      │ → ComprehensiveReport
│ Score + Dedup +  │   {product_score, fix_one_thing, issues, journeys}
│ Insights         │
└────────┬─────────┘
         │
         ▼ (if previous run exists)
┌──────────────────┐
│ DELTA ANALYSIS   │ → DeltaReport
│ "What changed?"  │   {score_delta, resolved, new, trend}
└──────────────────┘
```

### LLM Prompt Strategy by Phase

| Phase | Model | Temperature | Token Budget | Why |
|-------|-------|-------------|-------------|-----|
| Product Analysis | Sonnet | 0.3 | ~3K in, ~500 out | Analytical, needs precision |
| Scout | N/A (mostly Playwright) | — | ~1K per page | Mechanical, low LLM need |
| Persona Generation | Sonnet | 0.8 | ~2K in, ~500 out per persona | Creative, needs diversity |
| Task Assignment | Sonnet | 0.3 | ~2K in, ~300 out per persona | Analytical |
| Browser Session | Sonnet | 0.7 | ~2.5K in, ~200 out per action | Balanced: creative + grounded |
| Journey Reconstruction | Sonnet | 0.5 | ~3K in, ~800 out | Narrative synthesis |
| Feedback Synthesis | Sonnet | 0.3 | ~4K in, ~1K out | Structured extraction |
| Aggregation | Opus (optional) | 0.3 | ~8K in, ~2K out | Complex cross-persona reasoning |
| Delta Analysis | Sonnet | 0.2 | ~5K in, ~500 out | Comparative, needs accuracy |

### Estimated Cost (Redesigned Pipeline)

| Personas | Sonnet | Opus (aggregation only) | Total Time |
|----------|--------|------------------------|------------|
| 5 | ~$0.55 | +$0.15 | ~6 min |
| 10 | ~$1.00 | +$0.25 | ~12 min |
| 20 | ~$1.90 | +$0.40 | ~22 min |

The added phases (Product Analysis, Journey Reconstruction, Delta Analysis) add ~$0.10-0.20 per run — negligible for the value they provide.

---

## 9. The Competitive Moat — What This Design Creates

This v2 design creates advantages that are hard to replicate:

1. **Product-aware testing** — Personas derived from the product, not random. Nobody else does this.

2. **Task completion as the north star metric** — A single number (30% task completion) is more compelling than 20 issue descriptions. It's the metric that makes founders act.

3. **The Product Score** — A trackable, comparable, improveable number. Creates retention loops that no competitor has.

4. **Emotional journey mapping** — Not just "what happened" but "how did it feel at each step." This bridges the gap between LLM analysis and human UX insight.

5. **Delta tracking** — Turn a one-time tool into an ongoing practice. Every improvement is visible. Every regression is caught.

6. **Comparative intelligence** — "Your competitor does X better" is the most motivating feedback a founder can receive.

7. **The "Fix One Thing" insight** — The ultimate compression of a complex report into a single actionable sentence. This is what gets shared.

---

## 10. What Success Looks Like

**Minimum viable success**: A founder runs CrowdTest, finds ONE issue they didn't know about, fixes it, re-runs, and sees improvement. That's the atomic success loop.

**Product-market fit signal**: Founders run CrowdTest after every deploy. It becomes part of their workflow, not a one-time experiment.

**Viral success signal**: Founders share their CrowdTest scores unprompted. "Just hit 8.5/10 on CrowdTest" becomes a thing founders post about.

**The dream metric**: "CrowdTest users have 2x higher Day-7 retention on their products than non-users." If we can prove that CrowdTest feedback leads to measurably better products, we don't need marketing. The data sells itself.

---

*This document should make the previous PRD, technical design, and CEO review feel like rough drafts. Not because they were wrong — they were right about the fundamentals. But they were thinking about the PIPELINE. This document thinks about the PRODUCT.*

*CrowdTest isn't a testing pipeline. It's a mirror that shows founders what their users would see — if they had any users.*
