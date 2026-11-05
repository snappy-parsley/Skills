---
name: seismic-product-manuals
description: >-
  Build and maintain Seismic Source / DAQLink product user manuals with
  ManualBuilder (Markdown → PDF via Pandoc + Chrome). Use when creating a
  Manual/ folder, migrating Word .docx or legacy PDF manuals, fixing PDF
  layout, TOC or page numbers, leftover TOC 00 placeholders, figure captions,
  print.css, verifying a built manual PDF, or when the user mentions
  ManualBuilder, product manuals, or LineSetup/DataCollector manuals.
---

# Seismic product manuals (ManualBuilder)

## Before any manual work

Read these (in order):

1. `C:\Projects\ManualBuilder\docs\AUTHORING.md` — layout, toc.json, cover, migration, verify
2. `C:\Projects\ManualBuilder\docs\STYLES.md` — CSS and Chrome print gotchas

Also load the `pdfcraft` skill when verifying or extracting from PDFs.

Tool root: `C:\Projects\ManualBuilder`

Reference content trees:

- `C:\Projects\DAQLink\DataCollector5\Manual`
- `C:\Projects\DAQLink\LineSetup\Manual`

## Non-negotiables

- Product repos own **content only** (`Manual/*.md`, images, `toc.json`, `build.cmd`).
- Compile with `build-manual.ps1` — do not copy the toolchain into the product.
- Never hand-edit `Manual/pdf/*` (including with PdfCraft `edit` / `doc_save`).
- Change shared look-and-feel in `ManualBuilder\styles\print.css`, then rebuild.
- `build.cmd` must strip the trailing `\` from `%~dp0` before `-ManualRoot`.

## New product manual (summary)

1. Scaffold `Manual/` like LineSetup / DataCollector5.
2. Two-page cover with `"role": "cover"` in `toc.json`.
3. One chapter file per `h1`; real `###` subheads (not bold-only lines).
4. Images under `assets/images/` with stable names; one `![caption](…)` each.
5. Run `Manual\build.cmd` and verify (AUTHORING checklist + PdfCraft below).

## Word migration (summary)

Extract media → rename → Pandoc → split chapters → cover → fix captions → build → verify → only then delete legacy docx/pdf if requested.

## Legacy PDF migration

Prefer PdfCraft over one-off PyMuPDF snippets: `text` / `render` for content, `doc_export_images` (via `run --script`) for embeds. Land everything in Markdown under `Manual/`; ManualBuilder still produces the final PDF.

## Layout bugs — check STYLES.md first

Right-edge clipping, table borders, broken `<ol>` after headings, and TOC `00` placeholders are documented with known fixes in STYLES.md. Prefer fixing ManualBuilder once over per-product hacks. Never patch the generated PDF to “fix” layout — fix Markdown/`print.css` and rebuild.

## Verify PDF with PdfCraft

After `Manual\build.cmd`, use PdfCraft for agent-checkable verification. Producer remains ManualBuilder (Chrome print + page-numbers.py). CLI: `C:\Program Files\PdfCraft\pdfcraft-cli.exe`.

Map AUTHORING checklist → commands (`$pdf` = `Manual\pdf\<output>.pdf`):

| Check | PdfCraft |
|-------|----------|
| Page count / structure | `pdfcraft-cli info $pdf` |
| Cover → TOC → chapter order | `text --page 1`, `2`, TOC page, first body page |
| Leftover TOC `00` | `run --script` with `text_find` query `00` on TOC pages |
| Chapter text present | `text --page N` on first body chapter |
| Clickable TOC | `run --script`: `doc_open` then `link_list` |
| Figure/table right border, orphans | `render --page N --out page-N.png` then inspect image |

```powershell
$cli = "C:\Program Files\PdfCraft\pdfcraft-cli.exe"
$pdf = "C:\Projects\DAQLink\DataCollector5\Manual\pdf\DataCollector5-User-Manual.pdf"
$root = Split-Path $pdf
& $cli info $pdf
& $cli text $pdf --page 1
& $cli text $pdf --page 3
& $cli render $pdf --page 1 --out (Join-Path $root "verify-cover1.png")
```

Find leftover `00` placeholders and list TOC links (paths relative to `--root`):

```json
[
  {"tool": "doc_open", "args": {"path": "DataCollector5-User-Manual.pdf"}},
  {"tool": "text_find", "args": {"doc": 1, "query": "00", "start_page": 3, "limit": 50}},
  {"tool": "link_list", "args": {"doc": 1}}
]
```

```powershell
& $cli run --script verify.json --root $root --report
```

Fail the verify if TOC text is dominated by `00`, expected chapter headings are missing, or `link_list` is empty on a TOC that should be clickable. Visual border/orphan issues: render the suspect pages and read the PNGs.

For image export during legacy-PDF migration:

```json
[
  {"tool": "doc_open", "args": {"path": "Legacy.pdf"}},
  {"tool": "doc_export_images", "args": {"doc": 1, "folder": "extracted-images", "format": "png"}}
]
```
