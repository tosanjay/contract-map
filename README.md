# contract-map

A Claude skill that turns any system into a **living input/output contract** you can see, edit, comment
on, and check against the code.

Ask Claude to map a pipeline, architecture, or service graph, and it builds an interactive diagram as a
private claude.ai artifact: a flow of component blocks, each with typed **input** and **output** edges.
Every edge names the data, its shape (key fields), and the **producer** (`file:function`) that makes it.
You edit any block or edge on the page — on any device — and leave comments per block or per edge
("where does this value come from? why only this case?"). Claude reads the same shared store and answers
in the thread, or fixes the contract. Each block carries a status chip — **unchecked / verified /
drift** — that Claude updates when it walks the real code against what the contract claims, so the gaps
between intention and implementation become visible.

It's not a static picture. The point is the round-trip: a diagram you keep in sync with the code and use
to find where the two have drifted apart.

## What it's good for

- Mapping an unfamiliar codebase's components and the data between them.
- A shared, editable spec of a pipeline or service graph that a team keeps current.
- Spotting gaps between intended and actual behaviour (a declared output the code no longer produces, an
  input nothing consumes, a renamed producer).
- Onboarding: a concrete, correctable picture instead of a stale wiki diagram.

## Requirements

- Claude with artifact support and the artifact `db` capability (claude.ai / Claude Code artifacts).
- The map is a **private** claude.ai artifact; share it from the page's Share menu if others need it.

## Install (as a plugin)

```
/plugin marketplace add <your-github-user>/contract-map
/plugin install contract-map@contract-map
```

Or use it as a local skill: copy `skills/contract-map/` into your `~/.claude/skills/` (user-level, works
in every project) or a project's `.claude/skills/`.

## Use

Just ask, for example:
- "Map the components of this service and the contracts between them."
- "Diagram this pipeline stage by stage — what are the inputs and outputs of each?"
- "Turn the architecture into an editable spec I can comment on and check against the code."

Claude reads the code, builds and seeds the map, and gives you the link. Then:
- Tap a block title or any edge to edit it.
- Tap the 💬 on any block or edge to start a comment thread; Claude answers there.
- Ask Claude to "verify the contract against the code" to fill in the verified/drift statuses.

## Layout

```
contract-map/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── skills/
│   └── contract-map/
│       ├── SKILL.md
│       ├── assets/contract-map.html      # the interactive page (published as the artifact)
│       └── references/db-schema.md        # the db collections + ArtifactData recipes
├── README.md
└── LICENSE
```

## License

MIT — see [LICENSE](LICENSE).
