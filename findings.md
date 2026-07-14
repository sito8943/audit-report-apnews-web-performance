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

I also scored each fix with ICE — impact, confidence and ease, each 1 to 10, multiplied
(the scales and why I picked it are in `prioritization.md`). Highest score first, which
puts the order at 3, 8, 7, 1, 6, 2, 4, 5, 9. The pattern that comes out: the sure,
well-scoped fixes go first, and the two biggest problems (the JavaScript and the third
parties) end up near the bottom — not because they don't matter, but because they're huge
jobs and ICE punishes that hard.

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
Priority: impact 9, because this is the whole reading experience for most readers.
Confidence 8 — the field and the throttled lab agree mobile is where it hurts. But "send
less up front" touches almost everything on the page, so ease is only a 4.
ICE = 9 × 8 × 4 = 288.

## 2. The JavaScript is heavier on a phone

Affects users: on a real phone the page feels slower than the field INP (187 ms) makes it
look, because the phone is busy running all that code.
Metric: TBT, Speed Index (both on the 4× slower CPU).
Cause: there's 4.44 MB of JavaScript (161 requests). A phone CPU is about 4× slower than a
laptop, so the same scripts take roughly 4× longer to run, and that's what holds the page up.
Solution: drop the code that's not used, split the bundles, and defer anything not needed
for the first paint — helps mobile the most.
Priority: impact 7 — real, but the reader feels it through the loading more than anything
else. Confidence 7, because the "4× slower CPU means 4× longer scripts" part is my
inference from the lab setup, not something the field data says directly. And it's bundle
surgery, so ease 4. ICE = 7 × 7 × 4 = 196.

## 3. The main content shows up really late

Affects users: the headline/hero you opened the page for only appears after a long wait,
and it feels broken even though it isn't.
Metric: LCP (2.9 s on mobile, 3.6 s on desktop), Speed Index.
Cause: the LCP element is stuck waiting behind all that JavaScript and heavy media loading
at the same time.
Solution: prioritize it (fetchpriority=high, preload) and cut down the work happening
before it paints.
Priority: impact 8 — the headline is what you opened the page for. Confidence 9, since
mobile and desktop both show it and the cause (LCP element waiting behind everything else)
is visible in the waterfall. And fetchpriority/preload is a known, small change, so
ease 7. ICE = 8 × 9 × 7 = 504 — the top of the list, and it earns it: sure fix, cheap fix,
big win.

## 4. Way too much JavaScript

Affects users: the page looks done but you tap something and nothing happens for a moment,
because the browser is busy running scripts.
Metric: TBT, Time to Interactive.
Cause: JavaScript is 161 requests, 4.44 MB downloaded, and 15.20 MB of full size. That's
43% of everything downloaded, and with CSS it's 60% of the full size. That much code takes
real time to download and run.
Solution: drop the unused code, split the bundles, and defer whatever isn't needed for the
first render.
Priority: impact 8 and confidence 8 — 4.44 MB of JS is right there in the HAR, no guessing
needed. But cleaning up 161 script requests across a site this size is a monster, so
ease 3, and that sinks it: ICE = 8 × 8 × 3 = 192. It still needs to happen, it's just not
where you start.

## 5. Third parties are taking over

Affects users: ads and trackers fight the actual article for bandwidth and CPU, so the
thing you came to read loses.
Metric: TBT, LCP.
Cause: third-party requests add up to 866 requests and 7.28 MB downloaded, about 71% of
everything. Most of the page isn't even AP News's own content.
Solution: cut the ones that aren't needed, lazy-load the rest, and hold non-critical ones
until the page is usable.
Priority: impact 8 — 71% of the download not being the actual article is the biggest
single number in the audit. Confidence 7: the bytes are certain, but how much of it can
actually be cut is not up to engineering — ads are how a news site pays for itself. That's
also why ease is a 3: it's negotiation as much as code. ICE = 8 × 7 × 3 = 168.

## 6. Blank screen for too long at the start

Affects users: you stare at nothing for a couple of seconds before anything appears.
Metric: FCP (2.2 s on both), Speed Index.
Cause: scripts and CSS that block rendering run before the first paint, plus CSS that gets
downloaded but never used.
Solution: inline the critical CSS, defer the rest, and remove the unused rules.
Priority: impact 6 — two seconds of blank screen is bad, but fixing FCP alone doesn't make
the article arrive sooner. Confidence 7, since render-blocking resources show up clearly
in the lab but I haven't measured exactly how much each one blocks. Critical-CSS work is a
known technique but fiddly to keep right, so ease 5. ICE = 6 × 7 × 5 = 210.

## 7. The cache doesn't help on a second visit

Affects users: coming back to the page doesn't feel any cheaper or faster, it downloads
almost everything again.
Metric: repeat-load bytes, LCP, FCP.
Cause: there are no `304` responses and nothing served from cache, and about 31% of the
responses say `no-cache` or `no-store`, so the browser goes back to the network.
Solution: let static files be reused from cache with proper cache headers, and stop marking
everything `no-cache`.
Priority: confidence 9 — this is the finding I'm most sure about, the `no-cache`/`no-store`
headers and the missing `304`s are sitting right there in the HAR. Ease 8, because it's
header configuration, not code. Impact 5, since only returning readers feel it (though on
a news site people come back every day, so that's a lot of readers).
ICE = 5 × 9 × 8 = 360.

## 8. Images and fonts are a real byte problem

Affects users: photos and fonts take a big part of the download, especially on mobile.
Metric: LCP, Speed Index.
Cause: images are 215 requests and 3.01 MB (about 29% of the download), and the biggest one
alone is around 600 KB. Fonts add another 1.21 MB. These are already-compressed formats, so
gzip doesn't shrink them, they come through almost full size.
Solution: serve smaller responsive images, use AVIF/WebP, lazy-load images below the first
screen, and cut down the fonts.
Priority: impact 7 — over 4 MB of media on a phone connection is a lot of the wait.
Confidence 8, the byte counts are direct from the HAR and smaller images can only help.
Responsive images and modern formats are well-trodden ground, mostly pipeline work rather
than rethinking the site, so ease 7. ICE = 7 × 8 × 7 = 392.

## 9. There are way too many requests

Affects users: the page has hundreds of little things fighting to load, which makes it feel
busy and slow.
Metric: LCP, FCP, Speed Index.
Cause: the load has 926 requests, and most of them are third-party ad and tracker calls.
Solution: reduce third-party calls, lazy-load below-the-fold modules, and don't load
ad/comment/video systems before the main content.
Priority: impact 5 — most of the request count is the third parties again, so fixing
finding 5 fixes most of this too; on its own it's more noise than pain. Confidence 7 (the
926 requests are real, but how much request count itself hurts vs the bytes is harder to
pin down), and ease 4 since it's spread across everything. ICE = 5 × 7 × 4 = 140, last on
the list — fine, since it mostly comes along for free with 5 and 8.

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
