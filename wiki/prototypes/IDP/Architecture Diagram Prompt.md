---
title: IDP Architecture Diagram — Master Prompt
type: prompt
created: 2026-09-07
updated: 2026-09-07
tags:
  - prototype
  - idp
  - prompt
  - architecture-diagram
---

# Master Prompt — IDP Layered Architecture Diagram

Use with an AI image generator (Midjourney, DALL·E, Gemini/Nano Banana) or hand to a designer as a PowerPoint/Figma brief. Paste as one block.

> [!info] Source of truth
> Content is pulled directly from [[wiki/prototypes/IDP/IDP.md]] — this is a literal rendering of the built pipeline, not a marketing mockup. No taglines, no "benefits" callouts, no advantage bullets — just the working model, labeled accurately.

```
Create a professional enterprise software architecture diagram, classical corporate consulting style
(McKinsey/Big-4 deliverable quality), in a clean CLASSICAL WHITE THEME.

CANVAS & LAYOUT
- Landscape, 16:9, high resolution, presentation-ready (fit for a client deck slide).
- Pure white background (#FFFFFF), no gradients, no shadows, no 3D effects, no glow.
- Content organized as a strict LEFT-TO-RIGHT LAYERED PIPELINE, six vertical layer bands,
  each band the full height of the canvas, separated by thin hairline dividers (#1A1A1A, 0.5pt).
- Two thin horizontal "rail" boxes below the main pipeline for supporting/config layers, spanning the full width.

TYPOGRAPHY
- Times New Roman (serif) for ALL text — headers, labels, captions, footnotes. No sans-serif anywhere.
- Layer title: 20pt bold, small caps, navy (#1F3864).
- Component labels: 12pt regular, black (#1A1A1A).
- Annotations/notes: 9pt italic, grey (#555555).
- No emojis, no decorative icons, no cartoon illustration style.

COLOR PALETTE (muted, corporate, restrained — no bright/neon colors)
- Primary accent: Snowflake blue (#29B5E8) — used only for connecting arrows and the Snowflake logo.
- Layer fills: very light neutral tones per layer (near-white greys/blues, e.g. #F7F9FB, #EEF2F7) —
  just enough contrast to separate bands, not colorful.
- Lines/borders: black or navy hairlines only.
- One warm accent (#D9822B, amber) used ONLY to mark the "Model/LLM lane" boxes, to visually flag
  the only parts of the system that cost money to run.

LOGOS / MARKS (small, top-right corner of the relevant layer only — do not oversize or center them)
- Snowflake logo (official snowflake mark) on the outer canvas frame, top-right corner, labeled
  "Built entirely on Snowflake."
- Generic minimal icons (thin-line, monochrome, Times New Roman caption underneath) for: PDF/document
  file, database cylinder, magnifying glass (search), robot/chip outline (AI/LLM), padlock (governance),
  and a person silhouette (BI/human consumer). No colorful icon packs, no flat-design style.

LAYER-BY-LAYER CONTENT (left to right, exactly this order, exactly this content — this is the real system)

1. LANDING
   - Icon: inbound document stack.
   - Label: "Document Registry — state machine, MD5 dedup, immutable source copy."
   - Sub-box: DOC_REGISTRY.

2. BRONZE
   - Icon: raw text page.
   - Label: "Parsed text, zero interpretation."
   - Sub-box: DOC_PARSED (page count, OCR flag).

3. SILVER — split this band into two parallel lanes joined by a routing diamond:
   - Routing diamond labeled "Router: structural fingerprint (50%) + regex anchors (40%) +
     source hint (10%) → score".
   - Lane A (blue fill, no amber): "DETERMINISTIC LANE — regex/template extraction — ~75% of
     documents — $0 LLM cost."
   - Lane B (amber accent box): "MODEL LANE — LLM extraction via Cortex — ~25% of documents —
     ~$0.0002/doc."
   - Both lanes converge into a "VALIDATE" box: REQUIRED / ARITHMETIC / RANGE / FORMAT checks,
     with a small looping arrow labeled "escalate to model, one retry" going from Lane A validation
     failure into Lane B, and a branch labeled "MANUAL QUEUE" for second failures.

4. GOLD
   - Icon: linked nodes / entity graph.
   - Label: "Entity Resolution — embeddings (snowflake-arctic-embed-m) → cosine similarity blocking
     → auto-merge above 0.97 / LLM adjudication 0.85–0.97 → transitive closure."
   - Sub-box: ENTITY_CLUSTERS, cross-document aggregates.

5. SERVE
   - Icon: three small connected boxes — "SQL Views", "Semantic View (Cortex Analyst)",
     "Search Index (Cortex Search)".
   - Label: "Read-only consumption layer — no business logic."

6. CONSUMPTION (rightmost, outside the pipeline bands)
   - Two terminal nodes: "BI / Analysts" (person icon) and "IDP Agent on Snowflake CoWork"
     (chip icon) — agent box notes "grounded, cites doc_ids, refuses when data not held."

SUPPORTING RAILS (thin horizontal strips beneath the six-layer pipeline, spanning full width,
connected upward with dotted lines to every layer above them — NOT part of the main flow)
- Rail 1, labeled "CONFIG (data, not code)": Templates · Prompts · Settings.
- Rail 2, labeled "OPS (built day one)": Run Log · Cost Ledger · Error Log · Watermarks.

FLOW ARROWS
- Solid navy arrows connecting LANDING → BRONZE → SILVER → GOLD → SERVE → CONSUMPTION.
- Thin dotted grey arrows from CONFIG and OPS rails up into every stage they govern/monitor.
- Do not use arrowheads with 3D or glossy styling — simple thin triangular heads only.

FOOTER
- Bottom-left, 9pt italic Times New Roman: "Snowflake-native · Streams & Tasks orchestration ·
  11-task DAG · 75% deterministic lane share."
- Bottom-right: small Snowflake logo mark + "IDP Platform — Pibythree Technologies".

STRICT EXCLUSIONS
- No marketing language, no "why this matters" callouts, no benefit/advantage text, no bullet lists
  of value propositions, no customer logos other than Snowflake, no photographic elements, no people
  photos, no dark mode, no rounded "bubbly" shapes — rectangular boxes with sharp or lightly-rounded
  (2px radius max) corners only. This must read as an accurate engineering diagram of a working
  system, not a sales slide.
```

## Notes on use

- If your image tool truncates long prompts, split at the `LAYER-BY-LAYER CONTENT` heading and feed as two prompts, then composite.
- For a text-native diagram instead of an AI image (sharper for a real client deck), give this same content to a designer as a brief for PowerPoint/Figma/Visio — the section headers above map directly to slide/frame instructions.
- Full technical detail behind each layer lives in [[wiki/prototypes/IDP/IDP.md]] — pull exact numbers from there if the diagram needs updating after a deploy.
