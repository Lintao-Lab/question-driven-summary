---
name: question-driven-summary
description: Produce a question-driven summary of an academic or professional document (case, journal article, chapter, report, thesis) organised around the user's current research question — not a generic abstract. Use when the user asks for a research-focused / question-driven summary of a document, or wants a document digested "for the research" rather than in full. Runs entirely on subagents with independent contexts; no scripts, no API keys.
---

# Question-Driven Summary

Turn one source document into a structured summary organised around the **user's
current research question**. The research question is the lens: the same document
yields a different summary for a different question.

This skill is pure orchestration. There is **no code to run** — you (the main agent) are the
orchestrator, subagents with independent contexts do all the reading and summarising,
and a shell `cat` does the assembly. Requirements: file read/write tools, a shell, and
a subagent/task tool.

## Why this shape

- **Context economy:** the full document never enters your context. The architect reads
  it; each summariser reads it; you work only with the research question, the segment
  list, and short completion reports.
- **Independent contexts:** every summariser sees the **full document** plus its own
  narrow assignment, so it always knows where its segment sits in the whole — without
  sibling summaries contaminating its reading.
- **Mechanical assembly:** merging is `cat` in a fixed order. Never send the collected
  segments to an LLM for a "merge" or rewrite pass.

## Workflow

Run directory: `workspace/runs/<date>_<slug>/` (or the project's convention). All paths
handed to subagents must be absolute.

### 1. Intake

- Capture the **research question** from the user. If the user wants a pure summary
  with no presupposed question, use a neutral research focus that states exactly that —
  never invent a focus.
- Note the document path and judge the archetype: `case` (judicial/arbitral decision),
  `article` (journal article, book chapter), or `generic` (report, thesis, white paper,
  anything else).
- Create the run directory with a `stages/` subdirectory.

### 2. Architect (one subagent)

Build a prompt from `prompts/architect.md` (fill research focus, document path,
archetype, max segments — default 10) and launch ONE subagent. It reads the full
document in its own context and returns strict JSON:

```json
{"doc_type": "...", "architecture_note": "...",
 "issues": [{"title": "...", "location": "...", "reasoning_structure": ["..."]}]}
```

Save it verbatim as `<run dir>/segments.json`. If the returned JSON is malformed or the
segmentation is clearly wrong (e.g. a case whose titles are not "Whether ..."
questions), relaunch the architect with a correction note rather than hand-fixing —
but hand-editing `segments.json` is acceptable when the user wants a specific cut.

### 3. Fan-out (one subagent per segment, all in parallel)

For each issue in `segments.json`, build a prompt from `prompts/summariser.md` and
launch ALL of them **in a single parallel batch** (multiple subagent calls in one
message). Each prompt carries: research focus, document path, its segment's title /
location / reasoning_structure, and its output path
`<run dir>/stages/NN_slug.md` (NN = zero-padded index, slug = short lowercase hyphenated
title). Each summariser reads the full document, writes its own stage file, and returns
only a 2–3 line completion report — do not ask it to return the summary text.

After the batch: verify every expected stage file exists and is non-trivial. Relaunch
only the missing/failed segments.

### 4. Introduction (you write this)

Write `<run dir>/intro.md` yourself — 5–10 lines covering, in order:

1. What the document is and its posture or context (from the architecture_note).
2. Why it matters for the research focus.
3. Global-position synopsis: which segments are covered below, what is left out, and
   how the covered parts fit the whole.

You have everything needed (research question + architecture_note + segment list);
do not launch another subagent and do not read the full document for this.

### 5. Assemble (shell, not LLM)

Build `summary.md` from `templates/summary-template.md`: header (citation, source,
date, research focus, AI-generated disclaimer), Introduction, architecture note, then
one section per segment — segment title as heading, *Location* line, italic
guidance-only reasoning-structure bullets, then the stage file body — in
`segments.json` order. Generate per-segment scaffolding with a shell loop or your file
tools and concatenate with `cat`. If a summariser added its own heading, strip that
line. Do not otherwise edit, "smooth", or rewrite segment bodies.

### 6. QC (you, with search tools — deterministic, no LLM judgement)

- Extract every verbatim quote (quoted spans of 3+ words, blockquotes) from each stage
  file and search for it in the source document, whitespace-insensitively. For quotes
  containing ellipses or editorial brackets, check each retained fragment separately.
- A quote not found usually means an OCR artefact or typo in the *source* — check the
  source around the anchor before concluding the summariser hallucinated.
- Spot-check a couple of location anchors against the source.
- Append a QC footer to `summary.md`: `PASS`/`ISSUES`, quotes verified, unresolved
  items listed.

### 7. Delivery

Place `summary.md` where the user or project convention says (default: the project's
outputs directory). Run artefacts (`segments.json`, `intro.md`, `stages/`) stay in the
run directory.

## Multiple documents

Run one chain per document, each in its own run subdirectory. Chains are independent —
run them sequentially, or parallelise by delegating whole chains to subagents when the
host supports it.

## Failure handling

- Architect returns bad JSON → relaunch with a correction note.
- A summariser fails or its stage file is missing/thin → relaunch just that segment;
  the rest of the run is untouched.
- Never paper over a failed segment by writing its summary yourself from other
  segments' reports — you have not read the document.
