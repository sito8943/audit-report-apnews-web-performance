# Prioritization

I'm using **ICE** here: Impact × Confidence × Ease, each rated 1–10, multiplied together.
Higher score = do it first. (For my personal project I used WSJF, so this one is on
purpose a different system.)

The reason I picked ICE for this site is the Confidence rating. A lot of what I found on
AP News comes from lab runs on a throttled phone, and some of it is me connecting dots
(like "the CPU is 4× slower so the scripts take 4× longer") rather than something the
field data states directly. ICE lets me be honest about that: a finding backed by hard
evidence in the HAR gets a high confidence, a finding that's mostly inference gets marked
down, and the multiplication does the rest.

## The ratings

- **Impact** — if this got fixed, how much better does the page get for a reader?
  - _1–3_: they probably wouldn't notice
  - _4–6_: noticeable, the page feels somewhat better
  - _7–8_: a big chunk of the slowness goes away
  - _9–10_: it changes the whole experience of opening the page
- **Confidence** — how sure am I that the cause is what I think it is, and that the fix
  will actually move the metric?
  - _1–3_: mostly a guess
  - _4–6_: reasonable inference, but I'd want to measure again after
  - _7–8_: lab and field point the same way, or the data shows it directly
  - _9–10_: the evidence is right there in the responses (headers, byte counts)
- **Ease** — how cheap is it to do? (This is the opposite of effort — a 10 is trivial.)
  - _1–3_: a big project, lots of code or lots of teams
  - _4–6_: real work, but scoped
  - _7–8_: known technique, small change
  - _9–10_: basically a config flip

Score = Impact × Confidence × Ease, out of 1000.

Because it multiplies, one weak rating drags the whole score down — a huge fix nobody is
sure about, or a sure fix that's a monster to build, both sink. That's different from WSJF
(which adds the value side and divides by effort) and from a weighted reach/impact system:
ICE has no reach rating at all, no weights, and effort pushes the score down by
multiplication instead of division.

## Result

The scores and the reasoning for each rating are next to each finding in `findings.md`.
Ranked:

| # | Finding | Impact | Confidence | Ease | ICE |
|---|---------|--------|------------|------|-----|
| 3 | Main content shows up really late | 8 | 9 | 7 | **504** |
| 8 | Images and fonts are a real byte problem | 7 | 8 | 7 | **392** |
| 7 | Cache doesn't help on a second visit | 5 | 9 | 8 | **360** |
| 1 | Mobile is the one that's really slow | 9 | 8 | 4 | **288** |
| 6 | Blank screen for too long at the start | 6 | 7 | 5 | **210** |
| 2 | JavaScript is heavier on a phone | 7 | 7 | 4 | **196** |
| 4 | Way too much JavaScript | 8 | 8 | 3 | **192** |
| 5 | Third parties are taking over | 8 | 7 | 3 | **168** |
| 9 | There are way too many requests | 5 | 7 | 4 | **140** |

The order says: start with the well-understood, well-scoped fixes (prioritize the LCP
element, shrink the images, turn caching on), and leave the big JavaScript and third-party
cleanups for later — not because they don't matter (their impact ratings are the highest),
but because they're huge jobs and, in the third-party case, cutting ad scripts isn't
purely an engineering decision. ICE is blunt about that trade.
