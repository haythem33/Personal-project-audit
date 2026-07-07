# Baseline Findings — Core Web Vitals & PageSpeed Insights

**Page tested:** [https://diwansport.com/match](https://diwansport.com/match) (Live Matches / Scores page)
**Test conditions:** Mobile, Lighthouse 13.4.0, emulated Moto G Power, slow 4G throttling
**Overall Mobile Performance score:** 32 🔴

| Metric | Value | Status |
|---|---|---|
| First Contentful Paint (FCP) | 14.2 s | 🔴 |
| Largest Contentful Paint (LCP) | 22.3 s | 🔴 |
| Total Blocking Time (TBT) | 610 ms | 🔴 |
| Cumulative Layout Shift (CLS) | 0.175 | 🟠 |
| Speed Index | 14.2 s | 🔴 |

---

## Finding 1: Largest Contentful Paint (LCP) — 22.3s

**Metric(s) affected:** LCP — the single worst score on the page.

**How it affects users:** Users on the matches page won't see the main content — the live scores, team logos, match cards — for over 22 seconds. On a page whose entire purpose is showing live scores quickly, this is close to unusable; most users will bounce long before anything meaningful renders.

**Cause (likely):** The live match data probably isn't available until multiple API calls resolve (scores, team data, live status), and the page may be waiting on render-blocking JS/CSS before it can paint the largest element. Unoptimized images (team logos, banners) not being prioritized for early load likely compounds this.

**Solution (likely):** Preload the LCP element (e.g. `<link rel="preload">` on the hero/first score image), fetch and render initial match data server-side or with a fast initial payload rather than waiting on client-side JS, and defer everything non-critical until after the main content paints.

---

## Finding 2: First Contentful Paint & Speed Index — 14.2s (tied)

**Metric(s) affected:** FCP and Speed Index.

**How it affects users:** For over 14 seconds, users see a completely blank screen — no indication the page is even loading. This reads as a broken or frozen site, especially frustrating for a "check the live score quickly" use case.

**Cause (likely):** Heavy render-blocking resources (CSS/JS) loading before the browser can paint anything, likely combined with large third-party scripts (ads, analytics, live-score infrastructure) all competing for the critical rendering path.

**Solution (likely):** Inline critical above-the-fold CSS, defer/async non-critical JavaScript, and split large JS bundles so only what's needed for the first paint loads up front.

---

## Finding 3: Total Blocking Time — 610ms

**Metric(s) affected:** TBT (and ties directly to the failed mobile INP of 221ms seen in the field/Core Web Vitals data).

**How it affects users:** Even once content appears, the page stays unresponsive to taps/clicks for over half a second at a time — tapping a match filter or score card may feel like it's not registering, which is especially bad on a page meant to be checked quickly and repeatedly.

**Cause (likely):** Heavy JavaScript execution on the main thread — likely from live-score polling/websocket handling, ad scripts, and analytics/tracking all running long tasks that block user input.

**Solution (likely):** Break up long JavaScript tasks (code-splitting, `requestIdleCallback` for non-urgent work), defer third-party scripts until after interaction is possible, and audit/reduce ad and tracking script bloat.

---

## Finding 4: Cumulative Layout Shift — 0.175

**Metric(s) affected:** CLS — the least severe of the five metrics, but still outside the "good" range (≤ 0.1).

**How it affects users:** As live scores and images load in, page elements likely jump around — a real risk on this page, since users may go to tap a match card and hit something else as content shifts underneath their finger.

**Cause (likely):** Images (team logos, banners) or dynamically-injected content (live score updates, ads) loading without reserved space, pushing layout around after initial render.

**Solution (likely):** Set explicit `width`/`height` (or `aspect-ratio`) on all images and ad slots so space is reserved before they load, and avoid injecting live-updating content above existing content without a fixed-size container.
