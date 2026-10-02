# Files and drive items

## Exact identity, intent and outcomes

Apply identity checks to read-only finding and each comparison source as well as
mutations: full filename, file/folder type, parent location, requested time and
relevant content. A first-ranked near-match is not exact. Disambiguate duplicate
names; a bounded search cannot prove uniqueness or global absence.

Retain source and destination drive IDs separately. The copy recipe below is
same-drive; if drives differ, verify supported cross-drive copy before acting
and use the destination drive in the body. Move uses `update_entity`
`/drives/{sourceDriveId}/items/{sourceId}` with
`{"parentReference":{"id":"{folderId}"}}`, not a `/move` action. Verify same-drive
identity for move; never implicitly copy/delete to simulate a cross-drive move.
See [updates](update-entity-work-iq.md).

Escape embedded apostrophes in OData names by doubling them, then URL-encode
the literal once. Retain opaque IDs verbatim. A concrete unresolved source can
use one supported in-scope refinement or exact read, not recursive enumeration,
an always-download rule or denial bypass. Confirm the exact effect before writes.
`202` means accepted/pending; ambiguous results are not permission to replay.
Upload-session creation is not uploaded bytes. Do not expose preauthenticated
upload URLs. Apply [recovery](troubleshooting.md).

## Known endpoint recipes

Call counts below describe an unambiguous, authorized happy path, not a hard limit.
Identity, required confirmation, supported paging and requested completeness take
precedence. Endpoint query restrictions and payload shapes remain binding.

| Request | Example | Contract |
| --- | --- | --- |
| Creating an upload session for an existing OneDrive file | "Create an upload session to replace my file; do not upload content" | `call_function` once with `/me/drive/root/search(q='{urlEncodedExactName}')?$select=id,name,parentReference,file&$top=10` to resolve the exact driveItem and retain `parentReference.driveId` plus item `id`, then `do_action` `/drives/{driveId}/items/{itemId}/createUploadSession` with `{}`. This is a validated deployed contract: skip `search_paths` and `get_schema`, do not add an `item` wrapper, and do not upload file content. |
| Creating a folder in personal OneDrive | "Create a OneDrive folder named Project files" | Call `create_entity` exactly once with parent URL `/me/drive/root/children` and `{"name":"{requestedName}","folder":{},"@microsoft.graph.conflictBehavior":"fail"}`. This is a known deployed contract. Do not call `get_schema`, `search_paths`, fetch the root, or resolve a drive-scoped parent first. |
| Copying a named OneDrive file to a named folder | "Copy Q3 plan.txt to Shared" | Use two `call_function` calls to `/me/drive/root/search(q='{urlEncodedExactName}')?$select=id,name,parentReference,file,folder&$top=10`, retain the source `parentReference.driveId`, then `do_action` `/drives/{driveId}/items/{sourceId}/copy` with `{"parentReference":{"driveId":"{driveId}","id":"{folderId}"}}`. Skip `search_paths`, `get_schema`, and verification fetches. |
| Renaming a OneDrive file | "Rename Draft.txt to Final.txt" | `call_function` once with `/me/drive/root/search(q='{urlEncodedExactName}')?$select=id,name,parentReference,file&$top=10` to resolve the exact driveItem and retain `parentReference.driveId` plus item `id`, then `update_entity` `/drives/{driveId}/items/{itemId}` with `{"name":"Final.txt"}`. Skip `search_paths` and `get_schema`; do not PATCH `/me/drive/items/{id}`. |
| Deleting a named OneDrive file | "Remove Q3 plan.txt from my drive" | `call_function` once with `/me/drive/root/search(q='{urlEncodedExactName}')?$select=id,name,parentReference,file&$top=10`, select the exact file-name match, and copy its `parentReference.driveId` and `id` verbatim without truncating, reconstructing, or normalizing either value. Then call `delete_entity` exactly once on `/drives/{driveId}/items/{itemId}`. Do not add `eTag` or `@odata.etag` to `$select`; only when the normal lookup response includes an eTag, pass that returned value as `If-Match`. If a newly created file is not indexed yet, use at most one bounded `/me/drive/root/children` fallback before the same drive-scoped delete. Do not use `/me/drive/items/{id}`, `search_paths`, or malformed-id retries. |

## Reaching a SharePoint or OneDrive file — the working sequence

Resolve the drive, then address items **by id**. This structured route supplies
exact metadata when supported and permitted; it does not guarantee success.

**Hop 1 — get a `driveId`** (one `fetch`, pick the row that matches your source):

| You have | Call | Keep |
|---|---|---|
| A SharePoint site id | `/sites/{siteId}/drive` | its `id` = `driveId` |
| The user's own OneDrive | `/me/drive` | its `id` = `driveId` |
| A browser URL | the hostname + site name from it → `/sites/{host}:/sites/{siteName}` → then `/sites/{siteId}/drive` | `siteId`, then `driveId` |

**Hop 2 — address items by id, never by name:**

| Goal | Call |
|---|---|
| Find a file by name | `call_function` with `/drives/{driveId}/root/search(q='{urlEncodedExactName}')` — URL-encode the name; this is a function call, not a `fetch` path |
| List the drive's top level | `/drives/{driveId}/root` → take its `id` → `/drives/{driveId}/items/{id}/children` |
| List a folder | `/drives/{driveId}/items/{folderId}/children` |
| Item metadata | `/drives/{driveId}/items/{itemId}` |
| File bytes | `fetch_blob` `/drives/{driveId}/items/{itemId}/content` |

Two invariants: **a name goes in `q=` of a search, never in a URL segment**, and **every id
comes from a tool result in this conversation** — never assembled, guessed, or all-zeros.

Within SharePoint drive/item addressing, these variants are **not allowlisted** and fail, so
skip them and use the sequence above: a pasted browser URL; a folder or library name as a path
segment (`/sites/{siteId}/Shared%20Documents/...`); a colon path (`/root:/Folder/File`);
`/root/children` (children hang off `/items/{id}`); and `/sites/{siteId}/drive/items/...`,
which is a lookup rather than a prefix — switch to `/drives/{driveId}/...`.

Prefer the drive-scoped `/drives/{driveId}/items/...` form for SharePoint content. The
`/me/drive/...` forms remain the documented OneDrive convention (see
`references/fetch-blob-work-iq.md` and `references/call-function-work-iq.md`); they are
unreliable for SharePoint-hosted items, not invalid everywhere.

**Anything else — discover, never guess.** For an unknown route, use `search_paths`
with required `query` string, e.g. `"SharePoint sites and drive items"`, and use
only returned paths. Explicit access/policy denial stops; it never authorizes
discovery of another route. Cache supported templates rather than probing variants.
