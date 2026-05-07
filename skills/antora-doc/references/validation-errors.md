# Antora build errors — patterns and fixes

Common failure modes from the Antora build (`gradlew antora` or `npx antora <playbook>`) and how to resolve them. Load this file when an error message isn't immediately obvious.

## "target of xref not found"

```
target of xref not found: module:page.adoc
```

The xref target doesn't resolve. Check, in this order:

1. **Module name** — does `modules/<module>/` exist? Antora module names are lowercased and case-sensitive in xrefs.
2. **Page filename** — does `modules/<module>/pages/<page>.adoc` exist? Common typo: missing `.adoc`.
3. **Anchor** — if the xref has `#anchor`, does that anchor exist in the target page? Auto-generated anchors lowercase and hyphenate the heading text.
4. **Component** — if cross-component, the empty version `::` must be present: `xref:other-component::module:page.adoc[]`.
5. **Default module is `ROOT`** — `xref:install.adoc[]` from outside the ROOT module won't find a page in ROOT; use `xref:ROOT:install.adoc[]`.

## "include target not found"

```
include target not found: example$missing.java
```

Resource not found in the resource macro path:

| Macro | Looks in |
|---|---|
| `partial$x.adoc` | `modules/<m>/partials/x.adoc` |
| `example$x.java` | `modules/<m>/examples/x.java` |
| `image$x.svg` | `modules/<m>/images/x.svg` |
| `attachment$x.pdf` | `modules/<m>/attachments/x.pdf` |

If the file exists in another module, prefix with the module: `partial$snippet.adoc` → `other-module:partial$snippet.adoc`.

## "page in nav not found"

```
page in navigation file not found: my-page.adoc
```

`nav.adoc` references a page that doesn't exist. Either the page filename is wrong, or the page hasn't been created yet. Antora is strict about navigation entries — the build fails rather than rendering a broken link.

Fix: create the page, or correct the filename in `nav.adoc`.

## Tagged include — "tag not found"

```
tag 'config' not found in include
```

The `// tag::config[]` / `// end::config[]` markers are missing or misspelled in the source file. Tag markers are case-sensitive.

Also check: tags must be paired. An open `tag::` without a matching `end::` is a build failure.

## Heading hierarchy violation

```
section title out of sequence
```

You jumped from `==` to `====` without `===` in between. Antora doesn't accept skipped levels. Either insert the missing level or promote the lower section.

## "Unresolved attribute"

```
unresolved attribute: my-attr
```

`{my-attr}` is referenced but never defined. Check (in order):

1. Page-level attributes (just under the title)
2. Module `_attributes.adoc` (must be included from the page that uses it)
3. Component `antora.yml` → `asciidoc.attributes:`
4. Playbook `antora-playbook.yml` → `asciidoc.attributes:`

Define it at the lowest scope that covers all uses.

## "Component version not found"

```
component version not found: my-component
```

The xref points to a component that isn't published in the playbook's `content.sources`. Either the component name is wrong, or the playbook is missing a source pointing at the component's repo/branch.

## "Duplicate page"

```
duplicate page: module:my-page.adoc
```

The same page exists in two source branches that map to the same component+version. Usually means two branches both declared the same `version:` in `antora.yml`. Either change one branch's version or remove the duplicate.

## Mermaid / diagram includes

If the build complains about a missing rendered image but the source `.mmd` exists in `examples/`, the `.svg` hasn't been generated. Render manually:

```
mmdc -i modules/<m>/examples/diagram.mmd -o modules/<m>/images/diagram.svg
```

(The `mermaid-diagram-creator` agent handles this for new diagrams.)

## When the error is obscure

- **Run with verbose**: `gradlew antora --stacktrace` (Gradle) or `npx antora --stacktrace <playbook>` (Node), and/or set `runtime.log.level: debug` in the playbook.
- **Bisect**: comment out chunks of nav or includes to isolate which entry triggered the failure.
- **Check the cache**: stale Antora cache can mask fixed errors. `rm -rf .cache/antora` (or wherever the playbook's `runtime.cache_dir` points) and retry.

## After fixing

Always **re-run the full build** rather than trusting an incremental fix. Antora's resolver runs against the whole content set, so a fix in one place can unmask a problem elsewhere.
