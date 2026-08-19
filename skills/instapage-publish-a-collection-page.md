---
name: Publish an Instapage collection page
description: Add a placeholder-driven page to an existing Instapage collection, wire an image asset into it, publish it, and unpublish or delete it cleanly.
api: openapi/instapage-collections-openapi.yml
operations: [listCollections, getCollection, listCollectionPages, createCollectionPage, publishCollectionPage, unpublishCollectionPage, deleteCollectionPage, listImageFolders, listImagesInFolder, uploadImage]
generated: '2026-08-13'
method: generated
source: https://devdocs.instapage.com/
---

# Publish an Instapage collection page

A **collection** is a set of pages sharing one template, differentiated by named placeholders — the
programmatic-SEO / localized-campaign surface. A **collection page** is one instance with its own
URL and its own placeholder values. This is the closest Instapage gets to a content API.

Note the asymmetry: you can create, publish, unpublish and delete collection *pages* through the
API, but you cannot create a *collection* — the template must already exist, built in the editor.

## Before you start

- `Authorization: Bearer <personal API token>`, base `https://api.instapage.com/v1`.
- The token needs at least **editor** on the workspace; anything less returns `403`.
- Each collection counts as one published page against the plan, and the pages-per-collection cap is
  plan-tiered (15 on Create, 30 on Optimize, 50 higher up). Exceeding it returns `403`.

## Steps

1. **Find the collection.** `listCollections` — `GET /workspaces/{workspaceId}/collections`
   (`?page=` paginated). Read `data[].id`, `data[].name` and `data[].pagesCount.all` /
   `.published` so you know how close you are to the cap.

2. **Read the template's placeholders.** `getCollection` —
   `GET /workspaces/{workspaceId}/collections/{collectionId}`. `data.placeholders[]` gives each
   placeholder's `name` and `type` (`text` or `image`) plus its `defaultValueObject`. **The names
   you send in step 4 must match these exactly** — the placeholder set is defined by the template,
   not by you.

3. **Resolve any images first.** For an `image` placeholder you need a `mediaIndexId`, which is an
   image already in the workspace:
   - `listImageFolders` — `GET /workspaces/{workspaceId}/assets/images/folders`. Skip folders where
     `readOnly` is true; you cannot upload into them.
   - `listImagesInFolder` — `GET /workspaces/{workspaceId}/assets/images/folders/{folderId}/items`,
     read `data[].id`.
   - or `uploadImage` — `POST /workspaces/{workspaceId}/assets/images/folders/{folderId}/items`,
     `multipart/form-data` with the file field named exactly `image`. Returns `201` and
     `data.imageId`.

4. **Create the collection page.** `createCollectionPage` —
   `POST /workspaces/{workspaceId}/collections/{collectionId}/collection-pages`:

   ```json
   {
     "name": "Seattle — spring campaign",
     "urlSuffix": "seattle-spring",
     "placeholders": [
       {"name": "headline", "type": "text",
        "valueObject": {"textValue": "Landing pages for Seattle teams"}},
       {"name": "hero", "type": "image",
        "valueObject": {"mediaIndexId": 88213}}
     ]
   }
   ```

   Constraints: placeholder `name` max 255 characters, `textValue` max 65,000 characters,
   `mediaIndexId` must belong to the same workspace. `urlSuffix` is optional — omit it and Instapage
   generates one. Returns `201` with `id`, `status` (`not_published` on creation), `draftBaseUrl`,
   `draftUrlSuffix`, `publicBaseUrl`, `publicUrlSuffix`.

5. **Publish it.** `publishCollectionPage` —
   `POST .../collection-pages/{collectionPageId}/publication`. Returns **`202 Accepted`** — the page
   "will be available shortly", so it is not live at the moment of the response. Confirm by
   re-reading `listCollectionPages` — `GET .../collection-pages` — until `data[].status` is
   `published`. `400` means it was already published; `403` means the collection-page addition limit
   is exhausted.

6. **Understand the third status.** `published_has_changes` means the page is live but has unpublished
   edits — republish to push them.

## Taking one down

- `unpublishCollectionPage` — `DELETE .../collection-pages/{collectionPageId}/publication`. `202`
  on success, `409` if it is already unpublished.
- `deleteCollectionPage` — `DELETE .../collection-pages/{collectionPageId}`. `204` on success.
  Unpublish before deleting; there is no undo and no idempotency key, so record the ID you deleted
  before you send the request rather than after.
