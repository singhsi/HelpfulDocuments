# HelpfulDocuments Agent Instructions

## Repository Purpose

- Personal reference repository of developer guides and command cheat sheets.
- Content only: Markdown docs and SVG diagrams. No application code, build, lint, or test commands.
- Audience is software engineers; explain concepts in simple terms with practical examples.

## Structure

- One top-level folder per topic (`git/`, `java/`, `podman/`, `web/`); nested subfolders for subtopics (e.g. `java/build/maven/`).
- Each topic folder typically has `README.md` (beginner-friendly guide) and `commands.md` (quick-lookup commands).
- Extra focused files sit alongside them (e.g. `git/stash.md`, `java/java-se/performance.md`, `web/rest_api.md`).
- Folder `README.md` files contain a Contents table linking every file in that folder.
- Root `README.md` lists every topic folder with a `| File | Description |` or `| Subfolder | Description |` table.
- When adding, moving, or renaming a doc, update the Contents tables in the parent `README.md` files.

## Writing Style

- Start with `# Title`, a one-line summary, then a Table of Contents for longer guides.
- Separate major sections with `---`.
- Prefer tables for comparisons and reference data, fenced code blocks for commands.
- Use `>` blockquotes for tips, warnings, and "See also" links.
- Use relative links between docs (e.g. `[stash.md](stash.md)`, `[intro_web.md](intro_web.md)`).
- In `commands.md` files, use a `####` heading describing the task followed by the command.
- Use `<placeholder>` syntax for values the reader must replace.
- Keep examples accurate; do not invent flags, versions, or tool behavior.

## Diagrams

- Store diagrams in an `images/` folder inside the topic (e.g. `web/images/`).
- Use hand-written SVG so they render on GitHub and in the VS Code preview without build tools.
- Give every SVG a white background rect so it stays readable in dark themes.
- Prefix file names by guide and order (e.g. `01-big-picture.svg`, `rest-01-overview.svg`).
- Reference with descriptive alt text: `![What the diagram shows](images/<file>.svg)`.
- Validate with `xmllint --noout <file>.svg` and preview (e.g. `qlmanage -t -s 1000 -o <dir> <file>.svg` on macOS) to check for overlapping text.

## Working Style

- Touch only the docs required for the request.
- Do not restructure or reword existing docs unless asked.
- Verify that every relative link and image path resolves before finishing.
- Review changes with `git diff` before finishing.

## Git

- Commit messages are short, lowercase, past-tense sentences ending with a period (e.g. `moved build folder into java.`).
- `.DS_Store` is ignored; never commit it.
- Binary docs (e.g. `.docx`) exist in places; do not edit them unless asked.
