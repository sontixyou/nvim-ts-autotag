# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

nvim-ts-autotag is a Neovim plugin that uses Treesitter to automatically close and rename HTML/XML tags. It supports multiple filetypes including HTML, JSX/TSX, Vue, Svelte, and more.

## Development Commands

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

## Architecture

### Core Modules

- `lua/nvim-ts-autotag.lua` - Main entry point, exports `setup()` function
- `lua/nvim-ts-autotag/internal.lua` - Core logic for tag closing (`close_tag`), slash closing (`close_slash_tag`), and renaming (`rename_tag`). Handles buffer attachment/detachment and keymaps
- `lua/nvim-ts-autotag/config/plugin.lua` - Plugin setup and configuration (`Setup`). Defines default options, filetype aliases, and autocmds
- `lua/nvim-ts-autotag/config/init.lua` - `TagConfigs` registry that stores filetype configurations
- `lua/nvim-ts-autotag/config/ft.lua` - `FiletypeConfig` class defining tag patterns for each filetype

### How It Works

1. **Setup**: `require('nvim-ts-autotag').setup()` initializes configs and sets up autocmds for `InsertEnter`, `Filetype`, and `BufDelete`
2. **Attachment**: On supported filetypes, `internal.attach()` creates buffer-local keymaps for `>` (close tag) and `/` (close on slash) in insert mode
3. **Tag Detection**: Uses Treesitter queries with patterns defined in `FiletypeConfig` to find start/end tags
4. **Tag Operations**:
   - Close: Parses buffer, finds unclosed start tag, inserts closing tag
   - Rename: On `InsertLeave`, syncs start/end tag names

### Adding New Filetype Support

New filetypes are added in `config/plugin.lua` by:
1. Creating a new config with `base_cfg:extend("filetype", { patterns... })`
2. Or adding an alias: `Setup.aliases["new_ft"] = "existing_ft"`

Pattern fields: `start_tag_pattern`, `start_name_tag_pattern`, `end_tag_pattern`, `end_name_tag_pattern`, `close_tag_pattern`, `close_name_tag_pattern`, `element_tag`, `skip_tag_pattern`

## Testing

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
