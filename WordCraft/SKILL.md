---
name: wordcraft
description: >-
  Use when creating, editing, converting, inspecting, or reviewing Word documents
  (.docx, .doc, .odt, .rtf, .html, .md, .tex, .txt, .pdf), or when the user
  mentions WordCraft, wordcraft-cli, track changes, mail merge, or driving a
  Word document from the CLI or MCP.
---

# WordCraft

WordCraft is the installed Rust word processor. Every ribbon action is a **command id**. Drive documents through its CLI or MCP — do not unzip `.docx`, hand-edit OOXML, or reach for LibreOffice/pandoc when WordCraft can do the job.

## Binary (this machine)

| Role | Path |
|------|------|
| CLI | `C:\Program Files\WordCraft\wordcraft-cli.exe` (v0.15.1) |
| App | `C:\Program Files\WordCraft\wordcraft.exe` |

Prefer the full path. If `wordcraft-cli` is on `PATH`, that works too. Below, `wordcraft-cli` means that executable.

## Choose surface

| Need | Use |
|------|-----|
| Convert / extract / one-shot edit | CLI: `convert`, `text`, `inspect`, `info`, `render`, `run` |
| Multi-step edit in-session | MCP (`wordcraft-cli mcp`) if configured; else `run --script` |
| Drive the open desktop window | `wordcraft --control 7981` then `wordcraft-cli mcp --connect 127.0.0.1:7981` |

## CLI recipes

```powershell
wordcraft-cli inspect report.docx
wordcraft-cli text report.docx
wordcraft-cli info report.docx
wordcraft-cli convert report.docx report.pdf   # also odt, rtf, html, md, tex, txt, png, json
wordcraft-cli render report.docx page.png --page 1
wordcraft-cli run --template sample `
  --cmd 'select.text={"text":"Membership"}' --cmd format.bold --save out.docx
wordcraft-cli run --script job.json
wordcraft-cli commands --json                  # discover ids — never invent them
```

Password-protected files: `--password-file PATH` or `WORDCRAFT_PASSWORD` (avoid `--password` on the command line).

## Agent rules

1. **Use WordCraft.** After reading this skill, drive documents with CLI/MCP only.
2. **Verify before claiming success.** Re-read with `inspect` / `text` (or MCP `inspect_document` / `get_text`). Prefer inspect over screenshots.
3. **Styles over scattershot formatting.** Prefer `styles.*` and paragraph styles so TOC and restyles keep working.
4. **Track changes.** Set author with `--author` or `file.setAuthor`. Turn tracking on with `review.trackChanges {"value": true}` — a bare call *toggles* and can turn it off.
5. **Collapse selection** before `type_text` / `text.insert` when you must not replace selected text.
6. **MCP roots.** When configuring the server, set `--automation-read-root` and `--automation-write-root`. A missing write root blocks saves.
7. **Alpha software.** On failure, look up the id in `commands --json` / `list_commands`. Do not claim full Microsoft Word parity.

## MCP tools

| Tool | Role |
|------|------|
| `list_commands` | Search command ids |
| `execute` / `batch` | Run one or many commands |
| `type_text` / `select_text` | Type at caret / select nth match |
| `get_text` / `inspect_document` | Verify without screenshots |
| `render_page` | Page as PNG |
| `new_document` / `open_document` / `save_document` / `list_documents` | File lifecycle; multi-doc via `doc` id |
| `screenshot` / `click` / `key` / `ui_inspect` | App only (`--connect`) |

## Common mistakes

- Inventing command ids instead of listing them
- Screenshots as the only verification
- Forgetting `doc` when several documents are open
- Typing while a range is selected (replaces the selection)
- MCP with a read root but no write root, so saves fail

## More detail

Script JSON shape, CharProps/ParaProps fields, and MCP setup: [reference.md](reference.md).

Upstream: [For agents](https://github.com/storytold/wordcraft#for-agents-cli-and-mcp), [docs/mcp.md](https://github.com/storytold/wordcraft/blob/main/docs/mcp.md), [control protocol](https://github.com/storytold/wordcraft/blob/main/docs/control-protocol.md).
