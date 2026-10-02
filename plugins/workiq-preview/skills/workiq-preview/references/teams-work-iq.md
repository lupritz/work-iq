# Teams (chats, channel messages, reactions, presence)

Use entity tools for exact messages, listings and mutations. Ordinary caller-owned
team context uses available `retrieve` with explicit `strategy: "grounding"` under
[retrieval policy](retrieve-work-iq.md); [ask](ask-work-iq.md) is intentional
delegation only, not a synthesis default or failed-lookup resolver.

Establish intent first. Search-like "reply messages last week" does not authorize
posting or chat creation; remain read-only and clarify. Confirm exact targets,
recipients/content and effects before mutations. An absent user is not approval.
Known endpoint recipes below preserve inherited deployed contracts, not new live
validation; use the actual connected schema for unfamiliar shapes.

## Chats, channels and typed identities

| Surface | Meaning | Path root |
| --- | --- | --- |
| Chat | 1:1, group or meeting chat; flat messages, no replies endpoint | `/me/chats`, `/chats/{chatId}/messages` |
| Channel | A channel inside a team; threaded replies | `/teams/{teamId}/channels/{channelId}/messages` |

Resolve ambiguous names by actual topic, participants and IDs, not by guessing
which surface a name implies. Channel replies use
`/teams/{teamId}/channels/{channelId}/messages/{messageId}/replies`; do not invent
a chat replies endpoint. A chat reply posts a new message to the same flat chat.

Directory user IDs, conversation-member IDs, chat/team/channel IDs and tenant IDs
are distinct. Never use a semantic citation as mutation identity, infer a tenant
from an email domain, or decode an opaque member ID to guess a directory ID.
Missing required identity data blocks the action.

## Finding exact targets

### Channel

1. Fetch `/me/joinedTeams?$select=id,displayName` and match the exact team.
2. Fetch `/teams/{teamId}/channels?$select=id,displayName` and match the exact channel.

Do not select a similar name. Omit `$top` on joinedTeams and on chat/channel
message lists; these deployed endpoints reject it. Exact lookups do not use
semantic retrieval or `ask`.

### Person: authorized 1:1 create-or-return

For an authorized send or read-state workflow whose approval also covers possible
chat creation, the oneOnOne create-or-return contract returns the existing chat
for the pair or creates it if needed. It is a mutation, not a read-only lookup.
Never use it merely to read/find old messages, or add chat creation silently to
an edit/reaction. Without creation authorization, resolve an existing chat via
`/me/chats?$expand=members`, verify the counterpart and follow supported paging;
if unresolved, report the searched scope or ask for selection.

1. Resolve `/me?$select=id` plus the directory counterpart in a batch.
   For supplied email/UPN use
   `/users/{urlEncodedUserPrincipalName}?$select=id,displayName,mail,userPrincipalName`.
   Otherwise use
   `/users?$filter=displayName%20eq%20%27{odataEscapedAndUrlEncodedExactDisplayName}%27&$select=id,displayName,mail,userPrincipalName&$top=10`.
2. Require exactly one user matching the supplied email/UPN or full display name.
   Disambiguate duplicate names; a page bound is not proof of uniqueness. Do not
   use `/me/people`: fuzzy contacts are not directory-user identities.
3. After required confirmation, call `create_entity` on `/chats` with exactly the
   two returned directory IDs. Require a nonempty returned chat ID and
   `chatType == "oneOnOne"`; do not create a group chat for a single recipient.

```json
{"chatType":"oneOnOne","members":[{"@odata.type":"#microsoft.graph.aadUserConversationMember","roles":["owner"],"user@odata.bind":"https://graph.microsoft.com/v1.0/users('{signedInUserId}')"},{"@odata.type":"#microsoft.graph.aadUserConversationMember","roles":["owner"],"user@odata.bind":"https://graph.microsoft.com/v1.0/users('{counterpartUserId}')"}]}
```

These body binding URLs are not WorkIQ entity URL parameters.

### Group chat topic and signed-in member

Batch `/me?$select=id` with
`/me/chats?$filter=topic%20eq%20%27{odataEscapedAndUrlEncodedExactTopic}%27&$expand=members&$top=50`.
Require the complete topic/participant match and disambiguate duplicates. Follow
supported `@odata.nextLink` for required coverage; never invent `$skip` or a cursor.

For hide/read/unread, use the expanded member whose `userId` equals the signed-in
user ID. Reuse returned members; otherwise fetch exactly `/chats/{chatId}/members`
with no query string, including no `$select`, `$expand` or `$top`.
`userId` and `tenantId` are returned by that bare read but are not selectable
`conversationMember` fields. Set `teamworkUserIdentity.id` to that member's
`userId`, not its opaque `id` (often `MCMj...`), and retain the same member's
returned `tenantId`. Missing mapping/tenant means stop, not schema guessing.

### Message, marker and supplied URLs

After exact chat/channel resolution, fetch:

```text
/chats/{chatId}/messages?$select=id,createdDateTime,body
/teams/{teamId}/channels/{channelId}/messages?$select=id,createdDateTime,body
```

Match the complete message text before editing/reacting; use the returned ID.
For an exact marker request, filter locally to that exact marker; do not add
`$top` or `$orderby` or use semantic indexing as a substitute. Do not fetch
replies unless requested. Continue supported paging for requested complete
coverage or disclose partial results.

For supplied exact message URLs, batch every supported exact entity path in
one `fetch`, then synthesize locally:
`/teams/{teamId}/channels/{channelId}/messages/{messageId}` or
`/chats/{chatId}/messages/{messageId}`. Do not swap surfaces, guess opaque IDs from
an ambiguous browser link, or search broader history. Preserve each successful
batch result and report unresolved targets under [recovery](troubleshooting.md).

## Listings and discovery

For "show my chats", the bounded happy path is `/me/chats?$expand=members`.
Answer from returned topic, chatType and members; no enrichment after success.
Do not select `email` or `userId` on conversationMember. A cap or unfollowed
nextLink means partial coverage, not complete enumeration or absence.

Channel-member listing resolves the team/channel then fetches exactly
`/teams/{teamId}/channels/{channelId}/members`. Do not add `$top` or select
`email`/`userId`; answer from returned displayName and identity data. Do not probe
query variants after a 400. Nominal three-call budgets yield to disambiguation,
confirmation and supported completeness, not to unsupported query options.

For chat API inventory use `search_paths` with `{"query":"/chats"}`; for channel
inventory use `{"query":"/teams/{team-id}/channels"}`. Report every returned
operation and disclose unconfirmed categories. Inspect saved capped output if
available; do not make another discovery/schema call just to fill missing categories.

For channel-message create properties, call `get_schema` on
`/teams/{teamId}/channels/{channelId}/messages` with `operationType: "create"`.
Do not substitute chat/update schemas. Lead with confirmed user-supplied fields
(body, attachments, mentions where supported), not system-generated/read-only
resource fields such as IDs, timestamps, sender, reactions, replies or history.
If writability annotations are insufficient, state that limitation. Discovery
does not authorize creating a message.

## Send, reply, edit and react

After exact resolution, preparation and required confirmation:

| Effect | Tool and exact path |
| --- | --- |
| Send chat message | `create_entity` `/chats/{chatId}/messages` |
| Post channel message | `create_entity` `/teams/{teamId}/channels/{channelId}/messages` |
| Reply in a channel thread | `create_entity` `/teams/{teamId}/channels/{channelId}/messages/{messageId}/replies` |
| Edit a chat message | `update_entity` `/chats/{chatId}/messages/{messageId}` |
| Edit a channel message | `update_entity` `/teams/{teamId}/channels/{channelId}/messages/{messageId}` |
| React | `do_action` on the exact chat/channel message path plus `/setReaction` |

Known send/reply/edit body: `{"body":{"contentType":"text","content":"..."}}`.
For edits confirm ownership and supported fields. Known reaction body:
`{"reactionType":"👍"}`; the deployed action expects literal Unicode, not `like`.
Do not run path/schema preflights for these known shapes. A forbidden edit or
reaction is not proof that more consent will fix it; stop on explicit denial.

## Hide, mark read or unread

"Remove/delete/hide this chat from my list" is a per-user `hideForUser` action,
not `delete_entity` or deletion for other participants. Use exact topic/person
resolution above and the authoritative signed-in member. For a topic lookup,
reuse the expanded members and `/me` result; no redundant member/profile read.

Known body for `/chats/{chatId}/hideForUser` and `/markChatReadForUser`:

```json
{"user":{"@odata.type":"#microsoft.graph.teamworkUserIdentity","id":"{signedInUserId}","tenantId":"{signedInMemberTenantId}","userIdentityType":"aadUser"}}
```

For mark-unread, fetch `/chats/{chatId}/messages?$select=createdDateTime` (no
`$top`) and use the first returned message's `createdDateTime` in
`lastMessageReadDateTime` beside `user`. Do not use chat `lastUpdatedDateTime`;
it is not a message timestamp. If no usable timestamp is returned, stop.

```json
{"user":{"@odata.type":"#microsoft.graph.teamworkUserIdentity","id":"{signedInUserId}","tenantId":"{signedInMemberTenantId}","userIdentityType":"aadUser"},"lastMessageReadDateTime":"{returnedCreatedDateTime}"}
```

No empty body, missing tenantId, member-ID substitution or unknown identity.
These known action shapes go direct after prerequisites/confirmation. A
contract-defined `204` supports success without an unnecessary verification read.
After null, timeout or ambiguous 500, do not replay. Use a supported state read
only if the state is exposed; otherwise report outcome unknown. Do not invent
a verification endpoint.

## Presence and sync

Read `/me/presence` or `/users/{id}/presence` with `fetch`. For preferred presence,
confirm requested status/duration then `do_action`
`/me/presence/setUserPreferredPresence` with
`{"availability":"Busy","activity":"Busy","expirationDuration":"PT1H"}`.
`/me/presence/setPresence` is the application-session variant needing sessionId,
not an alternative after a failed preferred-presence write. No ambiguous replay
or denial bypass; a supported current-state read need not prove causation.

For explicit channel delta use `call_function`
`/teams/{teamId}/channels/{channelId}/messages/delta`. Preserve checkpoints,
next/delta links and removals under [functions](call-function-work-iq.md).
No checkpoint means initial sync, not changes since an arbitrary date. Semantic
catch-up does not automatically mean delta.
