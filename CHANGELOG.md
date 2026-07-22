# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.0] - 2026-07-22

### Added
- Pre-scan image detection: the PyMuPDF pre-scan now also produces an IMAGE MAP
  that flags substantial raster images (data-bearing figures, diagrams,
  screenshots, and scanned / image-only pages) while ignoring small logos,
  icons, and decorations via size and page-coverage thresholds.
- User warning when a PDF contains such substantial images, stating that the
  data inside those images is not reproduced by the text conversion and is lost
  in the Markdown, and advising to consult or use the original PDF for the
  affected pages.
- Optional traceability anchor before the image placeholder for flagged images
  (`<!-- image with possible data loss: consult the original PDF, page <n> -->`).

## [1.1.0] - 2026-07-20

### Changed
- Documents to convert may now come from either the project context or directly
  from chat attachments; both channels are treated as source documents. The
  inventory step deduplicates documents that appear in both.
- Broadened the skill description and trigger examples to cover attached
  documents (e.g. "convert the attached documents to Markdown").

## [1.0.0] - 2026-07-20

### Added
- Initial release of the `documents-to-markdown` skill.
- Conversion of PDF, Word, Excel, PowerPoint, txt, csv, and rtf into one Markdown file per source document.
- Invisible HTML-comment anchors for source and page, sheet, or slide.
- Revision-aware handling: colored (changed) text kept as final text, highlighted fill-in fields kept, strikethrough text omitted or optionally marked as `~~...~~`.
- Embedded detection pre-scan (PyMuPDF) that reports color, highlight, and strikethrough per page, with safeguards against watermark curves and colored insertion underlines.
- Vision-based transcription for pages that contain revision markings.
