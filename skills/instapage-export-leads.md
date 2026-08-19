---
name: Export Instapage form submissions (leads)
description: Page through form submission data for a set of landing pages over a time range, using the opaque nextPageToken cursor, and handle the PII and rate-limit rules correctly.
api: openapi/instapage-form-submissions-openapi.yml
operations: [listWorkspaces, listPages, retrieveFormSubmissions, deleteFormSubmissions]
generated: '2026-08-13'
method: generated
source: https://devdocs.instapage.com/
---

# Export Instapage form submissions (leads)

Form submissions are the lead records captured by forms on Instapage landing pages. Two things about
this endpoint surprise people: **it is a POST, not a GET**, and it does not use the `?page=`
pagination every other Instapage collection uses — it uses an opaque cursor in the request body.

## Before you start

- `Authorization: Bearer <personal API token>`, base `https://api.instapage.com/v1`.
- **This endpoint returns personal data.** `data[].fields` is whatever the customer's form collected
  — names, emails, phone numbers. Treat the response as PII: do not log it, do not put it in a
  prompt you would not put a customer record in, and do not persist it outside the user's control.
- 200 req/min per token and per IP; the daily quota is 5,000 calls on Create and 10,000 on Optimize.
  A full lead export can burn a meaningful fraction of a day's quota — count your pages first.

## Steps

1. **Resolve the workspace.** `listWorkspaces` — `GET /workspaces`, read `data[].workspaceId`.

2. **Resolve the page IDs.** `listPages` — `GET /workspaces/{workspaceId}/pages`. Collect
   `data[].id`. **The submissions endpoint accepts a maximum of 100 page IDs per request** — if the
   workspace has more, chunk into batches of 100 and repeat the whole loop below per batch.

3. **First page of results.** `retrieveFormSubmissions` —
   `POST /workspaces/{workspaceId}/submissions` with

   ```json
   {"pages": [9371, 9370], "timeframe": {"start": 1717259412, "end": 1727251907}}
   ```

   `timeframe.start` and `timeframe.end` are UNIX timestamps **in seconds**, UTC. Both are optional.

4. **Page through.** The response carries `meta.nextPageToken` and `meta.limit` (capped at 100).
   Re-POST the *same* body plus `"nextPageToken": "<token>"`. Stop when `meta.nextPageToken` is
   `null` or fewer than `limit` records come back. There is no `totalItemsCount` here — you cannot
   know the total up front, so write the loop with a hard iteration ceiling as well.

5. **Read the records.** Each item is `{id, pageId, variationName, variationCustomName, createdAt,
   fields}`. `createdAt` is UNIX seconds. `fields` is a free-form key/value map whose keys are the
   customer's form field names — do not assume any particular key exists.

## Deleting submissions — destructive, confirm first

`deleteFormSubmissions` — `DELETE /workspaces/{workspaceId}/submissions` with
`{"submissions": ["<id>", ...]}`, between 1 and 100 IDs per request, returns `204`.

**This is irreversible.** There is no soft delete, no undo, and no idempotency key. Always:

- get explicit human confirmation naming the count and the time range before the first call;
- delete in batches of ≤100 and record which IDs succeeded, because a retry after a timeout cannot
  be distinguished from a first attempt;
- expect `400` for an empty list or more than 100 IDs, and `403` when the token lacks permission.

## Rate-limit handling

On `429`, read `Retry-After` and sleep that many seconds. If the header is absent, back off
exponentially from 1s with a retry ceiling. Instapage warns that high-frequency requests may be
throttled *before* reaching 200/min, so pace a bulk export rather than bursting.
