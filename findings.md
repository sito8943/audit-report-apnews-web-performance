# Performance Findings

From the homepage baseline. The server itself is fast (root document ~30 ms), so the
problems are all on the front end: too much media, too much JavaScript, and too many
third parties. Layout is stable (CLS ~0.001), so shifting is not an issue.

## 1. The page is enormous

Affects users: on mobile or a slower connection the page eats a lot of data and takes a long time to finish, so content keeps appearing late.
Metric: Largest Contentful Paint, Speed Index.
Cause: about 20 MB total, with ~8 MB of media and ~7 MB of images. Images aren't in modern formats and aren't sized for the device.
Solution: compress the media, serve WebP/AVIF, use responsive sizes, and lazy-load anything below the fold.

## 2. Too much JavaScript blocks the main thread

Affects users: the page looks ready but doesn't respond to taps or scrolls for a long time.
Metric: Total Blocking Time, Time to Interactive.
Cause: main-thread work is ~48 s, most of it script evaluation, and about 2.1 MB of the JavaScript downloaded is never used.
Solution: remove the unused code, split bundles, and defer non-critical scripts so less runs up front.

## 3. Third-party scripts dominate the load

Affects users: ads and tracking compete with the actual article for bandwidth and CPU, making everything slower.
Metric: Total Blocking Time, Largest Contentful Paint.
Cause: third parties account for ~12.7 MB across ~157 requests (ads, analytics, social widgets).
Solution: cut the third parties that aren't essential, lazy-load the rest, and load non-critical ones after the page is usable.

## 4. Largest content shows up very late

Affects users: the main headline/hero the user came to read appears only after a long wait, so the site feels broken.
Metric: Largest Contentful Paint.
Cause: the LCP element has to wait behind all the JavaScript and heavy media loading at the same time.
Solution: prioritize the LCP image (fetchpriority=high, preload) and reduce the work happening before it paints.

## 5. First paint is slow because of render-blocking resources

Affects users: the screen stays blank for several seconds before anything appears.
Metric: First Contentful Paint, Speed Index.
Cause: render-blocking scripts and CSS, including ~125 KiB of unused CSS.
Solution: inline the critical CSS, defer the rest, and drop the unused CSS rules.
