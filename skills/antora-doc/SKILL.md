---
name: antora-doc
description: Use whenever the user asks to write, edit, restructure, validate, or build AsciiDoc documentation in an Antora project — including any work involving `.adoc` files, `nav.adoc`, `antora.yml`, `antora-playbook.yml`, the `src/docs/` tree, cross-references between modules (`xref:module:page.adoc[]`), partials/examples/images directories, or running the docs build (`gradlew antora` / `npx antora`). Trigger even when the user doesn't say "antora" explicitly — phrases like "add a page about X", "update the architecture docs", "fix the docs nav", "the docs build is broken", or "document this for the docs site" all qualify when an Antora project is present. Do NOT use for plain README/Markdown edits, Javadoc, or how-to docs that aren't part of an Antora component.
---

# Antora Documentation Skill

Pairs with the `asciidoc-antora-writer` subagent. The skill owns the **workflow** (when to engage, where files go, the validation loop, conventions). The agent owns **execution** (writing/editing `.adoc` files in parallel for batch work).

## When to engage

Engage when ALL of these hold:

- An Antora project is present. Detect via any of:
  - `antora-playbook.yml` (or `.yaml`) at the repo root
  - `src/docs/antora.yml` (or `*/antora.yml`)
  - A `modules/<name>/pages/*.adoc` tree
- The user wants to **author or restructure** documentation (not just read it).
- Or the user wants to **build/validate** the docs.

Don't engage for plain Markdown work, Javadoc, or one-off `.adoc` files outside an Antora component (those don't have nav, modules, or playbook concerns).

## Where files live

In this user's setup, **final documentation is rooted at `src/docs/`** in the standard Antora structure:

```
src/docs/
├── antora.yml                  # component descriptor (name, version, nav)
└── modules/
    ├── ROOT/                   # default module
    │   ├── pages/              # .adoc pages
    │   ├── partials/           # reusable fragments (include::partial$...)
    │   ├── examples/           # code samples + mermaid (.mmd) sources
    │   ├── images/             # generated/static images
    │   └── nav.adoc            # navigation tree for this module
    └── <other-module>/
        └── ...same shape
```

Always create new pages under `modules/<module>/pages/`. Always update the matching `nav.adoc` so the page is discoverable — an undiscovered page is a dead page.

## The validation loop

**Every edit cycle ends with a build.** This is non-negotiable in Antora work because cross-references, includes, and nav entries fail silently in the source but explode at build time.

```
gradlew antora
```

Either `gradlew antora` or `npx antora` is acceptable — pick whichever the project is set up for. If a `build.gradle` / `settings.gradle` references the Antora plugin, prefer `gradlew antora` so the build runs through the project's pinned versions; if there's only an `antora-playbook.yml` and a `package.json`, `npx antora <playbook>` is the right call.

Read the build output. If errors:

1. **Resolve them in source order** — earlier errors often cascade.
2. **Re-run `gradlew antora`** after each fix; don't batch fixes blindly.
3. **Common failure modes** are catalogued in `references/validation-errors.md` — load that when an error doesn't immediately make sense.

A docs change isn't "done" until `gradlew antora` runs clean.

## Cross-references

The most common source of broken builds. The full syntax cheatsheet is in `references/cross-refs.md` — load it when setting up links between modules or components, or when an `xref` fails.

Quick pointers:

- Same page: `<<section-id>>`
- Same module: `xref:other-page.adoc[]`
- Different module: `xref:other-module:page.adoc[]`
- Different component: `xref:other-component::module:page.adoc[]` (note the `::`)

If a link's text would be unclear from the page title alone, supply meaningful link text: `xref:install.adoc[Installation guide]`.

## Mermaid diagrams

**Externalise** — never inline mermaid in a `.adoc` page. Per project convention, mermaid sources live in `modules/<module>/examples/<name>.mmd` and the rendered `.svg` (or `.png`) goes in `modules/<module>/images/`.

```adoc
.Sequence: deploy flow
image::deploy-flow.svg[Deploy flow]
```

For new diagrams, prefer the `mermaid-diagram-creator` agent — it knows the externalise-to-`examples/` rule and validates with `mmdc`. For live edits to an existing `.mmd` (rename a node, change an arrow style, add a participant), the `mermaid-diagram` skill handles it inline without the subagent round-trip.

## Style

- **British spelling** — initialise, behaviour, organisation, colour, recognise. Catch Americanisms before commit.
- **Sentence-case headings** — `= Page title`, `== Section title`. Not `= Page Title`.
- **Concise paragraphs** that are scannable. Antora pages are read in browsers; long walls of text don't survive the medium.
- **Admonitions** (NOTE / TIP / IMPORTANT / WARNING / CAUTION) when they earn their place. A page with five WARNINGs has zero — readers tune out.
- **Source blocks** always specify the language: `[source,bash]`, `[source,java]`. Untagged blocks lose syntax highlighting.
- **Match existing style.** If the surrounding pages use a particular tone, attribute set, or admonition habit, follow it rather than inventing your own.

## When to spawn the agent vs handle inline

Use the `asciidoc-antora-writer` subagent when:

- **Multiple `.adoc` files** need writing or editing in one task — spawn N agents in parallel, one per page (or one per module). The agent works in isolation so it doesn't blow out main context.
- The user has handed over a **bulk migration** (e.g., "convert these 12 markdown files to Antora pages").
- A **large structural rewrite** (renaming a module, restructuring nav across many pages) — context isolation helps.

Handle inline (don't spawn the agent) when:

- A **single small edit** to one page.
- **Iterating live with the user** on phrasing or structure — the round-trip via subagent slows the conversation.
- **Investigating a build failure** — diagnostic work belongs in the main thread so the user can see what you're checking.

After the agent returns, **always** run `gradlew antora` yourself. The agent doesn't know to validate.

## Anti-patterns

These are easy to do by accident — don't:

- **Don't** write a new page without adding it to the matching `nav.adoc`.
- **Don't** skip heading levels (`=` then `===`). Antora's TOC and accessibility tooling rely on the hierarchy.
- **Don't** use absolute or relative file links between pages — always `xref:`.
- **Don't** inline mermaid source in a page — externalise to `examples/` and reference the rendered image.
- **Don't** declare a docs change "done" without a clean `gradlew antora`.
- **Don't** invent attributes — check `antora.yml`'s `asciidoc.attributes` and the playbook's global attributes first.

## Workflow

For most tasks:

1. **Locate the work** — which module, which existing page (if editing), or where the new page belongs.
2. **Check existing style** — read 1-2 nearby pages to match tone, admonition habits, attribute use.
3. **Write or edit** — inline for small jobs; spawn agents in parallel for batch work.
4. **Update nav** — `nav.adoc` for the affected module(s).
5. **Validate** — `gradlew antora`. Fix errors in source order.
6. **British-spelling pass** — quick grep for `behavior`, `color`, `organize`, `optimize`, `analyze`, `recognize`, `customize` (the `-ize`/`-or` Americanisms catch most cases). The exceptions are tool/command names like `flutter analyze` — leave those alone.
7. **Report** — file paths touched, nav changes, any cross-ref work, build status.

## References

- `references/cross-refs.md` — full xref/include syntax across same-page, module, component, and version boundaries.
- `references/validation-errors.md` — common `gradlew antora` error patterns and how to fix them.
