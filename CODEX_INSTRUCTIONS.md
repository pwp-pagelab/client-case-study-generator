# Case Study Generator — Codex Handover

This document is the source of truth for anyone (human or AI) extending the `case-study-generator.html` file. Read it before changing anything.

---

## What this is

A single-file, no-build HTML generator that produces **anonymized 1-pager PDF case studies** for performance marketing engagements. The user is an agency owner who wants to spin up polished, prospect-ready case studies fast — without ever exposing a real client's identity, and without falling into the "5 metrics, 4 of them say zero" trap that hits when real client data is messy or incomplete.

The generator does three things well:

1. **Anonymizes by design** — no client name field exists in the data model
2. **Adapts to whatever data the user has** — empty fields disappear from the output, never render as "SAR 0"
3. **Auto-generates smart copy** — headlines, challenges, and outcome callouts are drafted from the actual data, not generic templates

The deliverable is a print-to-PDF A4 page in one of three styles: Editorial (clean), Bold (dark/punchy), or Data-heavy (analytical).

---

## Why it's built this way — design principles (DO NOT VIOLATE)

These principles came from real failures with v1 of this generator. Each one is a guardrail.

### 1. Empty is not zero

A blank input field means **"not tracked"** — not "zero". The generator must never display empty inputs as "0" or "SAR 0" on the output page.

- **Check empty-vs-zero with `hasValue()`** before any calculation, render, or template trigger.
- **Hero stats, results table rows, and metric chips all filter out null values** before rendering.
- If a client only tracked leads and CPL (not closed deals or pipeline), the case study features only leads and CPL — and the "Closed deals" and "Pipeline" rows do not appear at all.

This rule is non-negotiable. Violating it produced case studies showing "Closed Deals: 0 · CAC: SAR 0 · Pipeline: SAR 0" for engagements that actually delivered 5,258 leads at SAR 3 CPL. That output is worse than no case study.

### 2. Client names never enter the system

There is **no `clientName` field** in the data model. The user works with anonymized descriptors only ("A US-based DTC skincare brand," "A 40-year-old Saudi construction material manufacturer"). The data model can't leak a name because there's nowhere to put one.

- Do not add a `clientName` field, even as optional.
- Do not add a "client logo upload" field. Agency logo is fine; client logo is not.
- The "Strict anonymization" toggle defaults to on. If you add fields later, gate them behind this toggle.

### 3. No fake testimonials

The endorsement section offers three modes:

- **`outcome`** (default) — agency-voice callout, no quote marks, no attribution. Always publishable. This is what most credible agencies actually use.
- **`testimonial`** — drafted in client voice, but clearly marked "for client approval" with a yellow warning box. The user must send to the client and get written approval before publishing.
- **`none`** — omit endorsement entirely.

**Fabricating attributed quotes (even anonymized ones like "Head of Marketing, client") is deceptive advertising under FTC, ASA, and most consumer-protection regulators.** Do not remove the warning. Do not change the default away from `outcome`. If you swap the template-based drafters for a real LLM, the LLM output for testimonials must still be labeled as draft-only.

### 4. Engagement type drives everything

The generator supports five focus types: `leadgen`, `ecommerce`, `creative`, `awareness`, `fullfunnel`. The active focus determines:

- Which performance input fields appear
- Which metrics are computed
- Which hero stats show on the page
- The wording of auto-drafted headlines, challenges, and outcome callouts

The `detectFocus()` function reads the user's filled fields and channel text to pick the right focus automatically. The user can override via the dropdown.

**When extending the generator, the right pattern is usually "add a new focus type," not "add a new field."** A new focus type defines its own field set, its own hero stat priorities, and its own draft copy templates.

### 5. Format dictates fidelity

The output is A4 (210mm × 297mm) portrait, fixed dimensions, one page. Do not change this unless the user explicitly asks for multi-page support. If you add content that pushes past one page in any of the three styles, trim or restructure — don't let it overflow.

---

## Architecture

Single HTML file. No build step. No external dependencies except Google Fonts.

```
case-study-generator.html
├── <style>                  All CSS, organized by section
├── HTML body
│   ├── .topbar              Sticky header with tab switcher + export button
│   ├── #view-input          Form (default visible)
│   └── #view-preview        Rendered 1-pager (hidden by default)
└── <script>
    ├── state                Single source of truth — all user inputs
    ├── helpers              hasValue, num, fmtCurrency, fmtNumber, fmtPct, fmtROAS, lift, liftInverse
    ├── calc()               All derived metrics, returns nulls for missing data
    ├── detectFocus()        Reads state, returns focus type
    ├── getAvailableMetrics() Returns only metrics with data, ranked by importance
    ├── buildHeroStats()     Returns top 4 available metrics
    ├── draft*()             draftHeadline, draftChallenge, draftOutcomeCallout, draftTestimonial
    ├── render*()            renderPerfFields, renderBaselineFields, renderComputed, renderPreview
    ├── layouts              editorialLayout, boldLayout, dataLayout (all return HTML strings)
    ├── tables/endorsement   buildResultsTableHTML, endorsementHTML
    └── bindAllInputs()      Event wiring + export
```

### Data flow

```
User types in form
   ↓
state object updates
   ↓
detectFocus() runs (if focus = 'auto')
   ↓
renderPerfFields() re-renders fields for current focus
   ↓
calc() computes all derivable metrics, null for missing inputs
   ↓
getAvailableMetrics() ranks non-null metrics by importance
   ↓
Form: renderComputed() shows metric chips
Preview: renderPreview() renders 1-pager with hero stats + table
```

### Empty-vs-zero handling (the most critical part)

```javascript
// hasValue: true only if the input is a real positive number
const hasValue = (v) => v !== '' && v !== null && v !== undefined
                     && !isNaN(Number(v)) && Number(v) > 0;

// num(key): returns the number, or null if not provided
const num = (k) => hasValue(state[k]) ? Number(state[k]) : null;

// calc() returns null for any metric that can't be computed
const out = {
  cpl: (spend && leads) ? spend / leads : null,
  cac: (spend && closedDeals) ? spend / closedDeals : null,
  // ... null propagates through everything
};

// Rendering filters nulls before showing
const allRows = [
  ['CPL', c.cpl !== null ? fmtCurrency(c.cpl, d.currency) : null, ...],
  // ...
].filter(row => row[1] !== null);  // <-- the critical filter
```

When you add new metrics, follow this pattern exactly. Don't shortcut it with `|| 0` fallbacks.

---

## Calculated metrics reference

| Metric | Formula | Available when |
|---|---|---|
| ROAS | Revenue ÷ Spend | Both spend AND revenue have values |
| Net ROI | (Revenue − Spend) ÷ Spend × 100 | Same as ROAS |
| CPA | Spend ÷ Conversions | Both spend AND conversions |
| CPC | Spend ÷ Clicks | Both spend AND clicks |
| CPM | (Spend ÷ Impressions) × 1000 | Both spend AND impressions |
| CTR | Clicks ÷ Impressions × 100 | Both clicks AND impressions |
| CVR | Conversions ÷ Clicks × 100 | Both conversions AND clicks |
| CPL | Spend ÷ Leads | Both spend AND leads |
| CAC | Spend ÷ Closed Deals | Both spend AND closed deals |
| Pipeline | Closed Deals × Avg Deal Value | Both closed deals AND avg deal value |
| Pipeline ROAS | Pipeline ÷ Spend | Pipeline AND spend |
| Lead → SQL | SQL ÷ Leads × 100 | Both SQL AND leads |
| SQL → Close | Closed Deals ÷ SQL × 100 | Both closed deals AND SQL |
| Cost per view | Spend ÷ Video views | Both spend AND video views |
| Cost per engagement | Spend ÷ Engagements | Both spend AND engagements |
| % Lift | (Current − Baseline) ÷ Baseline × 100 | Baseline must have value; inverted for cost metrics |

---

## Three style presets

| Style | When to use | Visual signature |
|---|---|---|
| `editorial` | Premium brands, calm/considered positioning | Cream background, Fraunces serif headlines, magazine-column layout |
| `bold` | High-energy agency pitch | Dark background, lime accent (#D4FF3F), Space Grotesk, big stat blocks |
| `data` | B2B/analytical buyers | White background, IBM Plex, blue accent (#0066FF), before/after table forward |

All three render into the same A4 frame. Switching styles must not change the data — only the visual treatment.

---

## What the user wants Codex to build next

In priority order:

### Priority 1 — Replace template drafters with real LLM calls

`draftHeadline`, `draftChallenge`, `draftOutcomeCallout`, and `draftTestimonial` currently use deterministic templates. Swap them for real LLM calls (Claude API or OpenAI). Keep the function signatures stable — they're called from the AI buttons in the form and from layouts as fallbacks when the user hasn't filled in the field.

Constraints:
- Pass the current `state` + computed metrics from `calc()` as context
- Prompt the LLM to read what data is present (not assumed) and write only about what's available
- Testimonial output must include a "draft only — requires client approval" instruction in the prompt
- Cache LLM responses so the user can re-click without re-paying

### Priority 2 — Saved engagement library (localStorage)

Each anonymized engagement should save as a record so the user doesn't re-type the agency header, descriptor, and channels every time. Add:
- "Save engagement" button that stores current state under a user-defined slug
- "Load engagement" dropdown that lists saved engagements and restores state
- "Delete engagement" with confirmation
- Export/import the library as JSON for backup

Storage key prefix: `casegen:engagement:{slug}`. Store the full state object.

### Priority 3 — Better PDF export

Current export uses `window.print()` which has occasional font-rendering quirks across browsers. Replace with `html2pdf.js` (CDN-loadable, no build step) for higher fidelity. Keep the print path as a fallback. Don't introduce a build step.

### Priority 4 — Agency branding customization

- Logo upload for the agency (file input → FileReader → base64 → render in header)
- Custom accent color picker (overrides the style preset accent)
- Custom font option (limited to Google Fonts to keep it embeddable)

**Do not add client logo upload.** That violates principle 2.

### Priority 5 — Platform API connectors (the big one)

Wire up read-only data pulls from ad platforms so the user picks a date range and the numbers populate. Manual entry stays as the fallback.

- **Meta Marketing API** — spend, impressions, clicks, results by date range, broken down by campaign
- **Google Ads API** — same fields + conversions
- **TikTok Ads API** — for clients using TikTok
- **GA4 Data API** — cross-channel attribution + revenue
- **HubSpot / Salesforce** — leads, SQLs, closed deals for lead-gen

Auth should be OAuth, tokens stored in localStorage (or migrate to a small backend if multi-device sync is wanted later). Each connector is independent — the user might only use Meta + GA4.

### Priority 6 — New focus types

The current five focus types cover most engagements but not all. Likely candidates for future focus types (add only when the user asks):
- **`subscription`** — for SaaS / subscription brands. Hero metrics: trials, paid conversions, LTV, payback period, churn rate
- **`appinstalls`** — for mobile app campaigns. Hero metrics: installs, CPI, in-app actions, retention curves
- **`retention`** — for retention/CRM engagements rather than acquisition. Hero metrics: returning customers, repeat purchase rate, LTV lift

Pattern for adding a focus: add to `detectFocus()` heuristics, add field set in `renderPerfFields()`, add metric calculations in `calc()`, add metric entries to `getAvailableMetrics()`, add draft templates to `draftHeadline()` / `draftOutcomeCallout()` / `draftChallenge()` / `draftTestimonial()`.

---

## What NOT to do

- **Do not add a client name field.** Not optional, not toggleable, not "for internal use only." It doesn't exist in this product.
- **Do not display empty inputs as zeros.** Every render path must check `hasValue()` or filter nulls.
- **Do not change the default endorsement to `testimonial`.** Outcome callout is the safe default for legal reasons.
- **Do not remove the testimonial approval warning.** Even if the UI looks cleaner without it.
- **Do not introduce a build step** (Webpack, Vite, etc.). The single-file no-build property is a feature — it lets the user host on GitHub Pages and edit directly.
- **Do not add tracking, analytics, or telemetry.** The user works with sensitive client data.
- **Do not add a multi-page output by default.** If the user asks for a long-form version, build it as a separate template, not a replacement.
- **Do not assume metrics exist.** Read what the user entered, not what your training data thinks a "good case study" should have.

---

## Hosting on GitHub Pages

```bash
# In the repo
mv case-study-generator.html index.html
git add . && git commit -m "Smart case study generator"
git push
```

Enable Pages in repo settings → point at `main` branch, root directory.

---

## Testing checklist for any change

Before merging anything, verify these scenarios still produce a clean case study:

1. **Sparse lead-gen** — only spend and leads entered, nothing else. Should show "Generated N leads at SAR X CPL" headline, no closed-deals or pipeline rows.
2. **Sparse ecommerce** — only spend and revenue. Should show ROAS, no CPA or conversion-rate rows.
3. **Creative-led, no conversions** — UGC channels, only spend + video views + engagements. Should pick creative focus, show cost-per-view in hero stats.
4. **Built from scratch** — `noBaseline` toggled on, no baseline values. Should show no "Change" column anywhere, headlines should pivot to "from a standing start" framing.
5. **Hide revenue** — `hideRevenue` on. Should show ratios (ROAS, CPL) but suppress dollar revenue and pipeline.
6. **Style switch** — all four scenarios above should render correctly in editorial, bold, and data styles.
7. **Export** — clicking Export should open a print dialog with one page, all three styles rendering with their respective fonts loaded.

Any failure on these = the change is not ready.
