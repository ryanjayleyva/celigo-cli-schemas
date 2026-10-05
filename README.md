# Celigo CLI Schemas

**NOTE:** This is an unaffiliated, personally maintained project. It is not
a Celigo product, is not endorsed by Celigo, and is not covered by Celigo
support.

JSON Schemas and editor configuration for Celigo CLI projects, covering
Neovim, VS Code, Cursor, and Zed. Drop these into any Celigo CLI project and
you get autocompletion, inline validation, and hover documentation for every
resource type while editing JSON files.

## What this is

A Celigo CLI project is a directory tree of JSON files, one per resource
(flows, exports, imports, connections, and so on). No editor has built-in
knowledge of what fields those files support. This repo supplies that
knowledge:

- `.celigo/schemas/` - a JSON Schema per resource type, shared by every
  editor below
- `.vscode/settings.json` - maps each resource folder to its schema for VS
  Code, Cursor, and Neovim users running neoconf.nvim (it imports
  `.vscode/settings.json` automatically)
- `.zed/settings.json` - the same mapping in Zed's configuration format

Once the relevant editor config is in place, you get validation as you type
and autocomplete for every property the schema defines.

## How this fits with the CLI

This repo is not the Celigo CLI and does not replace `celigo lint`, it only
makes the JSON files you hand to the CLI easier to write correctly in the
first place.

Say you are creating a connection by hand and run:

```bash
celigo connections create -f connection.json
```

The CLI checks the body before sending it and reports back what is missing
or wrong, for example a required field like `type`, or credential fields a
connector's own form expects. Without schema support in your editor, finding
out what's missing means reading that error, then cross-checking the API
docs to figure out the right field name and shape, then editing the file and
running the command again.

With `.celigo/schemas/connection.schema.json` wired into your editor, the
same information shows up while you type `connection.json`, before you ever
run the command: your editor flags a missing required field (`name` and
`type` are required at the top level) and autocompletes the valid property
names and enum values for the fields that exist. You still run the CLI
command to actually create the resource and catch anything the static
schema can't express (connector-specific conditional requirements, live
account state), but most of the back-and-forth of "what field am I missing"
is resolved before you hit enter.

## Setup

Every new Celigo CLI project needs its own copy of `.celigo/schemas/` plus
the config file or directory for whichever editor your team uses. Add them
once, right after you scaffold the project.

Clone this repo and copy the files into your project root.

```bash
git clone <this-repo-url> /tmp/celigo-cli-schemas
cp -r /tmp/celigo-cli-schemas/.celigo <path-to-your-project>/
cp -r /tmp/celigo-cli-schemas/.vscode <path-to-your-project>/
cp -r /tmp/celigo-cli-schemas/.zed <path-to-your-project>/
```

Copy `.celigo/schemas/` regardless of editor. Copy `.vscode/` and `.zed/`
only for the editors your team actually uses, there is no harm in keeping
both. `.vscode/` also covers Neovim users on neoconf.nvim, see below.

### Neovim

Neovim has no built-in JSON language server and no single config file
convention the way VS Code and Zed do. What you get depends on your setup.

If your Neovim config uses [neoconf.nvim][neoconf] to manage project-local
LSP settings, you do not need a separate file. neoconf.nvim ships with
`import = { vscode = true }` enabled by default, it reads the project's
`.vscode/settings.json` on its own and feeds the `json.schemas` array to
`jsonls`, the same JSON language server VS Code and Zed use under the hood.
Copying `.vscode/` as shown above is enough, confirm your neoconf.nvim setup
has not disabled the `vscode` import.

If you are not running neoconf.nvim, there is no repo file to drop in, the
mapping has to live in your own Neovim config since nothing auto-loads
project-local JSON files. Which API you use depends on your Neovim version.

**Neovim 0.11 and later** (including 0.12) configure LSP servers natively
with `vim.lsp.config()` and `vim.lsp.enable()`. The old
`require("lspconfig").jsonls.setup({...})` pattern is deprecated,
`nvim-lspconfig` now only supplies the server definitions that this native
API consumes. Add this to your config once, it applies to every Celigo CLI
project that follows this folder layout:

```lua
vim.lsp.config("jsonls", {
  settings = {
    json = {
      schemas = {
        {
          fileMatch = { "flows/*.json" },
          url = vim.fn.getcwd() .. "/.celigo/schemas/flow.schema.json",
        },
        {
          fileMatch = { "exports/*.json" },
          url = vim.fn.getcwd() .. "/.celigo/schemas/export.schema.json",
        },
        -- add the remaining resource types from the Resource type
        -- mapping table below
      },
    },
  },
})

vim.lsp.enable("jsonls")
```

**Neovim 0.10 and earlier** still use `nvim-lspconfig`'s `setup()` function:

```lua
require("lspconfig").jsonls.setup({
  settings = {
    json = {
      schemas = {
        {
          fileMatch = { "flows/*.json" },
          url = vim.fn.getcwd() .. "/.celigo/schemas/flow.schema.json",
        },
        {
          fileMatch = { "exports/*.json" },
          url = vim.fn.getcwd() .. "/.celigo/schemas/export.schema.json",
        },
        -- add the remaining resource types from the Resource type
        -- mapping table below
      },
    },
  },
})
```

`vim.fn.getcwd()` resolves against whichever project you open Neovim in, so
either snippet works across all of your Celigo CLI projects as long as each
one has `.celigo/schemas/` in place.

### VS Code

If your project already has a `.vscode/settings.json` with other settings,
merge the `json.schemas` array into it instead of overwriting the file.

Open the project in VS Code and reload the window. Open any file under a
mapped folder (for example `flows/my-flow.json`) and confirm you get
autocomplete on `Ctrl+Space` / `Cmd+Space`.

### Cursor

Cursor is a VS Code fork and reads the same `.vscode/settings.json`
natively. No separate configuration is needed, copying `.vscode/` is
sufficient.

### Zed

Zed does not read `.vscode/settings.json`. It reads schema associations
from `.zed/settings.json`, under
`lsp.json-language-server.settings.json.schemas`, which uses the same
`fileMatch` / `url` shape. If your project already has a `.zed/settings.json`,
merge the `schemas` array into it instead of overwriting the file.

Open the project in Zed, open any file under a mapped folder, and confirm
you get completions and validation.

## Resource type mapping

Each entry in `.vscode/settings.json` and `.zed/settings.json` matches a
folder name to a schema file, using the same mapping in both. Keep your
project's folder names aligned with this table or the schemas will not
apply.

| Folder                 | Schema                           |
| ---------------------- | -------------------------------- |
| `flows/`               | `flow.schema.json`               |
| `exports/`             | `export.schema.json`             |
| `imports/`             | `import.schema.json`             |
| `connections/`         | `connection.schema.json`         |
| `iclients/`            | `iclient.schema.json`            |
| `integrations/`        | `integration.schema.json`        |
| `apis/`                | `api.schema.json`                |
| `tools/`               | `tool.schema.json`               |
| `scripts/`             | `script.schema.json`             |
| `lookup-caches/`       | `lookupcache.schema.json`        |
| `mcp-servers/`         | `mcp-server.schema.json`         |
| `mcp-oauth-providers/` | `mcp-oauth-provider.schema.json` |
| `ai-agents/`           | `ai-agent.schema.json`           |
| `guardrails/`          | `guardrail.schema.json`          |
| `edi-profiles/`        | `ediprofile.schema.json`         |
| `file-definitions/`    | `filedefinition.schema.json`     |
| `stacks/`              | `stack.schema.json`              |
| `async-helpers/`       | `asynchelper.schema.json`        |
| `on-premise-agents/`   | `on-premise-agent.schema.json`   |

## Adding or updating a schema

1. Add or edit the schema file in `.celigo/schemas/`.
2. Add a matching `fileMatch` / `url` entry to `.vscode/settings.json` and
   `.zed/settings.json` if it is a new resource type. If you maintain the
   plain `nvim-lspconfig` snippet from the Neovim section in your own
   dotfiles, update it too.
3. Reload the editor to pick up the change.

## Source

https://developer.celigo.com/api
