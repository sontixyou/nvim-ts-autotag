# nvim-ts-autotag

> Forked from [windwp/nvim-ts-autotag](https://github.com/windwp/nvim-ts-autotag)

Use treesitter to **autoclose** and **autorename** html tag

It works with:

- html
- markdown
- tsx
- typescript
- xml

## Usage

```text
Before        Input         After
------------------------------------
<div           >              <div></div>
<div></div>    ciwspan<esc>   <span></span>
------------------------------------
```

## Setup

Requires `Nvim 0.11.5` and up.

Note that `nvim-ts-autotag` will not work unless you have treesitter parsers (like `html`) installed for a given
filetype. See [nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter) for installing parsers.

```lua
require('nvim-ts-autotag').setup({
  opts = {
    -- Defaults
    enable_close = true, -- Auto close tags
    enable_rename = true, -- Auto rename pairs of tags
    enable_close_on_slash = false -- Auto close on trailing </
  },
  -- Also override individual filetype configs, these take priority.
  -- Empty by default, useful if one of the "opts" global settings
  -- doesn't work well in a specific filetype
  per_filetype = {
    ["html"] = {
      enable_close = false
    }
  }
})
```

> [!CAUTION]
> If you are setting up via `nvim-treesitter.configs` it has been deprecated! Please migrate to the
> new way. It will be removed in `1.0.0`.

### A note on lazy loading

For those of you using lazy loading through a plugin manager (like [lazy.nvim](https://github.com/folke/lazy.nvim)) lazy
loading is not particularly necessary for this plugin. `nvim-ts-autotag` is efficient in choosing when it needs to load.
If you still insist on lazy loading `nvim-ts-autotag`, then two good events to use are `BufReadPre` & `BufNewFile`.

### Extending the default config

Let's say that there's a language that `nvim-ts-autotag` doesn't currently support and you'd like to support it in your
config. While it would be the preference of the author that you upstream your changes, perhaps you would rather not 😢.

For example, if you have a language that has a very similar layout in its Treesitter Queries as `html`, you could add an
alias like so:

```lua
require('nvim-ts-autotag').setup({
  aliases = {
    ["your language here"] = "html",
  }
})

-- or
local TagConfigs = require("nvim-ts-autotag.config.init")
TagConfigs:add_alias("your language here", "html")
```

That will make `nvim-ts-autotag` close tags according to the rules of the `html` config in the given language.

But what if a parser breaks for whatever reason, for example the upstream Treesitter tree changes its node names and now
the default queries that `nvim-ts-autotag` provides no longer work.

Fear not! You can directly extend and override the existing configs. For example, let's say the start and end tag
patterns have changed for `xml`. We can directly override the `xml` config:

```lua
local TagConfigs = require("nvim-ts-autotag.config.init")
TagConfigs:update(TagConfigs:get("xml"):override("xml", {
    start_tag_pattern = { "STag" },
    end_tag_pattern = { "ETag" },
}))
```

In fact, this very nearly what we do during our own internal initialization phase for `nvim-ts-autotag`.

### Enable update on insert

If you have that issue
[#19](https://github.com/windwp/nvim-ts-autotag/issues/19)

```lua
vim.lsp.handlers['textDocument/publishDiagnostics'] = vim.lsp.with(
    vim.lsp.diagnostic.on_publish_diagnostics,
    {
        underline = true,
        virtual_text = {
            spacing = 5,
            severity_limit = 'Warning',
        },
        update_in_insert = true,
    }
)
```

## Development

### Commands

```bash
# Run all tests
make test

# Run a specific test file
make test FILE=tests/specs/closetag_spec.lua

# Lint with stylua
make lint

# Format code with stylua
make format

# Clean test dependencies
make clean
```

Tests use plenary.nvim and run in headless Neovim. Test files are located in `tests/specs/`.

### Architecture

#### Core Modules

- `lua/nvim-ts-autotag.lua` - Main entry point, exports `setup()` function
- `lua/nvim-ts-autotag/internal.lua` - Core logic for tag closing (`close_tag`), slash closing (`close_slash_tag`), and renaming (`rename_tag`). Handles buffer attachment/detachment and keymaps
- `lua/nvim-ts-autotag/config/plugin.lua` - Plugin setup and configuration (`Setup`). Defines default options, filetype aliases, and autocmds
- `lua/nvim-ts-autotag/config/init.lua` - `TagConfigs` registry that stores filetype configurations
- `lua/nvim-ts-autotag/config/ft.lua` - `FiletypeConfig` class defining tag patterns for each filetype

#### How It Works

1. **Setup**: `require('nvim-ts-autotag').setup()` initializes configs and sets up autocmds for `InsertEnter`, `Filetype`, and `BufDelete`
2. **Attachment**: On supported filetypes, `internal.attach()` creates buffer-local keymaps for `>` (close tag) and `/` (close on slash) in insert mode
3. **Tag Detection**: Uses Treesitter queries with patterns defined in `FiletypeConfig` to find start/end tags
4. **Tag Operations**:
   - Close: Parses buffer, finds unclosed start tag, inserts closing tag
   - Rename: On `InsertLeave`, syncs start/end tag names

#### Adding New Filetype Support

New filetypes are added in `config/plugin.lua` by:
1. Creating a new config with `base_cfg:extend("filetype", { patterns... })`
2. Or adding an alias: `Setup.aliases["new_ft"] = "existing_ft"`

Pattern fields: `start_tag_pattern`, `start_name_tag_pattern`, `end_tag_pattern`, `end_name_tag_pattern`, `close_tag_pattern`, `close_name_tag_pattern`, `element_tag`, `skip_tag_pattern`

### Testing

Test cases in `tests/specs/` use a table-driven format:
```lua
{
    name = "test name",
    filepath = "./sample/index.html",
    filetype = "html",
    linenr = 10,
    key = [[>]],
    before = [[<div| ]],  -- | is cursor position
    after = [[<div>|</div>]],
}
```
