---
name: deep-learning-textbook
description: Use when the user wants to deeply learn a topic and asks for a rigorous, evidence-based, independently-verified learning textbook (single-file HTML+mermaid) — e.g. "深度学习/吃透/0→100/a textbook on X". Use when a single-pass explanation would be too shallow, unaudited, or lossy for a complex or high-stakes topic. Do NOT use for quick factual answers or single-concept explainers.
---

# Deep-Learning Textbook

## Overview
Produce a rigorous learning textbook via a **multi-agent pipeline**, not a monologue. Core principle: **a single agent is never enough for a non-trivial topic** — fan out research, verify with INDEPENDENT agents (never self-review), enforce a consistent auditable format, and persist the result.

## When to Use
- User says "深度学习 / 吃透 / 0→100 / 给我一套教材" on a topic.
- Topic is complex/multi-faceted/high-stakes (single-agent attention will fail).
- User wants a reusable artifact (HTML they re-open, with diagrams + evidence).
- NOT for: quick facts, single-concept explainers, conversational answers.

## The Pipeline (mandatory phases)
1. **Research (fan-out)** — N parallel subagents, one per facet/source. Beats single-agent attention ceiling. Each returns findings with citations / `file:line`.
2. **Adversarial verify (INDEPENDENT)** — separate verifier agents, default-to-skeptical, check claims against sources/code. **Self-review never counts as verification** (you miss your own errors).
3. **Synthesize** — one writer → single-file HTML + mermaid, applying Format Standards + cognitive contract.
4. **Expert review (multi-round)** — 2–3 expert personas (educator + architect + domain lead) critique; chief meta-review consolidates; writer applies.
5. **Persona launch-review** (for PRDs/proposals) — user-persona agents ask real review-meeting questions.
6. **Quality scan** — per-book: validate every mermaid block (mmdc/Playwright if available; else strict rule-check), fix TOC anchors, fix syntax. **Surgical edits, never full rewrite.**
7. **Commit + memory** — git commit the HTML; update memory (progress, ability map) so the next session continues.

## Format Standards (non-negotiable)
- **Single self-contained HTML** — one `.html`, minimal inline `<style>`, no external CSS, no packaging.
- **Mermaid via CDN** — `<script src="https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.min.js"></script>` + `<pre class="mermaid">`. Validate (see pipeline-templates.md).
- **Evidence** — `file:line` or source URL for every non-trivial claim.
- **`<aside class="note">`** difficulty annotations for the reader's cognitive style.
- **`<blockquote class="why">`** rationale + 反事实 (counter-factual) for key decisions.
- **TOC + anchors** — every `href="#..."` must resolve to an `id="..."`.

## Cognitive Contract (match the learner)
Model-first / cause-before-terminology / two-pass (skeleton then detail) / `introduced→practiced→can-decide` grading / on overload externalize to four truths (intent/repo/artifact/runtime). If a `learning-contract` memory/doc exists, read it first.

## Iron Law — "Done" is enforced, not claimed
A textbook is NOT done until ALL of:
- ✅ Independent verifier(s) signed off (not self-review).
- ✅ Quality scan passed (mermaid renders, TOC resolves).
- ✅ Committed + memory updated.
**No exceptions** for "it's basically done" / "I'll verify later" / "self-review caught it".

## Common Rationalizations — STOP
| Excuse | Reality |
|---|---|
| "I'll explain it in one pass" | Single agent hits the attention ceiling on big topics → shallow/wrong. Fan out. |
| "Self-review / self-adversarial is enough" | You miss your own errors. Independent verifier required. |
| "Prose is clearer than diagrams" | Model-first learners need structure. Mermaid is mandatory. |
| "Doc is done after writing" | No — review + quality scan + commit are part of "done". |
| "Advisory principles suffice" | Enforce (independent verify gate), don't just advise. |
| "Truncate to fit" | Write to a FILE (Write tool) — never truncate; large output auto-persists. |

## Reference
- `pipeline-templates.md` — workflow script skeleton, HTML `<style>` boilerplate, mermaid syntax rules, expert/persona schemas.
- Worked examples: `FunctuonOwnerLearning/textbooks/*.html` (A–L).
