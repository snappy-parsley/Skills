# WordCraft reference

Details agents load only when needed. Command catalog: run `wordcraft-cli commands --json` — do not paste or invent the full id list here.

## Script JSON (`wordcraft-cli run --script`)

```json
{
  "template": "sample",
  "commands": [
    {"command": "select.text", "params": {"text": "Membership"}},
    {"command": "format.bold"},
    "caret.docEnd",
    {"command": "text.insert", "params": {"text": "\nAppendix"}}
  ],
  "save": ["out.docx", "out.pdf"]
}
```

- `file` or `template` optional; CLI `--file` / `--template` override the script.
- A command is `{"command": id, "params": {…}}` or just the id string.
- `save` is a path or list; format follows the extension. Relative `file`/`save` resolve against the script folder.
- `file.new` / `file.open` open another document and make it active; later commands and saves use the active one. `view.switchWindows` lists open docs; `{"id": N}` switches.
- A failing command stops the run (exit 1); nothing is saved.
- Paths inside command params (e.g. a picture) are relative to the **working directory**, not the script folder.

## MCP setup

Headless (stdio):

```powershell
& "C:\Program Files\WordCraft\wordcraft-cli.exe" mcp --author "Cursor Agent" `
  --automation-read-root "C:\Projects" `
  --automation-write-root "C:\Projects"
```

Bridge to a running app:

```powershell
& "C:\Program Files\WordCraft\wordcraft.exe" --control 7981
& "C:\Program Files\WordCraft\wordcraft-cli.exe" mcp --connect 127.0.0.1:7981
```

Cursor MCP config sketch (`mcp.json`):

```json
{
  "mcpServers": {
    "wordcraft": {
      "command": "C:\\Program Files\\WordCraft\\wordcraft-cli.exe",
      "args": [
        "mcp",
        "--author", "Cursor Agent",
        "--automation-read-root", "C:\\Projects",
        "--automation-write-root", "C:\\Projects"
      ]
    }
  }
}
```

Roots must be existing folders. Omit a root → no access of that kind. `--connect` cannot combine with automation roots.

## CharProps / ParaProps (common fields)

Used by `format.set`, `para.set`, `styles.modify`. Lengths are **points**. Colour: `"RRGGBB"`, `"#RRGGBB"`, or `[r,g,b]`. Unknown field names are ignored — check spelling.

```json
{
  "command": "styles.modify",
  "params": {
    "style": "Heading1",
    "chr": {"font": "Georgia", "size": 18, "bold": true, "color": "1B1611"},
    "para": {
      "spaceBefore": 12,
      "lineSpacing": {"rule": "multiple", "value": 1.15},
      "keepNext": true
    }
  }
}
```

### CharProps (frequent)

| Field | Notes |
|-------|--------|
| `font`, `size` | Name; size in points |
| `bold`, `italic`, `strike`, `caps`, `smallCaps` | bool |
| `underline` | `none`, `single`, `double`, `dotted`, `dash`, `wave`, … |
| `color` | colour, `"Auto"`, or `{"theme": colour}` |
| `highlight` | `none`, `yellow`, `green`, … named highlights |
| `vertAlign` | `baseline`, `superscript`, `subscript` |
| `lang` | BCP 47 proofing tag |
| `link` | URL or `#bookmark` |

### ParaProps (frequent)

| Field | Notes |
|-------|--------|
| `style` | Paragraph style id |
| `align` | `left`, `center`, `right`, `justify`, `distribute` |
| `indentLeft`, `indentRight`, `indentFirst` | points; negative first = hanging |
| `spaceBefore`, `spaceAfter` | points |
| `lineSpacing` | `{"rule":"multiple\|atLeast\|exactly","value":…}` |
| `keepNext`, `keepLines`, `pageBreakBefore`, `widowControl` | bool |
| `outlineLevel` | 0–8 = Heading levels, 9 = body |
| `tabs` | `[{"pos":pt,"align"?:…,"leader"?:…}]` replaces inherited stops |

## Multi-document `doc` id

MCP tools (`execute`, `batch`, `type_text`, `select_text`, `get_text`, `inspect_document`, `render_page`, `save_document`) accept optional `doc` (open-document id). Omit → active document. `list_documents` returns ids.

## Upstream

- [docs/mcp.md](https://github.com/storytold/wordcraft/blob/main/docs/mcp.md) — full tool and property tables
- [docs/control-protocol.md](https://github.com/storytold/wordcraft/blob/main/docs/control-protocol.md) — JSON control channel, keys, `ui.*`
