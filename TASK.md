# TASK.md — Sprint 2: Persona Generation + Task Assignment + Browser Loop + Feedback

> ⚠️ DO NOT MERGE. Create PR and stop. 晚晚 reviews and merges.

## Sprint Goal

Implement Phases 2, 3, 4, and 6 in SKILL.md — the complete single-persona pipeline from "who would use this?" to "here's what they found."

After this sprint, a single persona can:
1. Be generated from the product profile
2. Get assigned a specific measurable task
3. Browse the product with emotional tracking
4. Produce grounded, evidence-linked feedback

## Context

- **Sprint 1 is done**: SKILL.md has complete Phase 0 (Product Analysis) and Phase 1 (Scout)
- Read `docs/product-design-v2.md` for the canonical design vision (especially sections 3, 4, 7, 8)
- Phase 2-8 already have stubs in SKILL.md — replace the stubs with full implementations
- **This is prompt writing, not code.** SKILL.md is instructions for an LLM to follow.

## What You're Building

Replace the Phase 2, 3, 4, and 6 stubs in SKILL.md with complete, detailed, step-by-step instructions.

---

### Phase 2: Persona Generation

**Input**: ProductProfile + SiteMap
**Output**: PersonaSet (array of Persona JSON)

**Key design from product-design-v2.md**: Personas are DERIVED from the product, not randomly combined from a dimension matrix.

**Instructions to write**:

1. Analyze ProductProfile to identify 5 persona **archetypes** relevant to THIS product:
   - **The First-Timer**: Never used this type of product. Tests onboarding clarity, time-to-value.
   - **The Switcher**: Coming from a competitor (use ProductProfile.competitors). Tests feature parity, migration friction.
   - **The Power User**: Experienced with this product category. Tests efficiency, shortcuts, advanced features.
   - **The Skeptic**: Doesn't trust new tools easily. Tests trust signals, security, social proof, cancellation policy.
   - **The Edge Case**: Unusual context — mobile-only, international, accessibility needs, rushed. Tests robustness.

2. For each archetype, generate 2 specific personas (total: 10 for default). Each persona must include:
   - Demographics tied to the ProductProfile's intended_user
   - A PURPOSE derived from ProductProfile.core_tasks
   - An ENTRY POINT — not all personas start at homepage:
     - 6/10: homepage (organic search)
     - 2/10: pricing or features page (comparison shopping)
     - 1/10: signup page (direct referral)
     - 1/10: a content/docs page (if exists in SiteMap)
   - An EMOTIONAL CONTEXT — why they're here NOW:
     - "Just got first client, needs to invoice them in 30 minutes"
     - "Evaluating 3 tools, will spend exactly 2 minutes"
     - "Manager told them to check this out, not personally motivated"
   - 3-5 BEHAVIORAL RULES as hard IF-THEN constraints (NOT personality adjectives)
   - A NARRATIVE paragraph (used as system prompt during browsing)

3. Cumulative diversity: each persona generation call includes summaries of ALL previously generated personas. Instruction: "Fill gaps in the collection — differ on ≥2 dimensions."

4. Diversity validation: No two personas should share the same (tech_level + purpose + device) combination.

**Output schema per persona**:
```json
{
  "id": "persona_001",
  "name": "Maria Chen",
  "archetype": "first_timer",
  "tech_level": "intermediate",
  "purpose": "Send first invoice to a client",
  "age": 34,
  "industry": "healthcare",
  "context": "Just landed first freelance client, needs to invoice them. Has 30 minutes before a meeting.",
  "entry_point": "/",
  "device": "mobile",
  "competitors_used": ["Wave"],
  "personality": {
    "patience": 0.3,
    "exploration_tendency": 0.8,
    "attention_to_detail": 0.6
  },
  "behavioral_rules": [
    "Skip any text longer than 2 sentences",
    "If you can't find what you need in 3 clicks, use search",
    "After 2 failures, express frustration out loud",
    "Ignore anything labeled 'enterprise' or 'team plan'",
    "You're in a hurry — go straight to your goal, don't explore"
  ],
  "emotional_state": "mildly_stressed",
  "narrative": "You are Maria Chen, a 34-year-old nurse practitioner who just started freelance consulting..."
}
```

---

### Phase 3: Task Assignment

**Input**: PersonaSet + ProductProfile
**Output**: PersonaTaskMatrix (specific task per persona)

**Key design**: Each persona gets a SPECIFIC, MEASURABLE task — not "explore the site."

**Instructions to write**:

1. For each persona, derive a primary task from ProductProfile.core_tasks, filtered by the persona's purpose and archetype:
   - First-Timer: "Sign up and [core_task_1]" (e.g., "Sign up and create your first invoice")
   - Switcher: "[core_task_1] and compare the experience to [competitor]"
   - Power User: "Complete [core_task_1] in the fewest steps possible"
   - Skeptic: "Find the privacy policy, pricing details, and a reason to trust this product"
   - Edge Case: A task that tests a boundary (mobile flow, unusual input, etc.)

2. Define SUCCESS CRITERIA for each task — how to determine COMPLETED vs FAILED:
   - "Invoice created with correct amount and recipient" = COMPLETED
   - "Reached signup form but couldn't submit" = PARTIALLY_COMPLETED
   - "Never found the invoice creation feature" = FAILED

3. Optional secondary tasks (1-2 per persona):
   - "Check if mobile experience works"
   - "Find pricing and understand what's free vs paid"

**Output schema**:
```json
{
  "persona_id": "persona_001",
  "primary_task": "Create an invoice for $500 to 'Acme Corp' and send it",
  "success_criteria": "Invoice created with correct amount and recipient, and 'send' action completed",
  "secondary_tasks": [
    "Find the pricing page and understand what's free vs paid"
  ],
  "max_actions": 20,
  "max_time": "3 minutes"
}
```

---

### Phase 4: Browser Sessions

**Input**: One Persona + their Task + SiteMap
**Output**: EnhancedActionLog

**This is the core testing phase.** The LLM embodies a persona and browses the product.

**State machine**: `INIT → NAVIGATE → ORIENT → ACT ⟷ OBSERVE → DECIDE → DONE`

**Instructions to write**:

1. **INIT**: Set up browser context
   - If persona.device is "mobile": resize viewport to 375x812 (iPhone)
   - If "tablet": resize to 768x1024 (iPad)
   - If "desktop": keep default viewport

2. **NAVIGATE**: Go to persona.entry_point (not always "/")
   - If cookie banner appears, dismiss it (same logic as Scout Phase 1)
   - Wait for page load

3. **ORIENT**: Take snapshot, identify what's available
   - Read the accessibility snapshot
   - Given the persona's task, plan the next action
   - Note: the LLM should think IN CHARACTER as the persona

4. **ACT**: Execute ONE action
   - Click, type, scroll, or navigate
   - Wait 500ms after each action
   - Take a new snapshot

5. **OBSERVE**: Check if action worked
   - Compare before/after snapshots
   - If unchanged → action may have failed → try alternative (RECOVER)
   - RECOVER options: scroll to reveal element, try parent element, use keyboard

6. **DECIDE**: Continue or stop?
   - Continue if: actions < 20 AND time < 3 min AND not stuck AND task not complete
   - Stop if: task completed OR task failed (stuck) OR limits hit

7. **EMOTIONAL TRACKING**: After each action, the persona reports their emotional state:
   - `confident` — "I know what to do"
   - `neutral` — "This is fine"
   - `confused` — "I don't understand what happened"
   - `frustrated` — "This isn't working"
   - `ready_to_leave` — "I'm about to give up"

**Action prompt structure** (what the LLM sees for each step):

```
System: You are {persona.narrative}
BEHAVIORAL RULES (MUST follow): {persona.behavioral_rules}

You are testing {product_url}. Your task: {task.primary_task}
Success criteria: {task.success_criteria}

Current page snapshot:
{accessibility_snapshot}

Your action history (last 5 steps):
{recent_actions_summary}

Issues already found by OTHER testers (find DIFFERENT ones):
{covered_issues_from_previous_personas}

Task progress: {task_progress_estimate}

What do you do next? Respond as JSON:
{
  "thinking": "your in-character thought process",
  "action": "click|type|scroll|navigate_back|done",
  "target": "element description from snapshot",
  "text": "(for type action only)",
  "emotional_state": "confident|neutral|confused|frustrated|ready_to_leave",
  "task_progress": "0-100% estimate"
}
```

**Circuit breakers** (hard stops):
- 20 actions → force DONE
- 3 minutes → force DONE
- 5 consecutive failed actions → force DONE, mark session as unstable

**Output schema** (EnhancedActionLog):
```json
{
  "persona_id": "persona_001",
  "entry_point": "/",
  "actions": [
    {
      "step": 1,
      "type": "click",
      "target": "hero CTA 'Get Started Free'",
      "expected": "Signup page",
      "actual": "Redirected to /signup with email form",
      "success": true,
      "emotional_state": "confident",
      "task_progress": "10%",
      "notes": "Good, clear CTA. Exactly what I expected."
    }
  ],
  "emotional_arc": ["confident", "confident", "neutral", "confused", "frustrated"],
  "task_result": "FAILED",
  "task_progress": "20%",
  "failure_reason": "Could not find invoice creation feature after 14 actions",
  "pages_visited": ["/", "/signup", "/dashboard"],
  "total_actions": 14,
  "duration_estimate": "~2.5 minutes",
  "confusion_events": [
    {
      "step": 7,
      "expected": "New invoice form",
      "got": "Settings dropdown",
      "gap": "'+' icon is ambiguous"
    }
  ]
}
```

---

### Phase 6: Feedback Synthesis

**Input**: Persona + EnhancedActionLog + Task
**Output**: PersonaFeedback

**Three-phase process**:

**Phase A**: Action log already exists from Phase 4 (ground truth) ✓

**Phase B**: The persona REVIEWS their action log and writes feedback IN CHARACTER:
- First impression (first 30 seconds / first 3 actions only)
- What worked well (with evidence from action log)
- What didn't work (with evidence)
- Task completion assessment
- Comparative observations (if persona has competitor experience)

**Phase C**: Extract structured JSON from the narrative:
- Tiered issues (BLOCKING → FRICTION → OBSERVATION)
- Evidence pointers to specific action log steps
- Absence observations ("there was no X")
- Scores

**Canary check**: After generating feedback, verify:
- "Could this feedback apply to ANY product?" → if yes, it's generic → regenerate once
- "Does every issue reference a specific action/element/page?" → if not → reject

**Output schema**:
```json
{
  "persona_id": "persona_001",
  "first_impression": "Landing page clearly explains what this does — send invoices. But on the dashboard, I was completely lost.",
  "task_completion": {
    "task": "Create and send an invoice for $500 to Acme Corp",
    "result": "FAILED",
    "steps_taken": 14,
    "failure_reason": "Could not find invoice creation entry point on dashboard"
  },
  "issues": [
    {
      "tier": 1,
      "type": "discoverability",
      "page": "/dashboard",
      "element": "Invoice creation entry point",
      "issue": "No visible 'Create Invoice' action on the dashboard. The '+' button opens settings, not creation.",
      "severity": "critical",
      "evidence": "Steps 7-14: exhaustive search of dashboard UI, tried '+' button (step 7), hamburger menu (step 9), sidebar links (steps 11-12)",
      "suggested_fix": "Add a prominent 'Create Invoice' button to the dashboard"
    }
  ],
  "absence_observations": [
    "No onboarding wizard for first-time users",
    "No tooltip explaining the '+' button",
    "No keyboard shortcuts visible"
  ],
  "positives": [
    {
      "feature": "Signup flow",
      "evidence": "Steps 2-4: completed signup in ~40 seconds",
      "why": "Minimal fields, clear labels, no unnecessary verification"
    }
  ],
  "comparative_feedback": [
    "Wave has a large 'Create Invoice' button on the dashboard — this product should too"
  ],
  "scores": {
    "first_impression": 8,
    "task_completion": 2,
    "navigation": 4,
    "trust": 7,
    "error_handling": 3,
    "nps": 4
  },
  "would_pay": false,
  "would_return": "maybe",
  "one_line_verdict": "Great product hidden behind a discoverability problem"
}
```

---

## Acceptance Criteria

- [ ] SKILL.md Phase 2 generates product-derived personas with archetypes, entry points, and behavioral rules
- [ ] SKILL.md Phase 3 assigns specific measurable tasks with success criteria
- [ ] SKILL.md Phase 4 has complete browser session state machine with emotional tracking
- [ ] SKILL.md Phase 4 includes the action prompt template showing what the LLM sees each step
- [ ] SKILL.md Phase 6 has three-phase feedback with canary check
- [ ] All output schemas match the JSON examples above
- [ ] Instructions are unambiguous — walk through them as if you were an LLM following them
- [ ] Phase 5, 7, 8 stubs preserved (implementation in Sprint 3)
- [ ] Cumulative diversity mechanism documented (previous persona issues fed to next persona)

## What NOT to Do

- Do NOT create application code — SKILL.md is the product
- Do NOT modify Phase 0 or Phase 1 (they're done from Sprint 1)
- Do NOT merge the PR
- Do NOT implement Phase 5 (Journey Reconstruction), 7 (Aggregation), or 8 (Delta)

## Test Mentally

Walk through the SKILL.md as if you're an LLM:
1. Given a ProductProfile for cal.com → can you generate 10 relevant personas?
2. Given persona "busy freelancer" → can you assign a specific task?
3. Given the task → can you follow the browser session instructions step by step?
4. Given the action log → can you write grounded feedback that references specific steps?
