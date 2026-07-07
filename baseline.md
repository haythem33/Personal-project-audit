# Performance Baseline

**Site:** Diwan Sport — [https://diwansport.com/](https://diwansport.com/)
**Primary page tested:** [https://diwansport.com/match](https://diwansport.com/match) (Live Matches / Scores page)
**Conditions:** **Mobile** — Lighthouse 13.4.0, emulated **Moto G Power**, **Slow 4G network throttling** with 4x CPU slowdown (lab data); Chrome DevTools Network tab (networking data). All metrics below are **mobile** unless a row is explicitly labeled Desktop.

---

## Metrics Summary

### PageSpeed Insights — Category Scores (Mobile vs. Desktop)

| Category | Mobile | Desktop |
|---|---|---|
| Performance | 32 🔴 | 62 🟠 |
| Accessibility | 87 🟠 | 85 🟠 |
| Best Practices | 100 🟢 | 100 🟢 |
| SEO | 100 🟢 | 100 🟢 |

### Core Web Vitals / Rendering — Lab Data (Mobile)

| Metric | Value | Status |
|---|---|---|
| First Contentful Paint (FCP) | 14.2 s | 🔴 |
| Largest Contentful Paint (LCP) | 22.3 s | 🔴 |
| Total Blocking Time (TBT) | 610 ms | 🔴 |
| Cumulative Layout Shift (CLS) | 0.175 | 🟠 |
| Speed Index | 14.2 s | 🔴 |

*Lab data is Mobile-only in this baseline (Moto G Power, Slow 4G, 4x CPU throttling) since this is the profile discussed in class and the one that best represents real-world constrained conditions.*

### Core Web Vitals — Field Data (Mobile vs. Desktop, real users, 28-day period)

| Metric | Mobile | Desktop |
|---|---|---|
| Largest Contentful Paint (LCP) | 2.2 s 🟢 | 1.5 s 🟢 |
| Interaction to Next Paint (INP) | 221 ms 🔴 | 78 ms 🟢 |
| Cumulative Layout Shift (CLS) | 0 🟢 | 0.1 🟠 |
| First Contentful Paint (FCP) | 1.9 s 🟠 | 1.2 s 🟢 |
| Time to First Byte (TTFB) | 1.1 s 🟠 | 0.5 s 🟢 |
| **Overall Assessment** | **Failed** 🔴 | **Passed** 🟢 |

*Note the gap between lab and field data: real users' devices/networks vary, and CrUX only reflects the origin/URL sample it collects — but the failed INP in the field confirms the lab-measured TBT is a real, mobile-specific, user-facing problem, not just a synthetic artifact.*

### Networking (Mobile — Chrome DevTools Network tab)

> Note: these captures reflect cache on/off state (as tested) but throttling wasn't explicitly confirmed for this pass. For full consistency with the lab rendering data above, re-run with Network throttling set to **Slow 4G** and device emulation set to **Moto G Power** in DevTools if you want networking numbers directly comparable to the PSI mobile profile.

| | Cold Load (cache disabled) | Soft Refresh (cache enabled) |
|---|---|---|
| Requests | 77 | 74 |
| Data Transferred | 6.3 MB | 2.4 MB |
| Total Resource Size | 7.5 MB | 7.5 MB |
| Load Time | 2.06 s | 1.20 s |

- **Compression reduction (cold load):** 7.5 MB → 6.3 MB ≈ **16%**
- **Caching reduction (soft refresh):** 6.3 MB → 2.4 MB ≈ **62%**, though request count barely drops (77 → 74)

**Breakdown by resource type (cached/soft-refresh session):**

| Type | Requests | Transferred | Resource Size | Share of Total |
|---|---|---|---|---|
| JS | 29 / 76 | 2,375 kB | 5,768 kB | ~77% |
| Images | 21 / 76 | 0 kB (cached) | 1,002 kB | ~13% |
| CSS | 6 / 76 | 0 kB (cached) | 457 kB | ~6% |
| Other (fonts/XHR/doc) | ~20 / 76 | ~15 kB | ~259 kB | ~3% |

---

## Findings

Each finding below represents one distinct, independently-observable root cause. Metrics that share a root cause (e.g. FCP, LCP, and Speed Index all delayed by the same render-blocking bundle) are grouped as a single finding rather than listed separately.

### Corrective Finding 1: A single oversized, unsplit JS bundle blocks first and largest paint

**Metric(s) affected:** First Contentful Paint (14.2s), Largest Contentful Paint (22.3s), Speed Index (14.2s)

**How it affects users:** For 14+ seconds users see a completely blank screen, and the actual live-score content doesn't appear for over 22 seconds. On a page whose entire purpose is checking scores quickly, this makes the page feel broken rather than just slow — most users will bounce before anything renders.

**Cause (likely):** A large JS bundle (5.77 MB, ~77% of total page weight) is not split by priority, so the browser must download and begin executing it before it can paint anything — including content unrelated to the initial view (ads, analytics, non-visible match data).

**Solution (likely):** Code-split the bundle so only what's needed for the first paint loads up front; defer ad/analytics scripts and below-the-fold logic; preload the actual LCP element (e.g. the first score card/hero image) directly rather than waiting on it to be requested by JS.

---

### Corrective Finding 2: Heavy main-thread JS execution blocks user interaction

**Metric(s) affected:** Total Blocking Time (610 ms, lab), Interaction to Next Paint (221 ms, field — confirmed "Failed" real-user assessment)

**How it affects users:** Even after content appears, taps on match filters or score cards may feel unresponsive for over half a second at a time — frustrating on a page meant to be checked quickly and repeatedly throughout the day.

**Cause (likely):** Long JavaScript tasks — likely live-score polling/websocket handling, ad scripts, and analytics all executing on the main thread — block input processing. This is distinct from Finding 1: the issue here is execution cost after load, not download/paint delay.

**Solution (likely):** Break up long tasks (code-splitting, `requestIdleCallback`/scheduler APIs for non-urgent work), defer third-party scripts until after the page is interactive, and audit/reduce ad and tracking script overhead.

---

### Corrective Finding 3: Layout shifts from unreserved space for images/dynamic content

**Metric(s) affected:** Cumulative Layout Shift (0.175, lab); field CLS is currently 0, suggesting this is an emerging/inconsistent issue worth monitoring rather than universally present for all users.

**How it affects users:** As live scores, team logos, or ads load in, nearby content likely shifts position — risking mis-taps on match cards as elements move underneath a user's finger.

**Cause (likely):** Images or dynamically-injected content (live score updates, ad slots) loading without reserved space in the layout.

**Solution (likely):** Set explicit `width`/`height` (or `aspect-ratio`) on all images and ad containers so space is reserved before content loads, and avoid injecting live-updating content above existing content without a fixed-size wrapper.

---

### Corrective Finding 4: JavaScript is not effectively cached on repeat visits

**Metric(s) affected:** Cached Transfer Size / repeat-visit load performance

**How it affects users:** On a soft refresh, images and CSS were served entirely from cache (0 kB re-downloaded), but JS still transferred 2,375 kB — nearly identical to the cold-load size. For a site people check repeatedly for live scores, this means almost every visit pays the full JS download cost again.

**Cause (likely):** JS files likely lack long-lived cache headers (`Cache-Control: max-age`/`immutable`) or aren't served with content-hashed filenames, so the browser can't safely reuse the cached copy.

**Solution (likely):** Serve JS with content-hashed filenames and long `max-age`/`immutable` cache headers so unchanged code is never re-downloaded; consider a service worker for repeat-visit speed.

---

### Corrective Finding 5: High request count adds latency overhead independent of payload size

**Metric(s) affected:** Number of Requests (77) — a contributor to overall load time distinct from total byte weight, especially over a throttled/high-latency mobile connection.

**How it affects users:** Each request carries connection/latency overhead. With JS concentrated in relatively few large files (29 requests for 5.77 MB), the bulk of the 77 total requests comes from images, CSS, and likely third-party scripts firing independently — each adding round-trip latency that compounds on a slow connection.

**Cause (likely):** Unbundled/unconsolidated first-party assets (icons, small images) combined with several independent third-party requests (ads, analytics, live-score/streaming infrastructure) rather than a consolidated delivery strategy.

**Solution (likely):** Consolidate first-party assets where practical (image sprites/icon fonts, bundling small CSS files), and audit third-party scripts to remove or defer any that aren't essential to the initial experience.

---

### Corrective Finding 6: Weak compression on first load relative to payload size

**Metric(s) affected:** Compression Ratio / Data Transferred (cold load), contributing to FCP, LCP, and Speed Index

**How it affects users:** Given JS makes up the bulk of page weight, an overall compression reduction of only ~16% suggests the largest assets aren't compressed as efficiently as they could be — users download more bytes than necessary before the page can render, which is especially costly on mobile networks.

**Cause (likely):** Assets may be served without Brotli compression (falling back to weaker/no compression), and images may not be served in modern, better-compressing formats (WebP/AVIF).

**Solution (likely):** Enable Brotli compression at the server/CDN level for all text-based assets (JS, CSS), and convert images to WebP/AVIF with appropriate quality settings.

---

### Good Finding 1: Images and CSS cache effectively on repeat visits

**Metric(s) affected:** Cached Transfer Size (Images and CSS: 0 kB transferred on soft refresh)

**Why this is good:** Unlike JS, images and CSS were served entirely from cache on the soft refresh — meaning the site's caching strategy works well for these asset types, and repeat visitors aren't re-downloading logos, banners, or stylesheets unnecessarily. This is a solid foundation to build on when fixing JS caching (Corrective Finding 4).

---

### Good Finding 2: Best Practices and SEO scores are excellent

**Metric(s) affected:** Best Practices (100/100), SEO (100/100)

**Why this is good:** Despite the significant performance issues, the site follows solid technical fundamentals — no flagged console errors, secure/valid protocol usage, and complete on-page SEO signals (crawlability, indexability, metadata). This suggests the performance problems are isolated to specific implementation choices (bundle size, caching, compression) rather than systemic engineering issues across the site.

---

## Mobile-Specific Findings

These findings are specifically about the *gap* between mobile and desktop, not just mobile numbers in isolation — grounded in the mobile-vs-desktop comparisons captured in the Metrics Summary above.

### Corrective Finding 7: CPU throttling disproportionately amplifies the JS execution cost on mobile

**Metric(s) affected:** Performance score (32 mobile vs. 62 desktop — a 30-point gap), Total Blocking Time (mobile lab), Interaction to Next Paint (221 ms mobile field vs. 78 ms desktop field)

**How it affects users:** Mobile users experience a dramatically worse version of the same page than desktop users — not just "somewhat slower," but a categorically different experience (Performance essentially doubles from red to orange, and Core Web Vitals go from Failed to Passed) purely by switching device class.

**Cause (likely):** The same ~5.77 MB JS bundle (Corrective Finding 1) is downloaded and executed on both platforms, but mobile testing applies 4x CPU throttling to simulate a mid-tier device — meaning the identical amount of JS work takes roughly 4x longer to parse and execute on mobile hardware, compounding the download-time penalty from Slow 4G network throttling on top of it.

**Solution (likely):** Beyond the general bundle-size fixes in Finding 1, consider adaptive loading specifically for mobile/low-end devices — e.g. using `navigator.connection` or `navigator.deviceMemory` to serve a lighter-weight bundle (fewer non-critical scripts, lower-resolution images) to constrained devices rather than shipping the same payload to every client regardless of capability.

---

### Finding 8: Real mobile users fail Core Web Vitals while desktop users pass

**Metric(s) affected:** Overall Core Web Vitals Assessment (Mobile: Failed, Desktop: Passed), driven by Interaction to Next Paint (221 ms mobile vs. 78 ms desktop)

**How it affects users:** This is field data — real Chrome users, not a lab simulation — confirming the mobile/desktop gap isn't a testing artifact. A meaningful share of real mobile visitors are having a genuinely unresponsive experience tapping match filters or score cards, while desktop visitors are not.

**Why this matters as a separate finding:** It's tempting to treat this as "the same issue as Finding 7," but it's independently valuable: it confirms the lab-measured mobile/desktop gap actually shows up in real-world usage at scale (CrUX's 28-day aggregate), not just in a single throttled test run — which changes how urgently the mobile-specific fixes in Finding 7 should be prioritized.

---

*Next steps: prioritize Corrective Findings 1 and 2 first (JS bundle size and execution cost), since they have the largest measured impact and touch the most metrics (FCP, LCP, Speed Index, TBT, and field-confirmed INP failure).*
