# Brain — LLM Wiki Schema

This is an LLM-maintained personal knowledge base. The LLM writes and maintains the wiki; the human curates sources, asks questions, and guides synthesis.

## Directory Structure

```
Brain/
  sources/       # Raw source documents (immutable — never modify)
  wiki/          # LLM-generated markdown pages (LLM owns this entirely)
  CLAUDE.md      # This file — schema and conventions
```

## Wiki Conventions

### Page Types

- **Source summaries** (`wiki/sources/`): One page per ingested source. Contains metadata, key takeaways, and links to entity/concept pages it touches.
- **Entity pages** (`wiki/entities/`): Pages for people, organizations, places, products — anything with a proper name.
- **Concept pages** (`wiki/concepts/`): Pages for ideas, frameworks, theories, recurring themes.
- **Synthesis pages** (`wiki/synthesis/`): Higher-order pages — comparisons, analyses, evolving theses, answers to big questions.
- **index.md** (`wiki/index.md`): Content catalog of every wiki page, organized by category.
- **log.md** (`wiki/log.md`): Append-only chronological record of all operations.

### Formatting Rules

- Use standard markdown with `[[wikilinks]]` for internal links (Obsidian-compatible).
- Every page should have a YAML frontmatter block with at minimum: `title`, `type`, `created`, `updated`.
- Use `tags:` in frontmatter for discoverability.
- Prefer short, clear page titles. Use the entity/concept's most common name.
- Filenames should match titles in lowercase-kebab-case (e.g., `reinforcement-learning.md`).

### Cross-Referencing

- When creating or updating a page, add `[[wikilinks]]` to every entity and concept that has its own page.
- If an entity or concept is mentioned but doesn't have a page yet, still wikilink it — it becomes a candidate for page creation during lint.
- At the bottom of each page, include a `## Related` section with the most important links.

## Operations

### Ingest

When the user adds a new source and asks to ingest it:

1. **Read** the source document in full.
2. **Discuss** key takeaways with the user — what's interesting, what to emphasize.
3. **Create** a source summary page in `wiki/sources/`.
4. **Create** new entity/concept pages for anything significant that doesn't have one yet.
5. **Update** existing entity/concept pages with new information from this source. Note where new data supports, extends, or contradicts existing content.
6. **Update** `wiki/index.md` with entries for any new pages.
7. **Append** to `wiki/log.md` with a summary of what was done.

### Query

When the user asks a question:

1. **Read** `wiki/index.md` to find relevant pages.
2. **Read** the relevant wiki pages.
3. **Synthesize** an answer with citations to wiki pages and original sources.
4. If the answer is substantial and reusable, **offer to file it** as a new synthesis page in the wiki.

### Lint

When the user asks for a health check:

1. Scan for **contradictions** between pages.
2. Find **orphan pages** (no inbound links from other pages).
3. Identify **red links** (wikilinks pointing to pages that don't exist yet) — suggest which to create.
4. Check for **stale content** that newer sources may have superseded.
5. Look for **missing cross-references** between related pages.
6. Suggest **questions to investigate** or **sources to find** that would fill gaps.
7. Log the lint pass in `wiki/log.md`.

## Log Format

Each entry in `wiki/log.md` uses this format for parseability:

```
## [YYYY-MM-DD] operation | Title

Description of what was done.
- Pages created: [[page1]], [[page2]]
- Pages updated: [[page3]]
```

Operations: `ingest`, `query`, `lint`, `synthesis`, `maintenance`.
