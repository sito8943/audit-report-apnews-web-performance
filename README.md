# AP News Performance Audit

AP News is a strong candidate for a performance audit because it is a content-heavy news site with many images, article pages, category pages, search pages, and interactive modules. The initial review shows that the site is functional, but it still struggles with rendering speed and main-thread overhead.

## Target Pages

- https://apnews.com/

  Is the main entry point

- https://apnews.com/entertainment

  It represents how users browse news by topic.

- https://apnews.com/hub/fifa-world-cup
- https://apnews.com/photo-gallery/world-cup-photos-mbappe-haaland-jimenez-57fd3b1070ed79152dfa89b7319f6139
- https://apnews.com/article/world-cup-schedule-results-news-81645977a722c4020c9644d17589bdbb

  These 3 from word cup are the hype today

- https://apnews.com/donate

  Because performance can affect user trust and conversion

- https://apnews.com/search?q=world+cup

  Because search is a key way users find content

## Reports

- `baseline.md`: Initial Core Web Vitals and PageSpeed Insights results.
- `findings.md`: Main performance issues found during the audit.
- `prioritization.md`: The ICE scoring I use to rank the fixes, and the resulting order.
