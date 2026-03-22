# TASK.md — Sprint 3: Multi-Persona + Journey Recon + Aggregation + Score + Report

> ⚠️ DO NOT MERGE. Create PR and stop. 晚晚 reviews and merges.

## Sprint Goal

Complete the SKILL.md pipeline by implementing:
1. **Multi-persona orchestration** — how to run N personas sequentially
2. **Phase 5: Journey Reconstruction** — turn action logs into narratives
3. **Phase 7: Aggregation + Scoring** — Product Score, "Fix ONE Thing", issue dedup
4. **Phase 8: Delta Analysis** — compare with previous run

After this sprint, `crowd-test` is feature-complete. One URL in → full report out.

## Context

- **Sprint 1 done**: Phase 0 (Product Analysis) + Phase 1 (Scout) ✅
- **Sprint 2 done**: Phase 2 (Persona Gen) + Phase 3 (Tasks) + Phase 4 (Browser) + Phase 6 (Feedback) ✅
- **SKILL.md is 1274 lines** — only Phase 5, 7, 8 remain as stubs
- Read `docs/product-design-v2.md` sections 5, 7, 8 for design vision

## What You're Building

Replace the Phase 5, 7, 8 stubs in SKILL.md with full implementations. Also add a **Multi-Persona Orchestration** section between Phase 4 and Phase 5 (or as a wrapper around Phases 4-6).

---

### Multi-Persona Orchestration

Add instructions for running N personas sequentially. This wraps Phases 4→5→6 in a loop.

**Instructions to write**:

1. After Phase 3 produces the PersonaTaskMatrix, process each persona one at a time:
   ```
   For persona 1..N:
     a. Run Phase 4 (Browser Session) for this persona
     b. Run Phase 5 (Journey Reconstruction) for this persona
     c. Run Phase 6 (Feedback Synthesis) for this persona
     d. Report progress: "✅ Persona {N}/{total}: {name} ({archetype}) — Task: {result}"
     e. Extract covered_issues from this persona's feedback
     f. Pass covered_issues to the NEXT persona's Phase 4 (cumulative diversity)
   ```

2. **Error isolation**: If a persona's browser session fails (page crash, CAPTCHA mid-session, etc.):
   - Log the failure
   - Skip to the next persona
   - Do NOT abort the entire run
   - Note the failed persona in the final report

3. **Progress output** (show after each persona completes):
   ```
   Persona 1/10: Maria Chen (first_timer, mobile)
     ├── Task: "Create invoice for $500" → ❌ FAILED
     ├── Issues found: 3 (1 critical)
     └── Emotional arc: confident → confused → frustrated

   Persona 2/10: James Wright (power_user, desktop)
     ├── Task: "Create invoice in <5 steps" → ✅ COMPLETED
     ├── Issues found: 2 (0 critical)
     └── Emotional arc: confident → neutral → confident
   ```

4. **Cumulative diversity**: Each persona receives the previous personas' issues as "already covered." This is critical — without it, 10 personas report the same 3 issues.

---

### Phase 5: Journey Reconstruction

**Input**: EnhancedActionLog (from Phase 4) for one persona
**Output**: JourneyNarrative

**Purpose**: Turn a flat action log into a STORY with emotional arc and key moments. This is what makes the report engaging — founders empathize with stories, not bullet lists.

**Instructions to write**:

1. **Identify key moments** from the action log:
   - `positive`: Something worked well or surprised the persona positively
   - `confusion`: emotional_state changed to "confused" or "frustrated"
   - `recovery`: confusion resolved, persona got back on track
   - `abandonment`: persona gave up (if applicable)
   - `delight`: an "aha" moment where something clicked

2. **Build the confidence curve**: Map the emotional_state sequence to numbers:
   - confident = 8
   - neutral = 6
   - confused = 4
   - frustrated = 2
   - ready_to_leave = 1

3. **Write a first-person journey narrative** (3-5 sentences):
   - Written AS the persona, in their voice
   - References specific pages and elements
   - Includes the "I expected X but got Y" moments
   - Mentions competitor comparison if persona has competitor experience

**Output schema**:
```json
{
  "persona_id": "persona_001",
  "confidence_curve": [8, 8, 6, 4, 2, 2, 4, 6, 8],
  "key_moments": [
    {"type": "positive", "step": 1, "description": "Landing page clearly explains the product"},
    {"type": "confusion", "step": 7, "description": "'+' button opened settings, not new invoice"},
    {"type": "recovery", "step": 10, "description": "Found create action through search"}
  ],
  "narrative": "I opened the site on my phone and immediately understood what it does — 'Send professional invoices in 30 seconds.' The signup was quick, just email and password. But the dashboard threw me — where do I create an invoice? The '+' icon opened settings, not creation. On Wave, there's a big green 'Create Invoice' button right there. I eventually found it through the search bar, but it shouldn't be that hard.",
  "journey_summary": "Good first impression → confusion at core task → partial recovery via search"
}
```

---

### Phase 7: Aggregation + Scoring

**Input**: All PersonaFeedback[] + all JourneyNarrative[]
**Output**: ComprehensiveReport

**This is where the magic happens — individual feedback becomes cross-persona intelligence.**

**Instructions to write**:

#### Step 1: Issue Deduplication

Collect all issues from all PersonaFeedback. Group semantically identical issues:
- "Can't find create button" and "Invoice creation is hidden" → same issue
- Keep the best evidence and clearest description from across personas
- Count how many personas independently reported each issue

#### Step 2: Severity Scoring

For each unique issue, calculate a weighted severity:
```
severity_score = base_severity × (1 + log2(affected_personas))

base_severity: critical=4, high=3, medium=2, low=1
```

Example: A "high" issue affecting 8/10 personas:
```
3 × (1 + log2(8)) = 3 × 4 = 12
```

Sort all issues by severity_score descending.

#### Step 3: Product Score (THE key deliverable)

Calculate 6 dimension scores (each 1-10, averaged across personas):

| Dimension | Source | Weight |
|-----------|--------|--------|
| First Impression | avg(persona.scores.first_impression) | 0.15 |
| Task Completion | (completed_personas / total_personas) × 10 | 0.30 |
| Navigation | avg(persona.scores.navigation) | 0.20 |
| Trust & Credibility | avg(persona.scores.trust) | 0.15 |
| Error Handling | avg(persona.scores.error_handling) | 0.10 |
| Overall NPS | avg(persona.scores.nps) | 0.10 |

```
Product Score = weighted sum of dimension scores, rounded to 1 decimal
```

#### Step 4: "If You Fix ONE Thing"

Identify the SINGLE highest-impact recommendation:
- The issue that: (a) affects the most personas, (b) has highest severity, (c) would most improve the Product Score
- Write it as ONE clear sentence
- Include WHY (how many personas affected + what impact fixing it would have)
- Include 2-3 persona quotes as evidence

#### Step 5: Task Completion Rate

```
Task Completion Rate = (COMPLETED count) / (total personas) × 100%
```

Also report: average steps to complete, average time, vs estimated optimal.

#### Step 6: Segment Analysis

Group insights by:
- **Tech level**: Do novices struggle more than experts? Where?
- **Device**: Are mobile users blocked on something desktop users aren't?
- **Archetype**: Which persona type has the worst experience?

#### Step 7: Generate the Report

Output the full report in Markdown format:

```markdown
# CrowdTest Report — {product_url}
## Score: {score}/10 — {one_line_assessment}

**Date**: {date} | **Personas**: {count} | **Pages visited**: {total_pages}
**Task Completion**: {rate}% ({completed}/{total}) | **Avg NPS**: {avg_nps}
**Cost**: ~${cost_estimate}

---

## 🎯 If You Fix One Thing

{highest_impact_recommendation}

**Why**: {N}/{total} personas were affected. {impact_description}

**Evidence**:
> "{persona_1_quote}" — {persona_1_name} ({archetype})
> "{persona_2_quote}" — {persona_2_name} ({archetype})
> "{persona_3_quote}" — {persona_3_name} ({archetype})

---

## 📊 Product Scorecard

| Dimension | Score | Assessment |
|-----------|-------|-----------|
| First Impression | {score}/10 | {brief_note} |
| Task Completion | {score}/10 | {brief_note} |
| Navigation | {score}/10 | {brief_note} |
| Trust & Credibility | {score}/10 | {brief_note} |
| Error Handling | {score}/10 | {brief_note} |
| Overall NPS | {score}/10 | {brief_note} |
| **OVERALL** | **{score}/10** | |

---

## 🔴 Critical Issues ({count})

### 1. {issue_title}
- **Severity**: Critical | **Affected**: {N}/{total} personas
- **Description**: {description}
- **Evidence**: {evidence_from_action_logs}
- **Suggested fix**: {fix}
- **Personas affected**: {names_and_archetypes}

### 2. ...

## 🟡 High Issues ({count})
## 🟠 Medium Issues ({count})
## 🟢 Low Issues ({count})

---

## 📖 Persona Journeys

### {persona_name} — {archetype} ({device})
**Task**: {task} → {result_emoji} {result}
**Confidence**: {confidence_curve_as_text_bar}
**Time**: {duration} | **Actions**: {count} | **NPS**: {nps}

> {journey_narrative}

(repeat for each persona)

---

## ✅ What's Working Well

| Feature | Mentions | Evidence |
|---------|----------|---------|
| {feature} | {N}/{total} | {quote} |

---

## 📊 Segment Insights

### By Tech Level
- **Novice users**: {finding}
- **Advanced users**: {finding}

### By Device
- **Mobile**: {finding}
- **Desktop**: {finding}

### By Archetype
- **First-Timers**: {finding}
- **Switchers**: {finding}

---

## 🔧 Recommended Fix Priority

1. {fix_1} — Impact: {estimate}
2. {fix_2} — Impact: {estimate}
3. {fix_3} — Impact: {estimate}
```

Save this report to `crowdtest-report-{domain}-{date}.md` in the current directory.

---

### Phase 8: Delta Analysis

**Input**: Current report + previous report (if exists)
**Output**: DeltaReport section appended to main report

**Instructions to write**:

1. Check if a previous CrowdTest report exists for the same URL in the current directory (glob: `crowdtest-report-{domain}-*.md`)
2. If no previous report → skip Phase 8, add note: "First run — no comparison available"
3. If previous report exists:
   - Compare overall Product Scores
   - Match issues: which old issues are resolved? Which persist? Which are new?
   - Compare Task Completion Rate
   - Compare NPS

**Output (appended to main report)**:

```markdown
## 📈 Delta from Previous Run

**Previous**: {date} | Score: {old_score}/10
**Current**: {date} | Score: {new_score}/10
**Change**: {delta} ({direction})

### ✅ Resolved Issues
- {issue_that_was_fixed}

### ⚠️ Persistent Issues
- {issue_still_present}

### 🆕 New Issues
- {issue_not_in_previous_run}

### Trends
- Task Completion: {old}% → {new}% ({delta})
- NPS: {old} → {new} ({delta})
```

---

## Acceptance Criteria

- [ ] Multi-persona orchestration wraps Phase 4→5→6 in a loop with progress output
- [ ] Cumulative diversity (covered_issues passed between personas) is documented
- [ ] Error isolation: failed persona doesn't abort the run
- [ ] Phase 5 produces JourneyNarrative with confidence_curve and key_moments
- [ ] Phase 7 calculates Product Score with 6 weighted dimensions
- [ ] Phase 7 produces "Fix ONE Thing" recommendation with evidence
- [ ] Phase 7 generates the FULL Markdown report matching the template above
- [ ] Phase 7 saves report to file
- [ ] Phase 8 compares with previous report if exists
- [ ] All output schemas match the JSON/Markdown examples
- [ ] SKILL.md is complete — all 8 phases are fully implemented

## What NOT to Do

- Do NOT modify Phase 0, 1, 2, 3, 4, or 6 (already implemented)
- Do NOT create application code
- Do NOT merge the PR
