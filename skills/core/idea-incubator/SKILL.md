---
name: idea-incubator
description: A meta-skill for taking raw ideas from "Grill" to "Living HTML Prototype" to "Architecture Plan."
---

# Idea Incubation Meta-Skill

Use this skill when the user presents a new project idea or wants to develop a concept within the **Brain** vault.

## Phase 1: The Grill
1.  **Activate**: Invoke `skills/core/grill-me/SKILL.md`.
2.  **Workflow**: Ask one pointed question at a time to resolve the design tree. Provide an expert recommendation with each question.
3.  **Completion**: Continue until a shared understanding is reached.

## Phase 2: The Living Prototype
1.  **Initial Design**: Create a high-fidelity, animated HTML/CSS/JS page in `wiki/prototypes/project-name.html`. Use `skills/frontend/claude-frontend-skill` for premium aesthetics.
2.  **Iterative Discussion**: As the user provides feedback, surgically update the `.html` file.
3.  **Evolution**: The page must reflect the "vibe" and core functionality of the idea.

## Phase 3: Architecture Planning
1.  **Activate**: Invoke `skills/engineering/improve-codebase-architecture`.
2.  **Synthesis**: Create a formal architectural plan in `wiki/synthesis/project-name-plan.md`.
3.  **Content**: Include executive summary, design decisions, roadmap, and necessary skills from the `skills/` library.

## Logging & Maintenance
- Log the start of an incubation as `maintenance | Idea Incubation: [Project Name]`.
- Update the `wiki/synthesis/idea-incubator.md` hub page.
- Ensure the final architecture is cross-linked with the Brain's existing `concepts/`.
