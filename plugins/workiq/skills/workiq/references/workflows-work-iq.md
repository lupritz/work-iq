# WorkIQ workflow index

Start with [the dispatcher](../SKILL.md). If both plugins are installed, load
`workiq-preview` and follow its routing for overlapping requests instead.
Standalone public semantic synthesis/discovery is ask-first; exact operations
use the domain reference, not semantic preflight.
For library columns use [metadata](sharepoint-library-metadata.md), not content
search. Use [Teams](teams-work-iq.md) for exact people/topics, marker messages,
member identity, paging and confirmed actions.

## Setup and selected catalog

The bundled `.mcp.json` configures server `workiq` and its hosted endpoint; no
local package/runtime is needed for MCP calls. The host supplies authentication.
Never put tokens in prompts, plugin files or tool arguments. Obtain account
identity from the user/host, not local git/OS data. Resume after a denial only
once reported remediation is complete and authorization still applies.

Resolve exact tool names and schemas from that configured server, not a guessed
prefix or another server's suffix match. Missing catalog/tools must be disclosed.
Use `search_paths` for explicit operation discovery and `get_schema` for explicit
fields/body questions; action request schemas do not establish response fields.
Report what was confirmed, not inferred Graph capabilities.

## Intent and cross-domain work

Finding existing content, suggested wording, persisted drafts and sends differ.
An ambiguous reply/message/draft phrase stays read-only pending clarification.
Resolve exact entities in their correct stores, inspect every batch result,
prepare and confirm required mutations, then execute once and report evidence.
Preserve successful sources/citations and name unresolved targets; no silent
comparison substitutes or fabricated prior context. A lookup is not permission
to act, and an absent user is not approval.

Directory users and personal contacts have incompatible IDs. Resolve personal
contacts from `/me/contacts` itself. Directory-managed fields are not editable
personal-contact fields; actual diagnostics govern privileges, not assumed
administrator/consent fixes. A missing personal contact does not authorize creation.

| Structured people request | Known route |
| --- | --- |
| Signed-in profile / manager | `fetch` `/me` or `/me/manager` with supported needed fields |
| Directory org chart / direct reports | `fetch` `/users/{id}/directReports`; preserve the actual directory ID and supported paging |
| Personal contacts | `fetch` `/me/contacts` with supported exact displayName filtering; resolve from this store before any confirmed edit |
| Outlook categories | `fetch` `/me/outlook/masterCategories`; writes require supported schema, intent and confirmation, and stop on denial |

## Known endpoint recipes

Call counts below describe an unambiguous, authorized happy path, not a hard limit.
Identity, required confirmation, supported paging and requested completeness take
precedence. Endpoint query restrictions and payload shapes remain binding.

| Request | Example | Contract |
| --- | --- | --- |
| Rolling up exact Mail, Calendar, and Teams entity URLs | "Use these exact entities to summarize status and blockers" | Use one batched `fetch` containing every supplied entity URL, then synthesize locally. Do not use `ask`, tenant-wide search, path discovery, or additional source lookups. |
| Signed-in user's profile photo metadata | "Show my profile photo dimensions and content type" | `fetch` `/me?$select=id`, then `fetch` `/users/{id}/photo?$select=id,width,height`. Do not use the policy-denied `/me/photo` alias, request `/$value`, or put `@odata.mediaContentType` in `$select`; read the media content type annotation returned with the metadata. |
