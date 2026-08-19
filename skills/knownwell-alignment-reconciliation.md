---
name: Reconcile Knownwell alignment reads against the score
description: Pull the weekly Red/Amber/Green alignment reads a client team files by hand, compare each one against the machine-computed Knownwell score, and surface the disagreements — the clients a human calls green that the data calls at-risk, and the reverse.
api: openapi/knownwell-alignment-api-openapi.yml
base_url: https://api.knownwell.com/ci/v1
operations:
  - list_client_alignments_v1_clients_alignment_get
  - get_client_alignment_v1_clients__client_id__alignment_get
  - get_clients_by_risk_v1_clients_by_risk__risk_level__get
  - get_trending_clients_v1_clients_trending_get
  - get_client_v1_clients__client_id__get
  - get_client_priorities_v1_clients__client_id__priorities_get
generated: '2026-08-13'
method: generated
source: openapi/_original/knownwell-ci-openapi-original.json + https://api.knownwell.com/ci/docs#alignment
---

# Reconcile Knownwell alignment reads against the score

Alignment is the only signal in Knownwell a human types in. Every other number — the Knownwell
score, the risk band, the trend — is computed. This skill exists to find where the two disagree,
because that gap is usually where a renewal gets lost.

## Before you start

- Every request needs the `X-API-Key` header. Base URL `https://api.knownwell.com/ci/v1`.
- An alignment read is `red`, `amber`, or `green`, filed against a **week** (the Monday of that
  week), by a named person. It **carries forward**: a read stays current until someone files a
  newer one, so "this week's read" may have been entered weeks ago.
- A client with no read at all comes back as `alignment: null`. That is a coverage gap, not a
  green.

## Steps

1. **Pull the current reads.** Call `list_client_alignments_v1_clients_alignment_get`. Leave
   `week` off to get the latest read per client. Pass `week=YYYY-MM-DD` (a Monday) to resolve the
   book as it stood in a given week — reads filed for later weeks are ignored, so this is a true
   point-in-time view, not a rewrite.

2. **Page the whole book.** `limit` defaults to 500 and maxes at 1000 on this endpoint — higher
   than the 500 cap elsewhere in the API. Use `total` to confirm you have everyone, and
   `offset` to walk the rest.

3. **Split the response three ways.** For each entry read `alignment.value`,
   `knownwellScore`, and `scoreSource`:
   - `alignment: null` → **unread**. Nobody has filed on this client.
   - read present → keep `alignment.week`, `alignment.updatedAt`, `alignment.updatedBy`.

4. **Flag stale reads.** Compare `alignment.week` to the current week. A green filed six weeks
   ago is not a current green; report it as stale and name `updatedBy` so someone can ask.

5. **Get the machine's opinion.** Call `get_clients_by_risk_v1_clients_by_risk__risk_level__get`
   for `high_risk` and `medium_risk`, and `get_trending_clients_v1_clients_trending_get` with
   `direction=declining`.

6. **Reconcile.** Produce two lists, and lead with the first:
   - **False comfort** — `alignment.value` is `green` but the client appears in `high_risk` or in
     the declining trend. These are the dangerous ones.
   - **Over-worry** — `alignment.value` is `red` but the score is on track and stable. Worth
     asking what the team knows that the data does not; often a real signal the model cannot see.

7. **Explain each disagreement.** For every client on either list call
   `get_client_v1_clients__client_id__get` for `spotlightSummary` and `topics`, and
   `get_client_priorities_v1_clients__client_id__priorities_get` for the open action items.
   Use `get_client_alignment_v1_clients__client_id__alignment_get` when you need one client's
   read on its own rather than re-pulling the list.

8. **Join back to their systems.** `externalClientId` carries the customer's own imported
   "Client ID" value when one was mapped. Include it in any output that has to line up against a
   CRM or billing system; it is null when no ID column was imported.

## Rules

- **Never treat `alignment: null` as green.** It means no human has looked. Report unread count
  as its own number — coverage of the alignment ritual is itself a finding.
- **`scoreSource`** is `chr1` or `chr2`. Do not compare scores across sources without saying so;
  a customer mid-cutover has both in the same list.
- **Pagination.** This endpoint's `limit` default is 500 / max 1000, unlike the 100/500 used by
  `/v1/clients`. Do not hard-code one paging rule across the API.
- **Rate limits.** 100/minute, 5,000/hour, 50,000/day. Steps 7 fans out one call per flagged
  client — watch `X-RateLimit-Remaining` and back off to `X-RateLimit-Reset` on a 429.
- **Errors.** `{"error", "detail", "status_code"}`. A malformed `week` fails validation with a
  422 and a `detail[]` array; the parameter is pattern-checked as `^\d{4}-\d{2}-\d{2}$`.
- **Read-only.** Alignment reads are filed in the Knownwell app, not through this API. You can
  read a read; you cannot write one.
