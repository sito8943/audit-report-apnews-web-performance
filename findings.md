# Performance Findings

These are from the homepage. The server responds fast (~30 ms for the document), so
nothing here is a backend problem, it's all on the front end: too much media, too much
JavaScript, and a lot of third-party stuff. Layout barely moves (CLS ~0.001) so I didn't
count that as a problem.

## 1. The page is just too heavy

Affects users: on a phone or slow wifi you're downloading a ton of data, and the content keeps trickling in for ages.
Metric: LCP, Speed Index.
Cause: the homepage transfers 11.58 MB and has 28.12 MB of total resource size. The main download is code and images.
Solution: compress everything, switch images to WebP/AVIF, serve responsive sizes, and lazy-load what's below the fold.

## 2. Way too much JavaScript

Affects users: the page looks done but you tap something and nothing happens for a while.
Metric: TBT, Time to Interactive.
Cause: the main thread is busy for ~48 s, mostly running scripts. In the HAR, scripts alone transfer 4.76 MB and add up to 15.8 MB of resource size.
Solution: drop the unused code, split the bundles, and defer whatever isn't needed for the first render.

## 3. Third parties are taking over

Affects users: ads and trackers fight the actual article for bandwidth and CPU, so the thing you came to read loses.
Metric: TBT, LCP.
Cause: third-party requests add up to 338 requests, 6.29 MB transferred, and 19.15 MB total resource size.
Solution: cut the ones that aren't needed, lazy-load the rest, and hold non-critical ones until the page is usable.

## 4. The main content shows up really late

Affects users: the headline/hero you opened the page for only appears after a long wait, and it feels broken.
Metric: LCP.
Cause: the LCP element is stuck waiting behind all that JavaScript and heavy media loading at the same time.
Solution: prioritize it (fetchpriority=high, preload) and cut down the work happening before it paints.

## 5. Blank screen for too long at the start

Affects users: you stare at nothing for a few seconds before anything appears.
Metric: FCP, Speed Index.
Cause: render-blocking scripts and CSS, plus ~125 KiB of CSS that isn't used.
Solution: inline the critical CSS, defer the rest, and remove the unused rules.

## 6. There are way too many requests

Affects users: the page has a lot of small things fighting to load, which makes the page feel busy and slow.
Metric: LCP, FCP, Speed Index.
Cause: the first load has 409 requests. The refresh has 560 requests, which is even worse.
Solution: reduce third-party calls, lazy-load below-the-fold modules, and avoid loading ad/comment/video systems before the main content.

## 7. The refresh is worse than the first load

Affects users: coming back to the page does not feel cheaper or faster.
Metric: repeat-load transferred bytes, LCP, FCP, Speed Index.
Cause: the first load transfers 11.58 MB, but the refresh transfers 15.86 MB. That is about 37% more, so caching is not helping in this capture.
Solution: let static assets reuse cache, avoid `no-cache` on everything, and stop loading fresh ad/video work on every refresh before the page is usable.

## 8. Images are a real byte problem

Affects users: article photos take a big part of the download, especially on mobile.
Metric: LCP, Speed Index.
Cause: images transfer 4.35 MB on the first load, almost 38% of everything downloaded. The biggest image alone is about 1.32 MB.
Solution: serve smaller responsive images, use AVIF/WebP where possible, lower quality for thumbnails, and lazy-load images below the first screen.

## 9. Video starts adding weight too early

Affects users: users who came to scan headlines still pay for video chunks.
Metric: LCP, Speed Index, bandwidth use.
Cause: media is 710 KB on the first load, then jumps to 2.85 MB on refresh because video chunks load again.
Solution: don't load video chunks until the player is visible or the user interacts with it.

## 10. Compression helps code, but not images and video

Affects users: compression reduces JS/CSS cost, but the media bytes still come through almost full size.
Metric: transferred bytes, LCP, Speed Index.
Cause: JS and CSS shrink well because they use `br`, `gzip`, or `zstd`. Images and video are already compressed formats, so transferred size is basically the same as resource size.
Solution: keep text compression, but fix media with better formats, smaller responsive sizes, lazy loading, and caching.
