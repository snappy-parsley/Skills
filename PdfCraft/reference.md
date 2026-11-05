# PdfCraft reference

Load when you need script shape, MCP setup, or UI control details. Full tool schemas: `pdfcraft-cli tools`.

## Script JSON (`pdfcraft-cli run --script`)

Array of steps; one session; `doc` ids from `doc_open`:

```json
[
  {"tool": "doc_open", "args": {"path": "in.pdf"}},
  {"tool": "text_find", "args": {"doc": 1, "query": "total due"}},
  {"tool": "comment_add", "args": {"doc": 1, "type": "highlight", "find": "total due"}},
  {"tool": "doc_save", "args": {"doc": 1, "path": "out.pdf"}}
]
```

```powershell
pdfcraft-cli run --script steps.json --root .\work --report
```

- `--root` confines every path the steps read or write (including PNG `out` paths). The script file itself may live outside the root.
- `--report` prints one JSON document: per step number, tool, status, result or error.
- One-off: `pdfcraft-cli run text_find doc=1 query=invoice` (`key=value`; values parse as JSON).
- Edits stay in memory until `doc_save`. Saving over the same path appends an incremental update (atomic write; original bytes preserved in the revision chain).

## MCP setup

Opt-in stdio server (no port). Prefer absolute CLI path on Windows:

```json
{
  "mcpServers": {
    "pdfcraft": {
      "command": "C:\\Program Files\\PdfCraft\\pdfcraft-cli.exe",
      "args": ["mcp", "--compact", "--root", "C:\\Projects"]
    }
  }
}
```

Separate read/write roots (Craft-suite style):

```powershell
& "C:\Program Files\PdfCraft\pdfcraft-cli.exe" mcp `
  --automation-read-root "C:\Projects" `
  --automation-write-root "C:\Projects\out"
```

- `--compact`: ~ten core tools plus `tool_search` / `tool_call` for the rest (fewer tokens).
- `--root` can combine with automation roots; a path must satisfy all given constraints.
- Roots must be existing folders. A root left out grants none of that access.

### Compact / command bridge helpers

| Tool | Role |
|------|------|
| `tool_search` / `tool_call` | Find and run any tool when `--compact` |
| `command_list` | Ribbon/command catalog (`filter?`) |
| `command_run` | `{"id":"page.rotate","params":{"doc":1,"degrees":90}}` |
| `command_batch` | Several commands; `stop_on_error` default true |
| `doc_inspect` | Session/document inspect |
| `render_preview` | Quick page PNG (`page` 1-based, `max_side` default 1024) |

Resources: `pdfcraft://document`, `pdfcraft://commands`.

## Drive the desktop UI

Control file must be in a folder only you can write (not a shared temp dir). On Windows:

```powershell
$ctrl = "$env:USERPROFILE\.pdfcraft-control.json"
& "C:\Program Files\PdfCraft\pdfcraft.exe" --control $ctrl
# other terminal:
pdfcraft-cli ui --control $ctrl inspect query=rotate
pdfcraft-cli ui --control $ctrl click label="Organize pages"
pdfcraft-cli ui --control $ctrl command id=comment.square
pdfcraft-cli ui --control $ctrl drag from='[400,300]' to='[600,420]'
pdfcraft-cli ui --control $ctrl screenshot --out window.png
pdfcraft-cli ui --control $ctrl quit
```

Loopback only; each connection presents the token in that file. `ui` refuses symlink control files. `quit` never silently discards unsaved work (`needs_confirmation` when a prompt would appear).

## Convenience CLI (no session script)

| Command | Role |
|---------|------|
| `info` / `text` / `render` | Inspect |
| `edit` | Rotate, delete, move, insert blank, metadata (`--out` required) |
| `combine` / `extract` / `split` | Organize files |
| `audit` | Faux-redaction check (nonzero exit if found) |
| `batch` | `convert`, `ocr`, `optimize` over files/dirs |
| `check` | Robustness sweep |

## Upstream

- [Built for agents](https://github.com/storytold/pdfcraft#built-for-agents-too)
- [docs/mcp.md](https://github.com/storytold/pdfcraft/blob/main/docs/mcp.md)
