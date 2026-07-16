# Baseline Results

## Summary

The page is not doing great, and mobile is the one I care about most here — that's where
most people read the news, so that's the experience I'm judging it by. It reacts fine once
it's up and it doesn't jump around, but the main content shows up too slowly and the screen
is blank for a bit at the start. So the problem is loading and rendering, not the server and
not layout. I ran the lab test on a throttled mobile setup (the same one we used in class),
so the numbers below reflect a real phone on a slow connection, not a laptop on fast wifi.

## Core Web Vitals — Field Data

This is the real-user data from Chrome (last 28 days). I looked at both mobile and desktop
because they came out a bit different.

Overall: **Failed**, because LCP is not in the good range on either one. Mobile is listed
first on purpose — it's the connection type most readers actually show up on.

### Mobile — Failed

- Largest Contentful Paint (LCP): **2.9 s** — needs improvement
- Interaction to Next Paint (INP): **187 ms** — good
- Cumulative Layout Shift (CLS): **0.04** — good
- First Contentful Paint (FCP): **2.2 s** — needs improvement
- Time to First Byte (TTFB): **0.2 s** — good

### Desktop — Failed

- Largest Contentful Paint (LCP): **3.6 s** — needs improvement
- Interaction to Next Paint (INP): **163 ms** — good
- Cumulative Layout Shift (CLS): **0.02** — good
- First Contentful Paint (FCP): **2.2 s** — needs improvement
- Time to First Byte (TTFB): **0.2 s** — good

## Lighthouse Lab Test

I ran the mobile column with the throttling we used in class: Slow 4G on the network (around
1.6 Mbps down, 750 Kbps up, 150 ms round trip) plus a 4× CPU slowdown, emulating a mid-range
phone. The desktop column is the normal desktop profile with no throttling. So the two
columns aren't really the same test — mobile is deliberately handicapped to match what a
phone on a cell connection actually goes through, which is why I trust the mobile number
more for judging real users.

|                | Mobile | Desktop |
| -------------- | ------ | ------- |
| Performance    | 40     | 28      |
| Accessibility  | 75     | 79      |
| Best Practices | 77     | 35      |
| SEO            | 85     | 85      |

## What this means

The page answers quickly (TTFB is 0.2 s), so the server is not the problem. It reacts fast
once it loads (INP is good) and it doesn't shift around (CLS is good). What's slow is
getting the content on the screen: the big content takes too long (LCP) and you stare at
nothing for a couple of seconds first (FCP). Desktop even scores lower than mobile in the
lab, which tells me it's not one slow file, it's just a lot of stuff loading and running at
the same time. And keep in mind the mobile lab number is already on a throttled phone, so
that's the honest version of what a reader on a phone gets — the real-user mobile data still
fails LCP (2.9 s) on top of that. So the work I'd prioritize is the mobile load: get the
content to paint sooner on a slow connection and stop shipping so much before it does.

## Network at `/`

I looked at the homepage HAR (`apnews.com-1.har`). It's one page load. Some requests don't
have a page tag on them, but those are just late ad and tracker calls that fire during the
same load, so I counted them in.

### How heavy the load is

The page makes **926 requests**.

It transfers **10.27 MB**, but the total resource size is **27.20 MB**. So compression is
helping — the browser downloads about **62.2% less** than the full size — but the page is
still huge.

### What the bytes are

Most of it is code and media.

- **JavaScript**: 161 requests, **4.44 MB** downloaded, 15.20 MB full size. That's 43% of
  everything downloaded.
- **Images**: 215 requests, **3.01 MB** downloaded, about 29%. The biggest single image is
  around **600 KB**.
- **Fonts**: 21 requests, **1.21 MB**.
- **CSS**: 19 requests, 0.16 MB downloaded, 1.13 MB full size.
- The rest is mostly tracker calls (documents and other).

JS and CSS together are **180 requests, 4.60 MB** downloaded and **16.33 MB** full size.
So code alone is about 45% of the download and 60% of the full size. Code is the biggest
thing on the page.

### Third parties

If I count only `apnews.com` and its subdomains as the site itself, then third parties are
**866 requests and 7.28 MB** — about **71%** of everything downloaded. So most of the page
is not even AP News's own stuff.

### Caching

The cache is basically not helping. There are **no `304` responses and nothing served from
cache**, and about **31%** of the responses say `no-cache` or `no-store`. So if you come
back, the browser downloads almost all of it again.

### Compression notes

The code compresses well. Most scripts and styles use `br`, `gzip`, or `zstd`, and that's
why the download is 62% smaller than the full size. The requests with no compression are
mostly images and fonts, which are already compressed formats, so they don't shrink again
over the network. For those, caching and smaller/lazy-loaded images matter more than gzip.

## Bundle outputs at `/`

The HAR tells me how many bytes came down, but not what's inside them, so for this part I
went back to the live page: pulled the homepage HTML, downloaded the first-party bundles,
and ran the Lighthouse coverage audits (unused JS/CSS, render-blocking, third-party
summary) on the same throttled mobile setup as before.

### JavaScript

AP News's own JavaScript is basically **one file**. The HTML loads a tiny
`webcomponents-loader` (2.7 KB) and then `All.min.<hash>.gz.js` — the name says it all.
It's **456 KB minified, 113 KB over the wire**, served from `assets.apnews.com`. Inside
there's a webpack build from their CMS (Brightspot — the script tag is even marked
`data-bsp-main-js`) with jQuery, the Flickity carousel, and all their web components in
the same file. Every page on the site gets this same bundle: no route splitting, no
component splitting, nothing loaded on demand.

Is that the right decision? Half of it is: the file name has a content hash and it's
served with `cache-control: public, max-age=31536000`, so it caches perfectly. But the
coverage numbers say the single bundle is too big for what one page uses — **79% of it
(about 89 of 113 KB) is unused on the homepage**. You're always paying for the whole site
to render one page.

Unused JS overall is much worse than the first-party bundle though: Lighthouse estimates
**~2.5 MB of unused JavaScript** on the load, and almost all of it is third parties —
Nativo's loader is 83% unused, Freestar's prebid.js 81%, JW Player's HLS code 85%,
Google's ad script 75%, reCAPTCHA 66%.

Source maps: **none**. The bundle has no `sourceMappingURL` comment and guessing the
`.map` URL returns a 404. Should they be there? Yes — an external `.map` file is only
downloaded when someone opens devtools, so it costs users nothing, and right now a
production error in that 456 KB of minified code points at nothing readable.

### CSS

Same story, literally the same name: **one** `All.min.<hash>.gz.css` for the whole site —
**801 KB minified, 107 KB over the wire, 6,264 rules**. On the homepage **90% of it is
unused** (96 of 107 KB). Same content-hash caching (good), same "every page carries the
whole design system" problem (not good). No source map either. Third-party CSS adds its
own waste on top: Viafoura's stylesheet is 98% unused, and the inline OneTrust and JW
Player style blocks are ~85% unused.

### Images

This is the part they got right. Images go through a resizing service
(`dims.apnews.com`) that crops and resizes server-side and converts to **WebP with a
JPEG/PNG fallback** via `<picture>`, with 1x/2x `srcset` variants, explicit
`width`/`height` on the tags, and the hero images preloaded as WebP with
`fetchpriority=high`. So yes — multiple sizes and formats, and yes, it's the right
decision. (The bytes are still heavy, but that's volume and quality settings — see
finding 8 — not a broken pipeline.)

Is the full-resolution file exposed? **Yes.** The resizer URL carries the original as a
plain `?url=` parameter, and that original downloads fine directly: the top hero traces
back to a **6000×4000, 8 MB JPEG** anyone can fetch. The page never ships it, so users
don't pay for it, but it's sitting one copy-paste away.

### Third parties

What's actually in the `<head>`, by job: OneTrust (consent), Kameleoon (A/B testing),
Parse.ly (experiments), Permutive and Quantcast (ad data), Sailthru and Wunderkind
(marketing/email popups), Google Tag Manager, Freestar/`pub.network` (ad bidding),
Nativo and Dianomi (sponsored content), Primis and JW Player (video), Viafoura
(comments), Zephr (paywall), reCAPTCHA, a quiz widget (Riverdrop), and one script from
`html-load.cc` I can't identify at all.

How they load is a mix, and the mix is the problem:

- **Render-blocking**: Parse.ly, the `apcdp.apnews.com` profile script (142 KB!), and
  OneTrust's auto-block file load synchronously before first paint — Lighthouse puts the
  cost at ~850 ms on mobile.
- **Loaded twice**: the Dianomi tag appears two times with the same `id`, and Primis has
  two synchronous tags.
- **Up front for no reason**: reCAPTCHA (383 KB, 66% unused) loads on the homepage where
  there's no form to protect, and Kameleoon is async but marked `fetchpriority=high`, so
  an A/B testing engine competes with the hero image.
- **Main-thread cost**: third-party code blocks the main thread for **~680 ms** in the
  mobile lab. The heavy ones: JW Player (578 KB, 672 ms of main-thread work), Viafoura
  (895 ms main-thread for a comments widget), Quantcast (139 ms blocking), Permutive
  (99 ms), Freestar (436 KB).

Do any seem inappropriate? The duplicated Dianomi tag is just a bug, reCAPTCHA doesn't
belong on the homepage, `html-load.cc` is a mystery domain running code on a news site,
and a synchronous quiz widget in the head is a strange thing to make every reader wait
for.

## Coverage, frames, and layers at `/`

For this part I measured the live homepage with the DevTools Coverage API and a scripted
scroll test, on the same mobile emulation as before (390px viewport, 4× CPU throttle).
Byte numbers in this section are **uncompressed** — that's what Coverage reports — so
they're bigger than the transfer sizes in the network section.

### Coverage — CSS

**Does it extract critical CSS? Not really.** The HTML has **18 separate inline `<style>`
tags (~35 KB)**, but reading them they're not an extracted critical path — they're ad-hoc
patches: theme variables, ad-slot spacing fixes, one-off widget tweaks. The actual
styling still comes from the single `All.min.css` bundle, and the 2.2 s of blank screen
(FCP) says the critical path isn't being served first. Piecemeal inlining isn't the same
thing as critical CSS extraction.

**How much unused CSS, and where from?** **88% of first-party CSS is unused at load**
(722 of 819 KB uncompressed). The sources:

- `All.min.css` — **89% unused** (700 of 782 KB). The whole-site design system shipped to
  every page, as already noted in the bundle section; Coverage confirms it rule by rule.
- Viafoura (comments widget) — three stylesheets, **~180 KB, 98–100% unused**, for a
  widget that isn't even visible on the homepage.
- Google Fonts CSS — **100% unused** (~27 KB): third-party widgets pull Roboto,
  Merriweather and Poppins that the page doesn't render with.

**Corrective finding:** extract real critical CSS for the homepage templates and load
`All.min.css` async; and stop letting Viafoura inject 180 KB of styles for a widget
below the fold.

### Coverage — JS

**How much unused JS, and where from?** First-party: **74% unused** (839 KB of 1.13 MB).
But first-party is a rounding error here — the tracked third-party JS is **13 MB
uncompressed, 66% unused (8.55 MB)**. Top offenders:

- Nativo's `s.ntv.io/serve/load.js` — **loaded twice**, 910 KB each time, 84% and 81%
  unused. That's ~1.8 MB of sponsored-content loader for one page, duplicated the same
  way the Dianomi tag is duplicated.
- reCAPTCHA — 870 KB, 72% unused, still with no form to protect.
- The ad stack: Freestar's `pubfig.engine.mobile.js` (614 KB, 82% unused), `prebid.js`
  (577 KB, 82% unused), Google's `pubads_impl.js` (617 KB, 78% unused).
- Wunderkind/BounceExchange (537 KB, 76% unused) and Viafoura's `vf-v2.js` (669 KB, 58%
  unused).

**Corrective finding:** the duplicated Nativo loader is a straight bug — one tag instead
of two saves ~900 KB of parse-and-execute without changing anything on the page.

### Frames — load, scrolling, interaction

I drove a continuous scroll through the page (300 frames sampled, CPU 4×) and watched
frame times and long tasks.

- **Load:** **22 long tasks totaling ~2.35 s** of main-thread blockage, the worst at
  242 ms. That's the ad stack, consent manager and trackers all booting at once, and
  it's exactly the "blank screen then late content" the metrics section describes.
- **Scrolling:** average frame **23 ms** (~43 fps), with **20 dropped frames of 300** —
  16 of them severe (>50 ms), worst **367 ms**. Alongside them: **14 long tasks** during
  the scroll, up to 287 ms. The causes line up with what's on the page: ad slots
  initializing and refreshing as they enter the viewport (Freestar/GPT), lazy embeds
  (JW Player, Viafoura) booting mid-scroll, all on the main thread.
- **Are they excessive or unexpected?** Both worse than Harbour.Space by a lot (20 vs 6
  dropped frames on the same test). Excessive: 1 frame in 15 misses, and several misses
  are 5–20× the frame budget. Unexpected: no — sadly this is what an ad-funded page with
  ~30 third parties does; but reading a news list should not hitch for a third of a
  second.

**Corrective finding:** ad slot initialization fires on the scroll path; moving slot
init/refresh off the interaction path (idle scheduling, bigger lazy margins so slots
init *before* they're visible) would remove most of the severe frame drops.

### Layers and animations

I scanned every element's computed style for layer-forcing properties and pulled the
compositor layer tree:

- **27 composited layers** on the loaded page. The creators are the usual suspects:
  the fixed header and leaderboard ad container (8 `position: fixed/sticky` elements
  total), the video player, and each animating widget.
- **`will-change` is actually disciplined here: 2 elements**, both the hamburger menu
  with `will-change: transform`. That's restrained — but it's declared in the
  stylesheet, so both layers exist permanently for a menu that's almost never open.
- **No `translate3d(0,0,0)` / `translateZ(0)` hacks** in computed styles, and only two
  `backface-visibility: hidden`. Nothing smells force-hacked.
- The animations I could identify (menu slide, Flickity carousel drag, fades) run on
  **transform/opacity — composition-driven**, not layout or paint. I didn't find
  layout-animating CSS (`top/left/width` transitions) in the shipped styles.

What I couldn't verify headless: whether anything *feels* janky first-frame on a real
device, and per-layer memory in the Layers panel — that's the manual DevTools pass still
to do. Given the scroll numbers above, any jank a user feels here is main-thread long
tasks, not compositing.

**Corrective finding:** scope the hamburger's `will-change: transform` to the open state
(add it on interaction, or via a class toggled when the menu opens) instead of keeping
two forced layers alive for the page's whole lifetime.
