# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Jimmy is a Python note conversion tool that converts notes from 40+ note-taking applications and file formats to Markdown with front matter. It produces standalone binaries via PyInstaller and runs offline without AI.

**Repository:** https://github.com/marph91/jimmy

## Development Commands

### Linting
```bash
./lint.sh  # Runs: ruff check -> mypy jimmy -> pylint jimmy
```

### Testing
```bash
pytest --capture=no --numprocesses=auto
```

Run a single test:
```bash
pytest test/test_convert.py::test_function_name -v
```

### Building
- **Package:** `hatch build` (uses Hatchling)
- **Binary:** PyInstaller via `jimmy.spec` - generates platform-specific executables
- **Dependencies:** External binaries (pandoc, one2html) downloaded via `download_binaries.py`

### Documentation
```bash
mkdocs  # Serve/build MkDocs documentation
```

## Architecture

### Three-Phase Conversion Pipeline

1. **Parsing Phase** (`jimmy/formats/*.py`):
   - Each format has a `Converter` class inheriting from `BaseConverter`
   - Converts source format to **Intermediate Format** (IMF) data models
   - Dynamic module loading: `jimmy.formats.{format_name}`

2. **Filtering Phase** (`jimmy/filters.py`):
   - Filters notes by title patterns and tags
   - Uses CLI flags like `--include-notes`, `--exclude-notes`, `--include-notes-with-tags`

3. **Writing Phase** (`jimmy/writer.py`):
   - **PathDeterminer** (first pass): Determines filesystem paths, creates note ID map
   - **FilesystemWriter** (second pass): Writes notes, resources, updates links

### Core Data Models (`jimmy/intermediate_format.py`)

```
Notebook       # Hierarchical container
├── child_notebooks: List[Notebook]
└── child_notes: List[Note]

Note           # Individual note
├── title, body, created, updated
├── tags: List[Tag]
├── resources: List[Resource]
└── note_links: List[NoteLink]

Resource       # Attached file/image
Tag            # Label/metadata
NoteLink       # Internal note reference
```

### Converter Pattern

- `BaseConverter` (ABC) in `jimmy/converter.py` defines the interface
- `DefaultConverter` handles pandoc-based formats
- Format-specific converters in `jimmy/formats/*.py` (e.g., `google_keep.Converter`)
- All converter methods should be decorated with `@common.catch_all_exceptions` for fault tolerance

### Entry Points

- **CLI:** `jimmy/jimmy_cli.py` - argparse-based command line interface
- **TUI:** `jimmy/jimmy_tui.py` - Textual-based terminal user interface
- **Orchestration:** `jimmy/main.py:run_conversion()` - main conversion flow

### Key Modules

| Module | Purpose |
|--------|---------|
| `jimmy/converter.py` | BaseConverter, DefaultConverter classes |
| `jimmy/intermediate_format.py` | Note, Notebook, Resource, Tag dataclasses |
| `jimmy/writer.py` | PathDeterminer, FilesystemWriter |
| `jimmy/filters.py` | Note/tag filtering logic |
| `jimmy/variables.py` | FORMAT_REGISTRY dict, VERSION constant |
| `jimmy/common.py` | Config dataclass, utility functions |
| `jimmy/md_lib/convert.py` | Pandoc wrapper for markup conversion |
| `jimmy/md_lib/links.py` | Markdown link parsing/generation |

## Configuration

The `common.Config` dataclass (defined in `jimmy/common.py`) contains all configuration:
- `interface`: "tui" or "cli"
- `input`: List[Path] of input files/folders
- `format`: Format name (for format-specific converters)
- `frontmatter`: "joplin", "obsidian", "qownnotes", or None
- `output_folder`: Target directory
- Filter options: `include_notes`, `exclude_notes`, `include_tags`, `exclude_tags`

## Adding a New Format Converter

1. Create `jimmy/formats/{format_name}.py`
2. Define a `Converter` class inheriting from `BaseConverter`
3. Implement `convert()` and `convert_note()` methods
4. Register format in `jimmy/variables.py:FORMAT_REGISTRY`
5. Add documentation in `docs/formats/{format_name}.md`

## Development Notes

- **Reproducibility:** Always sort iterators with arbitrary order (e.g., `sorted(file_or_folder.iterdir())`)
- **Error handling:** Use `@common.catch_all_exceptions` decorator on converter methods
- **Logging:** Use `self.logger` in converters, `LOGGER` in modules
- **Type checking:** mypy runs in "semi-strict" mode - some untyped code is allowed
- **Line length:** 100 characters (ruff)
- **Python version:** 3.14

## Intermediate Format Choice

The IMF uses HTML as an intermediate representation (not Pandoc AST) because:
- HTML is easily modifiable by beautifulsoup
- Supports wide range of elements reducible to Markdown
- No additional dependencies beyond beautifulsoup (already used)
- Pandoc AST libraries (Panflute, pandocfilters) have outdated table support
