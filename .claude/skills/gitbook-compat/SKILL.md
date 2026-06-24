---
name: gitbook-compat
description: Use when the user asks to audit docs for GitBook compatibility, fix broken links, validate SUMMARY.md, check .gitbook.yaml, or ensure their documentation site is ready for Git Sync publishing. Covers link resolution, URL encoding, structure validation, and silent-failure prevention.
tags:
  - gitbook
  - documentation
  - link-checker
---

# GitBook Compat Audit

A single-pass audit that checks a documentation repo against every GitBook rule that can silently fail.

## Invocation

Fire when the user mentions any of: "gitbook 兼容", "检查文档", "修复断链", "SUMMARY 校验", "gitbook 检查", "gitbook 适配".

## Steps

### 1. Validate `.gitbook.yaml`

**Completion**: the config passes all three checks, or each failure has a clear fix note.

Check:
- `root` points to a directory that exists, relative to repo root
- `structure.readme` resolves to an existing file from `root`
- `structure.summary` resolves to an existing file from `root`
- if `redirects` exist, every source path is a relative path (no leading `/`), and every target resolves

A broken `.gitbook.yaml` is the only failure that blocks all further checks — GitBook stops reading the repo if the config is invalid.

### 2. Audit SUMMARY.md

**Completion**: all links in SUMMARY.md resolve, all paths are URL-encoded, and the format is valid GitBook.

Check:
- **Header**: first line is exactly `# Summary`
- **Link format**: every link uses `* [title](path)` — no bare text beside links
- **Resolve**: decode each URL, verify the file exists under `root`
- **Encode**: every non-ASCII character in the URL portion must be percent-encoded. The test: `any(ord(c) > 127 for c in url)` must be false

**Rule**: GitBook auto-infers a TOC from directory structure if SUMMARY.md is absent, BUT a file not listed in SUMMARY.md will not appear in navigation — even if it's in the repo. This is the #1 cause of "my file isn't showing up."

### 3. Resolve cross-references

**Completion**: every internal link (`[text](relative/path)`) in every `.md` file resolves, or the specific broken link is reported.

Scope: all `.md` files under `root`. Skip external URLs (`http://`, `https://`) and anchors (`#fragment`).

For every link:
- construct the target from the source file's directory
- `unquote` any percent-encoding before filesystem lookup
- if the target doesn't exist, report: source file, line number (use grep), the bad URL, and the correct path if found nearby

**Rule**: GitBook uses standard markdown relative links. No special syntax is needed for cross-references. A `[text](../other/page.md)` link in a content page resolves exactly as the filesystem does.

### 4. Flag silent-failure traps

**Completion**: each detected trap is reported with severity and a fix.

Traps that cause silent failures (content exists but won't render or won't appear in nav):

| Trap | Detection | Severity |
|------|-----------|----------|
| File exists but not in SUMMARY.md | List all `.md` files under `root` not referenced in any SUMMARY.md link | **High** — won't appear in navigation |
| Multiple files map to same SUMMARY.md entry | Check for duplicate URLs in SUMMARY.md links | **Medium** — second file silently ignored |
| README.md created in GitBook UI | If `readme` file has GitBook metadata frontmatter (`description:`, `cover:`), it was created in the UI — Git Sync may overwrite it | **Medium** |
| `{% raw %}` blocks without closing tag | Grep for `{% raw %}` and count matching `{% endraw %}`; each pair should balance | **Low** — content after unclosed raw block is hidden |
| `.gitbook.yaml` outside root | If a `.gitbook.yaml` exists in a subdirectory of root but not IN root, that sub-tree uses a different implicit config | **Low** |

## GitBook rules (reference)

These are the canonical rules the audit steps enforce. Do not add rules beyond these without verification against the GitBook public docs.

### SUMMARY.md spec
- First line: `# Summary`
- Section groups: `## Group Name` (creates a labeled divider in the sidebar nav)
- Page entries: `* [Display Title](relative/path.md)` — use spaces (not tabs) for nesting, 4 spaces per level
- Paths: relative to the configured `root` in `.gitbook.yaml`
- If omitted, GitBook auto-generates a TOC from directory tree — but only for files that match its heuristic (it may skip deeply nested files)

### .gitbook.yaml spec
- All paths inside are relative to `root`
- `root`: directory containing the docs (relative to repo root; defaults to `./`)
- `structure.readme`: homepage markdown file (defaults to `README.md`)
- `structure.summary`: TOC file (defaults to `SUMMARY.md`)
- `redirects`: key-value map of `source: target` — source is a URL segment (no leading `/`), target is a file path

### File names and paths
- GitBook supports UTF-8 filenames at the filesystem level
- In SUMMARY.md links, non-ASCII characters MUST be percent-encoded
- Spaces in file names: the filesystem supports them, links must use `%20`
- Case-sensitive: `Page.md` ≠ `page.md` in Git Sync

### Content linking
- Standard markdown: `[text](relative/path.md)` — works everywhere
- GitBook-specific: `{% content-ref url="./" %} text {% endcontent-ref %}` — for rich page-link blocks (optional)
- Broken links within content pages are flagged by GitBook's change-request UI (a broken-link icon appears in the space header)

### Content features available in raw markdown
- Headings (`#` through `####`), lists, tables, code fences, images, standard links — all work
- Hints: `{% hint style="info|success|warning|danger" %}...{% endhint %}`
- Tabs: `{% tabs %} {% tab title="Name" %}...{% endtab %} {% endtabs %}`
- Code blocks with title: `{% code title="file.js" overflow="wrap" lineNumbers="true" %} ... {% endcode %}` (note: GitBook-annotated code blocks, not standard fenced blocks)

## Failure modes

- **Premature completion**: do not stop the audit after step 1. All four steps must complete before the audit counts as done. Each step prints its own count (scanned N files, found M issues).
- **Cross-reference false positives**: `{% content-ref %}` uses `url` not `href` — don't treat these as standard markdown links.
- **Encoding false positives**: a URL with `%20` is fine; a URL with a literal space is not. The test checks for literal non-ASCII chars or spaces in URL portions.
