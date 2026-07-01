# AP News Performance Audit

AP News is a strong candidate for a performance audit because it is a content-heavy news site with many images, article pages, category pages, search pages, and interactive modules. The initial review shows that the site is functional, but it still struggles with rendering speed and main-thread overhead.

## Audit Scope

The audit focused on the following pages:

- https://apnews.com/ — main entry point
- https://apnews.com/entertainment
- https://apnews.com/hub/fifa-world-cup
- https://apnews.com/photo-gallery/world-cup-photos-mbappe-haaland-jimenez-57fd3b1070ed79152dfa89b7319f6139
- https://apnews.com/article/world-cup-schedule-results-news-81645977a722c4020c9644d17589bdbb
- https://apnews.com/hub/quizzes
- https://apnews.com/donate
- https://apnews.com/search\?q\=world+cup

## Executive Summary

The main performance problem is slow rendering of the primary content. Largest Contentful Paint is the clearest issue, while layout stability remains acceptable. The site also appears to spend a large amount of time executing JavaScript, which contributes to a sluggish experience.

## Report Files

- baseline.md: Initial Core Web Vitals and Lighthouse findings.
- findings.md: Main performance issues and recommended improvements.

## Recommended Priorities

1. Optimize above-the-fold images and hero media.
2. Reduce JavaScript execution time and remove unused bundles.
3. Defer non-critical scripts and third-party embeds.
4. Improve caching and delivery of static assets.
5. Continue monitoring LCP, TBT, and Speed Index after each optimization pass.
