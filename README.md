# Question-Driven Summary

An agent skill that turns one source document into a structured summary organised
around **your current research question**, not a generic abstract.

The research question is the lens: the same document yields a different summary for a
different question.

## What it does

Given a document (judicial decision, journal article, book chapter, report, thesis)
and a research question, the skill produces a Markdown summary with:

- an introduction explaining what the document is and why it matters *for your
  question*;
- a document-architecture note showing where the relevant discussion sits in the whole;
- one section per segment — legal issues titled as "Whether …" questions for cases,
  semantic/functional sections for articles — each with precise location anchors,
  the author's reasoning structure, and 1–3 verified verbatim anchor quotes;
- an automated QC footer confirming every quote against the source.

## How it works

This skill is **pure orchestration: no scripts, no API keys, no model names**. Any AI
coding agent with file tools, a shell, and a subagent/task tool can run it.

1. **Architect** — one subagent reads the full document in its own context and returns
   a segmentation (segments + guidance-only reasoning structures + architecture note).
2. **Fan-out** — one subagent per segment, all in parallel. Each reads the **full
   document** (so it knows where its segment sits in the whole), summarises only its
   assignment, and writes its own output file.
3. **Introduction** — written by the orchestrating agent itself.
4. **Assembly** — a shell `cat` in a fixed order. No LLM ever "merges" or rewrites the
   collected segments.
5. **QC** — the orchestrator verifies every verbatim quote against the source with
   deterministic search, not LLM judgement.

The design goals are context economy (the full document never enters the orchestrator's
context), independent reading per segment (no cross-contamination between summarisers),
and mechanical, auditable assembly.

## Usage

Install like any agent skill: copy this directory into your agent's skills path
(e.g. `.agents/skills/question-driven-summary/` or `.claude/skills/question-driven-summary/`).

Then ask your agent something like:

> Use the question-driven-summary skill to summarise `materials/judgment.md` for my research question:
> "How do courts treat implied terms in informal commercial arrangements?"

The agent runs the chain and delivers `summary.md`; intermediate artefacts
(`segments.json`, per-segment stage files) stay in the run directory for audit.

## Requirements

- An AI agent host with: file read/write tools, a shell, and a subagent/task mechanism
  (e.g. Claude Code's Task tool, Kimi Code's Agent tool).
- No Python, no dependencies, no API keys.

## Licence

MIT — see [LICENSE](LICENSE).
