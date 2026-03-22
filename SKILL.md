---
name: crowd-test
description: "Generate diverse AI personas to browse-test your web product and find UX issues you can't see. Use when you need user testing but have no users, no budget, and no time. One URL in, prioritized issue report out."
---

# CrowdTest — AI Synthetic User Testing

> One URL. N personas. Grounded UX feedback. Zero real users needed.

## Prerequisites

- Playwright MCP server (`@playwright/mcp`) must be available
  - If not available, run: `npx @playwright/mcp@latest`
- Claude Sonnet 4.6+ (or Opus 4.6 for higher quality)

## Quick Start

```
/crowd-test https://myapp.com
```

Or invoke naturally:
```
Test https://myapp.com with 10 AI personas and find UX issues
```

## How It Works

CrowdTest runs a 5-phase pipeline:

1. **Scout** — Verify the page loads, detect auth/CAPTCHA, build a site map
2. **Generate Personas** — Create N diverse users with behavioral rules
3. **Browser Sessions** — Each persona browses the product, logging every action
4. **Feedback** — Each persona reviews their actions and writes grounded feedback
5. **Aggregate** — Deduplicate issues, rank by severity and consensus, generate report

---

## Phase 1: Scout

Before ANY persona runs, verify the target is testable.

### Steps

1. Navigate to the target URL
2. Wait for network idle (10 second timeout)
3. Take an accessibility snapshot
4. Check page health:
   - **Interactive elements < 5** → Page may not have loaded. Retry once after 3 seconds.
   - **Auth detected** (look for "sign in", "log in", "create account" in the snapshot) → Stop and ask user for `--storage-state` JSON file with cookies
   - **CAPTCHA detected** (look for "captcha", "recaptcha", "hcaptcha") → Stop and inform user CrowdTest cannot bypass CAPTCHAs
   - **Cookie banner** (look for "accept cookies", "consent", "cookie preferences") → Automatically dismiss by clicking Accept/Close/Got it
5. If page is ready:
   - Identify top-level navigation links
   - Visit top 3-5 pages, take a quick snapshot of each
   - Build a site map

### Output: SiteMap

```json
{
  "url": "https://example.com",
  "title": "My App",
  "status": "ready",
  "pages": [
    {
      "url": "/",
      "type": "landing",
      "interactive_elements": 15,
      "key_elements": ["search bar", "sign up CTA", "pricing link", "feature cards"]
    }
  ],
  "cookie_banner": false,
  "tech_hints": ["React"]
}
```

If status is NOT "ready", stop and report to the user with the specific issue.

---

## Phase 2: Generate Personas

Create N diverse personas based on the site map.

### Steps

1. Analyze the site map to infer product type, target audience, and core features
2. For each persona (one at a time):
   - Review ALL previously generated personas
   - Generate a NEW persona that fills gaps in the collection
   - Must differ on ≥ 2 dimensions from every existing persona
   - Generate 3+ behavioral rules in IF-THEN format

### Dimension Matrix

| Dimension | Values |
|-----------|--------|
| tech_level | novice, intermediate, advanced, developer |
| purpose | 3-5 values inferred from the product |
| age | 18-65+, ensure at least 3 different ranges |
| personality.patience | 0.1-0.9 |
| personality.exploration_tendency | 0.1-0.9 |
| personality.attention_to_detail | 0.1-0.9 |
| device | desktop (60%), mobile (30%), tablet (10%) |

### CRITICAL: Behavioral Rules

Do NOT write personality adjectives. Write executable IF-THEN constraints:

```
✅ "After 2 clicks without finding what you want, express frustration and try search if available"
✅ "Skip any text section longer than 3 lines without reading it"  
✅ "If a form has more than 5 fields, only fill the required ones"
✅ "After 5 total failed attempts at anything, abandon the product entirely"

❌ "You are impatient and easily frustrated" (LLMs can't act dumb)
```

### Output: Persona

```json
{
  "id": "persona_001",
  "name": "Maria Chen",
  "tech_level": "intermediate",
  "purpose": "compare pricing plans",
  "age": 34,
  "industry": "healthcare",
  "personality": { "patience": 0.3, "exploration_tendency": 0.8, "attention_to_detail": 0.6 },
  "behavioral_rules": [
    "After 2 clicks without progress, try search if available",
    "Skip text sections longer than 3 lines",
    "If a form has more than 5 fields, only fill required ones"
  ],
  "device": "mobile",
  "narrative": "You are Maria Chen, a 34-year-old nurse practitioner..."
}
```

---

## Phase 3: Browser Sessions

For each persona, run a browser session.

### State Machine

```
INIT → NAVIGATE → ORIENT → ACT ⟷ OBSERVE → DECIDE → done
```

### Steps (per persona)

1. **INIT**: Set up browser context matching persona's device
2. **NAVIGATE**: Open the URL, dismiss cookie banner if needed
3. **ORIENT**: Take snapshot, identify interactive elements, plan next action based on persona's purpose
4. **ACT**: Execute ONE action (click/type/scroll/navigate)
   - Wait 500ms after each action
   - Take new snapshot
5. **OBSERVE**: Did the snapshot change? If not → try alternative approach
6. **DECIDE**: Continue or stop?
   - Continue if: actions < 20 AND time < 3 minutes AND not stuck
   - Stop if: goal met OR stuck OR limit hit
7. Log every action to the action log

### Action Prompt (per step)

For each action decision, provide the persona with:
- Their narrative and behavioral rules
- The current accessibility snapshot
- Their last 5 actions summary
- Issues already found by OTHER personas (cumulative diversity)

Ask: "What do you do next?" → ONE action as JSON

### Circuit Breakers

- **Max 20 actions** per persona
- **Max 3 minutes** per persona
- **5 consecutive failures** → force stop

### Output: ActionLog

```json
{
  "persona_id": "persona_001",
  "actions": [
    {
      "step": 1,
      "type": "click",
      "target": "Pricing link in nav",
      "success": true,
      "notes": "I want to see how much this costs for my clinic"
    }
  ],
  "pages_visited": ["/", "/pricing", "/signup"],
  "abandoned": false
}
```

---

## Phase 4: Feedback

Each persona reviews their action log and writes grounded feedback.

### Three-Phase Process

**Phase A** (already done): Action log from browser session = ground truth

**Phase B**: Give the persona their complete action log. Ask them to write honest feedback AS THEIR CHARACTER. Requirements:
- First impression = first 30 seconds only
- Every issue MUST reference a specific action log step
- Be specific: "button X on page Y" not "the UI could be better"
- Include positives if any exist
- NPS score, would_pay, would_return

**Phase C**: Extract structured JSON from the narrative

### Self-Validation (Canary Check)

After generating feedback, ask: "Is this feedback specific and grounded in the action log, or is it generic and could apply to any product?"

If generic → regenerate once. Still generic → keep but flag it.

### Output: PersonaFeedback

```json
{
  "persona_id": "persona_001",
  "first_impression": "The landing page looks professional but I can't tell what this product does in the first 5 seconds",
  "issues": [
    {
      "page": "/pricing",
      "element": "CTA button",
      "issue": "Says 'Start Free' but the next page asks for credit card",
      "severity": "critical",
      "evidence": "Action log steps 7-8"
    }
  ],
  "positives": [{ "feature": "Search", "why": "Found what I needed in 2 keystrokes" }],
  "nps_score": 5,
  "would_pay": false,
  "would_return": true,
  "one_line_verdict": "Useful product hidden behind confusing onboarding"
}
```

---

## Phase 5: Aggregate & Report

### Steps

1. Collect all PersonaFeedback
2. **Deduplicate**: Group semantically identical issues (different personas may describe the same problem differently)
3. **Score severity**: `severity_weight × (1 + log2(affected_personas))`
4. **Consensus**: Issues mentioned by >50% of personas = consensus issues
5. **Segment analysis**: Group insights by tech_level, device, purpose
6. **Generate Markdown report**

### Report Structure

```markdown
# CrowdTest Report — {product_url}

**Date**: {date}
**Personas**: {count} | **Pages visited**: {total} | **Issues found**: {unique_count}
**Avg NPS**: {avg_nps} | **Would pay**: {pay_rate}% | **Would return**: {return_rate}%

## 🔴 Critical Issues ({count})

### 1. {issue_description}
- **Affected**: {N}/{total} personas
- **Pages**: {pages}
- **Evidence**: {evidence_summary}
- **Personas**: {persona_names}

## 🟡 High Issues ({count})
...

## 🟠 Medium Issues ({count})
...

## 🟢 Low Issues ({count})
...

## 📊 Segment Insights

### By Tech Level
- **Novice**: {finding}
- **Advanced**: {finding}

### By Device
- **Mobile**: {finding}
- **Desktop**: {finding}

## ✅ What's Working
- {positive} ({mention_count}/{total} personas)

## 📋 Individual Persona Reports
<details><summary>Maria Chen (NPS: 5)</summary>
{full_feedback}
</details>
```

---

## Configuration

| Option | Default | Description |
|--------|---------|-------------|
| `personas` | 10 | Number of personas to generate |
| `model` | sonnet | LLM model (sonnet / opus) |
| `audience` | (auto-detected) | Target audience hint |
| `output` | stdout | Output file path |
| `storage-state` | none | Browser cookies/auth JSON file |
| `verbose` | false | Show action logs in output |

---

## Site Compatibility

| Site Type | Expected Success | Notes |
|-----------|-----------------|-------|
| Static marketing sites | 90%+ | Best case scenario |
| Server-rendered apps | 80%+ | Usually works |
| SPAs (React/Vue/Angular) | 50-70% | Hydration delays possible |
| Auth-protected | Works with `--storage-state` | User provides cookies |
| Localhost/dev server | 70%+ | Must be running |
| CAPTCHA-protected | 0% | Cannot bypass |

---

## Cost Estimate

| Personas | Sonnet | Opus |
|----------|--------|------|
| 5 | ~$0.30 | ~$1.50 |
| 10 | ~$0.60 | ~$3.00 |
| 20 | ~$1.20 | ~$6.00 |

---

*CrowdTest: 20 junior product reviewers who each spend 2 minutes on your site. Not 20 real users — but infinitely better than testing alone at 11pm.*
