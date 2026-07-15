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
