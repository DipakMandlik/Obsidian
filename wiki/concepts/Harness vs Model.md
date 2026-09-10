---
title: Harness vs Model
type: concept
created: 2026-09-09
updated: 2026-09-09
tags: [ai-agents, llm, agentic-systems, tooling]
---

# Harness vs Model

A core distinction in modern AI products: the **model** and the **harness** (also called the scaffold, agent runtime, or agentic framework) are two separate layers, built by different people, upgraded on different schedules, and responsible for different failures.

- **Model**: the raw [[LLM]] weights (e.g. Claude Opus, GPT-5, Gemini). Stateless — takes a sequence of tokens in, predicts tokens out. No memory between calls, no tools, no internet, no file system. Everything it "knows how to do" is pattern-completion learned during training.
- **Harness**: the surrounding software that turns that stateless text-predictor into a usable product. It manages the conversation loop, decides what goes into the context window each turn, exposes and executes tools, enforces safety rules, retries failures, and renders output to the user.

Same model + different harness = different product. Same harness + different model = same product, smarter or dumber.

## Division of responsibility

| Capability | Who actually does it |
|---|---|
| Writing code, reasoning, drafting text | **Model** |
| Remembering earlier turns in a chat | **Harness** — re-sends prior messages/summaries as context each call; model has no persistent memory |
| Running a bash command, editing a file, browsing the web | **Harness** — executes the action and feeds the result back in; the model only ever "requests" a tool call as text |
| Reducing hallucination with fresh facts (RAG) | **Harness** — retrieves documents and injects them into context; model just conditions on what it's given |
| Rate limiting, retries on API errors, prompt caching | **Harness** |
| Content/safety filtering beyond the model's own training | **Harness** — extra classifiers, blocklists, guardrails |
| Multi-step planning, subagents, verification loops | **Harness** — orchestrates multiple model calls; each individual call is still a single stateless prediction |

## Real-world examples

1. **Claude Code vs claude.ai vs the Claude API** — all three can run on the identical Claude model, but Claude Code's harness gives it a filesystem, bash, subagents, and hooks; claude.ai's harness gives it a chat UI, memory, and artifacts; the raw API gives it nothing but a text-in/text-out endpoint. The model didn't change the harness did, and the capabilities look completely different.
2. **GitHub Copilot / Cursor / Windsurf**, all frequently built on the same underlying GPT/Claude model family, produce noticeably different coding-assistant quality. The difference is harness engineering: how much of the repo gets pulled into context, whether there's a plan-then-edit loop, whether edits are verified by running tests before being shown to the user.
3. **"The AI remembered what I said yesterday"** — it didn't. The harness stored the previous conversation (or a summary of it) and pasted it back into today's prompt. The model re-read it fresh, same as if you'd pasted it yourself.
4. **Hallucinated citations fixed by RAG** — the model itself didn't get more truthful; the harness started retrieving real documents and grounding the answer in them before the model ever generates a token.
5. **A model upgrade (e.g. Sonnet 4 → Sonnet 4.5) shipping as a one-line config change** in a well-built harness, versus breaking a product that hard-coded assumptions about the old model's quirks — the difference is how tightly the harness is coupled to a specific model's behavior.

## Why the distinction matters

- When an AI product fails (wrong answer, forgot context, made up a fact, couldn't do something), the first diagnostic question is: *is this the model being wrong, or the harness failing to give the model what it needed?* Most user-visible "the AI is dumb" complaints are actually harness bugs — bad context selection, missing tools, no verification step.
- Model quality is commoditizing fast across vendors; harness quality (context management, tool design, verification loops) is where most durable product differentiation now lives.

## Related
- [[LLM]]
- [[AI Agent]]
- [[Model Context Protocol|MCP]]
- [[RAG]]
- [[Claude]]
- [[wiki/log]]
