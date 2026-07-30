---
name: Drill into a Knownwell client's streams
description: Decompose one client relationship into its underlying streams — the workstreams, engagements, or goals scored beneath the account — and explain which of them is moving the client's overall score.
api: openapi/knownwell-ci-openapi-original.json
base_url: https://api.knownwell.com/ci/v1
operations:
  - search_clients_v1_clients_search_get
  - get_client_v1_clients__client_id__get
  - list_client_streams_v1_clients__client_id__streams_get
  - get_stream_v1_clients__client_id__streams__stream_id__get
  - list_all_streams_v1_streams_get
  - get_client_notes_v1_clients__client_id__notes_get
generated: '2026-07-19'
method: generated
source: openapi/knownwell-ci-openapi-original.json + https://api.knownwell.com/docs
---

# Drill into a Knownwell client's streams

Use this when a client's headline score does not tell the whole story and someone needs to know
which part of the relationship is responsible.

## Before you start

- Every request needs the `X-API-Key` header. Base URL `https://api.knownwell.com/ci/v1`.
- A stream belongs to exactly one client via `parentClientId`, and carries its own `score`,
  `streamState`, `goals`, `scoreChanges`, and `historicalData`.

## Steps

1. **Resolve the client.** If you were given a name rather than an id, call
   `search_clients_v1_clients_search_get` with `query=<name>` and pick the match. Confirm the
   choice with the user when more than one client comes back.

2. **Read the account-level picture first.** Call `get_client_v1_clients__client_id__get` to
   capture the client's `score`, `scoreChanges`, `spotlightSummary`, and `topics`. This is the
   number the stream detail has to explain.

3. **List the streams.** Call `list_client_streams_v1_clients__client_id__streams_get` with the
   client id. If the client has no streams, say so plainly and stop — the account score is not
   decomposable and the answer lies in the topics instead.

4. **Get each stream in detail.** Call
   `get_stream_v1_clients__client_id__streams__stream_id__get` for each stream. Read `score`,
   `streamState`, `goals`, and `scoreChanges`.

5. **Filter out the unscoreable.** Ignore any stream where `hasInsufficientData` is true when
   attributing score movement; note them separately as coverage gaps.

6. **Attribute the movement.** Rank the remaining streams by the magnitude of `scoreChanges`.
   The streams with the largest negative deltas are the ones dragging the account; call that out
   explicitly against the client-level change from step 2.

7. **Add the human context.** Call `get_client_notes_v1_clients__client_id__notes_get` for
   recorded observations that may explain the movement.

8. **Optional: compare across the book.** `list_all_streams_v1_streams_get` returns every stream
   in the account, so you can say whether a stream's score is bad in absolute terms or merely
   below this client's own average.

## Rules

- **Pagination.** `limit` (default 100, max 500) and `offset` on the list endpoints; use `total`
  for counts.
- **Rate limits.** 100/minute, 5,000/hour, 50,000/day. Step 4 is one call per stream — for a
  client with many streams, watch `X-RateLimit-Remaining` and back off to `X-RateLimit-Reset` on
  a 429.
- **Errors.** `{"error", "detail", "status_code"}`. A 404 on the stream call usually means the
  `stream_id` belongs to a different client — the stream path is nested under `client_id`, and
  both must match.
- **Freshness.** Scores update daily at 00:00 UTC; history spans up to 365 days.
- **Read-only.** No stream state, goal, or note can be written through this API.
