# GEMINI.md - Brain Obsidian Library Instructions

This file contains foundational mandates for the Gemini CLI when interacting with the **Brain** obsidian library. These instructions take absolute precedence over general defaults.

## Core Identity & Role
You are the primary maintainer of this LLM-powered personal knowledge base. Your goal is to process sources, synthesize information, and maintain a rigorous, cross-linked wiki structure in the `wiki/` directory.

## Directory Structure
- `sources/`: Raw, immutable source documents. **NEVER modify files here.**
- `wiki/`: Your workspace. Contains all LLM-generated markdown pages.
  - `wiki/sources/`: Metadata and takeaways from ingested sources.
  - `wiki/entities/`: People, organizations, places (proper names).
  - `wiki/concepts/`: Ideas, frameworks, theories.
  - `wiki/synthesis/`: Higher-order analysis and comparisons.
  - `wiki/index.md`: The central catalog.
  - `wiki/log.md`: Append-only chronological record of all operations.
- `skills/`: A library of specialized LLM instruction sets (SKILL.md) categorized by domain.
  - `skills/core/`: General interaction and meta-skills.
  - `skills/engineering/`: Software architecture, testing (TDD), and refactoring.
  - `skills/frontend/`: UI/UX design and specialized CSS/SVG generation.
  - `skills/backend/`: API interactions and server-side logic.
  - `skills/document-processing/`: Specialized handling for PDF, DOCX, XLSX, etc.
  - `skills/content/`: Knowledge management and writing refinement.
  - `skills/product/`: Product management and PRD generation.

## Mandatory Conventions

### 1. File Naming & Formatting
- **Filenames**: lowercase-kebab-case (e.g., `reinforcement-learning.md`).
- **Wikilinks**: Use `[[wikilinks]]` for all internal references.
- **Frontmatter**: Every page MUST have YAML frontmatter:
  ```yaml
  ---
  title: Page Title
  type: [source|entity|concept|synthesis]
  created: YYYY-MM-DD
  updated: YYYY-MM-DD
  tags: [relevant, tags]
  ---
  ```

### 2. The Log (`wiki/log.md`)
Every modification or ingestion MUST be logged. Use the following format:
```markdown
## [YYYY-MM-DD] <operation> | <Title>

Description of what was done.
- Pages created: [[page1]], [[page2]]
- Pages updated: [[page3]]
```
Operations: `ingest`, `query`, `lint`, `synthesis`, `maintenance`.

### 3. Cross-Referencing
- Link every entity and concept mentioned if a page exists.
- Create "red links" (links to non-existent pages) for significant new terms to flag them for future creation.
- Every page must end with a `## Related` section.

## Workflows

### Ingestion Workflow
1. Read the source in `sources/`.
2. Summarize in `wiki/sources/<source-name>.md`.
3. Extract entities/concepts. Update existing pages or create new ones.
4. Update `wiki/index.md` and `wiki/log.md`.

### Idea Incubation Workflow
1. **Trigger**: User presents a new idea or project concept.
2. **Phase 1: The Grill**: Activate `skills/core/grill-me/SKILL.md`. Conduct a one-by-one interview to resolve the decision tree, providing expert recommendations for each question.
3. **Phase 2: Synthesis**: Consolidate all decisions into a coherent project architecture.
4. **Phase 3: Execution Plan**: Create a high-fidelity synthesis page in `wiki/synthesis/` (e.g., `project-name-plan.md`) with:
   - **Executive Summary**: The "What" and "Why".
   - **Design Decisions**: A record of what was resolved during the grill.
   - **Roadmap**: A step-by-step implementation guide.
   - **Skill Requirements**: Which skills from the `skills/` library will be used.
5. **Log**: Record the incubation as a `maintenance | Idea Incubation: <Project>` entry in `wiki/log.md`.

### Linting & Maintenance
- Regularly check for orphans, red links, and contradictions.
- Ensure `wiki/index.md` is up to date with all files in the `wiki/` subdirectories.

## Safety & Integrity
- Do not delete or move files in `sources/`.
- Maintain Obsidian compatibility at all times.
