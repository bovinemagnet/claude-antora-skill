---
name: mermaid-diagram
description: Use when the user asks for a Mermaid diagram specifically — phrases like "mermaid", ".mmd", "mermaid flowchart", "mermaid sequence diagram", "render with mmdc", or "diagram for the docs site". Also use for live, in-conversation iteration on an existing `.mmd` file ("change the arrow to dotted", "rename that node", "add a participant called X"). For a fresh diagram from scratch — especially batches like "create six sequence diagrams for these flows" — prefer the `mermaid-diagram-creator` subagent (parallel + isolated context). For draw.io / drawio / `.drawio` requests, defer to the `drawio` skill.
---

# Mermaid Diagram Skill

Pairs with the `mermaid-diagram-creator` subagent. The agent owns **fresh creation and parallel batch work**. The skill owns **live iteration in the main thread** — quick edits to an existing `.mmd` where round-tripping via subagent would slow the conversation.

## Decide first: spawn the agent or handle inline

| Situation | Action |
|---|---|
| User wants a fresh diagram from a description | Spawn the `mermaid-diagram-creator` subagent. |
| User wants several diagrams at once | Spawn the agent in parallel — one per diagram. |
| User wants live edits to an existing `.mmd` ("dotted arrow", "rename node", "add a branch") | Handle inline. Subagent round-trips kill the iteration loop. |
| User wants a small one-off diagram with quick back-and-forth on structure | Handle inline. |
| User pasted an existing diagram and wants it tweaked | Handle inline. |

After **either** path: validate with `mmdc -i <file.mmd> -o /tmp/test.svg` (or rename to the target output). The skill is not "done" until the validation passes.

## File placement

- **In an Antora / AsciiDoc project**: `modules/<m>/examples/<name>.mmd`. The rendered image goes in `modules/<m>/images/<name>.svg` (or `.png`). The page references the rendered image via `image::<name>.svg[]`. Don't inline mermaid inside `.adoc`.
- **Other projects**: place `.mmd` files in a sibling `examples/` or `diagrams/` directory rather than scattered. Keep one diagram per file.
- **Filename convention**: kebab-case, descriptive — `auth-jwt-flow.mmd`, not `diagram1.mmd`.

## Inline syntax cheatsheet

Enough to handle live edits without loading a reference file. For the fuller catalogue (class, ER, gantt, journey, git-graph), load `references/syntax.md`.

### Flowchart

```
flowchart TD
  A[Start] --> B{Decision?}
  B -- yes --> C[Do thing]
  B -- no --> D[Other thing]
  C --> E([End])
```

- Direction: `TD` (top-down), `LR`, `BT`, `RL`.
- Node shapes: `A[rect]`, `A(rounded)`, `A([stadium])`, `A{diamond}`, `A((circle))`, `A>asymmetric]`, `A[/parallelogram/]`.
- Arrows: `-->` solid, `-.->` dotted, `==>` thick, `--text-->` labelled.

### Sequence

```
sequenceDiagram
  participant U as User
  participant A as API
  participant DB
  U->>A: POST /login
  A->>DB: lookup user
  DB-->>A: row
  A-->>U: JWT
  Note over U,A: Token expires in 1h
```

- `->>` solid arrow, `-->>` dashed reply, `-x` failed call, `-)` async.
- `activate` / `deactivate` for lifelines, or `+`/`-` shorthand on the arrow.
- `loop`, `alt`/`else`, `opt`, `par`, `rect rgb(...)` for grouping.

### State

```
stateDiagram-v2
  [*] --> Idle
  Idle --> Running: start
  Running --> Idle: stop
  Running --> Error: fail
  Error --> [*]
```

- `[*]` is the start/end pseudo-state. `state X { ... }` for nested states.

### Subgraphs (flowcharts)

```
flowchart LR
  subgraph Frontend
    A --> B
  end
  subgraph Backend
    C --> D
  end
  B --> C
```

## Style

- **British spelling** in labels — "behaviour", "initialise", "colour".
- **Keep it scannable** — one diagram should answer one question. Two smaller diagrams beat one tangled one.
- **Quote labels with special characters** — `B["Build & deploy"]` — bare labels with `&`, `:`, or `(` will trip the parser.
- **Use comments** (`%%`) for non-obvious choices, not for explaining what the diagram already shows.
- **Don't over-style** — colour the one node that matters, leave the rest at default. Mermaid's default theme is fine.

## Validation

Always validate after editing:

```bash
mmdc -i path/to/diagram.mmd -o /tmp/test.svg
```

Common failures and fixes are in `references/syntax.md`. The summary:

- **Parse error**: usually a special character in a label — wrap in quotes.
- **Cannot read property of undefined**: a node ID was used before declaration in some diagram types — declare participants/states up front.
- **mmdc not found**: tell the user; the agent normally handles install. Don't try to install it yourself.

For diagrams destined for Antora, render the `.svg` once validation passes and place it under the module's `images/`. Update the page's `image::` reference if the filename changed.

## Anti-patterns

- **Don't** inline mermaid inside `.adoc` pages — externalise.
- **Don't** silently skip validation. Mermaid's parser is forgiving in places and strict in others; visual eyeballing isn't enough.
- **Don't** use diagram-specific syntax in the wrong diagram type. `participant` only exists in sequence diagrams; `[*]` only in state.
- **Don't** spawn the subagent for a one-line edit — it's slower than just doing it.

## References

- `references/syntax.md` — full per-diagram-type syntax (class, ER, gantt, journey, git-graph) and validation troubleshooting.
