# Baseline Results

## Summary

The initial baseline suggests that the page is not meeting performance expectations, mainly because the main content appears too slowly to users.

## Core Web Vitals — Field Data

Assessment: Failed

- Largest Contentful Paint (LCP): 3.6 s
  Status: Needs improvement

- Interaction to Next Paint (INP): 156 ms
  Status: Good

- Cumulative Layout Shift (CLS): 0.03
  Status: Good

## Other Field Metrics

- First Contentful Paint (FCP): 1.9 s
  Status: Needs improvement

- Time to First Byte (TTFB): 0.2 s
  Status: Good

## Lighthouse Lab Test

- Performance: 30
- Accessibility: 79
- Best Practices: 54
- SEO: 85

## Interpretation

The page is responsive and visually stable, but the largest visible content takes too long to appear. This makes the experience feel slow even though interaction timing and layout stability are acceptable. The main issue is therefore related to loading and rendering performance rather than stability.

## Network baseline at `/`

I also checked the HAR for the homepage. The first file had some old Network entries in
it, so I only used the AP News page load from that HAR. For the refresh file, I used the
last AP News page load, because that is the actual refresh.

### Initial load

The first AP News load makes **409 requests**.

It transfers **11.58 MB**, but the total resource size is **28.12 MB**. Compression is
helping, but the page is still huge. The transfer is about **58.8% smaller** than the
full resource size.

Most of the downloaded data is code and images.

JS and CSS together are **117 requests**, **4.90 MB transferred**, and **16.81 MB total
resource size**. So code is about **42.3%** of the download and almost **60%** of the
total resource size.

Images are almost as heavy over the network: **79 requests**, **4.35 MB transferred**,
and **4.33 MB total resource size**. That is **37.6%** of the downloaded data.

Video/media is smaller in the first load, but it is still there: **6 requests** and
about **710 KB transferred**.

Third-party stuff is massive. If I count only `apnews.com` and its subdomains as
first-party, third parties are **338 requests**, **6.29 MB transferred**, and **19.15 MB
total resource size**. That is more than half of the data downloaded.

### Soft refresh

The refresh does not get lighter. It makes **560 requests** and transfers **15.86 MB**.
That is worse than the first load, not better.

So the cache reduction is basically **0%**. If I calculate it directly from bytes, it is
around **-37%**, because the refresh downloads about 4.28 MB more than the first load.

This looks like the refresh is not really benefiting from cache. There are no useful
`304` responses, and almost every request in the refresh has `Cache-Control: no-cache`,
so the browser goes back to the network again.

The refresh also loads more media than the first load. Media jumps from about **710 KB**
to **2.85 MB**, mostly from video chunks.

### Compression notes

JS and CSS are compressed. Most scripts use `br` or `gzip`, with a few using `zstd`.
Stylesheets use `br` or `gzip`.

Images are different. The main images are already compressed image formats like `webp`,
`jpeg`, `png`, or `gif`, so they do not really shrink again with transport compression.
That is why image transferred size and image resource size are basically the same.

Video/media is also already compressed. It transfers almost the same amount as its
resource size, so caching and lazy loading matter more than gzip here.
