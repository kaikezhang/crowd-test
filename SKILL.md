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

CrowdTest runs an 8-phase pipeline. Each phase feeds the next:

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

> Implementation in Sprint 2.

Create N diverse personas based on the ProductProfile and SiteMap.

### Steps

1. Analyze the ProductProfile to derive persona archetypes that test specific aspects:
   - The First-Timer (tests onboarding)
   - The Switcher (tests vs competitor expectations)
   - The Power User (tests efficiency)
   - The Skeptic (tests trust signals)
   - The Edge Case Finder (tests robustness)
2. For each persona (one at a time):
   - Review ALL previously generated personas
   - Generate a NEW persona that fills gaps in the collection
   - Must differ on >= 2 dimensions from every existing persona
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
"After 2 clicks without finding what you want, express frustration and try search if available"
"Skip any text section longer than 3 lines without reading it"
"If a form has more than 5 fields, only fill the required ones"
"After 5 total failed attempts at anything, abandon the product entirely"
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

## Phase 3: Task Assignment

> Implementation in Sprint 2.

Assign each persona a specific, measurable task based on the ProductProfile's `core_tasks`.

### Output: TaskAssignment

```json
{
  "persona_id": "persona_001",
  "task": "Find the pricing page and determine which plan fits a solo freelancer",
  "success_criteria": "Persona can name the recommended plan and its price",
  "max_actions": 20,
  "timeout_seconds": 180
}
```

---

## Phase 4: Browser Sessions

> Implementation in Sprint 3.

For each persona, run a browser session following their task assignment.

### State Machine

```
INIT -> NAVIGATE -> ORIENT -> ACT <-> OBSERVE -> DECIDE -> done
```

### Steps (per persona)

1. **INIT**: Set up browser context matching persona's device
2. **NAVIGATE**: Open the URL, dismiss cookie banner if needed
3. **ORIENT**: Take snapshot, identify interactive elements, plan next action based on persona's purpose
4. **ACT**: Execute ONE action (click/type/scroll/navigate)
   - Wait 500ms after each action
   - Take new snapshot
5. **OBSERVE**: Did the snapshot change? If not -> try alternative approach
6. **DECIDE**: Continue or stop?
   - Continue if: actions < 20 AND time < 3 minutes AND not stuck
   - Stop if: goal met OR stuck OR limit hit
7. Log every action to the action log

### Circuit Breakers

- **Max 20 actions** per persona
- **Max 3 minutes** per persona
- **5 consecutive failures** -> force stop

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

> Implementation in Sprint 3.

Each persona reviews their journey narrative and action log, then writes grounded feedback.

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

If generic -> regenerate once. Still generic -> keep but flag it.

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
