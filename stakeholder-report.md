# Diwan Sport Performance Audit — Executive Summary

**Site audited:** diwansport.com (Live Matches / Scores page)
**Prepared for:** Product/Engineering leadership
**Bottom line:** The live-scores page — the core product for a sports platform — is measurably slow on mobile, and real users are failing Google's Core Web Vitals today because of it. The good news: our scoring model identified the single highest-value fix as also the *cheapest* one to ship. This is a short, high-leverage punch list, not a rebuild.

---

## TL;DR — For the 30-Second Read

| | |
|---|---|
| **Current state** | Mobile Performance score of 32/100 (poor) on the live-scores page; real mobile users fail Core Web Vitals while desktop users pass |
| **Root cause** | An oversized JavaScript bundle that's both too big to begin with, and re-downloaded almost in full on every single repeat visit |
| **Is the foundation sound?** | Yes — caching works correctly for images and stylesheets, layout is visually stable on real devices, and the site's code quality/SEO fundamentals score perfectly (100/100) |
| **Cheapest high-impact fix** | A server configuration change (cache headers) that our scoring model ranks as the #1 priority fix — not because it's the scariest problem, but because it's nearly free and fixes the most-repeated pain point: this site is checked over and over throughout the day |
| **You can verify this yourself** | Paste `diwansport.com/match` into [pagespeed.web.dev](https://pagespeed.web.dev) right now |

---

## What This Is Costing the Business

- **Engagement/repeat-visit cost:** This is a live-scores platform — by design, users check it repeatedly throughout a match, not once. Right now, **every single repeat visit re-downloads almost the entire JavaScript payload** rather than benefiting from caching. That's a self-inflicted tax on exactly the usage pattern this product depends on.
- **Real-user reliability risk:** Google's own real-world data (not a lab estimate) shows mobile visitors **failing** Core Web Vitals, while desktop visitors pass. Since mobile is very likely the majority-usage device for checking live scores on the go, this gap disproportionately affects the platform's core use case.
- **Subscription/conversion risk:** Live streaming and on-demand content sit behind a subscription paywall on this platform. A visitor who has a poor first impression checking free live scores has less reason to trust the paid streaming experience will be any better — slow performance on the "front door" page undercuts the pitch for the product it's meant to funnel toward.

---

## The Evidence, In Terms Anyone Can Check

We used **Google PageSpeed Insights** (pagespeed.web.dev), the same public tool Google itself recommends. Anyone can re-run this in under a minute:

1. Go to pagespeed.web.dev
2. Paste `https://diwansport.com/match`
3. Click Analyze

**What it shows today:**

| | Mobile | Desktop |
|---|---|---|
| Overall Performance Score (0–100) | **32** | 62 |
| Real-user Core Web Vitals | **Failed** | Passed |

A mobile score of 32 sits in the tool's own "poor" (red) range — this is Google's categorization of Google's own measurement, not our interpretation.

---

## What's Already Working

- **Visual stability is genuinely good for real users.** While our stress-test conditions showed a borderline layout-shift number, actual visitors experience zero measurable layout shift — the page doesn't jump around as content loads.
- **Caching works correctly for images and stylesheets.** On a repeat visit, these assets are served instantly from cache exactly as they should be — proving the team already knows how to do this right; it's simply not been extended to JavaScript yet (see recommendations below).
- **Code quality and search fundamentals are excellent.** Best Practices and SEO both score a perfect 100/100 — no flagged security or crawlability issues. This is not a site with foundational engineering problems; it's a site with a specific, fixable performance gap.

---

## Ranked Recommendations

Ranked using a weighted scoring model (severity × how many users it touches × how confident we are in the diagnosis, divided by how hard the fix is) — deliberately built to surface the best return on investment first, not just the scariest-sounding problem.

### 🟢 Tier 1 — Do This First (Highest score in our model; low cost)

**1. Fix JavaScript caching so repeat visits don't re-download the whole app.**
This is the #1 ranked item — not because it's the most dramatic-sounding issue, but because it combines strong evidence, near-total user reach (every visit, and this is a repeat-visit-heavy product), and a genuinely low implementation cost (primarily a server/CDN configuration change, not a rewrite). This is the highest-leverage single fix available.

### 🟡 Tier 2 — High Value, Moderate Cost

**2. Reduce how much JavaScript blocks the page from responding to taps.**
Real mobile users are failing Google's responsiveness benchmark. This traces to how much work the browser has to do before it can react to a tap or click — a moderate engineering effort (script deferral and task-splitting), but directly tied to the failed real-user assessment.

**3. Reduce the overall size of the JavaScript sent to the page.**
The largest single technical problem in this audit: nearly 5.8 MB of JavaScript for one page. This is the most structurally involved fix in the list — it requires restructuring how code is packaged, not just a configuration tweak — but it's also the most severe issue measured.

### 🔵 Tier 3 — Worthwhile, Lower Urgency

**4. Improve server-side compression.** A relatively low-cost configuration change with a clear, measurable benefit.

**5. Consider a lighter mobile-specific experience.** Mobile users are disproportionately affected by the same underlying JS problem due to typical mobile hardware/network constraints — worth a follow-up investment once Tier 1–2 are addressed, to make sure the fixes actually close the mobile/desktop gap.

**6. Reduce the number of separate network requests the page makes.** Smaller, secondary opportunity.

**7. Fix minor layout-shift risk.** Lowest priority — real users aren't currently experiencing this as a problem, though our stress test flagged a borderline risk worth a look eventually.

---

## Bottom Line for Budget Decisions

- **If we fund nothing else this quarter:** fund Tier 1. It's the single best-scoring item in our model — cheap, high-confidence, and addresses the exact behavior pattern (repeat visits) this product is built around.
- **If we fund one larger initiative:** it should be Tier 2's JavaScript-size reduction — the most severe problem measured, and the one most likely to move the mobile score out of "poor" territory entirely.
- **What we're not recommending:** any rework of the platform's fundamentals. Code quality and SEO are already excellent; this is a targeted performance investment, not a rebuild.
