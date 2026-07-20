<!--
Thanks for contributing to documents-to-markdown.
Please do not include real tender or contract documents anywhere in this PR.
Use anonymized or self-made samples only.
-->

## Motivation

<!-- What problem does this change solve? Link any related issue. -->

## Changes

<!-- Summary of what you changed. -->

## Cases tested

Confirm the detection pre-scan still behaves correctly. Describe the output you saw.

- [ ] A document with genuine strikethrough (struck text reported and omitted)
- [ ] A document that marks changes with color and highlighting (reports COLOR / HIGHLIGHT, zero strikethrough)
- [ ] A document with a watermark (no false strikethrough)

<!-- Paste the relevant detection output here. -->

## Versioning

If behavior changed, confirm you updated:

- [ ] `skills/documents-to-markdown/SKILL.md`
- [ ] `CHANGELOG.md`
- [ ] `version` in the SKILL.md frontmatter (patch = fix, minor = new capability, major = behavior change)
