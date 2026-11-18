## How the investigation went

- Ground News' CDN (CloudFront) returned 403 to REA's automated Chrome, which looks like bot filtering. The repository's rules say REA may not get past bot protection, so I made no attempt to disguise the browser.
- A plainly labelled research client was served normally. I fetched 4 public pages and 62 code bundles, with pauses between requests and within robots.txt. I called none of their APIs: the endpoints below are strings found in their code.
- REA's code analyzer would not run because Windows Smart App Control blocks its native module. I scanned the bundles with my own scripts instead; REA would need a Linux machine.
- I didn't check production's .env overrides on the host, since that needs your approval. Hidden View settings below are the compose defaults.

## What I could verify about Ground News

- Feed-based discovery: location comes from the feed an article arrived through. Feed IDs run up to about 3,192.
- Speed: across 331 articles, the gap between the publisher's timestamp and Ground News' own record was 36 minutes at the median and 5.3 hours at the 90th percentile. 73% arrived within an hour.
- Wire copies are grouped, not dropped: one story's 150 sources were 77 articles plus 73 reprints folded under them. The reprints still count toward the coverage and bias figures.
- Translation: 45 of 77 articles on one story had their headline translated to English, which lets a single story span many languages.
- Stories are updated in place: each story keeps a stable ID while a revision counter goes up as articles join (7 stories carried it; median 225, highest 385).
- Human corrections: readers can report a wrong story, article or image, readers vote on whether a story is right, and editors can override.
- Publisher data: each outlet carries separate bias ratings from three raters (Media Bias/Fact Check, Ad Fontes, AllSides), an ownership chain, timezone, language, and satire and link-shortener flags.
- Serving: a server-rendered Next.js site behind CloudFront, an API on web-api-cdn.ground.news, and images on Bunny CDN.

The clustering method, the wire-detection method, the translation provider and what sits behind an endpoint named /public/vectors/getStoryLevelTimeline/ cannot be seen from outside. The report labels them as hypotheses.

## What Hidden View needs, in priority order

1. More feeds: move the feed list into a database table with a place per feed, seeded from the publishers already admitted and the feeds the crawler already finds.
2. Measure and improve freshness: record when each article was first seen, and turn on the adaptive feed scheduler and conditional GET. Both exist in the code but are off by default.
3. Group reprints by text similarity: today wire outlets are simply left out of clustering. Grouping their copies would keep the coverage signal Ground News keeps.
4. Cross-language clustering: translate headlines, or use a multilingual embedding model. Either means a model service or sending article text to another vendor, so it needs your approval.
5. Story versions and a last-modified time, so caches can refresh one story at a time.
6. A reader "this doesn't belong" report that feeds into clustering as a hard rule.
7. Bias stored per rater, plus timezone and flags on publishers.
8. Scaling the hosting: the server and pipeline limits only bind after steps 1–3. Changes to the AWS setup need your approval.

Nothing was committed; the saved pages, scripts and REA evidence are in the same scratch folder as the report. I can copy the report into docs/ or publish it as a page to share.

## Two corrections to my earlier report

- Translation is already live. Headline translation (Google provider) has run since 2026-10-07, and clustering reads the English headline (RWE_CLUSTER_USE_TITLE_EN; docs/ingestion/HEADLINE_TRANSLATION.md). Recommendation 4 adds nothing.
- "8 feeds" undercounts RSS. The mounted feed list has 8, but RSS also reaches about 151 more publisher feeds (docs/assets/user-facing-features/hidden-view/README.md:11). The breadth gap is real, but smaller than I implied.

The labels below follow the repository's convention: [M] measured (taken from the repository's audits, not re-measured on the host), [D] arithmetic on measured values, [P] projection with the assumption stated.

## Baseline

| Measure | Value | Source |
|---|---|---|
| Articles per day into the story window | ~12,200 (73,010 rows over 6 days, 2026-10-02) [M] | ACTION_ITEMS.md:123 |
| Outlets seen in a 6-day window | ~4,200–4,900, about 1.0 article per outlet per day [M] | SCALE_ROADMAP.md, SOURCE_PIPELINE_50K_AUDIT.md |
| Google News share of rows | 44% (6,963 of 15,808). Opaque links, headline only, no body ever [M] | GOOGLE_NEWS_ONLY_REVIEW.md |
| Publishers reached only through Google News | 181 publishers, 31% of rows [M] | same |
| Overlap of a publisher's own feed with its Google News rows | 27% [M] | same |
| Major publishers readable directly | 8 of 18 (44%) [M] | same |
| Story scan cap | 70,000 rows, already binding: the oldest 7.7 h were dropped [M] | ACTION_ITEMS.md:123 |
| Time from publication to ingest | Not measured. Ground News: 36 min median, 73% within 60 min [M, my sample] | — |

## Change by change

| # | Change | Discovery | Total ingested per day | Unique articles | Publisher coverage | Freshness | Effect |
|---|---|---|---|---|---|---|---|
| 1 | Feeds table, section feeds, place per feed | ↑↑ | +2,500 to +6,000 (+20–50%) [P] | +1,800 to +4,500 [P] | +0 to 150 new outlets. Mostly the same publishers, now with full text | small ↑ | Ingestion |
| 2 | Adaptive scheduler, conditional GET, first-seen metric | ↑ for busy feeds | +0 to 2% [P] | +0 to 2% | none | ↑↑: average wait for active feeds 5 min → 1 min [D] | Freshness (adds little volume) |
| 3 | Group wire copies by text similarity | none | 0 | content-unique count goes down; story membership goes up | wire outlets return to stories | none | Downstream |
| 4 | Translation | — | 0 (already live) | 0 | 0 | 0 | Done |
| 5 | Story version and last-modified | none | 0 | 0 | 0 | served stories update sooner | Downstream |
| 6 | Reader corrections | none | 0 | 0 | 0 | 0 | Downstream |
| 7 | Per-rater bias, timezone, satire and shortener flags | none | 0 or slightly ↓ (filters) | 0 | richer ratings | more accurate publish dates | Downstream |
| 8a | Crawl fleet over the 22,899-outlet index | ↑↑↑ | +2,700 to +14,000 [P] | most of it | +5,000 to +10,000 outlets (2–3×) [P] | per host, about every 2 h | Ingestion; needs owner (cost, AWS) |
| 8b | Raise the scan cap, or attach long-tail outlets to existing stories | none | 0 | 0 | — | — | Required for 1 and 8a to reach stories |

## Assumptions behind the projections

- Change 1: one feed carries about a quarter of a newsroom (the measured 27%). Section feeds for about 150 directly read publishers lift each 2–3×. Direct RSS is about 35–45% of today's rows; there is no per-channel count in the repository, so that share is an assumption. About a quarter of the new rows duplicate a Google News row, so total ingested rises faster than unique articles. Publisher counts stay correct because counts already collapse per outlet (story_service.py:1879).
- Change 2: the average wait from polling is half the interval. A 600 s interval gives 5 min; the scheduler's 120 s floor gives 1 min. Feeds that rarely change are stretched toward the 6 h ceiling, which costs little because they rarely publish. Extra articles come only from feeds that roll over faster than the 10-minute poll.
- Change 8a: 40–60% of index outlets are reachable (44% of majors were; guessed feed URLs return 404). The long tail publishes 0.3–1.0 articles a day, below the catalogue's average of 1.0, which the big outlets pull up. The 536-outlet test gave 9,764 new articles in one cycle [M], but that included backlog, so it is not a daily rate.
- Change 3: Ground News shows the scale. In the story I sampled, 73 of 150 sources (49%) were wire copies. Hidden View's share of excluded wire rows hasn't been measured.

## Combined projection

| Scenario | Articles per day | Change | Outlets in the window |
|---|---|---|---|
| Baseline | ~12,200 | — | ~4,200–4,900 |
| Changes 1 and 2 (no AWS) | ~14,700–18,500 | +20–50% | about the same, but ~31% of rows move from headline-only Google News toward full text |
| Changes 1, 2 and 8a | ~17,400–32,500 | +40–165% | ~9,000–15,000 |

## The scan cap

More ingestion means fewer days of story coverage unless change 8b lands first. The cap is a row count, so the window shrinks as volume grows [D]:

- Today, 70,000 ÷ 12,200 per day covers 5.7 days of the intended 6.
- At +50% volume it covers 3.8 days.
- At +100% volume it covers 2.9 days.

SCALE_ROADMAP.md already warns that this cap turns extra ingestion into less coverage without raising any error. Raising it is a deploy/.env change, so it needs your decision.

Freshness can't be compared with Ground News yet, because Hidden View doesn't record the gap between publication and first sight. Adding that metric (part of change 2) is the first step: without it, no freshness gain can be shown.

What increases ingestion: changes 1 and 8a, and 2 slightly. What improves processing downstream: changes 3, 5, 6, 7 and 8b; 4 is already live. The first step I'd take is to measure the current per-channel and per-publisher rows per day and the ingest lag on the host. That turns these [P] figures into [M] figures, and it needs your approval for read-only host access.

✻ Baked for 2m 21s · done Tuesday 7:51 PM

> Implement the proposed changes in Hidden View based on the findings discussed above.
