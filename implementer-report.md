# Diwan Sport Performance Audit — Implementer Report

**Site audited:** https://diwansport.com/match (Live Matches / Scores page)
**Audience:** Engineers implementing fixes

This document covers mechanism, reproduction steps, and effort for every finding. For business framing (cost/risk/priority for funding decisions), see the companion stakeholder report — same findings, different lens. Neither document is a summary of the other.

---

## How to Reproduce Everything in This Report

**Lab data (PageSpeed Insights):**
- Tool: [pagespeed.web.dev](https://pagespeed.web.dev), Lighthouse 13.4.0
- URL tested: `https://diwansport.com/match`
- Mobile profile: emulated **Moto G Power**, **Slow 4G** network throttling, 4x CPU slowdown (PSI default mobile profile)
- Desktop profile: no throttling

**Field data (Core Web Vitals):**
- Source: Chrome UX Report (CrUX), pulled automatically by PageSpeed Insights, 28-day rolling window, "This URL" scope

**Networking data:**
- Tool: Chrome DevTools → Network tab
- Cold load: "Disable cache" checked, hard refresh (Ctrl+Shift+R)
- Warm load: "Disable cache" unchecked, normal refresh (Ctrl+R)
- **Known gap:** these captures were not confirmed to be running under network/CPU throttling. If you need networking numbers directly comparable to the mobile PSI profile above, re-run with DevTools Network throttling set to Slow 4G and CPU throttling set to 4x, or device emulation set to Moto G Power.

**What we did NOT check** (flagging explicitly so it doesn't read as an oversight):
- **Offline/service worker support** — not evaluated. No specific product signal suggesting this is needed for a live-scores page; would need a separate scoping conversation if there's a business case for it.
- **Other pages on the site** — the README identifies news, standings/classement, on-demand video, subscription, and login pages as part of the broader site, but this baseline is scoped entirely to the `/match` live-scores page. We have not measured whether the same JS/caching issues apply site-wide, though given the shared JS delivery pattern it's a reasonable assumption worth confirming.
- **Detailed accessibility audit** — we have the PSI accessibility category score (87 mobile / 85 desktop) but did not expand individual accessibility audit items.
- **Performance flame charts, layer/compositing analysis, and animation jank** — not completed for this project. Flagging as an open item for a follow-up pass if animation/scroll smoothness becomes a concern.

---

## Findings

Each finding includes: **Mechanism**, **Reproduce**, **Fix**, **Effort**, and whether it's **Structural** (architectural change, broader buy-in) or **Local** (contained code/config change).

---

### Finding 1 — Oversized JS bundle blocks first and largest paint

**Metrics:** FCP 14.2s, LCP 22.3s, Speed Index 14.2s (all mobile lab)

**Mechanism:** Live match data likely isn't available until multiple API calls resolve (scores, team data, live status), and the page appears to wait on render-blocking JS/CSS before painting the largest element. Unoptimized/unprioritized images (team logos, banners) likely compound this.

**Reproduce:** PSI mobile on `https://diwansport.com/match` → Metrics section → FCP/LCP/Speed Index values, all flagged red.

**Fix:** Preload the actual LCP element (`<link rel="preload">` on the hero/first score image); render initial match data server-side or via a fast initial payload instead of waiting on client-side JS; defer everything non-critical until after main content paints.

**Effort:** Medium-High — the preload piece is quick; moving data-fetching to be server-rendered or fetched earlier is a more significant architectural change.

**Structural or Local:** Structural (data-fetching architecture).

---

### Finding 2 — Heavy main-thread JS execution blocks interaction

**Metrics:** TBT 610ms (lab), INP 221ms (field — confirmed "Failed" real-user assessment)

**Mechanism:** Long JavaScript tasks — likely live-score polling/WebSocket handling, ad scripts, and analytics — execute on the main thread and block input processing. This is distinct from Finding 1: the issue here is execution cost after load, not render delay.

**Reproduce:** PSI mobile → TBT value. For the field-confirmed version, check the Core Web Vitals field-data card at the top of the PSI report — INP shown separately from the lab metrics.

**Fix:** Break up long JS tasks (code-splitting, `requestIdleCallback`/scheduler APIs for non-urgent work); defer third-party scripts until after the page is interactive; audit ad/tracking script overhead.

**Effort:** Medium — requires identifying and restructuring the specific long-running tasks (likely the live-score update mechanism).

**Structural or Local:** Local, assuming the live-score update logic is first-party code (verify — if it's a third-party widget, this becomes Structural).

---

### Finding 3 — Layout shifts from unreserved space for images/dynamic content

**Metrics:** CLS 0.175 (lab); field CLS is currently 0

**Mechanism:** Images (team logos, banners) or dynamically-injected content (live score updates, ad slots) likely load without reserved space, pushing layout around after initial render. The field-data discrepancy (0 vs. 0.175) suggests this may be inconsistent — possibly dependent on network conditions or specific content states not always hit.

**Reproduce:** PSI mobile → CLS value (lab). Compare against the Core Web Vitals field-data card's CLS value for the same URL — if they diverge like this, treat it as an emerging/inconsistent issue rather than a guaranteed one, and consider testing at different times/match-states to see if it correlates with anything specific (e.g. live score update moments).

**Fix:** Set explicit `width`/`height` (or `aspect-ratio`) on all images and ad containers; avoid injecting live-updating content above existing content without a fixed-size wrapper.

**Effort:** Low — typically a straightforward CSS/markup fix once the specific shifting elements are identified via DevTools (Rendering tab → "Layout Shift Regions").

**Structural or Local:** Local.

---

### Finding 4 — JavaScript is not effectively cached on repeat visits

**Metrics:** On warm reload, JS accounts for 2,375 kB of the total 2,390 kB re-transferred (essentially 100%), while images and CSS show 0 kB re-transferred (fully cached)

**Mechanism:** JS files likely lack long-lived cache headers (`Cache-Control: max-age`/`immutable`) or aren't served with content-hashed filenames, so the browser can't safely reuse the cached copy across visits — even though images and CSS demonstrably do cache correctly under the current setup.

**Reproduce:** DevTools Network tab: do a cold reload (cache disabled) then a warm reload (cache enabled, normal refresh); filter by JS and compare the "Size" column between the two — JS should show near-identical transfer size both times, while filtering by Img or CSS should show near-zero transfer on the warm reload.

**Fix:** Serve JS bundles with content-hashed filenames and long `max-age`/`immutable` cache headers so unchanged code is never re-downloaded. Consider a service worker for additional repeat-visit speed. Model this on whatever configuration is currently working correctly for images/CSS (Good Finding 1 below) — same principle just needs extending to JS.

**Effort:** Low — this is primarily a CDN/server header configuration change, not a code rewrite. **Highest return-on-effort finding in this report.**

**Structural or Local:** Local.

---

### Finding 5 — High request count adds latency overhead independent of payload size

**Metrics:** 77 total requests (cold load): 29 JS, 21 images, 6 CSS, ~21 other

**Mechanism:** With JS concentrated in relatively few large files (29 requests for 5.77 MB), the bulk of the 77 total requests comes from images, CSS, and likely third-party scripts firing independently rather than a consolidated delivery strategy. Each adds connection/latency overhead, compounding on a throttled mobile connection.

**Reproduce:** DevTools Network tab, cold reload, read total request count from the bottom status bar; filter by type to see the breakdown.

**Fix:** Consolidate first-party assets where practical (image sprites/icon fonts, bundling small CSS files); audit third-party scripts to remove or defer any non-essential to the initial experience.

**Effort:** Medium — requires auditing what's actually generating the ~21 "other" requests before a specific fix can be scoped.

**Structural or Local:** Local, unless the "other" requests turn out to be predominantly third-party (would become Structural).

---

### Finding 6 — Weak compression on first load relative to payload size

**Metrics:** 7.5 MB of resources → 6.3 MB transferred on cold load ≈ 16% compression reduction

**Mechanism:** Given JS makes up the bulk of page weight, a ~16% overall compression reduction suggests the largest assets aren't compressed as efficiently as possible — likely served without Brotli (falling back to weaker/no compression), and/or images not served in modern formats (WebP/AVIF).

**Reproduce:** DevTools Network tab, cold reload, compare "Resources" total (uncompressed size) vs. "Transferred" total (bytes over the wire) in the bottom status bar. For a per-file check, inspect Response Headers on individual JS files for a `content-encoding: br` (Brotli) or `gzip` value — its absence or use of gzip instead of Brotli confirms the mechanism.

**Fix:** Enable Brotli compression at the server/CDN level for all text-based assets (JS, CSS); convert images to WebP/AVIF with appropriate quality settings.

**Effort:** Low — typically a server/CDN configuration toggle, not application code changes.

**Structural or Local:** Local.

---

### Finding 7 — CPU throttling disproportionately amplifies the JS execution cost on mobile

**Metrics:** Performance score 32 (mobile) vs. 62 (desktop) — a 30-point gap; TBT (mobile lab) and INP (221ms mobile field vs. 78ms desktop field) both show the same pattern

**Mechanism:** The same ~5.77 MB JS bundle (Finding 1's payload) is downloaded and executed on both platforms, but mobile PSI testing applies 4x CPU throttling to simulate a mid-tier device — the identical JS work takes roughly 4x longer to parse/execute on mobile hardware, compounding the download-time penalty from Slow 4G network throttling on top.

**Reproduce:** Compare Performance scores and TBT/INP values between the Mobile and Desktop PSI runs for the same URL — the gap itself is the evidence; no additional tooling needed beyond what's already been run.

**Fix:** Beyond the general bundle-size fixes in Finding 1, consider adaptive loading for mobile/low-end devices — using `navigator.connection` or `navigator.deviceMemory` to serve a lighter-weight bundle (fewer non-critical scripts, lower-resolution images) to constrained devices rather than shipping identical payloads to every client.

**Effort:** Medium-High — adaptive loading based on client signals is a genuine architectural addition, not a config change.

**Structural or Local:** Structural.

---

### Finding 8 (Informational, not corrective) — Real mobile users fail Core Web Vitals while desktop users pass

**Metrics:** Overall Core Web Vitals Assessment — Mobile: Failed, Desktop: Passed, driven by INP (221ms mobile vs. 78ms desktop)

**Mechanism:** This is field data — real Chrome users, not a lab simulation — confirming the lab-measured mobile/desktop gap (Finding 7) shows up in real-world usage at scale (28-day CrUX aggregate), not just in a single throttled test run.

**Reproduce:** PSI report → Core Web Vitals field-data card at the top → compare Mobile vs. Desktop tabs.

**Why this isn't listed as a separate corrective action:** It doesn't have its own independent fix — it's evidence that Finding 7's fix is genuinely urgent, not optional or theoretical.

---

## Good Findings (Don't Touch These)

**Images and CSS cache correctly on repeat visits** (0 kB re-transferred on warm reload for both). This is a working configuration — when implementing Finding 4's JS caching fix, replicate whatever cache-header/CDN setup is already correctly applied to images and CSS rather than inventing a new approach.

**Best Practices and SEO both score a perfect 100/100**, on both mobile and desktop. No flagged console errors, secure/valid protocol usage, complete on-page SEO signals. None of the fixes above should require touching anything related to these — if a proposed fix seems to risk this, that's a signal to reconsider the approach.
