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
