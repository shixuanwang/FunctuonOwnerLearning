# Pipeline Templates (deep-learning-textbook)

Heavy reference for `deep-learning-textbook`. Not loaded unless implementing.

## Workflow script skeleton (Workflow tool, JS)

```js
export const meta = {
  name: 'textbook-<topic>',
  description: '<topic> textbook (single HTML+mermaid) via fan-out research -> independent verify -> synthesize',
  phases: [
    { title: 'Research', detail: 'N parallel facets' },
    { title: 'Verify', detail: 'independent verifier agents, default-to-skeptical' },
    { title: 'Write', detail: 'one writer -> single-file HTML' },
  ],
}
const OUT = '<repo>/textbooks/<id>.html'
const STYLE = '<repo>/textbooks/B-joydrive-infra-0to100.html' // reuse style

const FACETS = [ /* {key, focus, ask} per source/dimension */ ]
const SCHEMA = { /* findings: [{topic, detail, source, url, evidence}] */ }

phase('Research')
const r = (await parallel(FACETS.map(F => () => agent('<research prompt w/ WebSearch/WebFetch or Read>', {label:'research:'+F.key, phase:'Research', schema:SCHEMA})))).filter(Boolean)

phase('Verify')            // INDEPENDENT — never skip
const v = await parallel(r.flatMap(b => b.findings.slice(0,3).map(f => () =>
  agent(`Adversarially verify: ${f.detail}. Default to refuted=true if uncertain. Check vs source/code.`, {label:'verify', phase:'Verify', schema:{/*verdict,evidence,correction*/}}))).filter(Boolean)

phase('Write')
const t = await agent(`<write prompt: synthesize r+v -> single HTML+mermaid per Format Standards + cognitive contract; Write to ${OUT}>`, {label:'write', phase:'Write', effort:'max'})
return { facets:r.length, verified:v.length, outPath:OUT, t }
```

Then (outside workflow or follow-up): expert multi-round review + persona launch-review + per-book quality scan + git commit + memory update.

## HTML `<style>` boilerplate (minimal, reuse across books)

Copy from `FunctuonOwnerLearning/textbooks/B-joydrive-infra-0to100.html` `<head>` — key classes: `nav.toc`, `main`, `h1/h2/h3`, `code`, `pre.mermaid`, `aside.note` (yellow), `blockquote.why` (blue), `table`, `.tag`, `.danger`. Mermaid init:
```html
<script src="https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.min.js"></script>
<script>mermaid.initialize({startOnLoad:true,theme:'neutral',flowchart:{useMaxWidth:true,htmlLabels:true,curve:'basis'},sequence:{useMaxWidth:true}});</script>
```

## Mermaid syntax rules (validate every block — common errors)

1. Node text with `( ) [ ] { } | / < > : "` → wrap in double quotes: `id["text (info)"]`.
2. Reserved words as node IDs (`end subgraph flowchart graph class click loop alt else state`) → rename.
3. `<br>` → `<br/>`.
4. Unclosed quotes → fix.
5. Dotted/labeled edge MUST be `A -. text .-> B` (dots hug the text) — NOT `A -.|text|.-> B` and NOT `A -.|text|-> B`. The `|text|` form is only for solid edges `A -->|text| B`.
6. Bare `&` → `&amp;` (with htmlLabels:true).
7. subgraph title with special chars → `subgraph id["title"]`.
8. `flowchart` node ID must be ASCII `[a-zA-Z0-9_]+` (non-ASCII IDs are version-fragile).
9. stateDiagram-v2: `[*]`, `state "name" as id`.
10. sequenceDiagram: `participant X as Name With Space`.

Verify method (best→worst): `mmdc` CLI real-render (needs puppeteer `--no-sandbox`) > Playwright + `window.mermaid.parse()` > `mermaid.parse()` in jsdom > manual rule-check + python regex (quotes paired, no bare `<br>`, no reserved-word IDs, no bare `&`).

## Schemas (compact)

expert review: `{role, verdict, strengths[], weaknesses:[{section,issue,severity,fix,rationale}], missing[], killer_questions[]}`
persona review: `{persona, launch_review_qa:[{question,intent,optimal_answer,basis}], top_concern, vote}`
quality-fix (per book): `{book, mermaid_blocks, mermaid_fixes:[{location,problem,fix}], toc_anchor_fixes[], hardfix_applied[], notes}`

## Discipline checklist (per book, before "done")
- [ ] Independent verifier signed off (not self-review)
- [ ] Every mermaid block validated (renders)
- [ ] Every TOC `href` resolves to an `id`
- [ ] `file:line` / source URL on non-trivial claims
- [ ] git committed + memory updated
