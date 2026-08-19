---
name: Report on Instapage page performance
description: Pull visit, conversion and lead statistics for a set of pages, correlate them with running experiments, and aggregate correctly across UTC time buckets.
api: openapi/instapage-analytics-openapi.yml
operations: [listWorkspaces, listPages, getAnalytics, listExperiments, listPersonalizations]
generated: '2026-08-13'
method: generated
source: https://devdocs.instapage.com/
---

# Report on Instapage page performance

Analytics is a single bulk endpoint. Like form submissions, **it is a POST** — the query lives in
the request body, not in query parameters.

## Before you start

- `Authorization: Bearer <personal API token>`, base `https://api.instapage.com/v1`.
- Every timestamp in and out is **UNIX seconds, UTC**. Daily, monthly and yearly aggregation windows
  are aligned to UTC — if you report in a local timezone you will mis-attribute traffic at the day
  boundary.
- 200 req/min per token and per IP, plus the daily plan quota.

## Steps

1. **Resolve the workspace and pages.** `listWorkspaces` — `GET /workspaces`; then `listPages` —
   `GET /workspaces/{workspaceId}/pages`. Use the filters rather than paging everything:
   `publishStatus=published`, `isDeleted=false`, and the `createdAfter`/`createdBefore`,
   `updatedAfter`/`updatedBefore`, `publishedAfter`/`publishedBefore` UNIX-second ranges.

2. **Request the statistics.** `getAnalytics` — `POST /workspaces/{workspaceId}/analytics`:

   ```json
   {
     "pages": [20476657, 10716956],
     "timeframe": {"start": 0, "end": 1726150655},
     "interval": "monthly",
     "grouping": ["pageId"],
     "traffic": "blended",
     "device": "any",
     "visited": 0
   }
   ```

   - `pages` — **maximum 100 page IDs per request.** Chunk beyond that.
   - `interval` — `hourly`, `daily`, `monthly` (default) or `yearly`.
   - `grouping` — `pageId` and/or `variationId`. Whatever you group by appears in `data[].key`; the
     key object's shape *changes with the grouping*, so read it defensively.
   - `traffic` — `blended` (default), `organic` or `paid`.
   - `device` — `any` (default), `desktop` or `mobile`.
   - `visited` — `1` counts only returning visitors, `0` counts only unique actions.

3. **Read the rows.** Each item is `{key, visit, conversion, leads}`. `key.pageId`, `key.date` and
   `key.variationId` are each present only when you grouped by them.

4. **Correlate with experiments.** `listExperiments` —
   `GET /workspaces/{workspaceId}/experiments?status[]=RUNNING`. Statuses are `DRAFT`, `RUNNING`,
   `ENDED`, `ARCHIVED`; the parameter repeats to filter on several. Join on `data[].pageId`. A page
   with a `RUNNING` experiment is splitting traffic across variations — report those grouped by
   `variationId`, or the page-level conversion rate blends two different experiences and means
   nothing.

5. **Account for personalization.** `listPersonalizations` —
   `GET /workspaces/{workspaceId}/pages/{pageId}/personalizations`. A page whose
   `totalPersonalizedExperienceCount` is greater than zero is serving different content per segment;
   `data[].isDefaultPersonalization` marks the fallback experience.

## Things that will bite you

- `conversion` and `leads` are different numbers and Instapage documents them separately — do not
  treat them as synonyms in a report.
- This endpoint is paginated by nothing: there is no `meta.pagination` and no cursor. The response is
  whatever the grouping produces, so a wide timeframe with `hourly` interval across 100 pages is one
  very large response. Narrow the timeframe rather than the page list where you can.
- `404` here means the workspace or a page ID was not found, `403` means the token's access level is
  insufficient for the workspace.
