# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-07-20

### Added
- Initial release of the `documents-to-markdown` skill.
- Conversion of PDF, Word, Excel, PowerPoint, txt, csv, and rtf into one Markdown file per source document.
- Invisible HTML-comment anchors for source and page, sheet, or slide.
- Revision-aware handling: colored (changed) text kept as final text, highlighted fill-in fields kept, strikethrough text omitted or optionally marked as `~~...~~`.
- Embedded detection pre-scan (PyMuPDF) that reports color, highlight, and strikethrough per page, with safeguards against watermark curves and colored insertion underlines.
- Vision-based transcription for pages that contain revision markings.
