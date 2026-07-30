---
name: Triage at-risk Knownwell clients
description: Find the clients whose Knownwell score puts the relationship at risk, pull the evidence behind each score, and hand back a ranked triage list with the people to contact.
api: openapi/knownwell-ci-openapi-original.json
base_url: https://api.knownwell.com/ci/v1
operations:
  - get_clients_by_risk_v1_clients_by_risk__risk_level__get
  - get_trending_clients_v1_clients_trending_get
  - get_client_v1_clients__client_id__get
  - get_client_history_v1_clients__client_id__history_get
  - get_client_priorities_v1_clients__client_id__priorities_get
  - get_client_key_people_v1_clients__client_id__key_people_get
generated: '2026-07-19'
method: generated
source: openapi/knownwell-ci-openapi-original.json + https://api.knownwell.com/docs
---

# Triage at-risk Knownwell clients

Use this when someone asks which client relationships are in trouble and what to do about them.

## Before you start

- Authenticate every request with the `X-API-Key` header. Keys are read-only; nothing in this
  skill mutates data.
- The base URL is `https://api.knownwell.com/ci/v1`.
- Scores refresh once a day at 00:00 UTC. Do not re-poll within a day expecting movement.

## Steps

1. **Pull the high-risk cohort.** Call `get_clients_by_risk_v1_clients_by_risk__risk_level__get`
   with `risk_level=high_risk`. Repeat with `medium_risk` only if the high-risk list is empty or
   the user asked for a wider net. Valid values are `high_risk`, `medium_risk`, `low_risk`,
   `on_track` — anything else returns 422.

2. **Add the decliners.** Call `get_trending_clients_v1_clients_trending_get` with
   `direction=declining`. A client that is still scored `on_track` but falling fast is often the
   more useful lead than one that has been red for months. Merge the two lists on `id`.

3. **Skip the noise.** Drop any client whose `hasInsufficientData` is true — Knownwell is telling
   you it does not have enough signal to score the relationship, so a low score there is not
   evidence of risk.

4. **Get the evidence per client.** For each surviving client, call
   `get_client_v1_clients__client_id__get`. Read `spotlightSummary` for the narrative, `topics`
   for which scoring dimensions are dragging, and `scoreChanges` for the size and direction of
   recent movement.

5. **Confirm the trend is real.** Call `get_client_history_v1_clients__client_id__history_get` to
   see the score curve. Up to 365 days are available. Distinguish a genuine slide from a single
   bad week.

6. **Collect the actions already on file.** Call
   `get_client_priorities_v1_clients__client_id__priorities_get`. Report what the team has
   already committed to before proposing anything new.

7. **Name the humans.** Call `get_client_key_people_v1_clients__client_id__key_people_get` so the
   output says who to contact, not just which account is red.

8. **Return a ranked list.** Rank by score change magnitude first, absolute score second. For
   each client give: name, score, direction and size of change, the dragging topics, the open
   priorities, the key contacts, and `lastContactDate`.

## Rules

- **Pagination.** List endpoints take `limit` (default 100, max 500) and `offset`. Requesting
  `limit` above 500 returns 422. Page through until you have fewer results than `limit`.
- **Rate limits.** 100 requests/minute, 5,000/hour, 50,000/day. Watch `X-RateLimit-Remaining`;
  on 429 wait until the Unix timestamp in `X-RateLimit-Reset`. Step 4 onward is one call per
  client, so cap the cohort or batch it across windows.
- **Errors.** Failures come back as `{"error": ..., "detail": ..., "status_code": ...}`. 401
  means the key is missing or invalid, 403 means the key is scoped to a different customer, 404
  means the `client_id` is wrong, 422 carries a `detail[]` array naming the bad parameter.
- **Read-only.** There is no write path. Never claim to have updated a score, closed a priority,
  or contacted anyone.
