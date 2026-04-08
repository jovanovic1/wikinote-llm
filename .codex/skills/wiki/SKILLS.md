---
name: wiki
description: >
  Handles all /wiki slash commands for maintaining a personal LLM wiki knowledge base.
  Use this skill whenever the user types /wiki ingest, /wiki ingest with a path, /wiki query,
  or /wiki stats. Also trigger when the user says things like "ingest this file into the wiki",
  "search the wiki for X", "query the wiki about Y", "how many pages does my wiki have",
  or any reference to wiki operations, ingestion, or querying their local markdown knowledge base.
  This skill is essential for the LLM Wiki pattern — always use it when wiki operations are requested.
---

# Wiki Skill

Handles four slash commands for the LLM Wiki system. Each command has a clear workflow below.

## Assumptions

- Wiki root is the current working directory, or the closest parent directory containing `AGENTS.md` + `wiki/` + `raw/`.
- If no wiki root is found, ask the user where their wiki lives before proceeding.
- All wiki pages are `.md` files. The wiki follows the structure defined in `AGENTS.md` — read it before any operation.

## Detecting wiki root

```bash
# Walk up from cwd looking for AGENTS.md + wiki/ + raw/
find . -maxdepth 3 -name "AGENTS.md" | head -1
```

If found, set `WIKI_ROOT` to that directory. If not found, ask the user.

---

## Command: `/wiki ingest` (no path)

Find all pending files in `raw/` not yet ingested, then ingest them — one at a time or in batch.

### Steps

1. **Find all pending files**
   ```bash
   # All files in raw/ (excluding assets/)
   find $WIKI_ROOT/raw -maxdepth 1 -type f | sort -t/ -k1 | grep -v assets
   
   # Check which are already ingested
   grep "^## \[" $WIKI_ROOT/wiki/log.md | grep "ingest"
   ```
   Cross-reference both lists. Build a list of pending files: files in `raw/` whose filename does not appear in any log entry.

2. **If 0 pending files**: Report "Nothing new in raw/ — all files have been ingested." List the 5 most recently ingested files from log.md for context.

3. **If 1 pending file**: Proceed directly as `/wiki ingest <path>` with no confirmation needed.

4. **If 2–5 pending files**: Show the list and confirm:
   > "Found 3 new files to ingest:
   > - raw/article-one.md
   > - raw/paper-two.pdf
   > - raw/notes-three.md
   > 
   > Ingest all 3 in sequence?"
   
   If confirmed, run `/wiki ingest <path>` for each file in order (oldest first by modification time). After each file, briefly report what was created/updated before moving to the next.

5. **If 6–20 pending files**: Show the full list grouped by file type, report the count, and ask:
   > "Found 14 pending files. Choose a mode:
   > - **Supervised** — I'll pause after each file for your review (slower, more control)
   > - **Batch** — I'll ingest all files back-to-back and give you a summary at the end (faster)
   > - **Select** — Tell me which files to ingest now (e.g. 'just the PDFs' or 'first 5')"

6. **Supervised mode**: Run `/wiki ingest <path>` per file. After each, print a one-paragraph summary of what changed, then ask "Continue to next file? (or type a note to add before proceeding)". User can type notes that get appended to the just-created source page before moving on.

7. **Batch mode**: Ingest all pending files sequentially with no pauses. Suppress the per-file discussion step (step 3 in the full ingest workflow) — write pages directly based on content. At the end, print a single consolidated report:
   ```
   Batch ingest complete — <N> files processed
   
   Pages created:  <total>
   Pages updated:  <total>
   
   Per-file summary:
   - raw/article-one.md → sources/article-one.md (concepts updated: X, Y)
   - raw/paper-two.pdf  → sources/paper-two.md   (entities created: A, B)
   ...
   
   Suggested next step: /wiki query to explore connections, or /wiki stats for overview.
   ```

   Batch mode does **not** relax the rest of the single-file ingest workflow. You must still do source synthesis, concept/entity extraction, cross-linking, index updates, and log updates for every file. The only step skipped in batch mode is the pre-write discussion with the user.

   Never treat "source pages only" as a completed ingest. If the corpus is too large for full enrichment in one pass, say so explicitly and ask whether to:
   - ingest a smaller subset properly now
   - do a staged ingest and mark the remainder as partially processed
   - switch to a supervised workflow

   If you ever perform a degraded ingest for speed, you must record it as incomplete in `log.md`, state exactly what is missing, and offer the cleanup pass immediately. Do not present the ingest as complete.

8. **Select mode**: Parse the user's selection (e.g. "just the PDFs", "first 5", "everything except notes-*.md"), confirm the resolved list, then run supervised or batch mode on that subset (ask which mode if more than 5 files in the selection).

---

## Command: `/wiki ingest <path>`

Full ingest workflow for a single source file.

### Steps

1. **Read AGENTS.md** to load schema conventions before touching any wiki files.

2. **Read the source**
   - Text files (`.md`, `.txt`, `.html`): read directly.
   - PDFs: use the pdf-reading skill if available, otherwise note the limitation.
   - Images: read as image, extract visible text and context.

3. **Discuss key takeaways** — Before writing anything, briefly summarise:
   - What is this source about? (2–3 sentences)
   - What are the 3–5 most important facts, claims, or ideas?
   - Does anything contradict existing wiki pages?
   - Ask the user: "Anything specific you want emphasised or filed differently?"

4. **Write the source summary page**
   Path: `wiki/sources/<slugified-title>.md`
   
   Template:
   ```markdown
   ---
   type: source
   title: <title>
   date_ingested: <YYYY-MM-DD>
   source_file: raw/<filename>
   tags: []
   ---
   
   # <Title>
   
   ## Summary
   <2–4 paragraph summary in your own words>
   
   ## Key points
   - <point 1>
   - <point 2>
   - ...
   
   ## Connections
   - [[<related wiki page>]] — <why related>
   
   ## Open questions
   - <question raised by this source>
   ```

5. **Update entity and concept pages**
   - Scan the source for named entities (people, companies, products, places) and concepts.
   - For each: check if a page exists in `wiki/entities/` or `wiki/concepts/`.
   - If it exists: append new information under a `## From <source title>` section, update any contradicted claims.
   - If it doesn't exist: create it only if the entity/concept is substantive enough to warrant its own page (appears multiple times or is central to the source).
   - Aim to touch 5–15 pages per ingest. Don't create stub pages for every passing mention.
   - The source page's `## Connections` section must point to the main concept/entity/topic pages updated during ingest. Do not leave source pages connected only to other source pages unless that is genuinely the strongest link.
   - Repeated patterns across multiple new sources should be promoted into shared concept or topic pages during the same ingest run, not deferred indefinitely.

6. **Update index.md**
   Add a line to the appropriate section:
   ```
   - [[sources/<slug>]] — <one-line description> (<YYYY-MM-DD>)
   ```

7. **Append to log.md**
   ```
   ## [<YYYY-MM-DD>] ingest | <Source Title>
   - File: raw/<filename>
   - Pages created: <list>
   - Pages updated: <list>
   - Key topics: <comma-separated>
   ```

8. **Report back** — Tell the user:
   - How many pages were created vs updated
   - Any contradictions found with existing content
   - 1–2 suggested follow-up questions or related sources to find

---

## Command: `/wiki query`

Answer a question using wiki content as the primary source.

### Steps

1. **Clarify the question** if `/wiki query` is given with no argument. Ask: "What would you like to know?"

2. **Read index.md** to identify which pages are likely relevant. Do not read all pages — use the index as a map.

3. **Read relevant pages** — Pull the 3–8 most relevant pages based on the index scan. Prefer concept and entity pages over source summaries for synthesis questions; prefer source summaries for "what did I read about X" questions.

4. **Synthesise an answer** with citations to wiki pages:
   ```
   According to [[concepts/attention-mechanism]], ...
   This contradicts what [[sources/paper-2024]] argues, which holds that...
   ```

5. **Offer to file the answer** — If the answer is substantive (a comparison, a synthesis, a novel connection), offer:
   > "This looks worth saving. Want me to file this as `wiki/synthesis/<slug>.md`?"

   If yes, write a synthesis page:
   ```markdown
   ---
   type: synthesis
   title: <question as a title>
   date: <YYYY-MM-DD>
   sources: [<list of pages consulted>]
   ---
   
   # <Title>
   
   <Answer prose>
   
   ## Sources consulted
   - [[...]]
   ```
   Then update index.md and log.md.

---

## Command: `/wiki stats`

Show a dashboard of the current wiki state.

### Steps

```bash
WIKI=$WIKI_ROOT/wiki

# Page counts by type
echo "=== Page counts ==="
echo "Sources:   $(ls $WIKI/sources/ 2>/dev/null | wc -l)"
echo "Concepts:  $(ls $WIKI/concepts/ 2>/dev/null | wc -l)"
echo "Entities:  $(ls $WIKI/entities/ 2>/dev/null | wc -l)"
echo "Synthesis: $(ls $WIKI/synthesis/ 2>/dev/null | wc -l)"
echo "Total:     $(find $WIKI -name '*.md' | grep -v index | grep -v log | wc -l)"

# Recent activity
echo ""
echo "=== Recent log entries ==="
grep "^## \[" $WIKI/log.md | tail -10

# Raw sources not yet ingested
echo ""
echo "=== Raw files not yet ingested ==="
# List raw/ files, check each against log.md
for f in $(ls $WIKI_ROOT/raw/ | grep -v assets); do
  if ! grep -q "$f" $WIKI/log.md 2>/dev/null; then
    echo "  PENDING: $f"
  fi
done

# Orphan pages (in wiki/ but not linked from index.md)
echo ""
echo "=== Possible orphan pages ==="
find $WIKI -name '*.md' | grep -v index | grep -v log | while read page; do
  slug=$(basename $page .md)
  if ! grep -q "$slug" $WIKI/index.md 2>/dev/null; then
    echo "  ORPHAN: $page"
  fi
done
```

Present the output as a clean summary, not raw terminal output. Format:

```
Wiki stats — <date>

Pages        Sources: N  |  Concepts: N  |  Entities: N  |  Synthesis: N  |  Total: N
Activity     Last ingest: <title> (<date>)
             Last query: <title> (<date>)
Pending      <N> raw files not yet ingested: <list>
Orphans      <N> pages not in index: <list> (offer to fix)
```

If orphans are found, offer: "Want me to add the orphans to index.md?"

---

## Error handling

| Situation | Response |
|-----------|----------|
| No wiki root found | Ask for path before proceeding |
| Source file not found | List what's in `raw/` and ask to confirm |
| AGENTS.md missing | Warn user and proceed with defaults; suggest creating a AGENTS.md |
| index.md missing | Create it with minimal structure, then proceed |
| log.md missing | Create it with an init entry, then proceed |
| PDF or binary file | Note limitation, extract what's possible, ask user to paste key text if needed |

---

## Style rules for wiki pages

- All pages use `[[wikilinks]]` for cross-references (Obsidian-compatible).
- YAML frontmatter on every page (type, title, date).
- Sentence case for headings.
- No raw URLs in body text — use `[anchor text](url)` or file under sources.
- Keep pages focused — if a concept page grows beyond ~200 lines, suggest splitting.
