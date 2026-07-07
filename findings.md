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

---
---

# Baseline Findings — Networking Stats

**Page tested:** [https://diwansport.com/match](https://diwansport.com/match) (Live Matches / Scores page)
**Tool:** Chrome DevTools → Network tab

## Overall Numbers

| | Cold Load (cache disabled) | Soft Refresh (cache enabled) |
|---|---|---|
| Requests | 77 | 74 |
| Data Transferred | 6.3 MB | 2.4 MB |
| Total Resource Size | 7.5 MB | 7.5 MB |
| Load time | 2.06 s | 1.20 s |

- **Compression reduction (cold load):** 7.5 MB of resources → 6.3 MB transferred ≈ **16% reduction**.
- **Caching reduction (soft refresh):** 6.3 MB → 2.4 MB transferred ≈ **62% reduction**, though the request count barely drops (77 → 74), meaning nearly every request still fires — it's just served from cache rather than skipped.

## Breakdown by Resource Type (cached/soft-refresh session)

| Type | Requests | Transferred | Resource Size | Share of total resources |
|---|---|---|---|---|
| JS | 29 / 76 | 2,375 kB | 5,768 kB | ~77% |
| Images | 21 / 76 | 0 kB (cached) | 1,002 kB | ~13% |
| CSS | 6 / 76 | 0 kB (cached) | 457 kB | ~6% |
| Other (fonts/XHR/doc) | ~20 / 76 | ~15 kB | ~259 kB | ~3% |

The key signal: on the cached reload, **JS made up 2,375 kB of the total 2,390 kB re-transferred** — virtually all of it — while images and CSS were served entirely from cache (0 kB transferred). JS dominates both the page's total weight and its repeat-visit cost.

---

## Finding 5: JavaScript dominates total page weight (~77% of all resources)

**Metric(s) affected:** Total Byte Weight / Resource Size — and by extension LCP, FCP, TBT, since the browser has to download and parse/execute this much JS before the page is usable.

**How it affects users:** Nearly 5.8 MB of JavaScript for a single page is a huge amount to download and process, especially on mobile/slower connections — directly explaining the very poor LCP (22.3s) and TBT (610ms) seen in the Core Web Vitals baseline. Users are paying a heavy download and parsing cost before they can even see or interact with live scores.

**Cause (likely):** A large, unsplit JS bundle — likely including the live-score/streaming logic, ad tech, analytics, and possibly duplicate or unused libraries — all shipped together rather than split by what's actually needed for the initial view.

**Solution (likely):** Code-split the bundle so only what's needed for first paint loads up front (e.g. defer ad/analytics scripts, lazy-load anything below the fold), tree-shake unused code, and audit for duplicate dependencies.

---

## Finding 6: JavaScript isn't benefiting from caching on repeat visits

**Metric(s) affected:** Cached Transfer Size / repeat-visit load performance.

**How it affects users:** On a soft refresh, images and CSS were served entirely from cache (0 kB re-downloaded), but JS still transferred 2,375 kB — almost identical to its cold-load size. For a site people check repeatedly throughout the day for live scores, this means every single visit pays nearly the full JS download cost again, instead of only the first one.

**Cause (likely):** JS files likely lack long-lived cache headers (`Cache-Control: max-age`/`immutable`) or aren't served with content-hashed filenames, so the browser can't safely reuse the cached copy — possibly combined with frequent redeploys invalidating the cache.

**Solution (likely):** Serve JS bundles with content-hashed filenames and long `max-age`/`immutable` cache headers so unchanged code is never re-downloaded, and consider a service worker to cache static assets for offline/repeat-visit speed.

---

## Finding 7: High total request count (77 requests) for a single page

**Metric(s) affected:** Number of Requests — a contributor to TBT and overall load time, especially over a throttled mobile connection.

**How it affects users:** Each request carries connection/latency overhead, and on a slow or high-latency mobile network, 77 separate round-trips compound quickly — adding to the already-poor FCP/LCP and making the page feel even more sluggish than the raw byte count alone would suggest.

**Cause (likely):** Likely a mix of unbundled/many small script chunks plus third-party requests (ads, analytics, live-score/streaming infrastructure) each firing independently rather than being consolidated.

**Solution (likely):** Bundle and consolidate first-party scripts where possible, and audit third-party scripts (ad networks, trackers) to remove or defer any that aren't essential to the initial experience.

---

## Finding 8: Weak overall compression on first load (~16% reduction)

**Metric(s) affected:** Compression Ratio / Data Transferred — directly affecting FCP, LCP, and Speed Index on the cold-load baseline.

**How it affects users:** Given that JS makes up the bulk of the page's weight, a compression reduction of only ~16% overall suggests the largest assets aren't being compressed as efficiently as they could be — meaning users download more bytes than necessary before the page can render, especially costly on mobile networks.

**Cause (likely):** Assets may be served without Brotli compression (falling back to weaker or no gzip compression), and/or images aren't being served in modern, better-compressing formats (WebP/AVIF) alongside the already-large JS payload.

**Solution (likely):** Enable Brotli compression (a stronger algorithm than gzip) at the server/CDN level for all text-based assets (JS, CSS), and convert images to WebP/AVIF with appropriate quality settings to shrink the ~1 MB of image weight further.
