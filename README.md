# AP News Performance Audit

**Website:** The Associated Press — apnews.com
**URL:** https://apnews.com/

## Why This Is a Good Candidate

AP News is a content-heavy news site where speed is the product: readers arrive from
links and searches, mostly on phones, and either the story paints fast or they bounce.
The site mixes several very different page templates — a dense homepage, topic hubs,
photo galleries, article pages, search, and a donation flow — each with its own
performance profile (images, ads, third-party embeds, client-side search). A first look
already showed the pattern worth auditing: the server answers instantly but the page
stays blank while a very large payload loads and runs. The homepage alone makes 926
requests and transfers 10.27 MB.

## Target Pages

Chosen so every template the site serves is covered once, not the same template seven
times:

- https://apnews.com/ — homepage, the main entry point and the densest template.
- https://apnews.com/entertainment — a section front; how users browse by topic.
- https://apnews.com/hub/fifa-world-cup — a topic hub, the template behind every
  ongoing-story page; picked live during the World Cup so it carried real traffic.
- https://apnews.com/photo-gallery/world-cup-photos-mbappe-haaland-jimenez-57fd3b1070ed79152dfa89b7319f6139
  — a photo gallery, the most image-heavy template on the site.
- https://apnews.com/article/world-cup-schedule-results-news-81645977a722c4020c9644d17589bdbb
  — an article page, the template most readers actually land on from links and search.
- https://apnews.com/donate — the conversion page; performance here is trust and money,
  not just reading comfort.
- https://apnews.com/search?q=world+cup — search, the one heavily interactive,
  client-side flow.

## How It Was Measured

- **Field:** Chrome real-user data (CrUX, last 28 days) via PageSpeed Insights, mobile
  and desktop.
- **Lab:** Lighthouse mobile on a throttled setup — Slow 4G (~1.6 Mbps down, 150 ms
  RTT) plus 4× CPU slowdown — so the numbers reflect a mid-range phone on a cell
  connection, not a laptop on office wifi.
- **Network:** homepage HAR export (`apnews.com-1.har`).
- **Unused code:** DevTools Coverage panel. Coverage reports uncompressed bytes and the
  HAR reports transfer bytes — the reports label which one every time they cite a
  number.

## Headline Numbers

- Field mobile LCP **2.9 s** (needs improvement) and FCP **2.2 s** — Core Web Vitals
  **failed** on both mobile and desktop, both because of LCP.
- Lighthouse performance: **40 mobile / 28 desktop**. TTFB is 0.2 s and INP/CLS are
  green — the server and the layout are fine; the problem is everything that loads and
  runs before content paints.
- Homepage: **926 requests, 10.27 MB transferred** (27.20 MB uncompressed).

## Reports

- `baseline.md`: Core Web Vitals (field, mobile + desktop), Lighthouse lab, and the
  network baseline from the homepage HAR.
- `findings.md`: Main performance issues found during the audit.
- `prioritization.md`: The ICE scoring I use to rank the fixes, and the resulting order.
- `executive-summary.pdf`: The stakeholder-facing version — what each problem costs and
  what fixing it buys.
- `developer-report.md` / `developer-report.pdf`: The developer-facing version —
  mechanism, reproduction steps, fix and verification for each finding.
