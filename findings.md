# Performance Findings

These are from the homepage. The server responds fast (~30 ms for the document), so
nothing here is a backend problem, it's all on the front end: too much media, too much
JavaScript, and a lot of third-party stuff. Layout barely moves (CLS ~0.001) so I didn't
count that as a problem.

## 1. The page is just too heavy

Affects users: on a phone or slow wifi you're downloading a ton of data, and the content keeps trickling in for ages.
Metric: LCP, Speed Index.
Cause: it's around 20 MB total. Most of that is ~8 MB of media and ~7 MB of images that aren't compressed or sized for the screen.
Solution: compress everything, switch images to WebP/AVIF, serve responsive sizes, and lazy-load what's below the fold.

## 2. Way too much JavaScript

Affects users: the page looks done but you tap something and nothing happens for a while.
Metric: TBT, Time to Interactive.
Cause: the main thread is busy for ~48 s, mostly running scripts, and roughly 2.1 MB of the JS that gets downloaded is never even used.
Solution: drop the unused code, split the bundles, and defer whatever isn't needed for the first render.

## 3. Third parties are taking over

Affects users: ads and trackers fight the actual article for bandwidth and CPU, so the thing you came to read loses.
Metric: TBT, LCP.
Cause: third-party requests add up to ~12.7 MB across ~157 requests (ads, analytics, social widgets).
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
