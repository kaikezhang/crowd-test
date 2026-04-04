---
name: crowd-test
description: "Generate diverse AI personas to browse-test your web product and find UX issues you can't see. Use when you need user testing but have no users, no budget, and no time. One URL in, prioritized issue report out."
---

# CrowdTest — AI Synthetic User Testing

> One URL. N personas. Grounded UX feedback. Zero real users needed.

## Prerequisites

- **Sub-agent dispatch capability** — at least one of:
  - OpenClaw with `sessions_spawn` (recommended)
  - `codex` CLI on PATH
  - `claude` CLI on PATH
- **Playwright** — sub-agents need browser automation:
  - Playwright MCP server (`@playwright/mcp`), or
  - Playwright installed in the worktree (`npx playwright install chromium`)
- Claude Sonnet 4.6+ (or Opus 4.6 for higher quality)

## Quick Start

```
/crowd-test https://myapp.com
```

Or invoke naturally:
```
Test https://myapp.com with 10 AI personas and find UX issues
```

Optional flags:
```
/crowd-test https://myapp.com --personas 5 --context "B2B invoicing tool for freelancers" --focus "onboarding flow" --competitors "FreshBooks, Wave"
```


## How It Works

CrowdTest runs a 9-phase pipeline (Phase 0-8). Each phase feeds the next:

```
Phase 1: Scout             → Is the site testable? Build a site map.
Phase 0: Product Analysis  → What IS this product? Who's it for?
Phase 2: Persona Generation → Who would use this? (derived from the product)
Phase 3: Task Assignment    → What should each persona try to do?
Phase 4-6: Sub-Agent E2E    → Dispatch agent per persona: browse, journal, feedback
Phase 7: Aggregation        → Product Score (X/10) + "Fix ONE Thing" report
Phase 8: Delta Analysis     → Compare with previous run if exists
```

**Execution order**: Phase 1 runs first (Scout needs raw browser data), then Phase 0 analyzes what Scout found. The numbering reflects conceptual priority — understanding the product (Phase 0) is foundational — but Scout must run first to gather the data.

**Execution model**: The main agent (orchestrator) runs Phases 0-3 and 7-8 directly. For Phases 4-6, the main agent dispatches one **independent sub-agent per persona**, each with its own browser. Sub-agents run serially — one persona at a time. Each sub-agent performs real E2E browser testing via Playwright and writes structured JSON output.

---

## Phase 1: Scout

> **Purpose**: Verify the target URL is testable and build a map of what's there. Without this step, ~30% of runs produce garbage because the page didn't load, requires auth, or has a CAPTCHA.

> **Input**: Target URL from the user
> **Output**: SiteMap JSON
> **Tools used**: `browser_navigate`, `browser_snapshot`, `browser_click`

### Step 1: Navigate to the target URL

Call `browser_navigate` with the user-provided URL.

- If the URL does not include a protocol, prepend `https://`.
- If `--storage-state` was provided, load the browser storage state first before navigating.

### Step 2: Wait for page load

Wait for the page to finish loading. Use a 10-second timeout for network idle.

**If navigation fails** (timeout, DNS error, connection refused):
1. Wait 3 seconds, then retry navigation once.
2. If retry also fails, produce the SiteMap with `status: "load_failed"` and stop:

```json
{
  "url": "https://example.com",
  "title": "",
  "status": "load_failed",
  "failure_reason": "Connection refused — is the server running?",
  "pages": [],
  "cookie_banner": false
}
```

Report to the user: "Could not reach {url}. Check that the URL is correct and the server is running. If testing localhost, make sure your dev server is active."

**STOP the pipeline here. Do not proceed to Phase 0 or any later phase.**

### Step 3: Take an accessibility snapshot

Call `browser_snapshot` to capture the page's accessibility tree. This is your primary observation tool — NOT screenshots. Accessibility snapshots are deterministic, token-efficient (~200-500 tokens vs 1500+ for screenshots), and capture the semantic structure of the page.

### Step 4: Run health checks on the snapshot

Analyze the accessibility snapshot for the following conditions, **in this order**:

#### 4a: Check interactive element count

Count all interactive elements in the snapshot: buttons, links (`<a>`), inputs, textboxes, selects, checkboxes, radio buttons, and other form controls.

- If **fewer than 5 interactive elements**: the page may not have fully loaded (common with SPAs that hydrate slowly).
  - Wait 3 seconds.
  - Call `browser_snapshot` again.
  - Re-count interactive elements.
  - If **still fewer than 5**: proceed anyway but add a note: `"warning": "Low interactive element count — page may not have fully loaded"` to the SiteMap.

#### 4b: Detect cookie/consent banner

Search the snapshot for any of these patterns (case-insensitive):
- "accept cookies", "accept all", "cookie", "consent", "cookie preferences", "cookie policy", "we use cookies", "got it", "I agree", "I understand"

If a cookie/consent banner is detected:
1. Find the **accept/dismiss button**. Look for elements with text matching: "Accept", "Accept All", "Accept Cookies", "Got it", "I Agree", "OK", "Close", "Dismiss", "I understand".
2. Call `browser_click` on that element.
3. Wait 1 second for the banner to dismiss.
4. Call `browser_snapshot` again to get a clean snapshot without the banner.
5. Set `cookie_banner: true` in the SiteMap output.

If you cannot find a clear accept/dismiss button, skip dismissal and note: `"cookie_banner_note": "Banner detected but could not find dismiss button"`.

#### 4c: Detect CAPTCHA

Search the snapshot for any of these patterns (case-insensitive):
- "captcha", "recaptcha", "hcaptcha", "verify you are human", "i'm not a robot", "prove you are human", "security check", "challenge"

If CAPTCHA is detected, produce the SiteMap with `status: "captcha_detected"` and stop:

```json
{
  "url": "https://example.com",
  "title": "Example App",
  "status": "captcha_detected",
  "failure_reason": "CAPTCHA detected — CrowdTest cannot bypass CAPTCHAs",
  "pages": [],
  "cookie_banner": false
}
```

Report to the user: "CAPTCHA detected on {url}. CrowdTest cannot bypass CAPTCHAs. Suggestions: (1) Test on a staging environment with CAPTCHA disabled, (2) Provide pre-authenticated cookies via `--storage-state`."

**STOP the pipeline here.**

#### 4d: Detect authentication wall

Search the snapshot for elements suggesting the page requires login. Look for **combinations** of these patterns (case-insensitive) that suggest the page ITSELF is an auth wall, not merely that a login option exists in the nav:

**Strong auth signals** (any one of these = auth wall):
- A prominent form containing BOTH a password field AND a username/email field
- "sign in to continue", "log in to continue", "please log in", "login required"
- The page contains ONLY a login/signup form with no other product content

**Weak auth signals** (need 2+ to flag as auth wall):
- "sign in", "log in", "sign up", "create account", "register"
- A password input field
- OAuth buttons ("Sign in with Google", "Continue with GitHub")

**NOT auth signals** (do not flag these):
- A "Login" or "Sign in" link in the navigation bar (this is normal for any site)
- "Sign up for free" as a CTA on a landing page with other content visible

If an auth wall is detected, produce the SiteMap with `status: "auth_required"` and stop:

```json
{
  "url": "https://example.com",
  "title": "Example App",
  "status": "auth_required",
  "auth_type": "login_form",
  "failure_reason": "Page requires authentication to access content",
  "pages": [],
  "cookie_banner": false
}
```

Report to the user: "Authentication required on {url}. To test authenticated pages, provide a browser storage state file: `/crowd-test https://example.com --storage-state ./cookies.json`. You can export cookies from your browser using Playwright's `storageState` API."

**STOP the pipeline here.**

### Step 5: Build the site map

If all health checks pass, the page is ready. Now build a map of the site.

#### 5a: Analyze the landing page

From the current accessibility snapshot, extract:
- **Page title**: from the page's `<title>` or main heading
- **Page type**: classify as one of: `landing`, `form`, `dashboard`, `content`, `pricing`, `features`, `documentation`, `other`
- **Interactive elements count**: total buttons, links, inputs, etc.
- **Key elements**: list the 3-7 most important interactive elements and content sections. Be specific — use the actual text from the snapshot. Examples: `"hero CTA 'Get Started Free'"`, `"nav: Features, Pricing, Login"`, `"search bar in header"`, `"testimonials carousel"`.

Add this as the first entry in the `pages` array.

#### 5b: Identify navigation links

From the snapshot, identify **top-level navigation links** — these are typically in a `<nav>` element, header, or sidebar. Focus on links that lead to distinct pages of the product (not external links, social media, or anchor links on the same page).

Select the **top 3-5 most important** navigation links to visit. Prioritize:
1. Core product pages (Features, Pricing, How it Works)
2. Pages that reveal the product's functionality (Dashboard, Demo, Docs)
3. Conversion pages (Signup, Get Started, Contact)

Skip: Blog, About Us, Terms of Service, Privacy Policy, social media links.

#### 5c: Visit each linked page

For each selected navigation link (up to 5):

1. Call `browser_navigate` with the link URL (resolve relative URLs against the base URL).
2. Wait for page load (5-second timeout — shorter than initial load since the site is already proven reachable).
3. Call `browser_snapshot`.
4. If navigation fails or times out, skip this page and note it as `"status": "unreachable"` — do NOT retry subpages.
5. From the snapshot, extract the same data as step 5a: page type, interactive element count, key elements.
6. Add to the `pages` array.

#### 5d: Identify the critical path

Based on all visited pages, identify the **critical path** — the sequence of pages a user would follow to complete the product's core action. Examples:
- E-commerce: `["/", "/products", "/product/123", "/cart", "/checkout"]`
- SaaS: `["/", "/signup", "/onboarding", "/dashboard"]`
- Content: `["/", "/articles", "/article/123"]`

The critical path should be 2-5 pages. If you can't determine a clear critical path, list the most likely user flow based on the CTAs and navigation structure.

#### 5e: Navigate back to the landing page

After visiting subpages, call `browser_navigate` to return to the original URL. This ensures the browser is in a clean state for the next phase.

### Output: SiteMap

Produce the following JSON as your structured output. This is consumed by Phase 0 (Product Analysis).

```json
{
  "url": "https://example.com",
  "title": "InvoiceNinja — Send Professional Invoices",
  "status": "ready",
  "pages": [
    {
      "url": "/",
      "type": "landing",
      "interactive_elements": 15,
      "key_elements": ["hero CTA 'Get Started Free'", "nav: Features, Pricing, Login", "testimonials section", "footer with social links"]
    },
    {
      "url": "/pricing",
      "type": "pricing",
      "interactive_elements": 8,
      "key_elements": ["3 plan cards", "annual/monthly toggle", "CTA 'Start Free Trial' per plan"]
    },
    {
      "url": "/features",
      "type": "features",
      "interactive_elements": 12,
      "key_elements": ["feature grid with icons", "demo video embed", "integration logos section"]
    }
  ],
  "critical_path": ["/", "/signup", "/dashboard"],
  "cookie_banner": true,
  "responsive_notes": "Nav collapses to hamburger on mobile viewport"
}
```

**Status values**:
- `"ready"` — page loaded, no blockers, site map built
- `"auth_required"` — page requires login (pipeline stops)
- `"captcha_detected"` — CAPTCHA blocks access (pipeline stops)
- `"load_failed"` — page could not be reached (pipeline stops)

If status is NOT `"ready"`, the pipeline **stops here**. Report the issue to the user with actionable instructions.

---

## Phase 0: Product Analysis

> **Purpose**: Understand what the product IS before generating personas. This is the difference between "AI randomly browses a website" and "freelancer tries to send an invoice using InvoiceNinja." Without this phase, personas are random and feedback is generic.

> **Input**: SiteMap from Phase 1 (specifically the accessibility snapshots and page data)
> **Output**: ProductProfile JSON
> **Tools used**: None (pure LLM reasoning over Scout data)

### Step 1: Extract product signals from the Scout data

Using the SiteMap and the accessibility snapshots collected during Phase 1, read and analyze:

- **Landing page headline and subheadline**: What does the product promise?
- **CTA button text**: What action does it want users to take? ("Get Started Free", "Book a Demo", "Buy Now")
- **Navigation labels**: What sections does the product think are important?
- **Pricing page** (if visited): What do the plan names, prices, and feature lists tell you about the target customer?
- **Features page** (if visited): What capabilities does the product offer?
- **Any audience-specific language**: "For freelancers", "Enterprise-grade", "For teams of all sizes"
- **Social proof**: Testimonials, logos, case studies — who are the existing users?

### Step 2: Answer the product analysis questions

Based on the signals extracted in Step 1, answer each of these questions explicitly. Do NOT guess or hallucinate — if information is not visible on the pages visited, say "Not visible from public pages."

1. **What does this product do?** Write ONE sentence. Be specific — "Helps freelancers create and send professional invoices" not "A business tool."

2. **Category**: Classify as one of:
   - `SaaS tool` (subscription software)
   - `e-commerce` (sells physical/digital products)
   - `content site` (blog, news, documentation)
   - `dev tool` (developer-facing product)
   - `social app` (social network, community)
   - `marketplace` (connects buyers and sellers)
   - `other` (specify)

   Add a subcategory after a dash: e.g., `"SaaS — invoicing & billing"`, `"e-commerce — fashion"`, `"dev tool — CI/CD"`.

3. **Intended user**: Who is the copy, pricing, and feature set aimed at? Be specific — "Freelancers and small business owners who need to invoice clients" not "businesses."

4. **Core tasks**: List 2-3 main things a user should be able to DO with this product. These must be actionable verbs: "Create and send an invoice", "Track payment status", "Manage client contacts." Derive these from the features page, CTAs, and navigation structure.

5. **Competitors**: What does this product resemble? Infer from the category, features, and positioning. List 2-4 competitors. If unsure, write "Unknown — insufficient signals."

6. **Maturity**: Classify as one of:
   - `landing_page_only` — just a marketing page, no functional product visible
   - `mvp` — basic functionality visible, limited features
   - `full_product` — comprehensive feature set, polished UI

7. **Pricing model**: Classify as one of:
   - `free` — entirely free, no paid tier visible
   - `freemium` — free tier with paid upgrades
   - `paid` — requires payment to use
   - `not_visible` — pricing not found on public pages

8. **Language**: Primary language of the site content (e.g., `"en"`, `"zh"`, `"es"`).

9. **Audience signals**: List 2-4 specific observations from the page content that support your "intended user" answer. Quote actual text where possible. Examples:
   - `"$0 base plan suggests indie/freelancer target"`
   - `"'For freelancers & small businesses' in footer"`
   - `"Enterprise features like SSO and audit logs in highest tier"`

### Step 3: Merge user-provided context (if any)

If the user provided any of these flags, merge their input with your analysis:

- `--context "description"`: Use this to refine your `value_proposition` and `intended_user`. The user knows their product better than you do — their context takes priority over your inferences.
- `--focus "area"`: Note this as a focus area. It will influence persona generation and task assignment in later phases.
- `--competitors "Competitor1, Competitor2"`: Replace your inferred competitors with the user's list.

If the user did NOT provide these flags, rely entirely on what you observed from the pages.

### Output: ProductProfile

Produce the following JSON. Every field must be populated — use "Not visible from public pages" for fields where information was genuinely unavailable.

```json
{
  "name": "InvoiceNinja",
  "url": "https://invoiceninja.com",
  "category": "SaaS — invoicing & billing",
  "value_proposition": "Send professional invoices in 30 seconds",
  "intended_user": "Freelancers and small businesses who need to invoice clients",
  "core_tasks": [
    "Create and send an invoice",
    "Track payment status",
    "Manage client contacts"
  ],
  "competitors": ["FreshBooks", "Wave", "Zoho Invoice"],
  "maturity": "full_product",
  "pricing_model": "freemium",
  "language": "en",
  "audience_signals": [
    "$0 base plan suggests indie/freelancer target",
    "No enterprise features visible",
    "'For freelancers & small businesses' in footer"
  ]
}
```

### Validation

Before proceeding to Phase 2, verify the ProductProfile:

1. **Is `value_proposition` specific?** "A great tool" = FAIL. "Send professional invoices in 30 seconds" = PASS.
2. **Are `core_tasks` actionable?** Each must start with a verb. "Invoicing" = FAIL. "Create and send an invoice" = PASS.
3. **Is `intended_user` specific?** "Everyone" or "businesses" = FAIL. "Freelancers and small business owners who invoice clients" = PASS.

If any check fails, revise the field before proceeding.

### How ProductProfile feeds downstream phases

The ProductProfile is critical context for all subsequent phases:
- **Phase 2 (Persona Generation)**: Personas are derived FROM the product's intended user, not randomly generated. A SaaS invoicing tool gets a freelancer persona, not a teenager browsing for fun.
- **Phase 3 (Task Assignment)**: Core tasks become the basis for what personas try to accomplish.
- **Phase 7 (Aggregation)**: The Product Score is evaluated against the product's own value proposition — does it deliver what it promises?

---

## Phase 2: Persona Generation

> **Purpose**: Generate diverse, product-derived personas that test specific aspects of the user experience. Personas are NOT random — they are derived FROM the ProductProfile's intended user, core tasks, and competitors.

> **Input**: ProductProfile (from Phase 0) + SiteMap (from Phase 1)
> **Output**: PersonaSet (array of Persona JSON objects)
> **Tools used**: None (pure LLM reasoning)

### Step 1: Derive persona archetypes from the ProductProfile

Analyze the ProductProfile and identify which of these 5 archetypes are relevant to THIS product. All 5 should be used for the default 10-persona run (2 personas per archetype):

| Archetype | What It Tests | How to Derive |
|-----------|--------------|---------------|
| **The First-Timer** | Onboarding clarity, time-to-value | Someone who matches `intended_user` but has never used this product category. They found it via Google search. |
| **The Switcher** | Feature parity, migration friction | Someone coming from a product in `competitors`. They already know how competitor X works and expect similar patterns. |
| **The Power User** | Efficiency, shortcuts, advanced features | Someone experienced with this product category. They want to complete `core_tasks` as fast as possible. |
| **The Skeptic** | Trust signals, security, social proof, pricing transparency | Someone evaluating the product but not yet convinced. They look for red flags before committing. |
| **The Edge Case** | Robustness, accessibility, unusual contexts | Someone with an unusual context: mobile-only, international user, accessibility needs, extremely rushed, or unusual input data. |

For each archetype, think about what SPECIFIC person would use THIS product. Use the ProductProfile fields:
- `intended_user` → demographics and industry
- `core_tasks` → what they want to accomplish
- `competitors` → what Switchers are coming from
- `category` → what Power Users already know
- `pricing_model` → what Skeptics will scrutinize

### Step 2: Generate personas one at a time

Generate personas **sequentially**, one at a time. For the default count of 10, generate 2 per archetype.

For each persona, you MUST include ALL of the following fields:

#### 2a: Demographics (tied to ProductProfile.intended_user)

- `name`: A realistic full name. Vary gender, ethnicity, and cultural background across the set.
- `age`: Between 18-65+. Ensure at least 3 different age ranges across the full set (e.g., 20s, 30s-40s, 50s+).
- `industry`: Derived from ProductProfile.intended_user — what industry would this person work in?
- `tech_level`: One of `novice`, `intermediate`, `advanced`, `developer`. Distribute across the set.

#### 2b: Purpose (derived from ProductProfile.core_tasks)

- `purpose`: A specific goal derived from one of the ProductProfile's `core_tasks`. Each persona should have a DIFFERENT purpose where possible. Frame it from the persona's perspective: "Send first invoice to a client" not "Test invoice creation."

#### 2c: Entry point (not everyone starts at homepage)

Assign entry points following this distribution across the full persona set:
- **6 out of 10**: Homepage `/` (organic search, direct visit)
- **2 out of 10**: Pricing or features page (comparison shopping — use a URL from SiteMap.pages)
- **1 out of 10**: Signup page (direct referral from a friend — use signup URL from SiteMap if available, otherwise `/signup` or `/register`)
- **1 out of 10**: A content/docs page (if one exists in SiteMap.pages, otherwise use homepage)

Use actual URLs from the SiteMap where possible.

#### 2d: Emotional context (why they're here NOW)

Write a 1-2 sentence scenario explaining why this persona is visiting the product RIGHT NOW. This creates urgency and motivation that drives realistic browsing behavior. Examples:
- "Just got first client, needs to invoice them in 30 minutes before a meeting."
- "Evaluating 3 tools today, will spend exactly 2 minutes on each before deciding."
- "Manager told them to check this out, not personally motivated — doing it because they have to."
- "Competitor just raised prices, looking for alternatives urgently."
- "Saw a tweet about this product, curious but skeptical — will leave at first red flag."

#### 2e: Behavioral rules (3-5 hard IF-THEN constraints)

**CRITICAL**: Do NOT write personality adjectives like "impatient" or "detail-oriented." Write executable IF-THEN constraints that an LLM can follow during browsing:

```
GOOD (executable):
"Skip any text longer than 2 sentences without reading it"
"If you can't find what you need in 3 clicks, use search if available"
"After 2 failed actions, express frustration out loud in your thinking"
"Ignore anything labeled 'enterprise' or 'team plan'"
"If a form has more than 5 fields, only fill the ones marked required"
"After 5 total failures, abandon the product entirely and mark task as FAILED"
"Always check pricing before signing up"
"If you see a competitor comparison, read it carefully"

BAD (not executable):
"You are impatient" — LLMs can't "be impatient"
"You are detail-oriented" — too vague to act on
"You like clean design" — not actionable
```

Each persona MUST have 3-5 behavioral rules. Rules should be consistent with the persona's archetype:
- First-Timers: rules about confusion thresholds, help-seeking behavior
- Switchers: rules about comparing to their previous tool
- Power Users: rules about efficiency, keyboard shortcuts, skipping tutorials
- Skeptics: rules about trust signals, privacy policy, pricing scrutiny
- Edge Cases: rules about their specific constraint (mobile, accessibility, etc.)

#### 2f: Personality traits (numeric)

```json
"personality": {
  "patience": 0.1-0.9,
  "exploration_tendency": 0.1-0.9,
  "attention_to_detail": 0.1-0.9
}
```

These numeric traits inform behavioral rules but are NOT used directly during browsing. They serve as metadata for aggregation analysis.

#### 2g: Device

Assign across the full persona set following this distribution:
- **6 out of 10**: `desktop`
- **3 out of 10**: `mobile`
- **1 out of 10**: `tablet`

Edge Case personas should lean toward mobile or tablet to test responsive design.

#### 2h: Competitors used (for Switcher archetypes)

For Switcher personas, populate `competitors_used` with 1-2 products from ProductProfile.competitors. For other archetypes, use an empty array `[]`.

#### 2i: Emotional state (starting state)

Set the persona's initial emotional state based on their context:
- `confident` — knows what they want, expects to find it
- `neutral` — no strong feelings, open to exploration
- `mildly_stressed` — has time pressure or frustration from elsewhere
- `skeptical` — expects to be disappointed

#### 2j: Narrative paragraph

Write a 3-5 sentence narrative in SECOND PERSON that will be used as the system prompt during the browser session. This paragraph must:
- Address the persona as "You are..."
- Include their name, age, background
- Include their specific purpose and emotional context
- Reference their behavioral tendencies (without using the IF-THEN rules verbatim — those are injected separately)
- Be vivid enough that an LLM can "stay in character"

Example:
```
"You are Maria Chen, a 34-year-old nurse practitioner who just started freelance health consulting on the side. You landed your first client yesterday and need to send them a professional invoice for $500 before your hospital shift starts in 30 minutes. You've never used invoicing software — you've been using Word templates until now. You're slightly stressed about the time pressure but excited about your new side business. You tend to rush through things and get frustrated when software doesn't have obvious buttons for what you need."
```

### Step 3: Cumulative diversity check

After generating each persona, compare it against ALL previously generated personas. The new persona MUST differ from every existing persona on **at least 2** of these dimensions:
- `tech_level`
- `purpose`
- `device`
- `archetype`
- `age` (different decade counts as different)
- `emotional_state`

If a newly generated persona is too similar, revise it before adding to the set.

### Step 4: Final diversity validation

After all personas are generated, verify:
1. **No duplicate combinations**: No two personas share the same `(tech_level, purpose, device)` tuple.
2. **Age diversity**: At least 3 different age decades are represented.
3. **Device distribution**: Roughly matches 60% desktop / 30% mobile / 10% tablet (±1 persona).
4. **Entry point distribution**: Roughly matches 60% homepage / 20% pricing-features / 10% signup / 10% content (±1 persona).
5. **All 5 archetypes represented**: Each archetype has at least 1 persona (2 for default 10-persona runs).

If any check fails, revise the persona set before proceeding to Phase 3.

### Output: Persona

For each persona, produce the following JSON:

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

The full output is a `PersonaSet` — an array of all Persona objects:

```json
{
  "personas": [ /* array of Persona objects */ ],
  "diversity_check": {
    "archetypes_covered": ["first_timer", "switcher", "power_user", "skeptic", "edge_case"],
    "tech_levels_used": ["novice", "intermediate", "advanced", "developer"],
    "devices": { "desktop": 6, "mobile": 3, "tablet": 1 },
    "age_ranges": ["20s", "30s", "50s", "60s"],
    "entry_points": { "/": 6, "/pricing": 2, "/signup": 1, "/docs": 1 }
  }
}

---

## Phase 3: Task Assignment

> **Purpose**: Assign each persona a specific, measurable task with clear success criteria. Generic tasks like "explore the site" produce generic feedback. Specific tasks like "create an invoice for $500 to Acme Corp" produce actionable insights.

> **Input**: PersonaSet (from Phase 2) + ProductProfile (from Phase 0)
> **Output**: PersonaTaskMatrix (array of TaskAssignment objects, one per persona)
> **Tools used**: None (pure LLM reasoning)

### Step 1: Derive primary tasks from ProductProfile.core_tasks

For each persona, select and customize a task from the ProductProfile's `core_tasks` list, filtered by the persona's archetype and purpose:

| Archetype | Task Template | Example |
|-----------|--------------|---------|
| **First-Timer** | "Sign up and {core_task_1}" | "Sign up and create your first invoice for $500 to 'Acme Corp'" |
| **Switcher** | "{core_task_1} and compare the experience to {competitor}" | "Create an invoice and note how it compares to FreshBooks" |
| **Power User** | "Complete {core_task_1} in the fewest steps possible" | "Create and send an invoice using the fastest path available" |
| **Skeptic** | "Find the privacy policy, pricing details, and a reason to trust this product" | "Determine if this product is trustworthy enough to enter payment info" |
| **Edge Case** | A task that tests a boundary condition | "Create an invoice on mobile with a long company name and special characters" |

**Make tasks CONCRETE**: Include specific values, names, amounts where applicable. "Create an invoice" is too vague. "Create an invoice for $500 to 'Acme Corp' with line item 'Consulting — March 2025'" is testable.

### Step 2: Define success criteria

For each task, write explicit criteria for three outcomes:

- **COMPLETED**: The task was fully accomplished. Be specific about what "done" looks like.
  - Example: "Invoice created with correct amount ($500), correct recipient (Acme Corp), and 'send' action completed"
- **PARTIALLY_COMPLETED**: The persona made meaningful progress but didn't finish.
  - Example: "Reached the invoice creation form but couldn't submit due to a required field issue"
- **FAILED**: The persona could not accomplish the task.
  - Example: "Never found the invoice creation feature after exhausting available navigation"

### Step 3: Assign secondary tasks (optional)

Each persona MAY have 1-2 secondary tasks — smaller things to check along the way. These are lower priority and should not distract from the primary task. Examples:
- "Check if mobile experience works for this flow"
- "Find the pricing page and understand what's free vs paid"
- "Look for a help/support option if you get stuck"
- "Note if there are any accessibility issues"

Secondary tasks should be relevant to the persona's archetype:
- Skeptics: "Find cancellation/refund policy"
- Edge Cases: "Check if the interface works with your constraint"
- Switchers: "Find an import/migration feature"

### Step 4: Set constraints

Each task assignment gets:
- `max_actions`: 20 (hard limit — circuit breaker)
- `max_time`: "3 minutes" (real users form opinions fast)

These are NOT configurable per persona — they are universal circuit breakers.

### Output: TaskAssignment

For each persona, produce the following JSON:

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

The full output is a `PersonaTaskMatrix` — an array of all TaskAssignment objects:

```json
{
  "tasks": [ /* array of TaskAssignment objects, one per persona */ ]
}
```

### Validation

Before proceeding to Phase 4, verify:
1. **Every persona has a task**: `tasks.length === personas.length`
2. **Every task is specific**: Contains at least one concrete value (a name, number, or specific action). "Explore the product" = FAIL. "Create an invoice for $500" = PASS.
3. **Every task has success criteria**: All three outcomes (COMPLETED, PARTIALLY_COMPLETED, FAILED) are defined.
4. **Tasks vary**: Not all personas are doing the same thing. At least 3 distinct primary tasks across the set.

---

## Persona Test Package

> **Purpose**: The main agent does NOT directly play every persona. Instead, it prepares a self-contained test package for a sub-agent, dispatches that sub-agent into an isolated worktree, and has the sub-agent perform real E2E testing with a fresh browser context.

For each persona, the main agent MUST write a `PERSONA-TASK.md` file in the persona worktree containing all required context.

### `PERSONA-TASK.md` Template

```markdown
# Persona Test Instructions

## Your Identity
- Name: {persona.name}
- Archetype: {persona.archetype}
- Age: {persona.age}
- Industry: {persona.industry}
- Tech level: {persona.tech_level}
- Device: {persona.device}
- Starting emotional state: {persona.emotional_state}

## Your Narrative
{persona.narrative}

## Your Behavioral Rules
{persona.behavioral_rules as numbered list}

## Your Primary Task
{task.primary_task}

## Success Criteria
{task.success_criteria}

## Secondary Tasks
{task.secondary_tasks}

## Target Product
- URL: {product_url}
- Entry point: {persona.entry_point}
- SiteMap: {site_map_json}
- ProductProfile summary: {product_profile_summary}

## Issues Already Found By Earlier Personas
{covered_issues summary, or "None yet"}

## Browser Instructions
1. Launch a fresh browser context.
2. Use Playwright MCP if available; otherwise use Playwright directly.
3. Set viewport by device:
   - desktop: 1920x1080
   - tablet: 768x1024
   - mobile: 375x812
4. Navigate to the entry point.
5. Run a real E2E session using ORIENT → ACT → OBSERVE → DECIDE.

## Session Rules
- Maximum 20 actions
- Maximum 3 minutes
- Stop after 5 consecutive failures
- Track emotional state after every action
- Use accessibility snapshots as the primary observation method
- If you encounter an already-known issue, you may note it, but prioritize discovering NEW issues or new evidence

## How to Test (Detailed E2E Instructions)

### State Machine
INIT → NAVIGATE → ORIENT → ACT ⟷ OBSERVE → DECIDE → DONE

### INIT
Set viewport by device type. Initialize action_count=0, consecutive_failures=0.

### NAVIGATE
Go to entry_point URL. If cookie banner appears, dismiss it. If navigation fails, retry once.

### ORIENT → ACT → OBSERVE → DECIDE (loop)
For each action:
1. **ORIENT**: Take accessibility snapshot, understand current page
2. **ACT**: Perform ONE action (click/type/scroll/navigate_back/done)
3. **OBSERVE**: Compare before/after snapshots. If action failed, try ONE recovery (scroll, parent element, keyboard, alternative path)
4. **DECIDE**: Continue or stop based on circuit breakers

After each action, report emotional state and log to action-log.

### Circuit Breakers
- Max 20 actions → force DONE
- Max 3 minutes → force DONE
- 5 consecutive failures → force DONE, mark FAILED

### Emotional States
confident(8) / neutral(6) / confused(4) / frustrated(2) / ready_to_leave(1)

When you become confused/frustrated, log a confusion_event:
{"step": N, "expected": "...", "got": "...", "gap": "..."}

## After Testing: Journey Reconstruction (journey.json)

1. **Key moments**: Tag significant events as positive/confusion/recovery/abandonment/delight
2. **Confidence curve**: Map emotional states to numbers [8,6,4,2,1] — one per action
3. **Narrative**: Write 3-5 sentences AS your persona in first person, referencing specific pages/elements
4. **Summary**: One-line arc using → arrows

## After Testing: Feedback Synthesis (feedback.json)

### Phase B: In-Character Review
Review your own action log and write honest feedback:
- **First impression** (first 3 actions only)
- **What worked well** (with step references)
- **What didn't work** (with step, page, element, expected vs actual)
- **Task completion assessment** (in your own words)

### Phase C: Structured Extraction
Extract from your narrative:
- **Issues**: tier(1-3), type, page, element, severity, evidence steps, suggested_fix
- **Positives**: feature, evidence steps, why it worked
- **Scores**: first_impression, task_completion, navigation, trust, error_handling, nps (0-10)
- **Verdict**: would_pay, would_return, one_line_verdict

### Canary Self-Validation
Before saving feedback.json:
1. Could any issue apply to ANY product? If yes → rewrite with specific evidence
2. Does every issue reference action log steps? If no → add evidence
3. Does one_line_verdict mention this specific product? If no → rewrite

## Required Output Files
Write these files in the current worktree before exiting:
1. `action-log.json`
2. `journey.json`
3. `feedback.json`
```

### Required Output Schemas

**`action-log.json`**
```json
{
  "persona_id": "persona_001",
  "entry_point": "/",
  "device": "mobile",
  "status": "COMPLETED|PARTIALLY_COMPLETED|FAILED|ERROR",
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
  "emotional_arc": ["confident", "neutral", "confused"],
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

**`journey.json`**
```json
{
  "persona_id": "persona_001",
  "confidence_curve": [8, 8, 6, 4, 2, 2, 4, 6, 8],
  "key_moments": [
    {"type": "positive", "step": 1, "description": "Landing page clearly explains the product"},
    {"type": "confusion", "step": 7, "description": "'+' button opened settings, not new invoice"},
    {"type": "recovery", "step": 10, "description": "Found create action through search"}
  ],
  "narrative": "I opened the site on my phone and immediately understood what it does...",
  "journey_summary": "Good first impression → confusion at core task → partial recovery via search"
}
```

**`feedback.json`**
```json
{
  "persona_id": "persona_001",
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
      "issue": "No visible 'Create Invoice' action on the dashboard.",
      "severity": "critical",
      "evidence": "Steps 7-14: exhaustive search of dashboard UI",
      "suggested_fix": "Add a prominent 'Create Invoice' button to the dashboard"
    }
  ],
  "positives": [],
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


## Multi-Persona Orchestration

> **Purpose**: The main agent runs one persona at a time by dispatching a sub-agent into an isolated git worktree. The sub-agent performs the actual E2E browser session and writes structured JSON output files. This preserves persona isolation while staying serial and resource-safe.

> **Input**: PersonaTaskMatrix + SiteMap + ProductProfile
> **Output**: All `action-log.json`, `journey.json`, and `feedback.json` outputs collected across personas
> **Execution**: Sequential — exactly one sub-agent active at a time

### Orchestrator Loop

After Phase 3 produces the PersonaTaskMatrix, process each persona one at a time:

```
Initialize:
  completed_personas = []
  failed_personas = []
  all_action_logs = []
  all_journeys = []
  all_feedback = []
  cumulative_covered_issues = []

For persona_index = 1 to N:
  1. Create isolated worktree:
     git worktree add -b crowdtest/persona-{persona_index} /tmp/crowdtest-persona-{persona_index} main

  2. Write PERSONA-TASK.md into the worktree using the template above.

  3. Dispatch ONE sub-agent using the best available mechanism:
     - OpenClaw: `sessions_spawn(... cwd=/tmp/crowdtest-persona-{persona_index})`
     - Codex CLI: `cd worktree && codex ...`
     - Claude Code CLI: `cd worktree && claude -p ...`

  4. Wait for completion. Then read:
     - action-log.json
     - journey.json
     - feedback.json

  5. If output files are missing, timeout occurs, or the sub-agent crashes:
     - mark persona as failed
     - log failure_reason
     - continue to the next persona

  6. If successful:
     - append results to all_action_logs / all_journeys / all_feedback
     - extract issue summaries into cumulative_covered_issues
     - report progress to the user

  7. Clean up the worktree:
     - git worktree remove /tmp/crowdtest-persona-{persona_index} --force
     - git branch -D crowdtest/persona-{persona_index}
```

### Sub-Agent Dispatch Rules

- Use a **fresh browser context** for each persona
- Serial only — do NOT run multiple personas in parallel
- Pass cumulative covered issues to each next persona so they prioritize unexplored paths
- If one persona fails, skip it and continue; do NOT abort the whole run

### Progress Output

After each persona completes, output:

```
Persona {persona_index}/{N}: {name} ({archetype}, {device})
  ├── Task: "{task_description}" → {result_emoji} {task_result}
  ├── Issues found: {issue_count} ({critical_count} critical)
  └── Emotional arc: {emotional_arc_summary}
```

### Failure Handling

- Timeout → mark failed, continue
- Missing output files → mark failed, continue
- Browser crash / page crash / CAPTCHA mid-session → mark failed, continue
- If **all** personas fail, stop before Phase 7 and report the site as untestable in the current environment

---

## Phase 5: Journey Reconstruction

Phase 5 is executed **inside the sub-agent session**. The sub-agent reads its own action log, reconstructs the confidence curve and key moments, and writes the result to `journey.json`.

The main agent does **not** regenerate the journey inline. It simply reads `journey.json`, validates that it exists, and includes it in Phase 7 aggregation.

---

## Phase 6: Feedback Synthesis

Phase 6 is executed **inside the sub-agent session**. The sub-agent transforms its own grounded session data into structured feedback and writes the result to `feedback.json`.

The main agent does **not** regenerate feedback inline. It reads `feedback.json`, validates that required fields are present, and passes the collected PersonaFeedback objects into Phase 7.

---

## Phase 7: Aggregation + Scoring + Report

**Input**: The main agent reads `action-log.json`, `journey.json`, and `feedback.json` from each persona's worktree (collected during the Multi-Persona Orchestration loop).

Normalize into the same three structures before aggregation:
- `EnhancedActionLog[]`
- `JourneyNarrative[]`
- `PersonaFeedback[]`

Once normalized, the aggregation, scoring, report generation, and delta analysis logic is identical.

> **Purpose**: Transform individual persona feedback into cross-persona intelligence. This phase produces the Product Score, "Fix ONE Thing" recommendation, and the full Markdown report. This is where the magic happens — individual feedback becomes actionable product insight.

> **Input**: All PersonaFeedback[] + all JourneyNarrative[] + all EnhancedActionLog[] + failed_personas[]
> **Output**: ComprehensiveReport (Markdown file)
> **Tools used**: None (pure LLM reasoning + file write)

### Step 1: Issue Deduplication

Collect ALL issues from ALL PersonaFeedback objects. Group semantically identical issues:

1. For each issue across all personas, extract: title, description, severity, page, evidence
2. Compare issues pairwise. Two issues are "the same" if they describe the same underlying problem, even with different wording:
   - "Can't find create button" and "Invoice creation is hidden" → SAME issue
   - "Signup form is confusing" and "Dashboard layout is confusing" → DIFFERENT issues
3. For each group of duplicates:
   - Keep the CLEAREST description (most specific, best evidence)
   - Merge evidence from all personas who reported it
   - Record `affected_personas`: list of persona_ids who independently found this issue
   - Record `affected_count`: number of personas affected

Output: `deduplicated_issues[]` — each with merged evidence and affected persona count.

### Step 2: Severity Scoring

For each deduplicated issue, calculate a weighted severity score:

```
severity_score = base_severity × (1 + log2(affected_count))

Where base_severity:
  critical = 4
  high     = 3
  medium   = 2
  low      = 1
```

Examples:
- A "high" issue affecting 8/10 personas: `3 × (1 + log2(8)) = 3 × 4 = 12`
- A "critical" issue affecting 1 persona: `4 × (1 + log2(1)) = 4 × 1 = 4`
- A "medium" issue affecting 4 personas: `2 × (1 + log2(4)) = 2 × 3 = 6`

Sort ALL issues by severity_score descending. This determines report order.

### Step 3: Product Score (THE key deliverable)

Calculate 6 dimension scores. Each dimension is scored 1-10, then weighted:

| Dimension | How to Calculate | Weight |
|-----------|-----------------|--------|
| First Impression | avg(persona.scores.first_impression) across all completed personas | 0.15 |
| Task Completion | (completed_personas / total_personas) × 10 | 0.30 |
| Navigation | avg(persona.scores.navigation) across all completed personas | 0.20 |
| Trust & Credibility | avg(persona.scores.trust) across all completed personas | 0.15 |
| Error Handling | avg(persona.scores.error_handling) across all completed personas | 0.10 |
| Overall NPS | avg(persona.scores.nps) across all completed personas | 0.10 |

```
Product Score = (first_impression × 0.15) + (task_completion × 0.30) + (navigation × 0.20)
              + (trust × 0.15) + (error_handling × 0.10) + (nps × 0.10)

Round to 1 decimal place.
```

Assessment labels based on score:
- 9.0-10.0: "Exceptional"
- 7.5-8.9: "Strong"
- 6.0-7.4: "Decent with notable issues"
- 4.0-5.9: "Needs significant work"
- 2.0-3.9: "Fundamentally broken"
- 0.0-1.9: "Unusable"

### Step 4: "If You Fix ONE Thing"

Identify the SINGLE highest-impact recommendation:

1. Look at the top-ranked issue by severity_score (from Step 2)
2. Consider: does fixing this issue improve Task Completion? (Task Completion has 0.30 weight — the highest)
3. The #1 fix is the issue that: (a) affects the most personas, (b) has highest severity, AND (c) would most improve the Product Score
4. Write it as ONE clear, actionable sentence
5. Include WHY: how many personas were affected + what impact fixing it would have
6. Include 2-3 direct quotes from different personas as evidence (from their journey narratives or feedback)

### Step 5: Task Completion Rate

```
Task Completion Rate = (COMPLETED count) / (total completed personas) × 100%
```

Also calculate and report:
- Average steps to complete (for personas who COMPLETED)
- Average time estimate (for personas who COMPLETED)
- Personas who PARTIALLY_COMPLETED vs FAILED (with reasons)

### Step 6: Segment Analysis

Group insights by three dimensions:

**By Tech Level** (from persona.tech_comfort_level):
- Compare average scores, completion rates, and common issues for novice vs advanced users
- Key question: "Do novices struggle more than experts? Where specifically?"

**By Device** (from persona.device):
- Compare mobile vs desktop experience
- Key question: "Are mobile users blocked on something desktop users aren't?"

**By Archetype** (from persona.archetype):
- Which persona archetype has the worst experience?
- Key question: "Which user type is most underserved?"

For each segment, write a 1-2 sentence finding with evidence.

### Step 7: Generate the Report

Output the full report in Markdown format. Use EXACTLY this template:

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

(Same format as Critical Issues)

## 🟠 Medium Issues ({count})

(Same format as Critical Issues)

## 🟢 Low Issues ({count})

(Same format as Critical Issues)

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

### Confidence Curve Text Bar

Render the confidence curve as a visual text bar for each persona:

```
Confidence: ████████░░░░████░░████████  (8→4→2→4→8)
```

Map values: 8=██, 6=█░, 4=░░, 2=░░, 1=░░
Or use a simpler representation: list the emotional states as arrows:
```
Confidence: confident → confident → neutral → confused → frustrated → confused → neutral → confident
```

### Failed Personas Section

If any personas failed during the orchestration loop, add after "Persona Journeys":

```markdown
### ❌ Failed Sessions

| Persona | Archetype | Failure Reason |
|---------|-----------|---------------|
| {name} | {archetype} | {reason} |
```

### Step 8: Save the Report

Save the generated report to the current directory as:
```
crowdtest-report-{domain}-{YYYY-MM-DD}.md
```

Where `{domain}` is extracted from the product URL (e.g., `myapp.com` → `myapp-com`).

Announce the save: "Report saved to crowdtest-report-{domain}-{date}.md"

---

## Phase 8: Delta Analysis

> **Purpose**: Compare the current run with a previous run (if one exists) to show improvement or regression. This creates the viral loop — founders fix issues, re-run, and see their score improve.

> **Input**: Current report + previous report (if exists in current directory)
> **Output**: DeltaReport section appended to the main report
> **Tools used**: File system (glob for previous reports)

### Step 1: Find Previous Report

Search the current directory for previous CrowdTest reports for the same URL:
```
Glob pattern: crowdtest-report-{domain}-*.md
```

- If no previous report exists → skip Phase 8, append to the report:
  ```markdown
  ---
  ## 📈 Delta from Previous Run

  *First run — no comparison available. Run CrowdTest again after making changes to see improvement.*
  ```
- If multiple previous reports exist → use the most recent one (by date in filename)
- Read the previous report to extract: Product Score, issues list, Task Completion Rate, NPS

### Step 2: Compare Scores

Extract from both reports:
- Overall Product Score
- Each dimension score (First Impression, Task Completion, Navigation, Trust, Error Handling, NPS)
- Task Completion Rate
- Average NPS

Calculate deltas for each metric.

### Step 3: Match Issues

Compare current issues with previous issues:

1. **Resolved Issues**: Issues in the previous report that are NOT in the current report
   - These represent fixes the team made between runs
2. **Persistent Issues**: Issues that appear in BOTH reports
   - These are still unfixed
3. **New Issues**: Issues in the current report that were NOT in the previous report
   - These are regressions or newly discovered problems

Match issues by semantic similarity (same as deduplication logic in Phase 7 Step 1).

### Step 4: Generate Delta Section

Append this section to the main report:

```markdown
---

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

Where `{direction}` is one of:
- "improved" if delta > 0
- "declined" if delta < 0
- "unchanged" if delta = 0

### Step 5: Save Updated Report

Re-save the report file with the delta section appended.

---

## Configuration

| Option | Default | Description |
|--------|---------|-------------|
| `personas` | 10 | Number of personas to generate |
| `model` | sonnet | LLM model (sonnet / opus) |
| `context` | (none) | Product description to improve analysis |
| `focus` | (none) | Specific area to focus testing on |
| `competitors` | (auto-detected) | Known competitors for comparison |
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

| Personas | Estimated Cost (Codex sub-agents) | Estimated Cost (CC sub-agents) |
|----------|-----------------------------------|---------------------------------|
| 5 | ~$1-2 | ~$3-5 |
| 10 | ~$2-4 | ~$5-10 |
| 20 | ~$4-8 | ~$10-20 |

**Note**: Each persona consumes sub-agent tokens (Codex / Claude Code) in addition to the main agent's orchestration tokens. Cost is higher than single-agent approaches but testing is more isolated and realistic.

---

## Cleanup

After the report is generated, clean up all browser state:

1. **Close all browser tabs** opened during the test (`browser_close`)
2. **Delete temporary files** — remove any screenshots, action logs, or intermediate JSON files created during the run (unless `--verbose` was set)
3. **Remove temporary worktrees** — delete `/tmp/crowdtest-persona-*` worktrees and corresponding temporary branches after results are collected
4. **Remove sub-agent artifacts** — clean `PERSONA-TASK.md`, copied snapshots, and transient logs once the final report is saved (unless `--verbose` was set)
5. **Summary to user** — print a one-line summary:
   ```
   ✅ CrowdTest complete — Score: X/10 | N issues found | Report: [path]
   ```
6. **If report was written to file** — confirm the file path
7. **If `--output` was not set** — the full report was already printed to stdout, just print the summary line

Do NOT leave browser windows open, worktrees, or orphaned sub-agent artifacts behind after the run finishes.

---

*CrowdTest: 20 junior product reviewers who each spend 2 minutes on your site. Not 20 real users — but infinitely better than testing alone at 11pm.*
