---
name: pdfcraft
description: >-
  Use when creating, editing, inspecting, combining, splitting, converting,
  redacting, signing, OCR-ing, or reviewing PDF files, or when the user
  mentions PdfCraft, pdfcraft-cli, PDF forms, comments, bookmarks, or driving
  a PDF from the CLI or MCP.
---

# PdfCraft

PdfCraft is the installed Rust PDF app. Engine features are exposed as JSON-Schema **tools** shared by the CLI, MCP, and Rust API. Prefer PdfCraft over ad-hoc PDF libraries, qpdf, or GUI clicking when the CLI/MCP can do the job.

## Binary (this machine)

| Role | Path |
|------|------|
| CLI | `C:\Program Files\PdfCraft\pdfcraft-cli.exe` (v0.18.1) |
| App | `C:\Program Files\PdfCraft\pdfcraft.exe` |

Prefer the full path. Below, `pdfcraft-cli` means that executable.

## Choose surface

| Need | Use |
|------|-----|
| Info / text / render / page ops | CLI: `info`, `text`, `render`, `edit`, `combine`, `extract`, `split`, `audit`, `batch` |
| Multi-step session (comments, forms, protect…) | `pdfcraft-cli run` / `run --script`, or MCP |
| Drive the open desktop window | App with `--control FILE` + `pdfcraft-cli ui --control FILE …` |

## CLI recipes

```powershell
pdfcraft-cli info  form.pdf
pdfcraft-cli text  paper.pdf --page 3
pdfcraft-cli render report.pdf --page 1 --out page.png
pdfcraft-cli edit  in.pdf --rotate 1,2:90 --delete 5 --title "Q3" --out out.pdf
pdfcraft-cli combine a.pdf b.pdf --out combined.pdf
pdfcraft-cli extract in.pdf --pages 1,3,5-8 --out highlights.pdf
pdfcraft-cli split   report.pdf --every 10 --out-dir parts/
pdfcraft-cli audit   redacted.pdf          # fails if faux redactions found
pdfcraft-cli tools                         # every tool + JSON Schema — never invent names
pdfcraft-cli run text_find doc=1 query=invoice
pdfcraft-cli run --script steps.json --root .\work --report
```

## Agent rules

1. **Use PdfCraft.** Drive PDFs with CLI/MCP tools; do not reverse-engineer COS objects by hand unless diagnosing a PdfCraft bug.
2. **Discover tools.** Run `pdfcraft-cli tools` (or MCP `tool_search`) before inventing a tool name or args.
3. **Save explicitly.** Session edits stay in memory and are undoable until `doc_save`. Unsaved changes are never discarded silently.
4. **Verify.** Re-check with `info` / `text` / `doc_inspect` / `text_extract` / `render` before claiming success. Prefer those over screenshots.
5. **Confine paths.** Prefer `--root DIR` on `run`/`mcp`, or `--automation-read-root` / `--automation-write-root`. A missing write root blocks saves.
6. **MCP is opt-in.** It only runs while `pdfcraft-cli mcp` is launched (stdio, no network port). Use `--compact` to shrink the tool list (`tool_search` / `tool_call` reach the rest).
7. **Redaction.** Prefer real redaction tools (`redact_mark` / `redact_apply`); use `audit` to catch faux redactions.
8. **Alpha / limits.** On failure, look up the tool schema. Do not claim full Adobe Acrobat parity.

## Core automation tools

Open → work → save. Common names (full list via `tools`):

| Tool | Role |
|------|------|
| `doc_open` / `doc_save` / `doc_close` / `doc_list` | Session lifecycle; docs addressed by integer `doc` |
| `doc_info` / `doc_inspect` | Structure and metadata |
| `text_extract` / `text_find` | Reading-order text / search |
| `page_render` / `render_preview` | Page images |
| `page_rotate` / `page_delete` / `page_move` / `page_extract` / `doc_combine` / `doc_split` | Organize |
| `comment_add` / `comment_list` / … | Comments (e.g. highlight via `find`) |
| `form_fields` / `form_fill` | Forms |
| `doc_protect` / `redact_*` / `sign_document` | Security |
| `command_list` / `command_run` / `command_batch` | Ribbon-style command bridge |

## Common mistakes

- Inventing tool names instead of running `tools`
- Forgetting `doc_save` after a script of edits
- Screenshots as the only verification
- MCP/`run` without a write-capable root, so saves fail
- Leaving the control file in a shared folder (use a private path only you can write)

## More detail

Script JSON, MCP/`--compact`, UI control, and confinement: [reference.md](reference.md).

Upstream: [Built for agents](https://github.com/storytold/pdfcraft#built-for-agents-too), [docs/mcp.md](https://github.com/storytold/pdfcraft/blob/main/docs/mcp.md).
