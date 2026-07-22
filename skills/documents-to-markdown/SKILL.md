---
name: documents-to-markdown
description: Converts documents available in the conversation — both in the project context and attached directly to the chat (PDF, Word, Excel, PowerPoint, txt, csv, rtf and other textual files) — into separate Markdown files, one per source document, preserving the full integral text and recognizable source and page marking via invisible HTML comments. Correctly handles revision markings (strikethrough, colored changes, highlighted fill-in fields) so that deleted text does not end up in the output, and extracts substantial images (scans, diagrams, screenshots) from a PDF as separate image files named after the source document, which the Markdown then references, so the text is converted while no image data is lost. Use this skill when the user explicitly asks to convert documents to Markdown or MD, for example "convert these documents to MD", "convert the PDFs to Markdown", "convert the attached documents to Markdown", or the Dutch equivalent "zet deze documenten om naar MD".
version: 1.3.1
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
3. Determine whether the document may contain revision markings (see "Handling revision markings"). For PDF documents that may have been amended, run the pre-scan from "Detection script" to determine per page which convention applies (COLOR / HIGHLIGHT / STRIKETHROUGH). The same pre-scan also produces the IMAGE MAP.
4. For every PDF, use the pre-scan's IMAGE MAP to detect substantial images that may contain relevant data (see "Extracting images that may contain relevant data"). Extract every flagged image to a separate image file named after the source document; these files are delivered alongside the `.md` file and referenced from it, so the text is converted to Markdown while no image data is lost.
5. Extract, per document, the full textual content:
   - If a page has no revision marking, plain text extraction suffices.
   - If a page does contain markings, transcribe that page via the image according to "Transcription via vision for revision markings".
6. Handle non-textual elements according to the rules under "Handling non-textual elements".
7. Add an invisible HTML comment as an anchor before each text block, so it remains recognizable from which document and from which page, slide, or sheet the text originates.
8. Deliver one separate `.md` file per source document according to the naming rules under "File naming", plus every image file extracted in step 4, delivered alongside the `.md` file and referenced from within it.

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

Write the code below to a temporary file (for example `revision_detect.py`) and run it as a pre-scan: `python revision_detect.py <file-or-folder>`. It reports two things per page: (1) the detected revision convention and which passages contain markings, so that only changed pages need vision; and (2) an IMAGE MAP flagging substantial raster images whose data would be lost on plain text conversion (see "Extracting images that may contain relevant data"). Add `--emit-images <dir>` to also extract every flagged image into `<dir>`, named after the source document; the IMAGE MAP then reports the saved file name per image so it can be referenced from the Markdown: `python revision_detect.py <file-or-folder> --emit-images <output-dir>`. Requires PyMuPDF (`pip install pymupdf`).

```python
#!/usr/bin/env python3
"""Revision preprocessor: detects color, yellow highlight, and safe strikethrough
in PDFs, plus substantial raster images that may hold data lost on conversion.
Curves (watermark) and colored bars (insertion underline) are ignored to avoid
false strikethroughs. With --emit-images it also extracts every flagged image to
a separate file named after the source document, for referencing from Markdown."""
import argparse, pathlib
import fitz  # PyMuPDF

BODY, WHITE = 0x000000, 0xFFFFFF

# Image detection thresholds. Tuned to flag data-bearing images (scans, diagrams,
# screenshots, photos) while ignoring small logos and decorations.
PAGE_COVER_MIN = 0.08   # image covers >= 8% of the page area, or
MIN_SIDE_PT    = 140    # is rendered >= 140 pt (~4.9 cm) on both sides,
MIN_PIXELS     = 100    # and is >= 100 px on its shorter side (enough to hold data).
FULLPAGE_COVER = 0.60   # >= 60% coverage counts as a (near) full-page image / scan.
SCAN_TEXT_MAX  = 50     # a full-page image plus < 50 chars of text = likely a scan.

def hexcolor(c): return f"#{c:06X}"

def substantial_images(page):
    """Return images likely to carry data (not small logos/decorations).
    Includes each image's xref and bbox so it can be extracted afterwards."""
    found = []
    page_area = abs(page.rect.width * page.rect.height) or 1.0
    for info in page.get_image_info(xrefs=True):
        r = fitz.Rect(info["bbox"])
        if r.is_empty or r.is_infinite:
            continue
        cover = abs(r.get_area()) / page_area
        pw, ph = info.get("width", 0), info.get("height", 0)
        short_px = min(pw, ph)
        if short_px < MIN_PIXELS:
            continue
        big_by_cover = cover >= PAGE_COVER_MIN
        big_by_size = min(r.width, r.height) >= MIN_SIDE_PT
        if big_by_cover or big_by_size:
            found.append({"cover": round(cover, 2), "w_pt": round(r.width),
                          "h_pt": round(r.height), "px": (pw, ph),
                          "fullpage": cover >= FULLPAGE_COVER,
                          "xref": info.get("xref", 0), "bbox": tuple(r),
                          "saved": None})
    return found

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

def analyse(doc):
    out = []
    for i, page in enumerate(doc):
        info = {"page": i + 1, "color": [], "highlight": [], "strike": [],
                "images": substantial_images(page),
                "text_len": len(page.get_text("text").strip())}
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
        info["scanned"] = (info["text_len"] < SCAN_TEXT_MAX
                           and any(im["fullpage"] for im in info["images"]))
        out.append(info)
    return out

def save_one(doc, page, im, out_dir, base):
    """Write a single flagged image to out_dir/base.<ext>. Prefer the original
    embedded bytes (lossless, native resolution); fall back to rendering the
    image's page region if the embedded object cannot be extracted. Returns the
    saved file name."""
    xref = im.get("xref", 0)
    if xref:
        try:
            ext_img = doc.extract_image(xref)
        except Exception:
            ext_img = None
        if ext_img and ext_img.get("image"):
            ext = ext_img.get("ext") or "png"
            path = out_dir / f"{base}.{ext}"
            path.write_bytes(ext_img["image"])
            return path.name
    # Fallback: render just the image's region on the page at high resolution.
    pix = page.get_pixmap(dpi=200, clip=fitz.Rect(im["bbox"]))
    path = out_dir / f"{base}.png"
    pix.save(str(path))
    return path.name

def emit_images(doc, pp, out_dir, stem):
    """Extract every flagged substantial image to out_dir, named after the
    source document as <stem>_p<page>_img<n>.<ext>. Records the saved file name
    on each image so the report can print it for referencing from Markdown."""
    out = pathlib.Path(out_dir)
    out.mkdir(parents=True, exist_ok=True)
    for p in pp:
        for idx, im in enumerate(p["images"], 1):
            im["saved"] = save_one(doc, doc[p["page"] - 1], im,
                                    out, f"{stem}_p{p['page']}_img{idx}")
    return pp

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

    img_pages = [p for p in pp if p["images"]]
    scan_pages = [p["page"] for p in pp if p["scanned"]]
    ti = sum(len(p["images"]) for p in img_pages)
    print(f"\n=== IMAGE MAP: {name} ===")
    if not img_pages:
        print("No substantial images (only small logos/decorations, if any).")
        return
    emitted = any(im["saved"] for p in img_pages for im in p["images"])
    print(f"Substantial images: {ti} on {len(img_pages)} page(s). "
          f"Likely scanned/image-only pages: {scan_pages or 'none'}.")
    if emitted:
        print("These images are extracted to separate files (see 'saved as' below). "
              "Reference each one from the Markdown with a Markdown image link at the "
              "matching spot, so the in-image data (text in scans, figures, tables as "
              "pictures) stays available alongside the converted text.")
    else:
        print("Re-run with --emit-images <dir> to extract these images to separate "
              "files (data inside them is NOT reproduced by text extraction).")
    for p in img_pages:
        tags = []
        for im in p["images"]:
            kind = "full-page/scan" if im["fullpage"] else "image"
            saved = f" -> saved as {im['saved']}" if im["saved"] else ""
            tags.append(f"{kind} {im['w_pt']}x{im['h_pt']}pt "
                        f"({int(im['cover']*100)}% of page){saved}")
        note = " [image-only page: essentially all content is in the image]" if p["scanned"] else ""
        print(f"\n<!-- source: {name} | page: {p['page']} -->{note}")
        print("  [IMAGE(S) -> extract & reference]:", "; ".join(tags))

if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("target")
    ap.add_argument("--emit-images", metavar="DIR",
                    help="also extract flagged substantial images into DIR, "
                         "named after the source document")
    a = ap.parse_args()
    d = pathlib.Path(a.target)
    files = [d] if d.is_file() else [p for p in sorted(d.rglob("*")) if p.suffix.lower() == ".pdf"]
    for f in files:
        doc = fitz.open(str(f))
        pp = analyse(doc)
        if a.emit_images:
            emit_images(doc, pp, a.emit_images, f.stem)
        report(f, pp)
```

## Transcription via vision for revision markings

Render each page that has revision markings to an image (170 dpi is sufficient) and reproduce the text from the image:

```python
import fitz
doc = fitz.open(pdf_path)
doc[page_number - 1].get_pixmap(dpi=170).save("page.png")
```

Reproduce the text in full according to all normal rules (anchors, tables, reading order), together with the revision rules above. If the image and the text layer conflict, the image prevails, because it shows color, highlight, and strikethrough as a reader sees them. Never let struck text disappear silently without this visual check.

## Extracting images that may contain relevant data

Text extraction reproduces the text layer of a PDF, but it cannot reproduce data that lives *inside* an image: text in a scanned page, numbers in a chart or diagram, a table captured as a screenshot, or a photographed document. If that data were dropped, it would be silently lost on conversion. Instead of warning about this loss, extract those images to separate files and reference them from the Markdown, so the text is converted while the image data stays available alongside it.

The pre-scan's IMAGE MAP flags which images to extract. It reports, per page, the substantial raster images, distinguishing:

- **Data-bearing images**: images that cover a meaningful part of the page (from `PAGE_COVER_MIN`, default ≥ 8% of the page area) or are rendered large (from `MIN_SIDE_PT`, default ≥ ~140 pt / ~4.9 cm on both sides), and are at least `MIN_PIXELS` px on the shorter side. These are photos, figures, diagrams, charts, and screenshots that plausibly contain information.
- **Scanned / image-only pages**: a near full-page image (`FULLPAGE_COVER`, default ≥ 60% coverage) on a page whose extractable text is nearly empty (`SCAN_TEXT_MAX`, default < 50 characters). Here essentially all content is in the image and text extraction returns little or nothing.

Small logos, icons, and decorations fall below these thresholds and are deliberately ignored, so they are not extracted.

When the IMAGE MAP reports one or more substantial images for a document:

1. Run the pre-scan with `--emit-images <dir>`, pointing `<dir>` at the same location where the `.md` files are delivered, so each flagged image is written to a separate file next to the Markdown. The images are named after the source document, `<source_stem>_p<page>_img<n>.<ext>` (see "File naming"), and the IMAGE MAP reports the saved file name per image.
2. In the `.md` file, at the spot where each extracted image appears in the reading order, insert a Markdown image link to the saved file, preceded by a traceability anchor (see "Handling non-textual elements"). This keeps the information intact: the converted text and a reference to the original image sit together at the right place.

The extracted image files are delivered alongside the `.md` file. Do not describe or invent the contents of the images; the reference to the extracted file is what preserves the data. If no substantial images are found, no image files are produced and no reference is added.

## Handling non-textual elements

- Tables: convert to correctly formatted Markdown tables. Preserve column headers, row order, and cell content exactly as in the original.
- Images:
  - If the pre-scan's IMAGE MAP flagged this image as substantial (data-bearing or a scanned/image-only page), it is extracted to a separate file (see "Extracting images that may contain relevant data"). At the spot where the image appears in the reading order, place a traceability anchor followed by a Markdown image link to the extracted file:
    `<!-- image extracted to separate file: <filename>, source page <n> -->`
    `![<caption or legible text, else: Image from <source> page <n>>](<filename>)`
    Use the image's visible caption, title, or legible text as the alt text when present; otherwise fall back to `Image from <source> page <n>`. Do not invent a visual description of the image contents.
  - Otherwise (small logos, icons, or decorations that were not flagged), do not extract a file; describe based on visible textual information such as captions, alt text, titles, or legible text in the image. Use the format:
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

For images extracted from a PDF (see "Extracting images that may contain relevant data"), name each file after the source document so the relationship stays obvious, and place it alongside the `.md` file:

`<original_filename_without_extension>_p<page>_img<index>.<ext>`

The `<ext>` is the image's native format as reported by the extraction (for example `png`, `jpeg`), and `<index>` numbers the flagged images on that page starting at 1. Examples:

- The first substantial image on page 7 of `Annual report 2025.pdf` becomes `Annual report 2025_p7_img1.png`, referenced from `Annual report 2025.md`.
- A scanned page 3 of `Contract.pdf` becomes `Contract_p3_img1.jpeg`, referenced from `Contract.md`.

## Output

Deliver the separate `.md` files, one per source document (whether it came from the project context or as a chat attachment), with the content and HTML comment anchors described above, together with any image files extracted from PDFs, placed alongside the `.md` file they are referenced from.

Do not give accompanying explanation, no summary of what you did, and no meta commentary outside the files. A short, factual list of the extracted image files is the only accompanying text permitted, and only when images were extracted.
