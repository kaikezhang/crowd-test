# TASK.md — Sprint 1: Scout + Product Analysis

> ⚠️ DO NOT MERGE. Create PR and stop. 晚晚 reviews and merges.

## Sprint Goal

Implement the SKILL.md orchestration prompt for the first two pipeline phases:
1. **Scout** — page readiness checker + site map builder
2. **Product Analysis** — understand what the product IS before generating personas

These run together as the "understand the product" step. Everything downstream depends on their output quality.

## Context

CrowdTest is an **OpenClaw Skill** (SKILL.md prompt). There is NO application code — the entire product is a well-structured prompt that orchestrates Playwright MCP browser tools and LLM reasoning.

Read these docs for context:
- `docs/product-design-v2.md` — the canonical product vision (READ THIS FIRST)
- `docs/technical-design.md` — architecture and data structures
- `docs/competitive-research.md` — what others have built

## What You're Building

**Update `SKILL.md`** with the complete Phase 0 (Product Analysis) and Phase 1 (Scout) prompt sections. The SKILL.md is the product — it tells the LLM how to orchestrate browser tools to test a website.

### Module 1: Scout (Phase 1)

**Purpose**: Verify the target URL is testable, build a map of what's there.

**The LLM following SKILL.md should**:

1. Navigate to the target URL using `browser_navigate`
2. Wait for page load (network idle, ~10s timeout)
3. Take accessibility snapshot via `browser_snapshot`
4. Run health checks on the snapshot:
   - Count interactive elements (buttons, links, inputs, textboxes)
   - If < 5 interactive elements → page may not have loaded, wait 3s and retry once
   - Detect auth: search for "sign in", "log in", "create account", "password" patterns
   - Detect CAPTCHA: search for "captcha", "recaptcha", "hcaptcha", "verify you are human"
   - Detect cookie banner: search for "accept cookies", "cookie", "consent", "preferences"
5. If cookie banner → click the Accept/Close/Got it button to dismiss
6. If auth detected → report `status: "auth_required"` and stop with instructions
7. If CAPTCHA detected → report `status: "captcha_detected"` and stop with instructions
8. If page ready:
   - Identify all top-level navigation links from the snapshot
   - Visit the top 3-5 linked pages (navigate, wait, snapshot each one)
   - For each page: classify type, count interactive elements, list key features
   - Identify the **critical path** — the sequence of pages leading to the core action

**Output format** (the LLM should produce this as structured thinking):

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
      "key_elements": ["hero CTA 'Get Started Free'", "nav: Features, Pricing, Login", "testimonials section"]
    },
    {
      "url": "/pricing",
      "type": "pricing",
      "interactive_elements": 8,
      "key_elements": ["3 plan cards", "annual/monthly toggle", "CTA per plan"]
    },
    {
      "url": "/features",
      "type": "features",
      "interactive_elements": 12,
      "key_elements": ["feature grid", "demo video", "integration logos"]
    }
  ],
  "critical_path": ["/", "/signup", "/dashboard"],
  "cookie_banner": true,
  "responsive_notes": "Nav collapses to hamburger on mobile viewport"
}
```

### Module 2: Product Analysis (Phase 0)

**Purpose**: Understand what the product IS before generating personas. This is the difference between "AI randomly browses a website" and "freelancer tries to send an invoice."

**The LLM following SKILL.md should** (using the Scout's snapshots as input):

1. Read the landing page content from the accessibility snapshot:
   - Headline text, subheadline, CTA button text
   - Navigation labels
   - Any pricing or audience signals
2. Answer these questions explicitly:
   - **What does this product do?** (one sentence)
   - **Category**: SaaS tool / e-commerce / content site / dev tool / social app / marketplace / other
   - **Intended user**: Who is the copy/pricing/features aimed at?
   - **Core tasks**: What are the 2-3 main things a user should be able to DO?
   - **Competitors**: What does this product resemble? (infer from features/category)
   - **Maturity**: Landing page only / MVP / Full product
   - **Pricing model**: Free / Freemium / Paid / Not visible
3. If the user provided `--context`, `--focus`, or `--competitors` flags, merge that info

**Output format**:

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

## SKILL.md Structure Requirements

The SKILL.md should be written as clear instructions that an LLM (Claude/GPT) can follow step-by-step. It should:

1. Start with a **Prerequisites** section (Playwright MCP must be available)
2. Have a **Quick Start** section ("Test https://url with N personas")
3. Clearly label each Phase with ## headings
4. Include the JSON output schemas inline
5. Include **error handling** for each failure mode (what to do when page doesn't load, auth detected, etc.)
6. Be self-contained — an LLM reading only SKILL.md should be able to run the full pipeline

For this sprint, fully flesh out **Phase 0 (Product Analysis)** and **Phase 1 (Scout)**. Keep the remaining phases (2-8) as stubs with their descriptions and schemas from the current SKILL.md, but add a note "Implementation in Sprint 2/3."

## Acceptance Criteria

- [ ] SKILL.md has complete Phase 0 and Phase 1 with detailed step-by-step instructions
- [ ] Phase 0 produces a ProductProfile JSON with all fields populated
- [ ] Phase 1 produces a SiteMap JSON with ≥ 3 pages for multi-page sites
- [ ] Scout correctly handles: ready / auth_required / captcha_detected / load_failed
- [ ] Cookie banner dismissal instructions are clear
- [ ] Critical path identification is included in SiteMap
- [ ] Error handling for each failure mode is documented in SKILL.md
- [ ] Remaining phases (2-8) are stubbed with schemas and "coming in Sprint N" notes
- [ ] The SKILL.md reads naturally as instructions an LLM can follow

## What NOT to Do

- Do NOT create any application code (Python, JS, etc.) — this is a prompt-only skill
- Do NOT remove existing content from SKILL.md that's still valid — enhance it
- Do NOT merge the PR

## Test It

After writing, mentally walk through the SKILL.md as if you were an LLM:
1. Given URL `https://cal.com` — can you follow the instructions to Scout it?
2. Given the Scout output — can you produce a ProductProfile?
3. Are the instructions unambiguous at every step?
