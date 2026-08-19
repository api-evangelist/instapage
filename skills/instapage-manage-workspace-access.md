---
name: Manage Instapage workspace access
description: Provision a workspace, bulk-invite team members at the right access level, set the developer quota-inheritance flag, and re-role or remove people safely.
api: openapi/instapage-team-members-openapi.yml
operations: [listWorkspaces, createWorkspace, getWorkspace, renameWorkspace, listTeamMembers, inviteTeamMembers, updateTeamMembers, removeTeamMembers]
generated: '2026-08-13'
method: generated
source: https://devdocs.instapage.com/
---

# Manage Instapage workspace access

Workspaces are Instapage's tenancy boundary — pages, integrations, domains and assets are all scoped
to one. The team-member endpoints are bulk-first: they take a **bare JSON array** as the request
body, not an object with a collection key.

## Before you start

- `Authorization: Bearer <personal API token>`, base `https://api.instapage.com/v1`.
- Only owners and managers can invite, re-role or remove members. A token cannot grant more than its
  creator holds, and if the creator's role changes the token's reach changes with it, silently.
- Access levels are `viewer`, `editor`, `manager` and `owner`. `owner` is not assignable through
  these endpoints — attempting to modify the owner's role is an explicit error.
- Workspace and team-member counts are plan-limited (1 workspace and 10 members on Create, 5
  workspaces on the 50,000-visitor band). Hitting a limit returns `422` for workspaces and `403` for
  team members.

## Steps

1. **List what exists.** `listWorkspaces` — `GET /workspaces`, optionally `?name=` (case-insensitive).
   Names are unique per owner but you can be *invited* to someone else's workspace with the same
   name, so always check `data[].accessLevel` and `data[].ownerId` before acting, not just the name.

2. **Create the workspace.** `createWorkspace` — `POST /workspaces` with `{"name": "..."}`.
   `201` on success; `409` if the name is taken under the same owner; `422` if the plan's workspace
   limit is reached. **No idempotency key** — a timed-out create must be reconciled with
   `listWorkspaces` before you retry, or you will get a 409 or a duplicate-looking estate.

3. **Rename if needed.** `renameWorkspace` — `PATCH /workspaces/{workspaceId}` with
   `{"name": "..."}`. `409` if the new name is taken.

4. **See who has access.** `listTeamMembers` — `GET /workspaces/{workspaceId}/team-members`.
   **This endpoint does not paginate** — the whole roster comes back at once. `data[].fullName` is
   `null` while `data[].invitationStatus` is `pending`; Instapage withholds the real name until the
   invitation is accepted, so do not treat a null name as a data problem.

5. **Invite people.** `inviteTeamMembers` — `POST /workspaces/{workspaceId}/team-members` with an
   array:

   ```json
   [{"email": "a@example.com", "accessLevel": "editor",
     "inheritOwnerContextInPublicApi": false}]
   ```

   Returns `201`. `400` on a malformed email, `403` when the caller lacks permission or the plan's
   member limit is reached.

6. **Set the developer flag deliberately.** `inheritOwnerContextInPublicApi: true` makes that
   person's *public API* calls consume the **workspace owner's** daily API quota instead of their
   own. Set it for service accounts and integration engineers so their automation draws on the
   owner's larger plan allowance; leave it `false` for ordinary marketers, or one runaway script
   drains the owner's quota for everyone.

7. **Re-role.** `updateTeamMembers` — `PUT /workspaces/{workspaceId}/team-members` with an array of
   `{"email", "targetAccessLevel", "inheritOwnerContextInPublicApi"}`. Returns `201`. Every email in
   a single request must be unique — duplicates are a `400`. Modifying the owner's role is a `403`.

8. **Remove.** `removeTeamMembers` — `DELETE /workspaces/{workspaceId}/team-members` with an array of
   `{"email"}`. Returns `201`. Including the workspace owner's email is an error. If the
   authenticated user includes themselves and is not the owner, they leave the workspace — check for
   that before sending, because you will lose access mid-script.

## Deleting a workspace

`deleteWorkspace` — `DELETE /workspaces/{workspaceId}`. This takes the workspace's pages,
integrations, domains and assets with it and there is no documented confirmation step or undo.
Require explicit human approval naming the workspace and its page count first.
