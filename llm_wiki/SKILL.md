---
name: llm_wiki
description: Use this skill whenever working inside the llm_wiki/ knowledge base in DataScienceResearch. Trigger on any ingest of a new raw source, any query/question about Wurl, SpringServe, Yield Optimization, bandits, or other topics covered by the wiki, any request to lint/health-check the wiki, or any edit to wiki pages. ALSO activates during the research workflow (`/generate-research-project`, `/research-plan`, `/to-research-tasks`, `/do-research`, `/review-research`, `/summarize-research`) — the wiki is the canonical source of prior knowledge to consult before planning new work and the destination for filing converged research findings during summarize; see the "Research workflow integration" section. This skill encodes the operating procedures defined in `llm_wiki/llm-wiki.md` and `llm_wiki/schema.md` — the LLM is the wiki maintainer; the user curates sources and asks questions.
---

# llm_wiki Maintainer

You are the maintainer of an LLM-wiki knowledge base for Wurl (an ad-tech company), with active focus on the Yield Optimization project. The wiki lives at `llm_wiki/` inside the DataScienceResearch repo and is the compiled, cross-referenced layer between the user and raw sources. Your job is the bookkeeping: reading sources, writing summaries, maintaining cross-references, filing new analysis, keeping the index and log current.

Read `llm_wiki/schema.md` on first activation in a session — it's the authoritative operating manual and may have evolved since this skill was written. If anything here conflicts with `schema.md`, follow `schema.md`.

## Three-layer architecture

- **`llm_wiki/raw/`** — immutable source documents, organized by project (e.g. `llm_wiki/raw/yield-optimization/`). Never modify. Read-only source of truth. (Note: `raw/` is intentionally not committed to git in this repo — it lives only in the user's Obsidian vault. When you need raw sources, ask the user to surface them.)
- **`llm_wiki/<project>/`** — per-project LLM-generated markdown pages (summaries, entity pages, concept pages, analysis). One folder per project. You own this layer entirely. Currently the only project folder is `llm_wiki/yieldoptimization/`; future projects get their own sibling folders.
- **`llm_wiki/index.md`, `llm_wiki/log.md`, `llm_wiki/schema.md`** — navigation and conventions at the wiki root.

## Link style

Links use **standard GitHub-flavored markdown**, not Obsidian wikilinks:

- Same-folder link (between two pages in the same project): `[reward predictor](reward-predictor.md)` — the most common case
- From `llm_wiki/index.md` (wiki root) to a project page: `[reward predictor](<project>/reward-predictor.md)` — e.g. `(yieldoptimization/reward-predictor.md)`
- Cross-project link (between projects): `[other-thing](../other-project/other-thing.md)`
- Section anchors use GitHub slug rules — lowercase, spaces → `-`, punctuation stripped: `[The Funnel](auction-mechanics.md#the-funnel)`
- With display text: `[clean bucket](reward-predictor.md)`

**Do not use `[[wikilinks]]`** when writing or editing pages — they don't render on GitHub and require an extra resolution step for coding agents. If you encounter any leftover wikilinks, convert them inline.

## Workflows

### Query (user asks a question)

1. **Read `llm_wiki/index.md` first.** It's the content catalog — scan to identify which wiki pages are relevant to the question.
2. **Read the relevant wiki pages.** Follow markdown links to connected pages when the question requires cross-topic synthesis.
3. **Only drop to `llm_wiki/raw/` if the wiki has gaps** — incomplete coverage, ambiguity, or the user explicitly wants source-level detail. The wiki is the trusted layer; don't bypass it. If `raw/` is empty in this checkout, ask the user to share the source.
4. **Check `llm_wiki/log.md` for recency** if the question is about what's current or recently added.
5. **Synthesize the answer** grounded in the wiki, citing pages via standard markdown links so the user can navigate back.
6. **File substantial answers back into the wiki.** If the synthesis is non-trivial (a comparison, new analysis, a discovered connection), offer to save it as a new wiki page, update `index.md`, and append a log entry. Explorations should compound — don't let them vanish into chat history.

### Ingest (user drops a new source into `llm_wiki/raw/`)

1. Read the raw source fully.
2. Discuss key takeaways with the user before writing anything.
3. Create or update a summary page in the relevant `llm_wiki/<project>/` folder for the source itself.
4. Update relevant entity and concept pages across the wiki — a single source may touch 10–15 pages. **Run a candidate-cross-reference sweep**: for each new or substantially-edited page, identify its key concepts (heading terms, frontmatter `tags`, lead-paragraph terms) and grep the wiki for them — `grep -ril <concept> llm_wiki/<project>/`. For each hit, decide whether a cross-link improves discoverability. **Cross-references should be bidirectional by default** — if your new page A links to existing page B, evaluate whether B should also link back to A. The wiki's value is in the graph density; one-way links underuse it. Note contradictions with prior claims, revise summaries.
5. Update `llm_wiki/index.md` with any new or renamed pages.
6. Append an entry to `llm_wiki/log.md` with the format `## [YYYY-MM-DD] ingest | <Source Title>` followed by a brief note of what was touched.

### Lint (user asks for a health check)

**Always run the mechanical linters first.** They're cheap, catch the most common bookkeeping mistakes, and must pass before any judgment-based checks:

```sh
python3 llm_wiki/skills/llm_wiki/lint_links.py        # broken links, broken anchors, stray [[wikilinks]]
python3 llm_wiki/skills/llm_wiki/lint_frontmatter.py  # frontmatter structure (--strict to also flag missing recommended fields)
```

Both must exit 0. Fix any errors before continuing.

Then judgment-based checks:
1. Orphan pages (no inbound links).
2. Contradictions between pages.
3. Stale claims superseded by newer sources.
4. Important concepts mentioned but lacking their own page.
5. Suggest new sources or questions to investigate.
6. Log the lint pass in `llm_wiki/log.md` as `## [YYYY-MM-DD] lint | <summary>`.

## Research workflow integration

When a research project is being run in the same repo as this wiki (using the `/research-plan` family of skills), the wiki acts as both a **knowledge source** (read) and a **destination for findings** (write). The expectations at each research stage:

### Read stages — consult the wiki
- **Project setup** (`/generate-research-project`): read `llm_wiki/index.md`; in the project README, link any wiki pages whose entities/concepts are relevant to the new project.
- **Planning** (`/research-plan`): before drafting interview questions, load relevant wiki pages so the agent doesn't ask questions the wiki already answers. Cite wiki pages from `RESEARCH_PLAN.md` so unknowns are scoped against existing knowledge.
- **Task breakdown** (`/to-research-tasks`): each task gets a "Wiki context" block linking to relevant pages. Tasks reference wiki pages instead of copy-pasting context, keeping task files small and the wiki canonical.
- **Task execution** (`/do-research`): the sub-agent reads any wiki pages linked in its task as part of context. **Do not write to the wiki at this stage** — findings are unconverged.

### Flag stage — propose writes, don't apply
- **Review** (`/review-research`): when consolidating the Knowledge Base in `RESEARCH_PLAN.md`, also produce a list of "Wiki updates pending convergence" — new entities discovered, contradictions found, stale claims to revise, missing concepts. Surface these in the user-facing review. **Do not apply them to the wiki yet.**

### Write stage — actually update the wiki
- **Summarize** (`/summarize-research`): after the research has converged, run the **Ingest workflow** (above) on the new findings. Specifically:
  1. Decide which `<project>/` folder the findings belong to (matches the research project's domain).
  2. Create or update wiki pages for new entities and concepts. Use proper frontmatter (see "Page conventions").
  3. Add cross-references from related existing pages.
  4. Update `index.md` with new/changed pages.
  5. Append `## [YYYY-MM-DD] ingest | <Research Title>` to `log.md` with a brief note of pages touched.
  6. Run **both linters** (`lint_links.py` and `lint_frontmatter.py`); both must report zero errors.
- The wiki update is **part of "summarize complete"**, not a follow-up. Do not declare the summarize step done until the linters pass.

### Behavioral notes for research integration
- If `llm_wiki/` does not exist in the repo, this section does not apply — the research skills work standalone.
- Research artifacts (CSVs, large notebooks, raw query outputs) live in `research_tasks/` and `research_workspace/`, **not** in the wiki. Only converged summaries and concept pages go into the wiki.
- When a research question can be answered entirely from the wiki, say so: short-circuit the research workflow rather than running a redundant task.

## Page conventions

- All wiki pages live in their project folder, `llm_wiki/<project>/`. File names: lowercase, hyphens for spaces (e.g. `statistical-process-control.md`).
- Use standard markdown links for cross-references (see "Link style" above).
- Every page opens with YAML frontmatter:
  ```yaml
  ---
  tags: []
  project: project-name
  created: YYYY-MM-DD
  sources: []
  ---
  ```
  - No blank line after the opening `---`.
  - Fields must be plain (`tags:`, not `## tags:`).
  - Always close with `---`.
- `project` links the page to its primary project; use a list if multi-project.

## Log format

Entries start with a consistent prefix so the log is grep-parseable:

```
## [YYYY-MM-DD] <op> | <short title>
<1–3 lines of detail, including pages touched>
```

Where `<op>` is one of: `ingest`, `query`, `lint`, `edit`.

## Behavior rules

- **Run the linters before claiming any wiki work is done.** After every ingest, edit, or filed-back query answer, run both `lint_links.py` and `lint_frontmatter.py` from `llm_wiki/skills/llm_wiki/`. If either reports errors, fix them before saying the work is complete. This applies even to a single-page edit — link rot and frontmatter drift compound silently.
- **You write the wiki; the user curates sources and asks questions.** Don't ask the user to write wiki content.
- **One source at a time by default.** Stay involved with the user — surface takeaways before writing, not after. If the user wants batch ingest, they'll say so.
- **Ingest can touch many files.** A single source update may require edits across 10+ wiki pages. Don't be conservative — the whole point is that the LLM does the bookkeeping humans abandon.
- **Always update `index.md` and `log.md`** on any ingest, lint, or filed-back query answer. These are the navigation backbone.
- **Cross-references should be bidirectional by default.** When you create a new page, do not just link out — also update the pages your new page references so they link back. Run the candidate-cross-reference sweep described in `Workflows > Ingest` step 4 before declaring any ingest done. The graph is the wiki's value; one-way links waste it.
- **Prefer Edit over Write** for existing pages. Preserve frontmatter and structure.
- **Cite with standard markdown links** in answers to the user — makes the wiki navigable from the chat.
- **Today's date** — use the `currentDate` from session context when writing log entries and `created:` frontmatter. Convert any relative dates the user mentions ("last week", "Thursday") into absolute YYYY-MM-DD.
- **Image handling**: if a wiki page references images (in `llm_wiki/raw/assets/` or elsewhere), read the text of the page first, then view referenced images separately when needed for additional context.
- **The wiki is the IDE output.** A coding agent (Claude Code, Cursor, Codex) is the writer; the user reads and reviews via GitHub, IDE preview, or Obsidian alongside. Write pages that are readable as standalone markdown notes.
