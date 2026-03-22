# TASK.md — Current Sprint: P0 Scout + Persona Generator

> ⚠️ DO NOT MERGE. Create PR and stop. 晚晚 reviews and merges.

## Sprint Goal

Implement the first two modules of the CrowdTest pipeline:
1. **Scout** — page readiness checker
2. **Persona Generator** — dimension-matrix-driven persona creation

These are the blocking prerequisites for everything else.

## Deliverables

### Module 1: Scout

**File**: `src/scout.md` (prompt module within SKILL.md) or standalone prompt section

**Input**: URL string + (optional) storage-state path
**Output**: SiteMap JSON

**Requirements**:

1. Navigate to URL using `browser_navigate`
2. Wait for network idle (10s timeout via `browser_wait_for`)
3. Take accessibility snapshot via `browser_snapshot`
4. Run page health checks:
   - Count interactive elements (buttons, links, inputs, textboxes)
   - If < 5 → flag as `load_failed`, retry once after 3s delay
   - Detect auth patterns: look for "sign in", "log in", "create account" in snapshot
   - Detect CAPTCHA: look for "captcha", "recaptcha", "hcaptcha", "verify you are human"
   - Detect cookie banner: look for "accept cookies", "cookie preferences", "consent"
5. If cookie banner detected → attempt to dismiss (click Accept/Close/Got it)
6. If auth detected → return `status: "auth_required"` with message
7. If CAPTCHA detected → return `status: "captcha_detected"` with message
8. If page loaded successfully:
   - Identify top-level navigation links
   - Visit top 3-5 linked pages (quick snapshot each)
   - Build site_map with page types and key elements

**Output schema**:
```json
{
  "url": "https://example.com",
  "title": "Example App",
  "status": "ready",
  "pages": [
    {
      "url": "/",
      "type": "landing",
      "interactive_elements": 12,
      "key_elements": ["search bar", "sign up button", "pricing link"]
    }
  ],
  "cookie_banner": true,
  "tech_hints": ["React", "SPA"]
}
```

### Module 2: Persona Generator

**Input**: SiteMap + (optional) audience_hint string + persona_count (default 10)
**Output**: Persona[] JSON array

**Requirements**:

1. Analyze SiteMap to infer:
   - Product type (SaaS, e-commerce, blog, dashboard, etc.)
   - Target audience (developers, consumers, business users, etc.)
   - Core features (from key_elements across pages)

2. Build dimension matrix:
   - `tech_level`: novice / intermediate / advanced / developer
   - `purpose`: 3-5 inferred from product type
   - `age`: sample from 18-65 range, ensuring variety
   - `personality.patience`: 0.1-0.9 range
   - `personality.exploration_tendency`: 0.1-0.9 range
   - `personality.attention_to_detail`: 0.1-0.9 range
   - `device`: desktop (60%) / mobile (30%) / tablet (10%)

3. Generate personas one at a time:
   - Each generation call includes summaries of ALL previously generated personas
   - Instruction: "Create a persona that FILLS GAPS in this collection"
   - Must differ from existing personas on ≥ 2 dimensions

4. For each persona, generate behavioral_rules (NOT personality adjectives):
   - At least 3 rules per persona
   - Rules must be IF-THEN format, testable
   - Example: "If you encounter a form with more than 5 fields, skip optional fields"
   - Example: "After 3 failed attempts, express frustration and try a completely different approach"

5. Generate narrative paragraph for each persona (used as system prompt during browsing)

**Output schema** (per persona):
```json
{
  "id": "persona_001",
  "name": "Maria Chen",
  "tech_level": "intermediate",
  "purpose": "compare pricing plans for her team",
  "age": 34,
  "industry": "healthcare",
  "personality": {
    "patience": 0.3,
    "exploration_tendency": 0.8,
    "attention_to_detail": 0.6
  },
  "behavioral_rules": [
    "After 2 clicks without finding what you want, express frustration and try search if available",
    "Skip any page section that requires scrolling more than twice",
    "If a form has more than 5 fields, only fill required ones"
  ],
  "device": "mobile",
  "narrative": "You are Maria Chen, a 34-year-old nurse practitioner who manages a small clinic..."
}
```

## Integration

Both modules will be orchestrated by the main SKILL.md prompt. For now, implement them as clearly separated prompt sections with:
- Input/output schemas documented
- Example invocations
- Error handling for each failure mode

## Acceptance Criteria

- [ ] Scout correctly identifies `ready` / `auth_required` / `captcha_detected` / `load_failed` on test URLs
- [ ] Scout dismisses cookie banners on common sites
- [ ] Scout builds a site_map with ≥ 3 pages for multi-page sites
- [ ] Persona generator produces N unique personas with valid JSON
- [ ] Each persona has ≥ 3 behavioral rules in IF-THEN format
- [ ] No two personas share the same (tech_level, purpose, device) combination
- [ ] Personas cover at least 3 different tech_levels and 3 different age ranges
- [ ] All outputs are valid JSON matching the schemas above

## References

- Architecture: `docs/technical-design.md` (sections 3.1, 3.2)
- Prompt designs: `docs/technical-design.md` (section 4)
- Data structures: `docs/technical-design.md` (section 2)
- Competitive insights: `docs/competitive-research.md` (Channel-Labs diversity pattern, PersonaHub format)
