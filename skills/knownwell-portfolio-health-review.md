---
name: Run a Knownwell portfolio health review
description: Assemble a portfolio-level health picture — overall statistics, per-portfolio rollups, and the score distribution across clients — for a recurring revenue or client-service review.
api: openapi/knownwell-ci-openapi-original.json
base_url: https://api.knownwell.com/ci/v1
operations:
  - get_portfolio_health_v1_clients_portfolio_health_get
  - get_statistics_v1_clients_statistics_overview_get
  - list_portfolios_v1_portfolios_get
  - get_portfolio_v1_portfolios__portfolio_id__get
  - list_clients_v1_clients_get
  - list_topics_v1_topics_get
  - export_clients_csv_v1_clients_export_csv_get
generated: '2026-07-19'
method: generated
source: openapi/knownwell-ci-openapi-original.json + https://api.knownwell.com/docs
---

# Run a Knownwell portfolio health review

Use this to produce the standing "how is the book of business doing" view rather than a
single-client answer.

## Before you start

- Send `X-API-Key` on every request against `https://api.knownwell.com/ci/v1`.
- Portfolio health metrics are calculated in real time; individual client scores refresh daily at
  00:00 UTC. Say which is which when you report numbers.

## Steps

1. **Start with the rollup.** Call `get_portfolio_health_v1_clients_portfolio_health_get` for
   aggregate health across the whole client base. This is the headline number.

2. **Add the overview statistics.** Call `get_statistics_v1_clients_statistics_overview_get` for
   the account-level counts and distribution that frame the rollup.

3. **Break it down by portfolio.** Call `list_portfolios_v1_portfolios_get`, then
   `get_portfolio_v1_portfolios__portfolio_id__get` for each portfolio you intend to report on.
   Each portfolio carries `name`, `clientCount`, `userIds`, and `isSystem` — mark system
   portfolios so readers know which groupings are automatic rather than curated.

4. **Get the client-level distribution.** Call `list_clients_v1_clients_get`, paging with
   `limit` and `offset`. Leave `include_archived` at its default of false unless the user
   explicitly asked to include archived clients. Bucket the results by score band and count how
   many carry `hasInsufficientData`.

5. **Explain what the score is made of.** Call `list_topics_v1_topics_get` once. It returns the
   fifteen topics and their categories (abbreviated SQP, RS, CA). Use the `displayName` and
   `description` fields when you name a dragging dimension, so the report reads in the customer's
   language rather than in raw ids.

6. **Offer the raw data.** If the user wants to work the numbers themselves, point them at
   `export_clients_csv_v1_clients_export_csv_get`, which returns the client list as CSV.

7. **Report.** Give the aggregate health figure, the per-portfolio breakdown with client counts,
   the score distribution, the count of unscoreable clients, and the topics most often dragging.
   State the as-of date of the score refresh.

## Rules

- **Pagination.** `limit` defaults to 100 and caps at 500; `offset` starts at 0. Keep paging
  until a page returns fewer rows than `limit`. Never infer a total from one page — use the
  `total` field.
- **Rate limits.** 100/minute, 5,000/hour, 50,000/day, signalled on `X-RateLimit-Limit`,
  `X-RateLimit-Remaining`, and `X-RateLimit-Reset`. Cache the result: scores only move once a
  day, so a cached review is valid for at least an hour.
- **Errors.** `{"error", "detail", "status_code"}`. 401 invalid or missing key, 403 wrong
  customer scope, 404 unknown `portfolio_id`, 422 validation (read `detail[]`), 429 rate limit,
  500 server-side.
- **Read-only.** Nothing here changes state. Do not offer to reassign clients or edit portfolios
  through the API — that happens in the Knownwell dashboard.
