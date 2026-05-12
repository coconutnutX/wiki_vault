# Code Architecture Wiki: LLM Wiki

Mode: B (GitHub / Repository)
Purpose: Document codebase architecture, decisions, patterns, and flows
Owner: aaa
Created: 2026-05-11

## Structure

```
vault/
├── .raw/              # immutable source documents (README, code dumps, issue exports)
├── wiki/
│   ├── modules/       # one note per major module / package / service
│   ├── components/    # reusable UI or functional components
│   ├── decisions/     # Architecture Decision Records (ADRs)
│   ├── dependencies/  # external deps, versions, risk assessment
│   ├── flows/         # data flows, request paths, auth flows
│   ├── entities/      # people, orgs, products, repos
│   ├── concepts/      # ideas, patterns, frameworks
│   ├── sources/       # one summary page per raw source
│   ├── questions/     # filed answers to user queries
│   ├── comparisons/   # side-by-side analyses
│   ├── meta/          # dashboards, lint reports, conventions
│   ├── index.md       # master catalog of all pages
│   ├── log.md         # chronological record of all operations
│   ├── hot.md         # hot cache: recent context summary (~500 words)
│   └── overview.md    # executive summary of the whole wiki
├── _templates/        # note templates (module, component, decision, etc.)
└── .obsidian/         # Obsidian vault config + CSS snippets
```

## Conventions

- All notes use YAML frontmatter: type, status, created, updated, tags (minimum)
- Wikilinks use [[Note Name]] format: filenames are unique, no paths needed
- .raw/ contains source documents: never modify them
- wiki/index.md is the master catalog: update on every ingest
- wiki/log.md is append-only: never edit past entries
- New log entries go at the TOP of the file
- **Every directory's _index.md must be updated when pages are added/removed.** This applies recursively: if you add a page to wiki/decisions/, update wiki/decisions/_index.md AND wiki/index.md
- **Inline wikilinks at the actual mention point in body text.** Do not append "Related Pages" sections at the end. Cross-page discovery is handled by _index.md.
- **Do not commit & push on every edit.** Only when the user explicitly asks.

## Operations

- Ingest: drop source in .raw/, say "ingest [filename]"
- Query: ask any question: Claude reads index first, then drills in
- Lint: say "lint the wiki" to run a health check
- Save: say "/save" to save current conversation context
- Archive: move cold sources to .archive/ to keep .raw/ clean
