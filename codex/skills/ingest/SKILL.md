---
name: ingest
description: Ingest PDFs, DOCX, markdown/text files, and URLs into a knowledge vault as atomic interconnected notes. Use when the user wants to turn research papers, documentation, URLs, or team docs into permanent knowledge notes with literature/internal formats.
---

# Document Ingestion

Turn external documents into structured, linkable knowledge notes.

## Quick Start

```text
ingest ~/Downloads/attention-paper.pdf
ingest https://example.com/api-docs --internal
ingest spec.docx --title "API Specification v2"
ingest paper.pdf --dry-run
```

## Arguments

- `file-or-url`: document path (`.pdf`, `.docx`, `.md`, `.txt`) or URL.
- `--internal`: create an internal note instead of a literature note.
- `--title "..."`: override note title.
- `--dry-run`: preview extraction and target path without writing.

## Workflow

1. Parse arguments and determine mode:
   - Literature: `literature/lit-{slug}.md`.
   - Internal: `internal/int-{slug}.md`.
2. Read the source:
   - `.md` / `.txt`: read directly.
   - `.pdf`: use available extraction tools such as `pdftotext` or `pandoc`.
   - `.docx`: use `pandoc -t plain` if available.
   - URL: browse/fetch the content when network access is available.
3. Find the vault root:
   - Prefer `~/.config/claude-note/config.toml` if present and it contains `vault_root`.
   - If no config exists, ask the user for the target vault path.
4. Extract knowledge:
   - Summary.
   - Key concepts.
   - Highlights or important findings.
   - Open questions.
   - Related topics.
5. Check for likely duplicates in the destination mode directory. If a similar note exists, ask whether to merge, create separately, or cancel.
6. Create the note using the relevant template unless `--dry-run` is set.
7. Report the created path or dry-run preview.

## Note Formats

- Literature notes: `references/literature-format.md`.
- Internal notes: `references/internal-format.md`.

## Optional CLI

If `claude-note` is installed, it may be used for extraction and semantic deduplication. Do not require it for basic ingestion; fall back to built-in file reads and common extraction tools.

## Resources

- `references/literature-format.md`
- `references/internal-format.md`
- `examples/sample-output.md`
