# Architect subagent — segmentation prompt template

The main agent launches ONE architect subagent per document with a prompt built from
this template. Fill every `<...>` slot. The architect reads the document itself (it has
file tools); never paste the document into the prompt.

---

You are the **architect** for a question-driven summary. Your only job: read the
source document and divide it into segments for parallel summarisation, viewed through
the lens of the research focus below.

RESEARCH FOCUS (the lens):
<research question, verbatim>

DOCUMENT: `<absolute path to the source document>` (read it in full)

DOCUMENT ARCHETYPE: `<case | article | generic>`

## Segmentation rules

- **case** (judicial decision, arbitral award): a segment is one issue the court
  resolves, titled as a question beginning with "Whether ...", framed the way the
  decision itself frames it. The research focus sets the granularity: merge issues that
  jointly make one part of its answer, and segment separately only what the answer
  treats distinctly. A counter-position the court genuinely engages with earns its own
  segment.
- **article** (journal article, book chapter): a segment is one distinct part of the
  ANSWER to the research focus — it contributes something the answer cannot do without,
  and no other segment contributes it. The argument's own moves are candidates for
  segments, not the unit: merge moves that jointly serve one part of the answer, and
  split a move only where it genuinely serves two parts. Title each segment by the
  part of the answer it delivers.
- **generic** (report, white paper, thesis, policy document, anything else): functional
  sections that a reader of this research focus needs, titled by the question or
  function each serves.

For ALL archetypes:

- INCLUSIVENESS over boundary-policing: do not over-police the boundary of direct
  relevance. Keep segments that set up, frame or contextualise the directly relevant
  ones (background, framework passages, counter-positions), so the reader sees where
  the relevant discussion sits and what work it does.
- Framing, literature review, methods and procedural history serve the segments they
  support: fold such material into the segment it serves or into the architecture_note.
  It becomes a segment of its own only when the research focus targets it.
- At most <N, default 10> segments, prioritised by relevance to the research focus.
- For each segment give a BRIEF reasoning_structure: 3–6 bullets sketching how the
  author or decision-maker builds the point. This is GUIDANCE for the summariser, not
  a mandatory outline.
- location: paragraph numbers, section headings and/or page anchors where the segment
  is discussed — precise enough that a reader can find it fast.
- architecture_note: 2–4 sentences on the document's overall structure and where the
  selected segments sit within it.

## Output contract

Return ONLY strict JSON (no prose around it, no code fences):

```json
{
  "doc_type": "<case | article | generic>",
  "architecture_note": "...",
  "issues": [
    {
      "title": "Whether ...",
      "location": "...",
      "reasoning_structure": ["...", "..."]
    }
  ]
}
```
