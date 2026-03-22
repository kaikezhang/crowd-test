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
Phase 4: Browser Sessions   → Each persona browses with task + emotional tracking
Phase 5: Journey Recon      → Action log → narrative arc + key moments
Phase 6: Feedback Synthesis  → Grounded, evidence-linked feedback per persona
Phase 7: Aggregation        → Product Score (X/10) + "Fix ONE Thing" report
Phase 8: Delta Analysis     → Compare with previous run if exists
```

**Execution order**: Phase 1 runs first (Scout needs raw browser data), then Phase 0 analyzes what Scout found. The numbering reflects conceptual priority — understanding the product (Phase 0) is foundational — but Scout must run first to gather the data.

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

## Phase 4: Browser Sessions

> **Purpose**: The core testing phase. The LLM embodies a persona and browses the product, executing their assigned task while tracking emotional state and logging every action. This produces the ground truth action log that all feedback is built on.

> **Input**: One Persona (from Phase 2) + their TaskAssignment (from Phase 3) + SiteMap (from Phase 1)
> **Output**: EnhancedActionLog (one per persona)
> **Tools used**: `browser_navigate`, `browser_snapshot`, `browser_click`, `browser_type`, `browser_scroll`, `browser_resize`
> **Execution**: Run ONE persona at a time, sequentially. After each persona completes, pass their `confusion_events` to the next persona via the action prompt (see "covered issues" below).

### State Machine

```
INIT → NAVIGATE → ORIENT → ACT ⟷ OBSERVE → DECIDE → DONE
```

Each persona's session follows this state machine. The loop is: ORIENT → ACT → OBSERVE → DECIDE → (back to ORIENT or DONE).

### Step 1: INIT — Set up browser context

Configure the browser viewport based on the persona's device:

- If `persona.device === "mobile"`: call `browser_resize` with width=375, height=812 (iPhone viewport)
- If `persona.device === "tablet"`: call `browser_resize` with width=768, height=1024 (iPad viewport)
- If `persona.device === "desktop"`: keep the default viewport (no resize needed)

Initialize tracking variables:
- `action_count = 0`
- `consecutive_failures = 0`
- `start_time = now()`
- `actions = []` (the action log array)
- `emotional_arc = []` (tracks emotional state over time)
- `pages_visited = []`
- `confusion_events = []`

### Step 2: NAVIGATE — Go to entry point

Call `browser_navigate` with the persona's `entry_point` URL (resolved against the product's base URL).

- If `persona.entry_point` is a relative path like `/pricing`, resolve it: `{product_url}{entry_point}`
- If the SiteMap recorded `cookie_banner: true`, be ready to dismiss the cookie banner using the same logic as Phase 1 Step 4b.
- Wait for page load.
- Add the entry point URL to `pages_visited`.

If navigation fails:
1. Retry once after 3 seconds.
2. If retry also fails, mark the session as `"task_result": "ERROR"` with `"failure_reason": "Could not load entry point"` and proceed to the next persona.

### Step 3: ORIENT — Observe and plan

Call `browser_snapshot` to get the current page's accessibility tree.

This is where the LLM thinks IN CHARACTER as the persona. Using the action prompt (see below), the LLM:
1. Reads the accessibility snapshot
2. Considers the persona's task and behavioral rules
3. Decides what action to take next
4. Reports their emotional state

### Step 4: ACT — Execute one action

Based on the action prompt response, execute exactly ONE browser action:

| Action Type | Browser Tool | Details |
|-------------|-------------|---------|
| `click` | `browser_click` | Click the element described in `target` |
| `type` | `browser_type` | Type `text` into the element described in `target` |
| `scroll` | `browser_scroll` | Scroll down to reveal more content |
| `navigate_back` | `browser_navigate` | Go back to the previous page |
| `done` | (none) | Persona has decided to stop |

After executing the action:
- Wait 500ms for the page to respond.
- Call `browser_snapshot` to capture the new state.
- Increment `action_count`.
- Track the current page URL — if it changed, add the new URL to `pages_visited`.

### Step 5: OBSERVE — Check if action worked

Compare the before and after snapshots:

- **Action succeeded**: The snapshot changed in a way consistent with the expected outcome.
  - Reset `consecutive_failures = 0`.
  - Log the action with `"success": true`.

- **Action failed**: The snapshot is unchanged or changed unexpectedly.
  - Increment `consecutive_failures`.
  - Log the action with `"success": false`.
  - Enter **RECOVER mode**:

#### RECOVER mode

When an action fails, try these recovery strategies in order:
1. **Scroll to reveal**: The target element may be off-screen. Call `browser_scroll` to scroll down, then retry.
2. **Try parent element**: The clickable area may be on a parent element. Look for a containing link or button.
3. **Use keyboard**: Try pressing Enter or Tab to interact with the focused element.
4. **Try alternative path**: Look for a different element that achieves the same goal.

Only attempt ONE recovery strategy per failed action. Log the recovery attempt as a separate action.

### Step 6: DECIDE — Continue or stop?

After each action-observe cycle, check the stopping conditions:

**Continue if ALL are true**:
- `action_count < 20` (max actions not reached)
- Elapsed time < 3 minutes (max time not reached)
- `consecutive_failures < 5` (not stuck in a failure loop)
- Task is not yet complete
- Persona has not decided to abandon (via `done` action)

**Stop (DONE) if ANY are true**:
- Task completed successfully → `task_result = "COMPLETED"`
- `action_count >= 20` → `task_result` based on progress, `failure_reason = "Max actions reached"`
- Elapsed time >= 3 minutes → `task_result` based on progress, `failure_reason = "Time limit reached"`
- `consecutive_failures >= 5` → `task_result = "FAILED"`, `failure_reason = "Session unstable — 5 consecutive failures"`, mark session as unstable
- Persona chose `done` action → `task_result` based on their assessment

When stopping, determine `task_result`:
- `"COMPLETED"` — success criteria fully met
- `"PARTIALLY_COMPLETED"` — meaningful progress was made but task not finished
- `"FAILED"` — could not accomplish the task

### Step 7: Emotional tracking

After EVERY action, the persona reports their emotional state. This is captured in the action prompt response and logged in both the individual action and the `emotional_arc` array.

Valid emotional states:
| State | Meaning | Triggers |
|-------|---------|----------|
| `confident` | "I know what to do next" | Clear UI, successful actions, obvious next step |
| `neutral` | "This is fine" | Default state, nothing remarkable |
| `confused` | "I don't understand what happened" | Unexpected result, unclear labels, no obvious path |
| `frustrated` | "This isn't working" | Multiple failures, confusing flow, broken elements |
| `ready_to_leave` | "I'm about to give up" | Extended confusion, no progress, trust broken |

When emotional state transitions to `confused` or `frustrated`, log a confusion event:
```json
{
  "step": 7,
  "expected": "New invoice form",
  "got": "Settings dropdown",
  "gap": "'+' icon is ambiguous — could mean 'add new' or 'more options'"
}
```

### Action Prompt Template

This is the EXACT prompt structure the LLM sees for each action step. The variables in `{braces}` are filled from the persona, task, and session state:

```
System: You are {persona.narrative}

BEHAVIORAL RULES (you MUST follow these):
{persona.behavioral_rules — listed as numbered items}

You are testing {product_url}. Your task: {task.primary_task}
Success criteria: {task.success_criteria}

Current page snapshot:
{accessibility_snapshot}

Your action history (last 5 steps):
{recent_actions_summary — step number, action type, target, success, emotional_state}

Issues already found by OTHER testers (find DIFFERENT ones):
{covered_issues_from_previous_personas — list of confusion_events from earlier personas, or "None yet" for the first persona}

Task progress: {task_progress_estimate — from the last action, or "0%" at start}
Actions remaining: {20 - action_count}

What do you do next? Think in character, then respond as JSON:
{
  "thinking": "your in-character thought process — what you see, what you're looking for, how you feel",
  "action": "click|type|scroll|navigate_back|done",
  "target": "exact element description from the snapshot above",
  "text": "(for type action only — the text to type)",
  "emotional_state": "confident|neutral|confused|frustrated|ready_to_leave",
  "task_progress": "0-100% estimate of how close you are to completing the task"
}
```

**Important notes on the action prompt**:
- `{recent_actions_summary}` shows only the LAST 5 actions to keep context window manageable. Include step number, action type, target, whether it succeeded, and the persona's emotional state at that step.
- `{covered_issues_from_previous_personas}` is the cumulative diversity mechanism. After each persona completes, their `confusion_events` are added to this list. This prevents all personas from reporting the same obvious issue and encourages discovering different problems.
- For the FIRST persona in the run, `covered_issues_from_previous_personas` is "None yet — you are the first tester."

### Circuit Breakers (hard stops)

These are NON-NEGOTIABLE limits that force the session to end:

| Breaker | Threshold | Action |
|---------|-----------|--------|
| Max actions | 20 actions | Force DONE, assess task_result based on progress |
| Max time | 3 minutes | Force DONE, assess task_result based on progress |
| Consecutive failures | 5 in a row | Force DONE, mark `task_result = "FAILED"`, flag session as unstable |

### Output: EnhancedActionLog

After the session ends, produce the following JSON:

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

## Phase 5: Journey Recon

> Implementation in Sprint 3.

Convert the raw action log into a narrative arc with confidence curve and key moments.

### Output: JourneyNarrative

```json
{
  "persona_id": "persona_001",
  "narrative_arc": "Confident start -> confusion at pricing -> recovery via search -> successful signup",
  "confidence_curve": [0.8, 0.7, 0.4, 0.3, 0.5, 0.7, 0.9],
  "key_moments": [
    { "step": 3, "type": "friction", "description": "Couldn't find pricing from homepage" },
    { "step": 7, "type": "delight", "description": "Search instantly found the pricing page" }
  ]
}
```

---

## Phase 6: Feedback Synthesis

> **Purpose**: Transform the raw action log into grounded, evidence-linked feedback. This is a THREE-PHASE process that ensures feedback is specific to THIS product and references actual events from the browsing session. Generic feedback like "the UI could be improved" is a failure mode this phase is designed to prevent.

> **Input**: Persona (from Phase 2) + EnhancedActionLog (from Phase 4) + TaskAssignment (from Phase 3)
> **Output**: PersonaFeedback JSON
> **Tools used**: None (pure LLM reasoning over session data)

### Phase A: Ground Truth (already exists)

The EnhancedActionLog from Phase 4 IS the ground truth. It contains:
- Every action taken, with success/failure status
- Emotional state at each step
- Confusion events with expected vs actual outcomes
- Task completion result
- Pages visited

**Do not modify or reinterpret the action log.** Phase A is already complete when Phase 6 begins.

### Phase B: In-Character Feedback Review

Give the persona their COMPLETE action log and task assignment. Ask them to write honest feedback AS THEIR CHARACTER — in first person, using their voice, referencing their specific experience.

The persona must address ALL of the following:

#### B1: First Impression (first 30 seconds / first 3 actions ONLY)

What did the persona think in their first 30 seconds on the site? This is based ONLY on the first 3 actions in the action log. Do not let later experience color the first impression.

Questions to answer:
- "Did I immediately understand what this product does?"
- "Did I know what to do next?"
- "Did the page feel trustworthy and professional?"

#### B2: What Worked Well (with evidence)

List things that went smoothly during the session. Each positive MUST reference a specific action log step or page:
- "The signup flow was fast — I completed it in 3 steps (steps 2-4)"
- "Search worked perfectly — found what I needed instantly (step 11)"

If nothing went well, say so honestly: "Nothing stood out as particularly smooth."

#### B3: What Didn't Work (with evidence)

List every friction point, confusion, and failure. Each issue MUST:
- Reference a specific action log step number
- Name the specific page and element involved
- Describe what was expected vs what happened
- Be specific enough that a developer could reproduce the issue

```
GOOD: "On /dashboard (step 7), I clicked the '+' icon expecting a 'New Invoice' form, but it opened a Settings dropdown. The icon is ambiguous."
BAD: "The dashboard was confusing." ← REJECT THIS
```

#### B4: Task Completion Assessment

In the persona's own words:
- Did they complete their task? Why or why not?
- How much effort did it take?
- Would they try again or give up?

#### B5: Comparative Observations (Switcher personas only)

If the persona has `competitors_used`, they MUST compare their experience:
- "In FreshBooks, creating an invoice is a single 'New Invoice' button on the dashboard. Here, I couldn't find it at all."
- "The signup was faster than Wave — Wave requires email verification first."

For non-Switcher personas, skip this section.

#### B6: Scores and Verdict

The persona provides:
- `would_pay`: boolean — would they pay for this product?
- `would_return`: `"yes"`, `"maybe"`, or `"no"` — would they come back?
- `one_line_verdict`: A single sentence summary of their experience

### Phase C: Structured Extraction

Extract structured JSON from the Phase B narrative. This is a SEPARATE step — do not combine it with Phase B. Read the narrative and extract:

#### C1: Tiered Issues

Classify each issue into a tier:

| Tier | Name | Definition | Example |
|------|------|-----------|---------|
| 1 | **BLOCKING** | Prevents task completion entirely | "Cannot find the core feature" |
| 2 | **FRICTION** | Slows the user down or causes confusion but doesn't block | "Ambiguous icon, had to try 3 things" |
| 3 | **OBSERVATION** | Minor annoyance or suggestion | "Footer links are hard to read" |

Each issue must include:
- `tier`: 1, 2, or 3
- `type`: classify as one of: `discoverability`, `usability`, `performance`, `trust`, `accessibility`, `content`, `navigation`, `error_handling`, `visual`, `other`
- `page`: the URL path where the issue occurred
- `element`: the specific UI element involved
- `issue`: clear description of the problem
- `severity`: `critical`, `high`, `medium`, or `low`
- `evidence`: reference to specific action log steps (e.g., "Steps 7-14: exhaustive search of dashboard UI")
- `suggested_fix`: a concrete suggestion for how to fix it

#### C2: Absence Observations

Things the persona expected to find but did NOT exist. These are valuable because they reveal unmet expectations:
- "No onboarding wizard for first-time users"
- "No tooltip explaining the '+' button"
- "No keyboard shortcuts visible"
- "No way to undo the last action"

#### C3: Positives (structured)

Each positive observation as a structured object:
- `feature`: what worked well
- `evidence`: reference to action log steps
- `why`: why it was good (be specific)

#### C4: Scores

Rate the product on these 6 dimensions (0-10 scale):

| Dimension | What It Measures |
|-----------|-----------------|
| `first_impression` | Clarity, professionalism, and immediate understanding in first 30 seconds |
| `task_completion` | How well the product enabled the persona's specific task |
| `navigation` | Ease of finding things, information architecture, menu clarity |
| `trust` | Social proof, security signals, pricing transparency, professional design |
| `error_handling` | How the product responds to mistakes, dead ends, and confusion |
| `nps` | Net Promoter Score (0-10): "How likely to recommend to a colleague?" |

### Canary Self-Validation

After generating the complete PersonaFeedback, run these checks:

**Check 1 — Specificity**: "Could this feedback apply to ANY product, or is it specific to THIS product?"
- Read each issue. If an issue could be copy-pasted into feedback for a completely different product without changing any words, it is **generic slop**.
- Examples of generic slop: "The UI could be improved", "Navigation was confusing", "The design feels dated"
- If ANY issue is generic → **regenerate that issue** with a specific reference to the action log.

**Check 2 — Evidence grounding**: "Does every issue reference a specific action log step, page, and element?"
- Every issue in the `issues` array MUST have a non-empty `evidence` field that references specific step numbers.
- If any issue lacks evidence → **reject it and rewrite** with proper evidence pointers.

**Check 3 — Product specificity**: Read the `one_line_verdict`. Does it mention the product's actual functionality or a specific experience? "Useful product" = FAIL. "Great invoicing tool hidden behind a discoverability problem on the dashboard" = PASS.

If regeneration is needed, regenerate ONCE. If the regenerated version still fails the canary checks, keep it but add `"canary_flag": true` to the output to signal that this feedback may be lower quality.

### Output: PersonaFeedback

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

## Phase 7: Aggregation

> Implementation in Sprint 3.

### Steps

1. Collect all PersonaFeedback
2. **Deduplicate**: Group semantically identical issues (different personas may describe the same problem differently)
3. **Score severity**: `severity_weight * (1 + log2(affected_personas))`
4. **Consensus**: Issues mentioned by >50% of personas = consensus issues
5. **Segment analysis**: Group insights by tech_level, device, purpose
6. **Product Score**: Rate 0-10 across 6 dimensions
7. **Fix ONE Thing**: Identify the single highest-impact fix
8. **Generate Markdown report**

### Report Structure

```markdown
# CrowdTest Report — {product_url}

**Date**: {date}
**Personas**: {count} | **Pages visited**: {total} | **Issues found**: {unique_count}
**Product Score**: {score}/10
**Avg NPS**: {avg_nps} | **Would pay**: {pay_rate}% | **Would return**: {return_rate}%

## Fix ONE Thing
{the single highest-impact recommendation}

## Critical Issues ({count})
### 1. {issue_description}
- **Affected**: {N}/{total} personas
- **Pages**: {pages}
- **Evidence**: {evidence_summary}

## High Issues ({count})
...

## Medium Issues ({count})
...

## Low Issues ({count})
...

## Segment Insights
### By Tech Level
- **Novice**: {finding}
- **Advanced**: {finding}

### By Device
- **Mobile**: {finding}
- **Desktop**: {finding}

## What's Working
- {positive} ({mention_count}/{total} personas)

## Individual Persona Reports
<details><summary>Maria Chen (NPS: 5)</summary>
{full_feedback}
</details>
```

---

## Phase 8: Delta Analysis

> Implementation in Sprint 3.

Compare the current run with a previous run (if exists) to show improvement or regression.

### Output: DeltaReport

```json
{
  "previous_score": 5.2,
  "current_score": 7.1,
  "delta": "+1.9",
  "resolved_issues": ["Pricing page CTA mismatch"],
  "new_issues": ["Mobile nav overlap on small screens"],
  "persistent_issues": ["Onboarding flow too long"]
}
```

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

| Personas | Sonnet | Opus |
|----------|--------|------|
| 5 | ~$0.30 | ~$1.50 |
| 10 | ~$0.60 | ~$3.00 |
| 20 | ~$1.20 | ~$6.00 |

---

*CrowdTest: 20 junior product reviewers who each spend 2 minutes on your site. Not 20 real users — but infinitely better than testing alone at 11pm.*
