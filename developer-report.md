# Developer Report — AP News Performance Audit

This is the version of the audit for the person who actually has to fix things. The
stakeholder summary says what the problems cost; this one says where each cost is
incurred, how to see it yourself, what to change, and how to know the change worked.
Everything here traces back to the measurements in `baseline.md` — if a number below
seems off, that file has the raw context.

One thing before the findings: nothing in this report is reproducible unless you run it
under the same conditions I did. The section right below is not optional reading.

---

## How to reproduce every measurement in this report

All the lab numbers come from the same setup. If you measure on your laptop on office
wifi you will get much nicer numbers and conclude I'm exaggerating. I'm not — I'm on a
simulated phone.

**The environment:**

- **Page:** `https://apnews.com/` (the homepage) unless a finding says otherwise. The
  other audited pages are listed in `README.md`.
- **Device emulation:** mobile, 390 px viewport.
- **Network throttling:** Slow 4G — ~1.6 Mbps down, 750 Kbps up, 150 ms RTT. In Chrome
  DevTools this is the "Slow 4G" preset in the Network panel; in Lighthouse it's the
  default mobile throttling.
- **CPU throttling:** 4× slowdown (Performance panel → gear icon → CPU: 4× slowdown).
  This is the difference between a laptop and a mid-range phone, and it is the reason
  the JavaScript findings feel invisible on a dev machine.
- **Cache:** disabled ("Disable cache" checked in the Network panel) for first-load
  measurements. For the caching finding (7) you deliberately leave it on — that's the
  point of that test.

**The tools, per kind of measurement:**

- **Field data (what real users get):** PageSpeed Insights → enter `apnews.com` →
  the "Discover what your real users are experiencing" section is CrUX, last 28 days.
  This is the source of LCP 2.9 s / INP 187 ms / CLS 0.04 on mobile. Field data moves
  slowly — expect a fix to take up to 28 days to fully show here. Use lab for fast
  iteration, field for confirmation.
- **Lab scores:** Lighthouse in DevTools, mobile config, throttling as above. Source of
  the Performance 40 (mobile) / 28 (desktop) scores, the unused-JS/CSS estimates, the
  render-blocking list, and the third-party summary.
- **Network weight:** load the homepage with cache disabled, let it settle (~30 s, the
  ad stack keeps firing), export the HAR. My export is `apnews.com-1.har`: 926 requests,
  10.27 MB transferred, 27.20 MB uncompressed.
- **Unused code:** DevTools → Cmd+Shift+P → "Show Coverage" → reload with recording.
  Note Coverage reports **uncompressed** bytes; the HAR reports transfer bytes. Both
  appear in this report and I say which is which each time.
- **Long tasks and frames:** Performance panel recording during load, and a second
  recording while scrolling the full page. CPU 4× for both.
- **Headers:** `curl -I` against the page and asset URLs, or just the Headers tab in
  the Network panel.

**Rule for verifying any fix:** measure, change one thing, measure again on the same
setup. Every finding below ends with a "how you know it worked" that names the number
that should move.

---

## The findings, in the order I'd work them

The order is the ICE ranking from `prioritization.md` (3, 8, 7, 13, 1, 11, 10, 6, 2, 4,
12, 5, 9). Numbering matches `findings.md` so the two documents cross-reference.

Three of the thirteen (1, 2, 9) are umbrella findings — real problems, but their fix *is*
the other findings. I've kept them in place and said exactly which work covers them, so
nobody opens a ticket that duplicates another ticket.

---

### Finding 3 — LCP: the main content paints too late

**Mechanism.** This is not a "missing preload" problem — the tag-level hygiene is
already right. The hero image is preloaded, served as WebP via `<picture>`, and marked
`fetchpriority=high` (see finding 17). The LCP is late because of **contention**: the
hero has to fight through

1. ~850 ms of render-blocking scripts in the `<head>` (Parse.ly, the 142 KB
   `apcdp.apnews.com` profile script, OneTrust's auto-block — detailed in finding 13),
2. a **2.4 MB HTML document** (the server renders ~148 story promos into the homepage —
   every phone parses all of them before the fold is done), and
3. at least one third-party script (Kameleoon, an A/B tester) *also* marked
   `fetchpriority=high`, so it competes with the hero at the same priority.

**Reproduce.** Run Lighthouse mobile on `/`. Open the LCP audit and expand the **phase
breakdown** (TTFB / load delay / load time / render delay). TTFB will be tiny (~0.2 s);
the bulk sits in load delay + render delay — i.e., the browser knew about the image but
was busy. Then check the Network panel waterfall: filter by `Doc` and `Img`, and watch
what downloads in the same window as the hero.

**Fix.**
- Remove or defer the render-blocking head scripts (that work is itemized in finding 13
  — doing 13 largely *is* doing this finding).
- Strip `fetchpriority=high` from Kameleoon. Nothing except the LCP element should
  carry it.
- Shrink the document: render the first 2–3 screens of story promos server-side and
  load the long tail on scroll. This one is **structural** — it changes what the
  Brightspot templates emit, so it's a conversation with whoever owns the CMS templates,
  not a solo patch.

**How you know it worked.** Lighthouse LCP phase breakdown: render delay collapses.
Mobile lab LCP moves first; field LCP (2.9 s → target < 2.5 s) follows within a CrUX
window.

---

### Finding 8 — Images and fonts: 4.2 MB of already-compressed bytes

**Mechanism.** Images are 215 requests / 3.01 MB transferred (29% of the page), largest
single image ~600 KB; fonts are 21 requests / 1.21 MB. These formats don't gzip — they
arrive essentially full size — so the only levers are *fewer pixels, better codecs,
fewer files, later loading*. The pipeline itself is fine (finding 17); the problem is
volume and settings.

**Reproduce.** In the HAR (or Network panel filtered to `Img` and `Font`), sort by
size. The ~600 KB image and the 21 font files are immediately visible. For the fonts,
note the Coverage finding: the Google Fonts CSS (Roboto, Merriweather, Poppins) is
**100% unused** — third-party widgets pull font families the page never renders with.

**Fix.**
- Add AVIF as the first `<source>` in the existing `<picture>` setup — the
  `dims.apnews.com` resizer already does format conversion, so this is a pipeline
  setting, not new infrastructure.
- Audit the resizer quality parameter; a 600 KB above-the-fold image on mobile means
  quality is set higher than a phone screen can show.
- Confirm below-the-fold images carry `loading="lazy"`; anything inside the 148-promo
  tail definitely should.
- Fonts: subset to the character sets actually used, cut families/weights nobody
  renders with, and take the unused Google Fonts CSS up with the widget vendors that
  inject it (it's their request, not yours — which makes it part of the finding 5
  conversation).

**How you know it worked.** Re-export the HAR: image bytes and font bytes drop
directly. Lab LCP and Speed Index improve because the hero competes with less.

**Scope.** Local — resizer settings and template attributes. No build changes.

---

### Finding 7 — Caching: a returning reader re-downloads the site

**Mechanism.** In the entire 926-request HAR there are **zero 304 responses and zero
cache hits**, and ~31% of responses carry `no-cache` or `no-store`. On top of that,
article/hub documents are served `max-age=30, s-maxage=31536000` — great for the CDN,
but after 30 seconds the *browser's* copy expires, and because the documents ship **no
working `ETag`/`Last-Modified` revalidation path**, expiry means a full re-download
(~1 MB per article) instead of a 304.

To be precise about ownership: the hashed first-party assets (`All.min.<hash>.js/css`)
are already `max-age=31536000` and cache perfectly — don't touch those. The
`no-cache`/`no-store` bulk is mostly third-party trackers you don't control. The part
that's yours to fix is **document revalidation**.

**Reproduce.** Load `/`, then reload *without* "Disable cache". Network panel: status
column — you'll see fresh 200s where 304s or memory-cache hits should be. For the
document path: `curl -I https://apnews.com/article/<any-article>` — check
`cache-control`, then note there's no usable validator to send back.

**Fix.** Emit `ETag` (or `Last-Modified`) on document responses and make sure the CDN
passes conditional requests through (or answers them itself). Keep `max-age=30` — the
short freshness window is correct for news — but pair it with revalidation so expiry
costs a 304, not a megabyte.

**How you know it worked.** The repeat-load HAR shows 304s on documents and dramatically
lower repeat-visit bytes. This is the finding I'm most confident in (headers are right
there in the responses) and one of the cheapest — it's config, not code.

**Scope.** Local — CDN/origin header configuration.

---

### Finding 13 — Third-party scripts loaded the wrong way

**Mechanism.** Separate from *how much* third-party code there is (finding 5), this is
about load order and duplication — pure waste with no revenue attached. The specific
items, each independently verifiable in the page source:

| Item | What's wrong | Cost |
|---|---|---|
| Parse.ly, `apcdp.apnews.com` profile script (142 KB), OneTrust auto-block | Synchronous in `<head>`, before first paint | ~850 ms render-blocking (Lighthouse, mobile) |
| Dianomi tag | Included **twice**, same `id` | Duplicate download + execute |
| Nativo `s.ntv.io/serve/load.js` | Loaded **twice**, 910 KB (uncompressed) each | ~1.8 MB parse/execute for one page; ~900 KB free by deleting one tag |
| Primis | Two synchronous tags | Duplicate blocking work |
| reCAPTCHA | Loads on the homepage, which has **no form** | 383 KB transfer (870 KB uncompressed), 66–72% unused |
| Kameleoon | `async` but `fetchpriority=high` | An A/B tester competing with the hero image |

**Reproduce.** View source on `/`. Search for `dianomi` (two hits, same id), `ntv.io`
(two hits), `primis`, `recaptcha`, `fetchpriority`. For the blocking cost, Lighthouse →
"Eliminate render-blocking resources" lists the three head scripts with their ms cost.

**Fix.** In order of effort: delete the duplicate Dianomi and Nativo tags (two one-line
deletions, ~900 KB of parse saved on the Nativo one alone); move reCAPTCHA to the pages
that have forms; drop Kameleoon's `fetchpriority`; convert Parse.ly and the `apcdp`
script to `defer` (or load them after first paint). OneTrust is the one to be careful
with — consent has legal ordering constraints, so confirm with whoever owns compliance
before making it non-blocking.

**How you know it worked.** The render-blocking audit shrinks toward zero; FCP and LCP
move together. The duplicates are binary: view source, count tags.

**Scope.** Each fix is small and local, but the tags have different owners (ads, video,
consent, experimentation), so the work is coordination-heavy rather than code-heavy.

---

### Finding 1 — Mobile is where it's actually slow (umbrella)

**Mechanism.** Not a separate defect — it's the compound effect of everything else on a
slow radio and a slow CPU. Field mobile fails LCP (2.9 s); the throttled lab run scores
40. The 10.27 MB / 926 requests simply don't fit through Slow 4G before a reader gives
up.

**What to actually do with this finding.** Two things, neither of which duplicates
other tickets:

1. **Adopt the throttled-mobile setup above as the team's regression baseline.** Every
   fix in this report should be verified on it, not on desktop.
2. **Set a performance budget** on that baseline (e.g., lab LCP and total transfer
   bytes) and wire it into CI or a scheduled Lighthouse run, so the wins from the other
   findings don't erode.

The load-reduction work itself lives in findings 3, 8, 13, 11, 10, 5.

---

### Finding 11 — One CSS bundle for the whole site, 90% unused

**Mechanism.** All first-party CSS is a single `All.min.<hash>.gz.css`: 801 KB
minified, 107 KB on the wire, 6,264 rules. Coverage on the homepage: **90% unused**
(89% by uncompressed bytes — 700 of 782 KB). CSS is render-blocking by nature, so every
page pays download+parse on the entire site's design system before it can paint.

**Reproduce.** DevTools Coverage panel, reload on `/`, find `All.min.css` in the list —
the unused percentage is right there, and clicking it shows rule-by-rule red/green.

**Fix.** Two viable shapes, pick one with the CMS team:
- Split the stylesheet per page template (homepage, article, hub, search), or
- Keep one bundle but extract **real** critical CSS per template, inline it, and load
  the bundle `media="print"`-swap / `rel="preload"` async.

Either way the existing content-hash + immutable caching keeps working.

**How you know it worked.** Coverage unused % drops; render-blocking CSS bytes in
Lighthouse drop; FCP moves.

**Scope.** **Structural.** The bundling is inside the Brightspot CMS build
(`data-bsp-main-js` marks the build), and 6,264 accumulated rules mean any split risks
breaking pages nobody looks at. This is a team-lead conversation, and it needs visual
regression coverage across templates before shipping.

---

### Finding 10 — One JS bundle for the whole site, 79% unused

**Mechanism.** Same architecture as the CSS, same name: `All.min.<hash>.gz.js` —
456 KB minified, 113 KB on the wire — containing jQuery, the Flickity carousel, and
every web component of every page. Coverage on the homepage: **79% unused** (~89 of
113 KB). Every page ships the whole site's behavior to run one page's worth.

**Reproduce.** Coverage panel again; or fetch the bundle from `assets.apnews.com` and
read it — the webpack module list makes the contents obvious.

**Fix.** Split into a small shared core plus per-template or per-component chunks
loaded on demand (dynamic import at web-component definition time is the natural seam,
since the site is already component-structured). Content-hash caching survives the
split unchanged.

**How you know it worked.** Coverage unused % on the entry bundle drops; TBT improves
on the 4× CPU run.

**Scope.** **Structural** — same CMS build as finding 11, same conversation. Worth
doing 10 and 11 as one project since they're the same build change twice.

---

### Finding 6 — Blank screen: no real critical CSS

**Mechanism.** FCP is 2.2 s on both mobile and desktop. The document arrives fast
(TTFB 0.2 s) but rendering waits on the blocking head resources: the full CSS bundle
(finding 11) plus the synchronous scripts (finding 13). The page *looks* like it does
critical CSS — there are **18 inline `<style>` tags (~35 KB)** in the HTML — but read
them: they're ad-hoc patches (theme variables, ad-slot spacing, widget tweaks), not an
extracted critical path. The actual above-the-fold styling still comes from the
blocking bundle.

**Reproduce.** View source, count the `<style>` tags and skim their contents. Then
Lighthouse's render-blocking audit for the FCP cost.

**Fix.** Generate real critical CSS per template (tooling like `critical`/`penthouse`
against the top 2–3 templates), inline it, async-load the bundle, and — important —
delete the 18 ad-hoc blocks as their contents get absorbed, or the inline CSS grows
without bound.

**How you know it worked.** FCP in the throttled lab drops well under 2 s; the
render-blocking audit no longer lists first-party CSS.

**Scope.** Sits between local and structural: the extraction tooling is standard, but
keeping the extract correct as templates change needs a build step, which lands in the
same CMS build as 10/11. Bundle these three.

---

### Finding 2 — JavaScript costs 4× more on a phone (umbrella)

**Mechanism.** Same code as findings 4/10/13, measured on the CPU axis instead of the
network axis: a mid-range phone CPU is ~4× slower, so the ~2.35 s of long tasks in the
load recording is a laptop-invisible, phone-dominant cost.

**What to do with it.** No separate fix — the code reduction happens in 10 (first-party)
and 13/5 (third-party). The developer action here is measurement discipline: **always
profile with CPU 4× on**, and track TBT on that setting as the budget metric. Without
throttling you will conclude the JS is fine. It isn't.

---

### Finding 4 — Too much JavaScript overall (mostly not yours)

**Mechanism.** 161 JS requests, 4.44 MB transferred, 15.20 MB uncompressed — 43% of all
transferred bytes. But the composition matters for who fixes it: Lighthouse estimates
**~2.5 MB of unused JS**, and almost all of it is third-party (Nativo 83% unused,
Freestar's prebid.js 81%, JW Player's HLS code 85%, Google ads 75%, reCAPTCHA 66%).
The first-party share is one 113 KB bundle (finding 10).

**Reproduce.** Lighthouse "Reduce unused JavaScript" audit — the per-script table is
the evidence above.

**Fix routing.** First-party → finding 10. Third-party duplication and load order →
finding 13. Third-party *existence* → finding 5. There is no separate ticket here; this
finding exists so the 4.44 MB headline number doesn't get triaged as one giant task.

---

### Finding 12 — No source maps in production

**Mechanism.** Neither `All.min.js` nor `All.min.css` carries a `sourceMappingURL`
comment, and guessing the `.map` URL 404s. Production errors point into 456 KB of
minified code — unreadable stack traces, which slows down fixing every *other* finding.

**Reproduce.** `curl -s <bundle-url> | tail -c 200` — no map comment. Append `.map` to
the bundle URL — 404.

**Fix.** Turn on external source map emission in the webpack build (`devtool:
'source-map'`), upload the `.map` files next to the bundles. External maps are only
fetched when DevTools is open, so readers never download a byte. If exposing them
publicly is a concern, restrict by referer/IP or upload to the error tracker instead —
but ship them somewhere.

**How you know it worked.** The `.map` URL returns 200 and DevTools shows original
sources; error-tracker stack traces become readable.

**Scope.** A build flag plus an upload step. The cheapest item in this report.

---

### Finding 5 — The third-party population itself

**Mechanism.** 866 of 926 requests and 7.28 MB of 10.27 MB — **71% of the page** — is
third-party. The head alone loads: OneTrust, Kameleoon, Parse.ly, Permutive, Quantcast,
Sailthru, Wunderkind, GTM, Freestar/pub.network, Nativo, Dianomi, Primis, JW Player,
Viafoura, Zephr, reCAPTCHA, a quiz widget (Riverdrop), and one script from
`html-load.cc` nobody could identify. Main-thread cost: ~680 ms blocked by third
parties in the mobile lab; JW Player alone 672 ms of main-thread work, Viafoura 895 ms
— for a comments widget that isn't visible on the homepage, which also injects ~180 KB
of CSS that Coverage measures 98–100% unused.

**Related mechanism — scroll jank.** The scroll recording (300 frames, CPU 4×) shows
20 dropped frames, 16 severe (>50 ms), worst 367 ms, with 14 long tasks during the
scroll. The tasks line up with ad slot init/refresh (Freestar/GPT) and lazy embeds
(JW Player, Viafoura) booting exactly when they enter the viewport — i.e., on the
interaction path.

**Reproduce.** Lighthouse third-party summary for the per-vendor cost table. For scroll:
Performance panel, record while scrolling the full homepage, look for long tasks
stacked at ad-slot positions.

**Fix — the engineering part** (the *which vendors stay* part is a business decision;
flag it up, don't decide it in a PR):
- **Facades for the heavy widgets:** JW Player behind a click-to-load poster; Viafoura
  loaded only when the comments container approaches the viewport — that alone removes
  ~1.5 s of main-thread work and 180 KB of dead CSS from the homepage.
- **Move slot init off the scroll path:** schedule ad init/refresh in `requestIdleCallback`
  and widen the lazy-load rootMargin so slots initialize *before* they're visible
  instead of mid-scroll. This targets the 367 ms frame drops directly.
- **Identify or remove `html-load.cc`.** An unidentifiable script executing on a news
  site is a security question, not just a performance one. Escalate it.

**How you know it worked.** Third-party main-thread ms in Lighthouse drops; the scroll
recording's severe-frame count drops toward the ~6 a comparable page gets.

**Scope.** **Structural and cross-team.** The facades and idle scheduling are
engineering; the vendor list is revenue. Separate the two in your ticket, or the whole
thing stalls on the revenue conversation.

---

### Finding 9 — 926 requests (umbrella)

Most of the count is the third-party population (finding 5) and the image volume
(finding 8). No separate work: as those land, re-export the HAR and watch the request
count fall as a tracking metric, not a target.

---

## Two quick wins that didn't make the numbered list

Small items from the deep-dive (`baseline.md`), each nearly free:

- **Scope the hamburger menu's `will-change: transform`.** It's declared in the
  stylesheet, so two compositor layers exist for the page's entire lifetime for a menu
  that's almost never open. Add the property via a class when the menu opens, remove on
  close. Verify in the Layers panel: 2 fewer permanent layers.
- **The hero's 8 MB original is one copy-paste away.** The `dims.apnews.com` resizer
  URL carries the source as a plain `?url=` parameter, and the original 6000×4000 JPEG
  downloads directly. Readers never receive it, but it's exposed for hotlinking. Sign
  or restrict the origin parameter. (Cost risk, not page speed.)

---

## Domains I checked that need no fix — and why

So nobody re-audits these or assumes I skipped them:

- **Layout stability (CLS).** 0.04 mobile / 0.02 desktop, both good. Images and ads get
  reserved space (`width`/`height` on tags). Nothing to do except not regress it.
- **Interactivity (INP).** 187 ms mobile / 163 ms desktop, both good. The JS problem is
  a *loading* cost, not a responsiveness cost — post-load, the page reacts fine.
- **Server / TTFB.** 0.2 s. The origin+CDN answer fast; no backend work is warranted
  anywhere in this report.
- **Text compression.** Scripts and styles come over br/gzip/zstd; transfer is 62%
  below resource size. The uncompressed requests are images/fonts, which don't compress
  again. Already correct.
- **Image pipeline mechanics.** Server-side resize, WebP + fallback via `<picture>`,
  1x/2x srcset, dimensions set, hero preloaded with `fetchpriority=high`. The pipeline
  is right; finding 8 is about its settings and volume, not its design.
- **Compositing and animation.** 27 layers, all explainable (fixed header, ads, video,
  widgets); no `translateZ(0)`-style hacks; the animations run on transform/opacity.
  The jank in the scroll test is main-thread long tasks, not paint or compositing —
  which is why the fix lives in finding 5, not in CSS.
- **Rendering strategy.** SSR from the CMS + long CDN cache + purge-on-publish is the
  right architecture for a news site, full stop. The two costs it has grown — the
  2.4 MB homepage document and the missing revalidation path — are covered in findings
  3 and 7 respectively. Do not let anyone propose an SPA rewrite off the back of this
  report.

**Considered, judged out of scope:**

- **Offline support / service worker.** I didn't audit for one, and I don't recommend
  building one for this: anonymous news reading is link-in, read, leave — a service
  worker adds a cache-invalidation liability on a site whose whole model is
  purge-on-publish freshness. If the team ever wants "read offline" as a product
  feature, that's a product decision first, not a performance fix.
- **What I couldn't verify headless:** real-device feel (first-frame jank on actual
  hardware) and per-layer memory in the Layers panel. The scroll numbers strongly
  suggest any felt jank is long tasks, but a manual pass on a physical mid-range phone
  is the remaining confirmation step. If someone picks up finding 5, do that pass
  first — it's an afternoon and it hardens the ticket.

---

## Suggested working order, restated as a plan

1. **Week-one, no permission needed:** duplicate tags out (13), source maps on (12),
   Kameleoon priority off (13), `will-change` scoped, document `ETag`s (7).
2. **Sprint-sized, own-team:** image quality/AVIF/lazy tail (8), reCAPTCHA relocation
   (13), JW Player/Viafoura facades and idle-scheduled ad init (5, engineering half).
3. **Structural, needs the CMS/build owners:** bundle splitting + real critical CSS
   (10, 11, 6 as one project), homepage promo-tail rendering (3's document half).
4. **Cross-org:** the vendor list (5, business half), including the `html-load.cc`
   escalation.

Verify every step on the throttled mobile setup at the top of this file. The two
numbers that tell the overall story: mobile lab Performance score (40 today) and field
mobile LCP (2.9 s today, target < 2.5 s).
