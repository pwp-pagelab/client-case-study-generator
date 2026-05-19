# Case Study Generator — Performance Marketing

A single-file, framework-free HTML generator that produces anonymized 1-pager PDF case studies for performance marketing engagements.

**Live file:** `case-study-generator.html` — open it in a browser, no build step.

---

## Design principles (read before changing anything)

1. **No client names, ever.** Clients are always referred to by anonymous descriptors (e.g. *"A US-based DTC skincare brand"*). The data model has no `clientName` field by design. Don't add one.
2. **No fake testimonials.** AI-drafted endorsements come in three modes:
   - `outcome` — agency-voice callout, no quote marks, no attribution. Always publishable.
   - `testimonial` — drafted in client voice but **explicitly marked as "for client approval"**. Must never be published without written client sign-off. UI warns about this.
   - `none` — omit entirely.
3. **Partial data is normal.** Real engagements often lack revenue, baseline, or both. The generator handles:
   - `hideRevenue` — clients who won't share dollar amounts; ratios and lifts still work.
   - `noBaseline` — clients who started from scratch; reframes the narrative as "0 → achieved".
4. **All numbers are derived where possible.** User enters raw values (spend, impressions, clicks, conversions/leads/deals); the generator computes ROAS, CPA, CPL, CAC, CTR, CVR, CPC, lifts.

---

## File structure

It's intentionally one file — easier to host on GitHub Pages, easier for Codex to extend, easier to embed.

```
case-study-generator.html
├── <style>           — all CSS, organized by section
├── HTML body         — input form (view-input) + preview stage (view-preview)
└── <script>
    ├── state         — single source of truth
    ├── formatters    — fmtCurrency, fmtNumber, fmtPct, fmtROAS, lift, liftInverse
    ├── calc()        — all derived metrics
    ├── draft*()      — AI-style template drafters for headlines, outcome callouts, testimonials
    ├── render*()     — DOM updates for inputs, computed strip, 1-pager preview
    ├── layouts       — editorialLayout, boldLayout, dataLayout
    └── bindAllInputs — event wiring + export
```

---

## Calculated metrics

| Metric | Formula | Notes |
|---|---|---|
| ROAS | Revenue ÷ Spend | Ecommerce only; hidden if `hideRevenue` |
| Net ROI | (Revenue − Spend) ÷ Spend × 100 | Ecommerce only |
| CPA | Spend ÷ Conversions | Both modes |
| CTR | Clicks ÷ Impressions × 100 | Both modes |
| CVR | Conversions ÷ Clicks × 100 | Both modes |
| CPC | Spend ÷ Clicks | Both modes |
| CPM | (Spend ÷ Impressions) × 1000 | Both modes |
| CPL | Spend ÷ Leads | Lead-gen only |
| CAC | Spend ÷ Closed Deals | Lead-gen only |
| Pipeline ROAS | (Closed Deals × Avg Deal Value) ÷ Spend | Lead-gen only |
| Lead → SQL % | SQL ÷ Leads × 100 | Lead-gen only |
| SQL → Close % | Closed ÷ SQL × 100 | Lead-gen only |
| % Lift | (Current − Baseline) ÷ Baseline × 100 | Inverted for cost metrics |

---

## Three style presets

| Style | When to use |
|---|---|
| `editorial` | Premium brands, calm/considered positioning, magazine vibe |
| `bold` | High-energy agency pitch, dark with lime accent |
| `data` | B2B/analytical buyers, before-after table forward |

All three render into the same A4 (210×297mm) frame and export via `window.print()`.

---

## Roadmap (suggested next steps for Codex)

1. **Replace template drafters with real LLM calls.** The `draftHeadline`, `draftOutcomeCallout`, `draftTestimonial` functions currently use deterministic templates. Swap them for calls to your LLM of choice (Claude API, OpenAI, etc.). Keep the function signatures stable so the UI doesn't change.
2. **Saved client library.** Add localStorage persistence so each anonymized engagement saves as a record. Add a "Load engagement" dropdown.
3. **Multi-page case studies.** Some buyers want 2–3 pages with channel breakdowns. Add a page-count selector and split layouts.
4. **Logo upload for agency branding.** File input → FileReader → base64 → render in header. Client logo is **never** added (per anonymization rule).
5. **Platform API connectors** (the big one). Wire up:
   - Meta Marketing API → spend/impressions/clicks/CTR/results by date range
   - Google Ads API → same fields + conversions
   - GA4 Data API → cross-channel attribution
   - HubSpot/Salesforce → leads, SQLs, closed deals for lead-gen
   
   Build these as optional. Manual entry stays as the fallback.
6. **PDF rendering via `html2pdf.js` or `pdf-lib`** for higher fidelity than `window.print()`. The print path works fine for now but has occasional font-rendering quirks across browsers.
7. **Theme customization** — allow agency to define accent color, fonts, and logo placement to make case studies on-brand.

---

## Hosting on GitHub Pages

```bash
# In a new repo
echo "case-study-generator.html" > index.html.placeholder
mv case-study-generator.html index.html
git add . && git commit -m "Initial case study generator"
git push
```

Then enable Pages in repo settings, point at `main` branch, root directory. Done.

---

## Legal note on anonymization

Anonymizing case studies still requires the client's permission **unless** the descriptor is generic enough that the client cannot reasonably be identified. "A US DTC skincare brand we worked with in 2026" describes hundreds of companies and is generally safe. "The only DTC men's beard-oil brand backed by [VC]" identifies one company and is not. Use the size + vertical + geography descriptors conservatively.

For testimonials — even anonymized ones — get written client approval before publishing. The FTC, ASA, and most consumer-protection regulators treat fabricated endorsements as deceptive advertising regardless of attribution detail.
