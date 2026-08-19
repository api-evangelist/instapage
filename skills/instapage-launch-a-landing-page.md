---
name: Launch an Instapage landing page
description: Create a landing page from exported Instapage JSON, file it in a group, publish it to a custom domain, and confirm the publish landed.
api: openapi/instapage-pages-openapi.yml
operations: [listWorkspaces, listGroups, createGroup, createPageFromJson, updatePage, publishPage, getPage]
generated: '2026-08-13'
method: generated
source: https://devdocs.instapage.com/
---

# Launch an Instapage landing page

Instapage has no "create a blank page" API. The only programmatic way to bring a new page into
existence is `createPageFromJson`, which imports an Instapage JSON document you obtained from
`exportPageAsJson` or from the Instapage editor. Plan for that: you need a source page first.

## Before you start

- Base URL `https://api.instapage.com/v1`. Every request carries `Authorization: Bearer <token>`.
- The token inherits its creator's permissions per workspace. Publishing needs at least editor.
- 200 requests/minute per token *and* per IP, plus a daily plan quota (5,000/day on Create,
  10,000/day on Optimize). A 429 carries `Retry-After` — honour it; do not tight-loop.
- **There is no idempotency key.** If a `createPageFromJson` call times out, do not blindly retry —
  call `listPages` and check whether the page already exists, or you will create a duplicate.

## Steps

1. **Find the workspace.** `listWorkspaces` — `GET /workspaces`. Filter with `?name=` if you know
   the name. Read `data[].workspaceId`. Check `data[].accessLevel` is `editor`, `manager` or
   `owner`; a `viewer` token will 403 later, not now.

2. **Pick or create the group (folder).** `listGroups` — `GET /workspaces/{workspaceId}/groups`.
   If the folder does not exist, `createGroup` — `POST /workspaces/{workspaceId}/groups` with
   `{"name": "..."}`. A duplicate name returns 409; an over-long name returns 413.

3. **Get the page content.** `exportPageAsJson` — `GET /workspaces/{workspaceId}/pages/{pageId}/json`
   against the template page. Read `data.content` — that array is what you pass on. AMP pages cannot
   be exported and return 400.

4. **Create the page.** `createPageFromJson` — `POST /workspaces/{workspaceId}/pages/json` with
   `{"title": "...", "content": [...]}`. Title max 255 characters. `is_amp` must not be true.
   Success is `200` (not 201) with `data.pageId`.

5. **File it in the group.** `updatePage` — `PATCH /workspaces/{workspaceId}/pages/{pageId}` with
   `{"groupId": <id>}`. Success is `201`. Page and group must be in the same workspace, or 404.
   Passing `{"groupId": null}` removes the page from all groups.

6. **Publish it.** `publishPage` — `POST /workspaces/{workspaceId}/pages/{pageId}/publication` with
   `{"targetUrl": "...", "publicationMethod": "customDomain"}`. Methods are `customDomain`,
   `freeDomain`, `wordpress`, `drupal`; for `wordpress`/`drupal` set `targetUrl` to `null`, and for
   `freeDomain` the URL follows `name.pagedemo.co`. Confirm the custom domain first with
   `listDomains` (`GET /workspaces/{workspaceId}/domains`) — a domain whose `connection` is not
   `connected` will not serve the page.

7. **Confirm.** Publication returns `202 Accepted` — it is asynchronous and the page is NOT live
   yet. Poll `getPage` — `GET /workspaces/{workspaceId}/pages/{pageId}` until `data.publishStatus`
   is `published` and `data.url` is non-null. Back off between polls; polling burns daily quota.

## Things that will bite you

- Publishing or unpublishing through the API **clears any scheduling** configured on that page.
- `403` on publish usually means the plan's published-pages limit is exhausted, not a token problem.
- To change the URL of a page that is already live, use `updatePublishedPageUrl`
  (`PUT .../publication`) — do not unpublish and republish. It 403s if the page has running
  experiments.
- Errors come back as `{"title", "details", "meta"}` with no machine-readable code. Branch on the
  HTTP status, never on the `title` string.
