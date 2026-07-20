---
name: documents-to-markdown
description: Converts documents that are available in the conversation — both documents in the project context and documents attached directly to the chat (PDF, Word, Excel, PowerPoint, txt, csv, rtf and other textual files) — into separate Markdown files, one per source document, preserving the full integral text and recognizable source and page marking via invisible HTML comments. Correctly recognizes and handles revision markings (strikethrough text, colored changes, highlighted fill-in fields) so that deleted text does not end up in the output. Use this skill when the user explicitly asks to convert documents to Markdown or MD, for example "convert these documents to MD", "convert the PDFs to Markdown", "turn the documents in the project context into MD files", "convert the attached documents to Markdown", or the Dutch equivalents such as "zet deze documenten om naar MD".
version: 1.1.0
license: Apache-2.0
---

# Documents to Markdown

Convert all available documents into separate Markdown files, one per source document, preserving the full integral textual content and recognizable source and page marking.

## Which documents to convert

Documents can reach this skill through two channels, and both are in scope:

- **Project context**: documents that are part of the project the conversation runs in.
- **Chat attachments**: documents attached directly to the conversation (uploaded to the chat).

Treat both as source documents. When the user does not narrow the set, convert every document from both channels. When the user points to a specific document, format, or subset (for example only the attached files, or only the PDFs), limit the conversion to that. If the exact same document appears in both channels, convert it only once.

## Purpose

Deliver, per source document, one separate `.md` file containing the complete textual content of that source document in full. Do not add information, do not summarize, do not rewrite, and do not interpret. Reproduce the text exactly as it appears in the source document. The only exception is the handling of revision markings, described under "Handling revision markings".

## Supported document types

Handle all common document types that may appear in the project context or as a chat attachment, including at least:

- PDF (`.pdf`)
- Word (`.docx`, `.doc`)
- Excel (`.xlsx`, `.xls`)
- PowerPoint (`.pptx`, `.ppt`)
- Plain text (`.txt`)
- CSV (`.csv`, `.tsv`)
- Rich Text Format (`.rtf`)
- Other textual documents present in the project context or attached to the conversation

## Workflow

1. Inventory all documents to convert, from both channels: the documents in the project context and the documents attached to the conversation (chat). Deduplicate documents that appear in both. If the user pointed to a specific document or subset, limit the inventory to that.
2. Determine, per document, the type and its page structure or layout:
   - PDF, Word, RTF: per page
   - Excel, CSV: per sheet (for CSV the whole file counts as one unit)
   - PowerPoint: per slide
   - Plain text: as a single whole
3. Determine whether the document may contain revision markings (see "Handling revision markings"). For PDF documents that may have been amended, run the pre-scan from "Detection script" to determine per page which convention applies (COLOR / HIGHLIGHT / STRIKETHROUGH).
4. Extract, per document, the full textual content:
   - If a page has no revision marking, plain text extraction suffices.
   - If a page does contain markings, transcribe that page via the image according to "Transcription via vision for revision markings".
5. Handle non-textual elements according to the rules under "Handling non-textual elements".
6. Add an invisible HTML comment as an anchor before each text block, so it remains recognizable from which document and from which page, slide, or sheet the text originates.
7. Deliver one separate `.md` file per source document according to the naming rules under "File naming".

## Source and page marking

Use HTML comments as invisible anchors above each text block. This keeps the reading flow of the original text fully intact. Use the following formats:

- PDF, Word, RTF:
  `<!-- source: filename.pdf | page: 7 -->`
- Excel:
  `<!-- source: filename.xlsx | sheet: Sheet_Name -->`
- PowerPoint:
  `<!-- source: filename.pptx | slide: 3 -->`
- CSV, TXT and files without pagination:
  `<!-- source: filename.csv -->`

Place the anchor directly above the text it applies to. Place a new anchor at every new page, sheet, or slide.

Do not add visible headers, titles, or separators to mark pages, sheets, or slides. The HTML comments are the only marking.

## Handling revision markings

Documents may show changes that are invisible or misleading in the extracted text layer. This is common in contracts and documents amended in response to a tender addendum (in Dutch: "Nota van Inlichtingen"). There are three conventions:

- **Colored text** (for example red): this is the new, final text. It stays in full. Do not reproduce the color. Place an anchor before the changed block:
  `<!-- revision: <color>, applied per addendum dated <date if stated> -->`
- **Highlight** (yellow): a field still to be filled in ([...] / <...>). Reproduce the content as it is and place an anchor before the field:
  `<!-- fill-in field: highlighted, to be completed -->`
- **Strikethrough text**: deleted text that no longer applies. Omit it from the final text. If you want to preserve traceability, render the struck text as `~~struck text~~` instead of omitting it.

Why this must be handled separately: the text layer of a PDF contains no reliable information about strikethrough (a strikethrough is usually a vector line separate from the text), and purely geometric detection of it is misled by watermarks, underlines, and table borders. Therefore pages with markings are transcribed visually.

## Detection script

Write the code below to a temporary file (for example `revision_detect.py`) and run it as a pre-scan: `python revision_detect.py <file-or-folder>`. It reports, per page, the detected convention and which passages contain markings, so that only changed pages need vision. Requires PyMuPDF (`pip install pymupdf`).

```python
#!/usr/bin/env python3
"""Revision preprocessor: detects color, yellow highlight, and safe strikethrough
in PDFs. Curves (watermark) and colored bars (insertion underline) are ignored
to avoid false strikethroughs."""
import sys, argparse, pathlib
import fitz  # PyMuPDF

BODY, WHITE = 0x000000, 0xFFFFFF

def hexcolor(c): return f"#{c:06X}"

def yellow_rects(page):
    R = []
    for dr in page.get_drawings():
        f = dr.get("fill")
        if f and f[0] > 0.8 and f[1] > 0.8 and f[2] < 0.5:
            R.append(fitz.Rect(dr["rect"]))
    return R

def _dark(color):
    if color is None: return True
    return all(c <= 0.35 for c in color[:3])

def strike_strokes(page):
    strokes = []
    for dr in page.get_drawings():
        lc, fc = dr.get("color"), dr.get("fill")
        for it in dr["items"]:
            if it[0] == "l":
                p1, p2 = it[1], it[2]
                if abs(p1.y - p2.y) < 1.5 and abs(p2.x - p1.x) > 4 and _dark(lc):
                    strokes.append((min(p1.x, p2.x), max(p1.x, p2.x), (p1.y + p2.y) / 2))
            elif it[0] == "re":
                r = fitz.Rect(it[1])
                if r.height < 2.5 and r.width > 4 and _dark(fc):
                    strokes.append((r.x0, r.x1, (r.y0 + r.y1) / 2))
    return strokes

def is_struck(box, strokes):
    x0, y0, x1, y1 = box
    lo, hi = y0 + (y1 - y0) * 0.25, y0 + (y1 - y0) * 0.75
    for sx0, sx1, sy in strokes:
        if lo <= sy <= hi:
            ov = min(x1, sx1) - max(x0, sx0)
            if ov > 0 and ov / (x1 - x0) >= 0.5:
                return True
    return False

def analyse(path):
    doc = fitz.open(str(path))
    out = []
    for i, page in enumerate(doc):
        info = {"page": i + 1, "color": [], "highlight": [], "strike": []}
        yellow, strokes = yellow_rects(page), strike_strokes(page)
        for b in page.get_text("dict")["blocks"]:
            for l in b.get("lines", []):
                for s in l["spans"]:
                    t = s["text"].strip()
                    if not t: continue
                    if s["color"] not in (BODY, WHITE):
                        info["color"].append((t, hexcolor(s["color"])))
                    if any(fitz.Rect(s["bbox"]).intersects(g) for g in yellow):
                        info["highlight"].append(t)
        if strokes:
            for w in page.get_text("words"):
                if is_struck((w[0], w[1], w[2], w[3]), strokes):
                    info["strike"].append(w[4])
        out.append(info)
    return out

def report(path, pp):
    name = path.name
    tc = sum(len(p["color"]) for p in pp)
    th = sum(len(p["highlight"]) for p in pp)
    ts = sum(len(p["strike"]) for p in pp)
    conv = [n for n, v in (("STRIKETHROUGH", ts), ("COLOR", tc), ("HIGHLIGHT", th)) if v]
    print(f"\n=== CHANGE MAP: {name} ===")
    print("Convention(s):", ", ".join(conv) or "no revision marking")
    print(f"(color={tc}, highlighted={th}, struck={ts})")
    for p in pp:
        if not (p["color"] or p["highlight"] or p["strike"]): continue
        print(f"\n<!-- source: {name} | page: {p['page']} -->")
        if p["strike"]:
            print("  [STRUCK -> remove]:", " ".join(p["strike"]))
        if p["color"]:
            cl = {c for _, c in p["color"]}
            print(f"  [CHANGED (color {', '.join(cl)}) -> keep]:",
                  " ".join(t for t, _ in p["color"])[:500])
        if p["highlight"]:
            print("  [FILL-IN FIELD (highlight) -> to be completed]:", " | ".join(p["highlight"]))

if __name__ == "__main__":
    ap = argparse.ArgumentParser(); ap.add_argument("target"); a = ap.parse_args()
    d = pathlib.Path(a.target)
    files = [d] if d.is_file() else [p for p in sorted(d.rglob("*")) if p.suffix.lower() == ".pdf"]
    for f in files:
        report(f, analyse(f))
```

## Transcription via vision for revision markings

Render each page that has revision markings to an image (170 dpi is sufficient) and reproduce the text from the image:

```python
import fitz
doc = fitz.open(pdf_path)
doc[page_number - 1].get_pixmap(dpi=170).save("page.png")
```

Reproduce the text in full according to all normal rules (anchors, tables, reading order), together with the revision rules above. If the image and the text layer conflict, the image prevails, because it shows color, highlight, and strikethrough as a reader sees them. Never let struck text disappear silently without this visual check.

## Handling non-textual elements

- Tables: convert to correctly formatted Markdown tables. Preserve column headers, row order, and cell content exactly as in the original.
- Images: describe based on visible textual information such as captions, alt text, titles, or legible text in the image. Use the format:
  `[Image: short rendering of visible text or caption]`
  Do not invent visual descriptions. If no textual information is visible, use:
  `[Image without caption]`
- Charts and diagrams: use only a placeholder; do not include any self-invented interpretation. Format:
  `[Chart: title or caption if present]`
  If there is no title or caption:
  `[Chart]`
- Footer, header, page numbers of the original: reproduce these once per page as they appear, without duplicating or translating them.
- Blank pages: mark with the anchor and the line `[Blank page]`.

## Strict content rules

- Reproduce all text in full. Do not change wording, spelling, capitalization, or punctuation.
- Do not add explanation, summary, conclusion, or context.
- Do not invent missing information.
- Do not translate anything, even if the document is multilingual. The instructions in this skill are in English, but the transcribed document content is always kept in its original language.
- Do not remove repetitions, footers, or boilerplate present in the original.
- Preserve the reading order of the original.
- Only permitted deviation: the handling of revision markings. Struck text is omitted (or marked as `~~...~~`); colored changes and highlighted fill-in fields stay, with an anchor before them. Do not reproduce the color formatting itself.

## File naming

Deliver one `.md` file per source document. Use this format for the file name:

`<original_filename_without_extension>.md`

Examples:

- Source document `Annual report 2025.pdf` becomes `Annual report 2025.md`
- Source document `Budget.xlsx` becomes `Budget.md`
- Source document `Minutes meeting Q1.docx` becomes `Minutes meeting Q1.md`

Preserve the original file name exactly as it is, including spaces, capitals, and special characters. Only replace the original extension with `.md`. Do not add a version suffix, date, or any other addition to the file name.

## Output

Deliver only the separate `.md` files, one per source document (whether it came from the project context or as a chat attachment), with the content and HTML comment anchors described above.

Do not give accompanying explanation, no summary of what you did, and no meta commentary outside the files.
