# Personal Project Performance Audit Report

## Website

**Diwan Sport** — [https://diwansport.com/](https://diwansport.com/)

Diwan Sport is the official streaming/media platform for Tunisia's Ligue 1 and Ligue 2 football, covering live match scores, video streaming/highlights, standings, and sports news.

## Why this site is a good audit candidate

Diwan Sport is a non-tech (sports media/broadcasting) company, and its platform touches nearly every content type an auditor cares about:

- **Static content** — news articles, "About/Privacy" pages
- **Dynamic content** — live match scores and schedules that update in real time, standings tables
- **Interactive features** — match filters (by date/competition/country), on-demand video playlists, live video streaming player
- **In-page loaders** — live score widgets that refresh without a full page reload, lazy-loaded match cards and team logos
- **Authentication** — subscription sign-up/login required to access live streams and on-demand content
- **Third-party embeds** — video player/streaming infrastructure, analytics

This mix of a real-time live-scores dashboard, video streaming, and subscription gating already produces a confirmed poor mobile score on at least one page, while simpler pages like the news section should score much better — giving a good spread for the audit.

## PageSpeed Insights Scores

Tested against **https://diwansport.com/match** (live matches/scores page):

| Category | Mobile | Desktop |
|---|---|---|
| Performance | **32** 🔴 | 62 🟠 |
| Accessibility | 87 🟠 | 85 🟠 |
| Best Practices | 100 🟢 | 100 🟢 |
| SEO | 100 🟢 | 100 🟢 |

> Run **[pagespeed.web.dev](https://pagespeed.web.dev)** against `https://diwansport.com/` (the homepage) as well, and add that row below for comparison against the confirmed red match-page score.

| Category | Mobile | Desktop |
|---|---|---|
| Performance | _TBD_ | _TBD_ |
| Accessibility | _TBD_ | _TBD_ |
| Best Practices | _TBD_ | _TBD_ |
| SEO | _TBD_ | _TBD_ |

## Pages to Include in the Audit

1. **Homepage** — [https://diwansport.com/](https://diwansport.com/)
   The main entry point, mixing news, live scores previews, and streaming promotion; sets the baseline for the rest of the audit.

2. **Live Matches / Scores Page** — [https://diwansport.com/match](https://diwansport.com/match)
   **Confirmed red mobile Performance score (32).** The real-time score widgets, filters, and constant data refresh make this the heaviest dynamic page on the site — the core "poor performance" example for the report.

3. **News Page** — [https://diwansport.com/news](https://diwansport.com/news)
   Mostly static/article-style content; useful as a comparison point to show what the site looks like without live data widgets.

4. **Standings / Classement Page** — [https://diwansport.com/classement](https://diwansport.com/classement)
   A structured data table (league standings) — good for testing how well tabular, semi-static content performs versus the live match page.

5. **On-Demand Video Page** — [https://diwansport.com/ondemand](https://diwansport.com/ondemand)
   Video playlists and highlights; tests how the site handles heavy media thumbnails and streaming previews.

6. **Subscription Page** — [https://diwansport.com/subscription](https://diwansport.com/subscription)
   Represents the monetization/paywall flow, with pricing tiers and sign-up forms.

7. **Login / Account Page**
   Represents the authentication flow gating access to live streams and on-demand content — important to test separately from public pages.

8. **A Live Stream / Individual Match Page** — click into any live match from the matches page
   Combines authentication gating, an embedded video player, and real-time score updates all on one page — likely the most demanding page on the whole site.

---
*Fill in the homepage PageSpeed Insights table above, and consider testing a couple of the other pages (e.g. news, classement) to round out the comparison before finalizing the report.*
