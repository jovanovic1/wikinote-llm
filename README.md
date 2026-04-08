# WikiNote LLM

Build your own LLM knowledge-base wiki based on the idea shared by Andrej Karpathy in this [X post](https://x.com/karpathy/status/2039805659525644595).

This repository helps turn raw source material into structured, reusable insight instead of leaving learning trapped in chat or scattered notes.

## Quick Start

1. Download [Obsidian](https://obsidian.md/).
2. Clone this repo.
3. Add your source files to `raw/articles/`.
4. Ask your coding agent to run the wiki ingest workflow.
5. Open the `wiki/` folder in Obsidian to browse the generated pages.

Note: Obsidian is optional but recommended for browsing and editing the wiki. The underlying files are plain markdown and can be edited with any text editor.

## Capturing Sources From Chrome

If you collect reading material from the web, the [Obsidian Web Clipper](https://obsidian.md/clipper) Chrome extension is a useful input path for this repo.

With Obsidian Web Clipper, you can save web pages directly as Markdown instead of copying and pasting content by hand. A simple workflow is:

1. Install Obsidian Web Clipper in Chrome.
2. Open an article or page you want to keep.
3. Use the clipper to export the page as Markdown.
4. Save the resulting file into `raw/articles/`.
5. Run the wiki ingest workflow on the new file.

This keeps your source material in clean Markdown format from the start, which makes the ingest step more reliable.

## Agent Compatibility

This repo is compatible with both Codex and Claude Code.

|                     | Codex                  | Claude Code                    |
|---------------------|------------------------|--------------------------------|
| Instructions        | `AGENTS.md`            | `CLAUDE.md`                    |
| Skills              | `.codex/skills/wiki/`  | `.claude/skills/wiki/`         |
| Skill entry file    | `SKILLS.md`            | `SKILL.md`                     |
| Invoke skill        | `$wiki ingest`         | `/wiki ingest`                 |

## What This Repo Contains

- `raw/` - source material waiting to be ingested
- `wiki/sources/` - one page per ingested source
- `wiki/concepts/` - reusable abstract ideas that recur across sources
- `wiki/entities/` - concrete things like companies, products, and people
- `wiki/topics/` - synthesis pages that connect multiple sources and concepts
- `wiki/index.md` - the main map of the knowledge base
- `wiki/log.md` - ingest and maintenance history
- `AGENTS.md` - the operating rules for how the wiki should be maintained
- `.codex/skills/wiki/SKILLS.md` - the command workflow for ingest, query, and stats

## How The Wiki Works

The ingestion workflow is designed to produce more than summaries.

For each source, the project should:

- create a source page in `wiki/sources/`
- extract important ideas, patterns, and claims
- update or create relevant concept pages
- update or create relevant entity pages
- connect the source into broader topic pages
- update `wiki/index.md` and `wiki/log.md`

The core maintenance rules are:

- prefer editing existing pages over creating duplicates
- use `[[wikilinks]]` aggressively
- trace claims back to source pages
- keep pages concise but high-signal
- turn recurring patterns into shared concept or topic pages

## Current State

The wiki currently contains a large Paul Graham essay corpus with:

- a broad `sources/` layer
- a developed `concepts/` layer
- an `entities/` layer for recurring people, companies, and platforms
- a smaller `topics/` layer for cross-source synthesis

Recent cleanup work focused on raising page quality to the standard in `AGENTS.md`, especially:

- synthesized source summaries instead of lead-paragraph extraction
- concept pages expanded into full articles with clear leads
- stronger cross-linking between sources, concepts, entities, and topics
- linting for missing links, weak pages, and structural gaps

## Working With It

The commands are the same regardless of agent:

- `ingest` / `$wiki ingest` - process pending files from `raw/`
- `ingest <path>` - process a specific file
- `query ...` - answer questions using the wiki as the primary source
- `stats` - inspect coverage, activity, and gaps

Before making changes, read [`AGENTS.md`](./AGENTS.md).

For command behavior and exact ingest/query workflows, read [`SKILLS.md`](./.codex/skills/wiki/SKILLS.md).

## Goal

The goal of this project is not just to archive reading material. It is to build a persistent, compounding knowledge system where raw inputs become linked intelligence that is easier to query, refine, and reuse over time.
