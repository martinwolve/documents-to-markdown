# Contributing

Thanks for your interest in improving `documents-to-markdown`. This document explains how to propose changes and what to keep in mind.

## Ground rules

- **Never commit real tender or contract documents.** Use anonymized or self-made samples only. This applies to issues, pull requests, and the `examples/` folder.
- Keep the skill a single, self-contained `SKILL.md`. The detection code is embedded in the file on purpose, so the skill stays installable as one file.
- The skill reproduces document content verbatim and never translates it. Only the skill's own instructions are in English.

## Development setup

The only dependency for the detection pre-scan is PyMuPDF:

```
pip install -r requirements.txt
```

To try the detection code, copy the Python block from the "Detection script" section of `skills/documents-to-markdown/SKILL.md` into a file (for example `revision_detect.py`) and run it against a sample PDF:

```
python revision_detect.py sample.pdf
```

It should report, per page, the detected convention (COLOR / HIGHLIGHT / STRIKETHROUGH) and the affected passages.

## Testing a change

Before opening a pull request, verify on at least:

- a document with genuine strikethrough (struck text must be reported and omitted),
- a document that marks changes with color and highlighting (must report COLOR / HIGHLIGHT and zero strikethrough),
- a document with a watermark (must not produce false strikethrough).

Describe in your pull request which cases you checked and what the output was.

## Submitting changes

1. Fork the repository and create a branch for your change.
2. Make your change. If you change behavior, update `SKILL.md`, the `CHANGELOG.md`, and the `version` in the SKILL.md frontmatter.
3. Follow [semantic versioning](https://semver.org/): patch for fixes, minor for new capabilities, major for behavior changes.
4. Open a pull request using the template, describing the motivation and the cases you tested.

## Reporting issues

Use the issue templates. For bugs, include the detection output and, if possible, an anonymized sample that reproduces the problem.
