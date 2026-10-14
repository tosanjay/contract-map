# Contract Map — database schema & ArtifactData recipes

The page (`assets/contract-map.html`) renders entirely from the artifact's shared database and declares
`capabilities: {db: {}, user: {}}`. Read, seed, and verify it with the **ArtifactData** tool, passing the
artifact's `url` on every call. Three collections:

## `blocks` — one per component/stage

| field | type | meaning |
|---|---|---|
| (doc id) | string | short stable key, e.g. `deep_re`, `auth_service` |
| `name` | string | display name |
| `role` | string | one line: what this component does |
| `order` | number | position in the chain (ascending) |
| `lane` | string | optional grouping label (subsystem / layer / phase); blank = omitted |
| `status` | string | `unchecked` \| `verified` \| `drift` |
| `statusNote` | string | what you verified, or where it drifted |
| `statusDate` | string | `YYYY-MM-DD` of the last check (blank until checked) |

## `edges` — one per hand-off (output of `source`, input of `target`)

| field | type | meaning |
|---|---|---|
| (doc id) | string | any stable id, e.g. `e01` |
| `label` | string | the data's name |
| `type` | string | its shape / key fields |
| `producer` | string | `file:function` that produces it |
| `source` | string | a block id, or `ext:in` (external input) |
| `target` | string | a block id, or `ext:out` (final output) |
| `note` | string | optional |

A block's **inputs** are the edges whose `target` is that block; its **outputs** are the edges whose
`source` is that block. That's the whole model.

## `comments` — discussion threads, keyed to a block or edge

| field | type | meaning |
|---|---|---|
| (doc id) | string | any id; `add` mints one, or use `c_...` |
| `target` | string | the block id or edge id this comment is about |
| `author` | string | `user` (shown "You") or `claude` (shown "Claude") |
| `text` | string | the comment |
| `ts` | number | epoch milliseconds (ordering) |
| `resolved` | boolean | optional; open user comments show as "open Q" |

## Seeding (batch)

`action: "batch"`, up to 50 writes per call, no `if_version` when creating. Example:

```
ArtifactData action=batch url=<artifact url> writes=[
  {"op":"set","collection":"blocks","doc_id":"parse","data":{"name":"Parser","role":"Reads the request body into a typed record.","order":1,"lane":"ingest","status":"unchecked","statusNote":"","statusDate":""}},
  {"op":"set","collection":"blocks","doc_id":"auth","data":{"name":"Authorizer","role":"Checks the token and scopes.","order":2,"lane":"ingest","status":"unchecked","statusNote":"","statusDate":""}},
  {"op":"set","collection":"edges","doc_id":"e01","data":{"label":"raw request","type":"{headers, body bytes}","producer":"server.py:handle","source":"ext:in","target":"parse","note":""}},
  {"op":"set","collection":"edges","doc_id":"e02","data":{"label":"ParsedRequest","type":"{userId, action, payload}","producer":"parse.py:parse_request","source":"parse","target":"auth","note":""}}
]
```

Split blocks and edges into separate batches if you have more than ~50 rows total.

## Verifying (update a block's status)

Every read returns the doc's `version`; pin it on the write so you don't clobber a concurrent edit by
the user. Read first, then update:

```
ArtifactData action=get url=<url> collection=blocks doc_id=auth
# → shows {... , "version": 3}
ArtifactData action=update url=<url> collection=blocks doc_id=auth if_version=3 data={
  "status":"drift","statusNote":"producer renamed: auth.py:check_token is now verify_bearer; scopes field dropped","statusDate":"2026-10-14"}
```

Use `action: "list"` on `blocks` / `edges` (page with `query.cursor`) to pull the whole current contract
before a verification pass.

## Answering a comment

Read open comments, then write your reply (mint an id or use `set` with your own):

```
ArtifactData action=query url=<url> collection=comments query={"where":[["resolved","==",false]]}
ArtifactData action=set url=<url> collection=comments doc_id=c_reply1 data={
  "target":"auth","author":"claude","ts":<epoch ms>,"resolved":false,"text":"It comes from parse.py:parse_request — the token is on ParsedRequest.headers, not a separate field. Fixed the edge type."}
```

Get an epoch-ms timestamp with `date +%s` then append `000`, or any current-time source.
