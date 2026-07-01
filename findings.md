# Performance Findings

## Summary

The site shows clear performance bottlenecks around rendering, JavaScript execution, and resource delivery. The biggest opportunity is to reduce the time required for the page to become visually useful.

## Key Findings

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

## Likely Causes

- Too much main-thread work
- High JavaScript execution time
- Large amount of unused JavaScript
- Large network payload
- Render-blocking requests
- Image optimization opportunities
- Cache lifetime improvements

## Recommended Actions

1. Optimize and compress hero images and large media assets.
2. Minimize JavaScript execution by removing unused code and deferring non-critical scripts.
3. Reduce third-party and render-blocking resources where possible.
4. Improve caching and asset delivery to speed up repeat visits.
5. Re-test after each change to confirm improvements in LCP, TBT, and Speed Index.
