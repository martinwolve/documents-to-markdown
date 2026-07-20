# documents-to-markdown

A Claude [Agent Skill](https://agentskills.io) that converts documents from a project's context (PDF, Word, Excel, PowerPoint, txt, csv, rtf) into separate Markdown files, one per source document, preserving the full integral text and recognizable source and page marking via invisible HTML comments.

Its distinguishing feature is correct handling of **revision markings** in amended documents, such as contracts adjusted after a tender addendum (in Dutch: a *Nota van Inlichtingen*).

## Why this skill exists

Plain text extraction silently mishandles the ways a document shows changes:

- **Strikethrough** (deleted text) is usually a vector line drawn over the text, separate from the text stream. Text extraction keeps the struck words, so a contract ends up containing both the old and the new clause and contradicts itself.
- **Colored text** (e.g. red changes) and **highlighted fields** carry meaning that a plain `.md` conversion loses.

Purely geometric strikethrough detection is fragile: it is misled by watermarks (e.g. a diagonal DRAFT), by colored insertion underlines, and by table borders, producing false positives that would delete valid text. This skill therefore combines a conservative detection pre-scan with visual (image-based) transcription for pages that actually contain markings.

## What it does

- One `.md` file per source document, with the text reproduced in full and unchanged.
- Invisible HTML-comment anchors marking source and page/sheet/slide, e.g. `<!-- source: file.pdf | page: 7 -->`.
- Revision-aware output:
  - Colored (changed) text is kept as the final text, with a `<!-- revision: ... -->` anchor.
  - Highlighted fill-in fields are kept, with a `<!-- fill-in field: ... -->` anchor.
  - Struck text is omitted from the final text (or optionally marked as `~~...~~`).
- Tables converted to Markdown; images and charts represented by text placeholders.

## Requirements

- The skill's detection pre-scan uses [PyMuPDF](https://pymupdf.readthedocs.io/). Install it in the environment where the skill runs:

  ```
  pip install -r requirements.txt
  ```

- Visual transcription requires a Claude environment that can render and view page images (Claude.ai with the project context).

## Installation

### Claude.ai

1. Download `skills/documents-to-markdown/SKILL.md`.
2. In Claude, go to Settings and add or update a skill, selecting this `SKILL.md`.
3. Keep the file name `SKILL.md`. The detection code is embedded in the file, so no extra files are required.

### Claude Code

Place the `skills/documents-to-markdown/` folder where your Claude Code skills are discovered (personal or project skills), then invoke it by asking Claude to convert documents to Markdown.

## Usage

With documents in the project context, ask for example:

- "Convert these documents to MD"
- "Turn the PDFs in the project context into Markdown files"
- Dutch equivalent: "zet deze documenten om naar MD"

For an amended document, the skill first runs the detection pre-scan to see which pages carry color, highlight, or strikethrough, then transcribes those pages from their rendered image so the markings are handled correctly.

## Project structure

```
skills/
  documents-to-markdown/
    SKILL.md          # the skill, with embedded detection code
examples/             # optional, anonymized sample material only
requirements.txt      # PyMuPDF
CHANGELOG.md
LICENSE
```

## Versioning

This project uses [semantic versioning](https://semver.org/). The version in the SKILL.md frontmatter is kept in sync with the git tag and the `CHANGELOG.md`. Patch releases for fixes, minor for new capabilities, major for behavior changes.

## Contributing

Issues and pull requests are welcome. Please do not include real tender or contract documents in examples or issues; use anonymized or self-made samples.

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE).
