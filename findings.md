# Performance Findings

These are from the homepage. The server is fast (TTFB 0.2 s), so nothing here is a backend
problem. It's all front end: too much JavaScript, too much media, and a ton of third-party
stuff. Layout barely moves (CLS 0.04 / 0.02) so I didn't count that as a problem.

I'm looking at this from mobile, since that's where most people read the news. The lab
numbers I trust were taken on a throttled phone (Slow 4G, 4× slower CPU — the setup from
class), not a fast laptop. The first two are the mobile ones; the rest hit both but hurt
mobile more.

I tried to keep each of these as its own separate thing. When a few problems really just
show up as the same metric (like the couple of things that push LCP out), I put them
together instead of listing them twice.

---

## Things to fix

## 1. Mobile is the one that's really slow

Affects users: on a phone the article shows up late and you wait on a blank screen first.
That's what most readers get.
Metric: mobile LCP (2.9 s), FCP (2.2 s). The throttled mobile run is what drags the lab
score to 40.
Cause: the whole page (10.27 MB, 926 requests) has to come down a slow phone connection
instead of fast wifi. On Slow 4G that's just a lot to pull in, so the content paints late.
Solution: aim at mobile first — send less up front, load the hero early (fetchpriority=high,
preload), and hold the rest until the page is usable.

## 2. The JavaScript is heavier on a phone

Affects users: on a real phone the page feels slower than the field INP (187 ms) makes it
look, because the phone is busy running all that code.
Metric: TBT, Speed Index (both on the 4× slower CPU).
Cause: there's 4.44 MB of JavaScript (161 requests). A phone CPU is about 4× slower than a
laptop, so the same scripts take roughly 4× longer to run, and that's what holds the page up.
Solution: drop the code that's not used, split the bundles, and defer anything not needed
for the first paint — helps mobile the most.

## 3. The main content shows up really late

Affects users: the headline/hero you opened the page for only appears after a long wait,
and it feels broken even though it isn't.
Metric: LCP (2.9 s on mobile, 3.6 s on desktop), Speed Index.
Cause: the LCP element is stuck waiting behind all that JavaScript and heavy media loading
at the same time.
Solution: prioritize it (fetchpriority=high, preload) and cut down the work happening
before it paints.

## 4. Way too much JavaScript

Affects users: the page looks done but you tap something and nothing happens for a moment,
because the browser is busy running scripts.
Metric: TBT, Time to Interactive.
Cause: JavaScript is 161 requests, 4.44 MB downloaded, and 15.20 MB of full size. That's
43% of everything downloaded, and with CSS it's 60% of the full size. That much code takes
real time to download and run.
Solution: drop the unused code, split the bundles, and defer whatever isn't needed for the
first render.

## 5. Third parties are taking over

Affects users: ads and trackers fight the actual article for bandwidth and CPU, so the
thing you came to read loses.
Metric: TBT, LCP.
Cause: third-party requests add up to 866 requests and 7.28 MB downloaded, about 71% of
everything. Most of the page isn't even AP News's own content.
Solution: cut the ones that aren't needed, lazy-load the rest, and hold non-critical ones
until the page is usable.

## 6. Blank screen for too long at the start

Affects users: you stare at nothing for a couple of seconds before anything appears.
Metric: FCP (2.2 s on both), Speed Index.
Cause: scripts and CSS that block rendering run before the first paint, plus CSS that gets
downloaded but never used.
Solution: inline the critical CSS, defer the rest, and remove the unused rules.

## 7. The cache doesn't help on a second visit

Affects users: coming back to the page doesn't feel any cheaper or faster, it downloads
almost everything again.
Metric: repeat-load bytes, LCP, FCP.
Cause: there are no `304` responses and nothing served from cache, and about 31% of the
responses say `no-cache` or `no-store`, so the browser goes back to the network.
Solution: let static files be reused from cache with proper cache headers, and stop marking
everything `no-cache`.

## 8. Images and fonts are a real byte problem

Affects users: photos and fonts take a big part of the download, especially on mobile.
Metric: LCP, Speed Index.
Cause: images are 215 requests and 3.01 MB (about 29% of the download), and the biggest one
alone is around 600 KB. Fonts add another 1.21 MB. These are already-compressed formats, so
gzip doesn't shrink them, they come through almost full size.
Solution: serve smaller responsive images, use AVIF/WebP, lazy-load images below the first
screen, and cut down the fonts.

## 9. There are way too many requests

Affects users: the page has hundreds of little things fighting to load, which makes it feel
busy and slow.
Metric: LCP, FCP, Speed Index.
Cause: the load has 926 requests, and most of them are third-party ad and tracker calls.
Solution: reduce third-party calls, lazy-load below-the-fold modules, and don't load
ad/comment/video systems before the main content.

---

## Things it does well

## 10. The layout doesn't jump around

The page stays stable while it loads (CLS 0.04 on mobile, 0.02 on desktop, both good).
Nothing moves under your finger when you go to tap something. Worth keeping this as they
add more content, by giving images and ads their space up front.

## 11. It reacts fast once it's up

Even with all that JavaScript, interactions feel quick (INP 187 ms on mobile, 163 ms on
desktop, both good). So the pain is really in the loading, not in using the page after it
loads.

## 12. Fast server and good compression on the code

The server answers quickly (TTFB 0.2 s), so the slowness isn't coming from the backend. And
the text compression is doing its job, the page downloads 62% less than its full size
because the JS and CSS are compressed. So that part is already fine, what's left to fix is
the media and the caching.
