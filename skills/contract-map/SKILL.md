---
name: contract-map
description: >-
  Build an interactive, editable component input/output contract for any system or codebase as a
  claude.ai artifact: a flow of component/stage blocks, each with typed input and output edges (the
  data's name, its type/key fields, and the producer as file:function), plus per-block and per-edge
  comment threads and a verified/drift status kept in sync with the code. Use this whenever the user
  wants to map, diagram, or specify a pipeline, architecture, data flow, service graph, or a system's
  components and the contracts between them — whether reverse-engineering existing code or specifying a
  new or in-progress system up front and checking the code against that spec as it gets built. Also use
  it when the user wants a living spec they can edit, comment on, and verify against the implementation,
  or asks things like "map the components", "diagram the pipeline", "what are the inputs and outputs of
  each stage", "spec the system before I build it", "turn the architecture into an editable spec",
  "where does this value come from", or "give me a shared contract to spot gaps between intent and
  implementation". Prefer this over a static diagram whenever the user will keep, edit, comment on, or
  verify the result.
---

# Contract Map

Make a system legible as a **contract**: each component declares what it consumes and what it produces,
so you (and the user) can see where intention and implementation diverge. The result is a live claude.ai
artifact the user edits on any device, comments on per block or per edge, and that you re-read and check
against the code over time.

Why an artifact and not a static picture: the value is the round-trip. The user corrects a block or asks
"where does this input come from?" on the page; you read the same shared store and answer or fix it. A
PNG can't do that.

## What you build

A vertical signal-chain of **blocks** (components/stages). Between them run **edges**; an edge is an
output of its source block and an input of its target block. Each edge carries three things, because a
contract needs all three to be checkable:
- **name** — the data itself (e.g. `AnalystReport`, `auth token`, `parsed rows`)
- **type** — its shape / key fields (e.g. `{userId, scopes, exp}`)
- **producer** — where the code makes it, as `file:function` (the anchor for drift-checking)

Blocks carry a **status** chip — `unchecked`, `verified`, or `drift` — plus a note. That is how "check
from time to time" is made visible: you set it when you walk the code against the contract.

The data lives in the artifact's shared database (the `db` capability), in three collections: `blocks`,
`edges`, and `comments`. See `references/db-schema.md` for the exact field shapes and ready-to-use
`ArtifactData` calls — read it before you seed or verify.

## Workflow

### 1. Scope the map — from the code, or as intended design

Two ways in, and the tool is valuable both directions:

- **Reverse-engineer (existing code).** Read the system and seed the map from what the code actually
  does — the real components, the data between them, and the `file:function` that produces each edge.
  Follow the real calls; a value produced in one place and consumed in another is exactly an edge worth
  drawing. Seed blocks `unchecked`; the verification pass confirms or flags them later.

- **Contract-first (new or in-progress project).** When there is little or no code yet, author the
  *intended* contract — the components you plan to build and the data they should hand off. Seed blocks
  `unchecked`, and leave `producer` blank or set it to the planned location. As the code gets written,
  the verification pass marks each block `verified` or `drift`, so the map becomes a running check of the
  implementation against intent. This is often the more valuable mode: it surfaces unbuilt or diverged
  pieces early, while they are cheap to fix.

Either way, aim for the altitude the user asked for — usually 5–20 blocks, one per genuine
component/stage/service, not per function. If the system is large, unfamiliar, or not yet built, ask the
user for the entry points, the component list, or the intended design, then fill in the edges. A map
seeded honestly (real producer refs where code exists, blank/planned where it doesn't) is what makes the
user's corrections and the later checks meaningful.

### 2. Publish the artifact

Copy the template to a working file so you can set its name, then publish it:

- Copy `assets/contract-map.html` to a scratch path (e.g. the session scratchpad).
- Optionally set the page name to the system: change `<title>Contract Map</title>` and the `<h1>` and
  `.sub` line to the system's name and one-line description. Keep it short.
- Publish with the `Artifact` tool: the scratch file, `capabilities: {"db": {}, "user": {}}`, and
  `icon: "flowchart"`. Give it a `title` matching the system.

The page renders an empty state first, then fills from the store — so seed the store next.

### 3. Seed the blocks and edges

Use the `ArtifactData` tool, `action: "batch"` (up to 50 writes per call), into the `blocks` and
`edges` collections. Give blocks `status: "unchecked"` to start — in reverse-engineer mode you have not
verified them yet; in contract-first mode they are intent, not yet built. Edge `source`/`target` are
block ids, or the sentinels `ext:in` (external input) and `ext:out` (final output). Exact field names and
example batches are in `references/db-schema.md`.

### 4. Hand it over

Tell the user the URL and that it's private to them (others need the Share menu). In one or two lines,
how it works: tap a block title or any edge to edit it; the 💬 on every block and edge opens a comment
thread; the status line shows open questions. Say plainly whether the seeded content is read from real
code (and may be slightly off — correct it) or is the intended design to be filled in as they build.

### 5. Verify against the code (the recurring job)

When the user asks (or proactively, when you've been in the code), do a verification pass:
- Read the current `blocks` and `edges` with `ArtifactData`.
- For each block, check the real code against its declared inputs and outputs: does the producer exist at
  that `file:function`? Does it still produce that shape? Is a declared input actually consumed? In
  contract-first mode, has it been built at all?
- Set each block's `status` to `verified` or `drift`, with a short `statusNote` saying what you checked
  or where it diverged (in contract-first mode, `drift` also covers "not yet implemented" — say so), and
  `statusDate` = today. Update via `ArtifactData` `action: "update"` (pin `if_version` — the schema
  reference shows how).
- Report the drifts to the user; those are the gaps between intent and implementation this whole thing
  exists to surface.

### 6. Answer comments

The user asks questions on blocks and edges. Read the `comments` collection (filter by `target` = the
block or edge id), answer by writing a comment with `author: "claude"` and a millisecond `ts`. Your reply
appears in that thread on the page. Keep answers grounded in the code, and when a comment reveals the
contract is wrong, fix the block/edge too, not just the reply.

### 7. Keep the contract current with decisions (avoid false drift)

The contract can go stale a second way, not just the code moving: a decision made in conversation (here in
the CLI, or in a design doc or a commit message) changes the *intended* contract but never reaches the map.
A later verification pass then reports "drift" that is really the spec being out of date, not a code
violation — a false alarm that erodes trust in the map. Two disciplines prevent it:

- **Write decisions into the map as they happen.** When a conversation changes what a component should
  consume or produce, or adds, removes, or retires a stage, update the `blocks`/`edges` in the same step —
  the way you'd update a design doc — and record *why* as a comment on that block or edge. The comment
  thread is the audit trail of what the contract says and why; don't let an agreed change live only in chat.
- **Reconcile intent before you flag code.** A verification pass is code against *current intent*, so first
  bring the contract up to date with any agreed-but-unapplied decisions (check the recent conversation,
  design docs, and commits since the block's `statusDate`). A mismatch that matches a decision means the
  contract was stale — update it and mark it `verified` against the new intent, don't report the code. Only
  a mismatch with no corresponding decision is real code drift.

## Notes

- This produces a **claude.ai Artifact** and needs the `db` + `user` capabilities (declared on publish).
  It is private by default; the user shares it from the page's Share menu.
- To update an existing map (new version of the page, or more blocks), publish the same artifact `url`
  and/or write more `blocks`/`edges` rows — don't make a second artifact unless the user wants one.
- Keep producer refs at `file:function`, not exact line numbers: lines drift, function names are stabler
  anchors for the recurring check.
- The map is a tool for finding gaps, not a trophy. A half-built map with honest `unchecked`/`drift`
  statuses is more useful than a tidy one that claims a verification that never happened.
