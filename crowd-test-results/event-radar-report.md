# CrowdTest Report — Event Radar

## Score: 5.8/10 — Needs significant work

**Date**: 2026-03-22 | **Personas**: 3 | **Pages visited**: 12
**Task Completion**: 67% (2/3) | **Avg NPS**: 5.3
**Cost**: ~$0.18

---

## 🎯 If You Fix One Thing

**Fix the feed crash: `items?.map is not a function` in `AlertCard.tsx:595` — the feed crashes when navigating back from other pages.**

**Why**: 2/3 personas hit this crash. It completely breaks the core product experience — users cannot return to the feed without a full page reload. For a real-time event detection tool, feed reliability is existential.

**Evidence**:
> "The whole page crashed! I see a big error message with a stack trace. That's terrifying for a first-time user." — Jake Morrison (First-Timer)
> "ANOTHER CRASH. Same error. If I were trading and this crashed, I'd miss alerts. The feed is fragile." — Marcus Webb (Skeptic)
> Diana Reeves (Power User) was the only persona who didn't hit this crash — she navigated directly via URL reload.

---

## 📊 Product Scorecard

| Dimension | Score | Assessment |
|-----------|-------|-----------|
| First Impression | 7.3/10 | Strong value prop visible immediately — "Track market-moving events before they hit the headlines." Daily Briefing is a nice touch. But no onboarding for returning users who skipped setup. |
| Task Completion | 6.7/10 | 2/3 personas completed their primary task. The crash blocked task flow for 2/3 personas but both recovered via reload. |
| Navigation | 5.7/10 | 5 clear nav items (Feed/Watchlist/Scorecard/History/Settings). But sort preferences don't persist, "What is Smart Feed?" button does nothing visible, and feed crashes on return navigation. |
| Trust & Credibility | 5.0/10 | Source Journey transparency is excellent. But: no footer, no About page, no Privacy Policy, no Terms of Service, no company info. Verification data is almost entirely "Pending." |
| Error Handling | 3.3/10 | Unhandled React crash with raw stack trace shown to users. No error boundary. No graceful degradation. Save button gives no confirmation feedback. |
| Overall NPS | 5.3/10 | First-Timer: 6, Power User: 6, Skeptic: 4. Product has genuine value but trust and stability issues hold it back. |
| **OVERALL** | **5.8/10** | **Needs significant work** |

*Score calculation: (7.3 × 0.15) + (6.7 × 0.30) + (5.7 × 0.20) + (5.0 × 0.15) + (3.3 × 0.10) + (5.3 × 0.10) = 1.10 + 2.01 + 1.14 + 0.75 + 0.33 + 0.53 = 5.8*

---

## 🔴 Critical Issues (2)

### 1. Feed crashes with `items?.map is not a function` on return navigation
- **Severity**: Critical | **Affected**: 2/3 personas
- **Description**: Navigating back to the Feed (`/`) from Scorecard, Watchlist, or other pages intermittently causes an unhandled React crash in `SourceDetailStrip` component at `AlertCard.tsx:595`. The entire page is replaced with a raw error stack trace and a "Hey developer 👋" message.
- **Evidence**: Jake hit this at Step 9 (returning from Watchlist after adding USO) and Marcus hit it at Step 6 (returning from Scorecard). Both required a full page reload to recover.
- **Suggested fix**: Add an ErrorBoundary around the Feed component. Fix the root cause — `items` in `SourceDetailStrip` receives a non-array value (likely `undefined` or an object). Add a type guard: `Array.isArray(items) ? items.map(...) : []`.
- **Personas affected**: Jake Morrison (First-Timer), Marcus Webb (Skeptic)

### 2. No error boundary — raw React stack traces shown to users
- **Severity**: Critical | **Affected**: 2/3 personas
- **Description**: When the feed crashes, users see a raw developer error page with stack traces, file paths (`AlertCard.tsx:595`), and a message saying "Hey developer 👋 — You can provide a way better UX..." This is the React Router default error page, not a product error page.
- **Evidence**: Both Jake (Step 9) and Marcus (Step 7) saw the full `TypeError` stack trace with internal file paths exposed.
- **Suggested fix**: Implement a React ErrorBoundary component that shows a user-friendly "Something went wrong — click to reload" message instead of raw stack traces. This also prevents leaking internal file structure.
- **Personas affected**: Jake Morrison (First-Timer), Marcus Webb (Skeptic)

---

## 🟡 High Issues (3)

### 3. No About page, Privacy Policy, Terms of Service, or company information
- **Severity**: High | **Affected**: 1/3 personas (but would affect all trust-sensitive users)
- **Description**: The site has no footer at all. There is no way to find out who operates Event Radar, where the company is based, what data is collected, or what the terms of use are. For a financial information product, this is a serious trust gap.
- **Evidence**: Marcus (Steps 1-2) scrolled to the bottom of the feed looking for footer links and found none. He searched navigation for an About or Legal section — none exists.
- **Suggested fix**: Add a minimal footer with: About, Privacy Policy, Terms of Service, and a "Not financial advice" disclaimer. For a financial product, this isn't optional — it may be legally required.
- **Personas affected**: Marcus Webb (Skeptic)

### 4. "What is Smart Feed?" button produces no visible response
- **Severity**: High | **Affected**: 1/3 personas
- **Description**: On the Feed page, clicking the "What is Smart Feed?" button does nothing visible — no tooltip, no modal, no explanation appears. The button exists but seems non-functional.
- **Evidence**: Jake (Step 2) clicked this button as a first-timer trying to understand what Smart Feed means. The page did not change. No tooltip or popover appeared in the accessibility tree.
- **Suggested fix**: Either make the button trigger a tooltip/popover explaining Smart Feed, or remove the button if the feature isn't implemented yet.
- **Personas affected**: Jake Morrison (First-Timer)

### 5. Save button on Settings page gives no confirmation feedback
- **Severity**: High | **Affected**: 1/3 personas
- **Description**: After entering a Discord webhook URL and clicking "Save" on the Settings page, there is no visual confirmation — no toast notification, no "Saved!" message, no color change on the button. The user has no way to know if the save succeeded.
- **Evidence**: Diana (Step 11) entered a webhook URL and clicked Save. The page did not change. No success/error message appeared.
- **Suggested fix**: Show a toast notification ("Settings saved") or change the Save button text temporarily to "Saved ✓" with a green color. Also consider auto-saving like the Notification Budget section does (which shows "autosave" label).
- **Personas affected**: Diana Reeves (Power User)

---

## 🟠 Medium Issues (3)

### 6. Sort preference ("Highest severity") doesn't persist across navigation
- **Severity**: Medium | **Affected**: 1/3 personas
- **Description**: When Diana selected "Highest severity" from the sort dropdown, navigated to Scorecard, and returned to Feed, the sort had reset to "Latest first." User preferences should persist at least for the session.
- **Evidence**: Diana (Steps 1, 13) selected "Highest severity" sort, left the page, returned — sort was back to "Latest first."
- **Suggested fix**: Persist the sort preference in localStorage or URL query params so it survives navigation.
- **Personas affected**: Diana Reeves (Power User)

### 7. Watchlist order changes unexpectedly after adding a ticker
- **Severity**: Medium | **Affected**: 1/3 personas
- **Description**: After Jake added USO via the search dialog, the watchlist ticker order changed — NVDA moved to the top, displacing AAPL. The reorder was not initiated by the user and was confusing.
- **Evidence**: Jake (Step 7) noticed "6 tickers tracked" but the order was NVDA, AAPL, TSLA, META, XLE, USO — different from the original AAPL, META, NVDA, TSLA order.
- **Suggested fix**: Preserve existing watchlist order when adding new tickers. New tickers should append to the bottom.
- **Personas affected**: Jake Morrison (First-Timer)

### 8. Evidence tab on event detail shows duplicate summary instead of actual evidence
- **Severity**: Medium | **Affected**: 1/3 personas
- **Description**: On the Iran/Strait of Hormuz CRITICAL event detail page, clicking the "Evidence" tab shows the same summary paragraph that's already on the Summary tab. No additional evidence — no source URL, no corroborating sources, no raw data.
- **Evidence**: Jake (Step 12) clicked the Evidence tab and saw only the same AI-generated summary. Expected: source links, screenshots, cross-references.
- **Suggested fix**: Populate the Evidence tab with: original source URL, raw source text, corroborating sources if available, and timestamps. If no evidence beyond the summary exists, say "No additional evidence available" rather than repeating the summary.
- **Personas affected**: Jake Morrison (First-Timer)

---

## 🟢 Low Issues (2)

### 9. Inconsistent event detail richness across events
- **Severity**: Low | **Affected**: 1/3 personas
- **Description**: The Iran/Strait of Hormuz event (from Truth Social) has minimal detail — just a summary and no Bull/Bear case. The PTC event (from Breaking News) has Bull/Bear case, price tracking, similar past events, and market regime context. The experience is inconsistent.
- **Evidence**: Jake (Step 11) saw minimal detail on the Iran event. Marcus (Step 9) saw rich detail on the PTC event.
- **Suggested fix**: Ensure all CRITICAL and HIGH events have Bull/Bear case analysis, even if brief. If data is unavailable, show "Analysis pending" rather than omitting the section entirely.
- **Personas affected**: Jake Morrison (First-Timer), Marcus Webb (Skeptic)

### 10. Daily Briefing card disappears permanently after dismissal
- **Severity**: Low | **Affected**: 1/3 personas
- **Description**: Diana dismissed the Daily Briefing, and it did not reappear. There's no way to re-read the daily briefing after dismissing it.
- **Evidence**: Diana (Step 2) clicked "Dismiss briefing." The briefing was gone on subsequent page loads.
- **Suggested fix**: Add a "Show today's briefing" toggle or make briefing accessible from a dedicated section.
- **Personas affected**: Diana Reeves (Power User)

---

## 📖 Persona Journeys

### Jake Morrison — First-Timer (desktop)
**Task**: Set up watchlist with XLE and USO, find the most critical oil event → ⚠️ PARTIALLY_COMPLETED
**Confidence**: confident → neutral → confused → frustrated → confident → confident
**Time**: ~2 min | **Actions**: 14 | **NPS**: 6

> I opened Event Radar and it looked promising — "Track market-moving events before they hit the headlines." I tried clicking "What is Smart Feed?" but nothing happened. Then I found the "Add XLE to watchlist" button right on the event card — that was smooth. I went to the Watchlist to add USO, the search worked great and I added it instantly. But when I went back to the feed, the entire page crashed with a developer error. I had to reload. After the reload, I found the CRITICAL Iran event about the Strait of Hormuz and read the detail page. The Trust tab showing the AI pipeline was cool, but the crash killed my confidence.

### Diana Reeves — Power User (desktop)
**Task**: Sort by severity, review Scorecard accuracy, configure Discord webhook → ✅ COMPLETED
**Confidence**: confident → confident → neutral → confident → confident → neutral → confident
**Time**: ~2.5 min | **Actions**: 13 | **NPS**: 6

> I went straight to sorting by severity — works fine, but doesn't actually reorder things since events are grouped by date. I found the keyboard shortcuts (j/k, /, ?) — nice for power users. The Scorecard is genuinely impressive — 10,795 events, source accuracy breakdown, rolling trend charts, and they're honest about limitations ("calibration layer, not a victory lap"). I configured my Discord webhook URL but when I hit Save, there was zero confirmation. Did it work? No idea. The notification budget settings with daily caps and quiet hours are well-designed. This product has serious depth but needs UX polish.

### Marcus Webb — Skeptic (desktop)
**Task**: Evaluate trustworthiness — check sources, accuracy data, find privacy/legal info → ❌ FAILED
**Confidence**: skeptical → neutral → confused → frustrated → neutral → frustrated → ready_to_leave
**Time**: ~2 min | **Actions**: 11 | **NPS**: 4

> I came in looking for reasons to trust this tool. First red flag: no footer, no About page, no Privacy Policy — who runs this? Then I checked an event detail and was pleasantly surprised by the Trust tab showing the AI pipeline (Rule Filter → AI Judge → Enriched → Delivered in 37s). That's genuine transparency. But then the feed crashed with a raw developer error showing file paths — unprofessional and concerning. The Scorecard shows 41.8% directional hit rate — barely better than a coin flip. And only 79 outcomes verified out of 10,795 events — that's 0.7%. I appreciate the honesty but I'm not trading on this yet.

---

## ✅ What's Working Well

| Feature | Mentions | Evidence |
|---------|----------|---------|
| Source Journey / Trust transparency | 3/3 | All personas found the Trust tab's pipeline visualization (source → filter → AI judge → delivery) genuinely impressive |
| Watchlist search & add from feed | 2/3 | Jake and Diana both found adding tickers intuitive — inline "Add to watchlist" buttons on cards + search dialog |
| Daily Briefing | 2/3 | Jake and Diana noted the Daily Briefing as a helpful orientation — "2 events detected in the last 24h for your watchlist" |
| Scorecard depth | 2/3 | Diana and Marcus found the source accuracy breakdown, rolling trend charts, and signal/confidence buckets valuable |
| Keyboard shortcuts | 1/3 | Diana found j/k navigation, / for search, ? for help — good power user feature |
| Real-time event content | 3/3 | All personas noted the events were genuinely current (Iran sanctions, PTC earnings, oil prices) with real ticker associations |
| Notification budget controls | 1/3 | Diana noted quiet hours, daily caps, and non-watchlist toggle as well-designed |

---

## 📊 Segment Insights

### By Tech Level
- **Novice (Jake)**: Successfully added tickers but was blocked by the feed crash. The "What is Smart Feed?" dead button was his only attempted learning aid. Needs better onboarding.
- **Advanced (Diana)**: Navigated efficiently, used keyboard shortcuts, configured Discord. Main friction was missing save confirmation. The Scorecard was a highlight.
- **Expert skeptic (Marcus)**: Focused entirely on trust signals. Found good transparency (Trust tab, Scorecard) but was alarmed by missing legal pages and the crash exposing internal file paths.

### By Device
- **Desktop**: All 3 personas used desktop. No mobile testing in this run.

### By Archetype
- **First-Timer**: Successfully completed partial task but needs better onboarding and error handling. The "What is Smart Feed?" dead button is the only help feature and it doesn't work.
- **Power User**: Most satisfied persona. The depth of Scorecard data, keyboard shortcuts, and notification controls serve this audience well. Needs UX polish (save confirmation, sort persistence).
- **Skeptic**: Most underserved persona. The product has genuine transparency (Trust tab) but sabotages itself with missing legal pages and raw crash errors. A skeptic who sees developer stack traces will not trust the product with financial decisions.

---

## 🔧 Recommended Fix Priority

1. **Fix the `AlertCard.tsx:595` crash** — Impact: Eliminates the #1 trust-breaking experience. 2/3 personas hit this. Likely a simple null/type guard fix.
2. **Add React ErrorBoundary** — Impact: Even with other bugs, users should never see raw stack traces. Prevents future crashes from being trust-breaking.
3. **Add footer with About/Privacy/Terms** — Impact: Addresses the fundamental trust gap for any financially-oriented product. May be legally required.
4. **Fix "What is Smart Feed?" button** — Impact: First-timers' only learning aid is non-functional.
5. **Add save confirmation to Settings** — Impact: Users need feedback when saving critical settings like notification webhooks.
6. **Persist sort preference** — Impact: Low effort, improves power user experience.

---

## 📈 Delta from Previous Run

*First run — no comparison available. Run CrowdTest again after making changes to see improvement.*
