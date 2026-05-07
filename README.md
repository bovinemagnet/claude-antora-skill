# claude-antora-skill

Authoring assets for [Claude Code](https://claude.com/claude-code) — Anthropic's terminal-based coding agent — that turn it into a competent collaborator on [Antora](https://antora.org/) documentation projects.

This repository ships two **skills** and two **subagents**. Drop them into your Claude Code configuration tree and Claude will recognise Antora projects, follow project conventions, and offload bulk authoring work to specialised subagents that run in parallel without polluting the main conversation context.

- **Author**: Paul Snow
- **Version**: 0.0.1
- **Targets**: Claude Code (CLI, IDE extensions, desktop app) — anywhere skills and subagents are loaded from `~/.claude/` or `<project>/.claude/`.

## What you get

| Asset | Type | Owns | Pairs with |
|---|---|---|---|
| `antora-doc` | Skill | Antora workflow — when to engage, where files live, validation loop, conventions, anti-patterns | `asciidoc-antora-writer` |
| `mermaid-diagram` | Skill | Live, in-thread iteration on `.mmd` files (rename a node, change an arrow style, add a participant) | `mermaid-diagram-creator` |
| `asciidoc-antora-writer` | Subagent | Writing/editing `.adoc` pages, nav, partials, examples — runs in isolation, parallelisable for batch work | `antora-doc` skill |
| `mermaid-diagram-creator` | Subagent | Fresh diagram creation, mmdc validation, parallel batch work — produces ready-to-include `.mmd` sources and `image::` directives | `mermaid-diagram` skill |

The split is deliberate: **skills carry workflow knowledge** (loaded into the main conversation when their trigger phrasing matches), and **subagents carry execution** (spawned in isolated contexts so the main conversation stays clean during bulk work).

## Why use this

Claude Code is a general-purpose coding agent. Out of the box it has no opinion about Antora's quirks: cross-references that fail silently in source but explode at build time, modules that need their `nav.adoc` touched whenever a page is added, mermaid diagrams that should be externalised rather than inlined, and a build command that varies per project (`gradlew antora` vs `npx antora <playbook>`). These skills bake those conventions in so you don't have to re-explain them every conversation.

Concretely, after installing:

- "Add a page about JWT refresh to the integration module" → Claude finds the right module, drafts the page in AsciiDoc, updates `nav.adoc`, runs the build, and reports back.
- "Convert these twelve markdown files into Antora pages" → the `antora-doc` skill spawns twelve `asciidoc-antora-writer` subagents in parallel, then validates the result with one build.
- "Create a sequence diagram for the booking flow and drop it in the architecture page" → the `mermaid-diagram-creator` subagent writes the `.mmd`, runs `mmdc` to validate, renders an SVG into `images/`, and returns the `image::` directive ready to paste.
- "Change the dotted arrow in `auth-flow.mmd` to thick" → the `mermaid-diagram` skill handles it inline, no subagent round-trip.

## Prerequisites

- **Claude Code** installed and authenticated. See the [Claude Code quickstart](https://docs.claude.com/en/docs/claude-code/quickstart) for setup.
- **An Antora project** in your working directory. The skill auto-detects via the presence of `antora-playbook.yml` (or `.yaml`), `src/docs/antora.yml`, `*/antora.yml`, or a `modules/<name>/pages/*.adoc` tree. Without one, the skill stays out of the way.
- **Antora build tooling** for whichever invocation your project uses — either Gradle with the Antora plugin, or Node + `npx antora`.
- **mmdc** (the Mermaid CLI) on `PATH` if you intend to author or validate diagrams. Install via `npm install -g @mermaid-js/mermaid-cli`.

## Install

Pick **one** of the two locations depending on whether you want the skills available everywhere or scoped to a single project.

### User-global (recommended for solo use)

Available in every Claude Code session you start, regardless of working directory.

```bash
mkdir -p ~/.claude/skills ~/.claude/agents
cp -r skills/antora-doc ~/.claude/skills/
cp -r skills/mermaid-diagram ~/.claude/skills/
cp agents/asciidoc-antora-writer.md ~/.claude/agents/
cp agents/mermaid-diagram-creator.md ~/.claude/agents/
```

### Project-scoped (recommended for team-shared repos)

Lives inside the project, can be checked into version control, and applies only when Claude Code is invoked from that project.

```bash
cd /path/to/your/antora/project
mkdir -p .claude/skills .claude/agents
cp -r /path/to/claude-antora-skill/skills/antora-doc .claude/skills/
cp -r /path/to/claude-antora-skill/skills/mermaid-diagram .claude/skills/
cp /path/to/claude-antora-skill/agents/asciidoc-antora-writer.md .claude/agents/
cp /path/to/claude-antora-skill/agents/mermaid-diagram-creator.md .claude/agents/
```

Commit `.claude/skills/` and `.claude/agents/` so your teammates pick them up automatically.

## Verify the install

Start a Claude Code session in your Antora project and ask:

> What skills and agents are available?

You should see `antora-doc`, `mermaid-diagram`, `asciidoc-antora-writer`, and `mermaid-diagram-creator` in the listing. If any are missing, the most likely cause is a mistyped path during the copy or an unexpected directory layout — `SKILL.md` must sit directly inside `.claude/skills/<name>/`, and agent files must sit directly inside `.claude/agents/`.

A quick functional check:

> Add a placeholder page called "Test page" to the ROOT module.

Claude should locate `src/docs/modules/ROOT/pages/`, create the page, update `nav.adoc`, and run the project's Antora build. If it does that without you having to explain Antora's directory layout, the skill is loaded and triggering correctly.

## How triggering works

Skills and agents in Claude Code are matched by their **description field**, not their name. When you type a request, Claude scans the descriptions of every loaded skill and agent and decides which (if any) apply. That's why the descriptions in this repository enumerate natural phrasings — "add a page about X", "fix the docs nav", "the docs build is broken", "create a sequence diagram for Y" — rather than just the canonical name.

You don't need to mention "Antora" or "Mermaid" to trigger them. Phrases like *"document the new auth endpoint"* or *"draw the deploy flow"* are enough when the project context fits.

If you want to **force** invocation, ask explicitly: *"use the antora-doc skill to ..."* or *"spawn the asciidoc-antora-writer subagent to ..."*.

## When subagents get spawned

The skills follow a deliberate split between inline work and subagent dispatch:

**`antora-doc` spawns `asciidoc-antora-writer` when:**

- Multiple `.adoc` files need writing or editing in one task — one subagent per page (or per module), in parallel.
- A bulk migration is involved (e.g., "convert these markdown files to Antora pages").
- A large structural rewrite is requested (renaming a module, restructuring nav across many pages).

**`mermaid-diagram` spawns `mermaid-diagram-creator` when:**

- A fresh diagram is requested from a description.
- Several diagrams are requested at once, in parallel.

**Both skills handle work inline (no subagent) when:**

- It's a single small edit to one file.
- You're iterating live with the assistant on phrasing or structure — subagent round-trips slow that loop down.
- It's a diagnostic task (investigating a build failure, for example) — diagnostic work belongs in the main thread so you can see what's being checked.

## Conventions the skills enforce

Worth knowing up front, because Claude will follow these without being told:

- **British spelling** in prose and labels (`behaviour`, `colour`, `organise`, `recognise`, `initialise`).
- **Sentence-case headings** (`= Page title`, not `= Page Title`).
- **Mermaid externalised** — never inlined in `.adoc`. Sources live in `modules/<m>/examples/<name>.mmd`; rendered images live in `modules/<m>/images/<name>.svg`.
- **`xref:` for all internal links** — never relative or absolute file paths between pages.
- **Every nav-discoverable page in `nav.adoc`** — an undiscovered page is a dead page.
- **Build must run clean before declaring done** — every documentation change ends with `gradlew antora` or `npx antora <playbook>` (whichever your project uses) running without errors.

If your project conventions diverge from any of these, override them by stating the preference in the conversation, in a project `AGENTS.md`, or in a `.claude/instructions` file. User instructions outrank skill conventions.

## Build commands

The skill accepts either of:

```bash
gradlew antora                  # if your project has the Antora Gradle plugin
npx antora antora-playbook.yml  # if your project is Node-only
```

It will prefer whichever your project is set up for; if both are present, `gradlew antora` is preferred so the build runs through the project's pinned versions.

## Updating

This repository is the source of truth. To pick up changes:

```bash
cd /path/to/claude-antora-skill
git pull
# Then re-run the install commands from above to overwrite the deployed copies.
```

If you've customised your local copies, diff before overwriting — the deploy is a copy, not a symlink.

## Repository layout

```
claude-antora-skill/
├── skills/
│   ├── antora-doc/
│   │   ├── SKILL.md                          # Antora workflow, conventions, anti-patterns
│   │   └── references/
│   │       ├── cross-refs.md                 # xref/include syntax cheatsheet (loaded on demand)
│   │       └── validation-errors.md          # build error patterns and fixes (loaded on demand)
│   └── mermaid-diagram/
│       ├── SKILL.md                          # mermaid workflow, inline syntax cheatsheet
│       └── references/
│           └── syntax.md                     # extended diagram-type reference (loaded on demand)
└── agents/
    ├── asciidoc-antora-writer.md             # AsciiDoc/Antora writer subagent
    └── mermaid-diagram-creator.md            # Mermaid diagram creator subagent
```

The `references/*.md` files inside each skill are loaded **on demand** by the skill body when the situation calls for them, not eagerly. This keeps the always-loaded surface area small.

## Further reading

- [Claude Code documentation](https://docs.claude.com/en/docs/claude-code) — installation, configuration, and full feature reference.
- [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills) — how skills are discovered, loaded, and triggered.
- [Claude Code subagents](https://docs.claude.com/en/docs/claude-code/sub-agents) — how subagents are dispatched and isolated.
- [Antora documentation](https://docs.antora.org/) — the documentation pipeline these assets target.
- [AsciiDoc syntax quick reference](https://docs.asciidoctor.org/asciidoc/latest/syntax-quick-reference/) — the markup the writer agent produces.
- [Mermaid documentation](https://mermaid.js.org/) — the diagram syntax the diagram skill and agent use.
- [Anthropic](https://www.anthropic.com/) — the company behind Claude and Claude Code.

## Licence

Copyright 2026 Paul Snow.

Released under the [Apache License 2.0](LICENSE). You're free to use, modify, and redistribute these assets — including in commercial and proprietary projects — provided you preserve the copyright notice and licence text. See `LICENSE` for the full terms.
