# Performance Findings

## Key findings

- Largest Contentful Paint (LCP): 13.4 s
  The largest visible content takes too long to appear.

- Total Blocking Time (TBT): 4,580 ms
  The browser main thread is blocked for too long, which can make the page feel slow or unresponsive.

- Speed Index: 11.2 s
  The page takes too long to visually complete.

- First Contentful Paint (FCP): 1.5 s
  The first content appears relatively early, but the rest of the page is delayed.

- Cumulative Layout Shift (CLS): 0.064
  Layout stability is good, so visual shifting is not the main issue.

## Likely causes

- Too much main-thread work
- High JavaScript execution time
- Large amount of unused JavaScript
- Large network payload
- Render-blocking requests
- Image optimization opportunities
- Cache lifetime improvements
