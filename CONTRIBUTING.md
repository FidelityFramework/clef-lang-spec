# Contributing to this project

## Writing a specification

Writing a good spec is an art in itself. You must be very precise while using natural language, which by its nature is imprecise. Have a look at other parts of the Clef spec, or better at the [C# spec](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/readme), which was created with much more effort by many more people.

## Guidelines for editing the markdown sources

The source for the spec is the collection of markdown files in the spec folder.

#### Markdown flavor

We aim to use [CommonMark](https://spec.commonmark.org/0.31.2/) markdown, plus the [section links](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax#section-links) and [tables](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/organizing-information-with-tables#creating-a-table) of github-flavored markdown.

#### Headings

We use [ATX headings](https://spec.commonmark.org/0.31.2/#atx-headings) without closing # characters.

Keep source headings unnumbered. The website owns their presentation.

#### Links

Intra-spec links are made like this: `[§](inference-procedures.md#constraint-solving)`.

As such, the links work directly in the spec sources (like on github or in VS Code preview).

Check rendered links in the website preview when changing chapter paths.

#### Other markdown guidelines

All [fenced code blocks](https://spec.commonmark.org/0.31.2/#fenced-code-blocks) should carry one of the following info strings.

- `fsharp` for F# code samples
- `csharp` for C# code samples
- `fsgrammar` for F# grammar
- `fsother` for other code (like pseudo code in a few places)

Show _defined terms_ in italics.

For `inline code` (including e.g. file and type names) use code spans.

## Viewing the result

`main` is the canonical Clef specification branch. Create contribution branches
from `main`; there is no special `dev` publishing branch.

Review source changes in a Markdown viewer. The rendered specification is
published at [clef-lang.com/spec/draft/](https://clef-lang.com/spec/draft/) by the
sibling [clef-lang-site](https://forge.spkez.dev/FidelityFramework/clef-lang-site)
repository. Its Hugo module mounts this repository's `spec/` directory; its F#
deployment CLI refreshes the module from `main` and publishes to Cloudflare Pages.

For a local website preview, follow the site's development instructions.
Uncommitted spec edits require a local Hugo module replacement; refreshing the
module from `main` only includes committed, pushed changes. This repository no
longer generates a `gh-pages` branch or runs the inherited MkDocs deployment.
