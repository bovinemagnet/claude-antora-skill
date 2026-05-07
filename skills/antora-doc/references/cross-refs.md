# Antora cross-references and includes

Antora's cross-reference syntax is the most common source of broken builds. This file is the reference; load it whenever you need to link or include something across page, module, component, or version boundaries.

## Cross-references (`xref:`)

| Target | Syntax |
|---|---|
| Same-page section | `<<section-id>>` or `<<section-id,Custom text>>` |
| Same module, another page | `xref:other-page.adoc[]` |
| Same module, anchor in another page | `xref:other-page.adoc#section-id[]` |
| Different module, same component | `xref:other-module:other-page.adoc[]` |
| Different component | `xref:other-component::module:other-page.adoc[]` (the `::` is intentional — empty version) |
| Different component, specific version | `xref:other-component:1.2@module:other-page.adoc[]` |
| Different component, version + ROOT module | `xref:other-component:1.2@other-page.adoc[]` (ROOT module name omitted) |

### Custom link text

Default link text is the target page's title. Override when the title alone wouldn't make sense in context:

```adoc
xref:install.adoc[Installation guide]
xref:config:auth.adoc#jwt-flow[the JWT flow section]
```

### Anchors

Section anchors are auto-generated from heading text (lowercased, hyphenated). Override with an explicit ID for stability:

```adoc
[#jwt-flow]
=== JWT authentication flow
```

Now `xref:auth.adoc#jwt-flow[]` is stable even if the heading text is reworded.

## Includes (`include::`)

Includes pull content from another file at build time. The path uses **resource macros** (`partial$`, `example$`, `image$`, `attachment$`) instead of relative filesystem paths.

| Resource | Lives in | Include syntax |
|---|---|---|
| Reusable AsciiDoc fragment | `modules/<m>/partials/snippet.adoc` | `include::partial$snippet.adoc[]` |
| Code sample | `modules/<m>/examples/sample.java` | `include::example$sample.java[]` |
| Cross-module partial | `modules/other/partials/snippet.adoc` | `include::other:partial$snippet.adoc[]` |
| Cross-component example | `modules/<m>/examples/sample.java` in another component | `include::other-component::example$sample.java[]` |

### Tagged regions

Include only a tagged section of a source file:

```java
// tag::config[]
public static final int PORT = 8080;
// end::config[]
```

```adoc
[source,java]
----
include::example$server.java[tag=config]
----
```

Use `tags=config;startup` to include multiple tags. Use `lines=10..20` for line ranges.

### Callouts inside source blocks

```adoc
[source,bash]
----
gradlew antora <1>
gradlew antora --to-dir=build/site <2>
----
<1> Default build, output to `build/antora-site/`.
<2> Override the output directory.
```

## Images (`image::`)

| Use | Syntax |
|---|---|
| Block image | `image::deploy-flow.svg[Deploy flow]` |
| Block image with caption | `.Sequence: deploy flow` line above the `image::` |
| Inline image | `image:icon.svg[]` (single colon) |
| Cross-module image | `image::other-module:diagram.svg[]` |

Images live in `modules/<m>/images/`. Always include alt text — it's a hard requirement for accessible builds and shows up in screen readers.

## Attachments (`link:`)

Downloadable files in `modules/<m>/attachments/`:

```adoc
link:{attachmentsdir}/spec.pdf[Download the spec (PDF)]
```

The `{attachmentsdir}` attribute is built-in.

## Page attributes vs global

- **Page-level**: define just under the title, before the first paragraph.
  ```adoc
  = My page

  :version: 2.4.0
  :sourcedir: ../../examples
  ```
- **Module-level**: shared across all pages in a module — `modules/<m>/_attributes.adoc`, included from each page.
- **Component-level**: in `antora.yml` under `asciidoc.attributes:`.
- **Global**: in `antora-playbook.yml` under `asciidoc.attributes:`.

When in doubt about whether an attribute exists, grep for it before defining a new one.

## Version-aware cross-references

In a multi-version component, the version in an xref defaults to the **same version as the source page** unless specified. For "always link to the latest" behaviour, omit the version with `::` (e.g., `xref:other-component::module:page.adoc[]`).

## Things that look like xrefs but aren't

- `link:https://example.com[]` — external URLs use `link:`, not `xref:`.
- `<<other-page>>` (without `xref:`) — only works for same-page anchors. Across pages you need `xref:`.
- Markdown-style `[text](url)` — Antora ignores these silently. Will look broken.
