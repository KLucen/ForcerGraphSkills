---
name: researchgraph-force-graph
description: Reuse or extend ResearchGraph's force-directed LightGraph renderer, simulation, visual semantics, interaction states, and persisted layouts for single-paper research graphs.
---

# ResearchGraph force graph

Use this skill when changing the reusable graph renderer or when adding a single-paper LightGraph view that must preserve ResearchGraph's established physics, interaction, visual, and persistence behavior.

Read [references/engine-design.md](references/engine-design.md) before changing force parameters, graph projection, layout persistence, or interaction behavior. The reference records the real implementation and the boundaries inherited from the earlier full graph work.

## Core boundary

- LightGraph is a compact concept-navigation graph inside PaperCard for one paper and one literature version.
- PaperCard and LightGraph are produced by one LLM call. The renderer does not start another extraction pipeline.
- Keep semantic graph data separate from layout data. Stable node IDs may restore positions only for the same paper and literature version; never reuse layout across papers or versions.
- The LLM may propose labels, short descriptions, relation types, and `EXPLICIT`/`INFERRED` basis. It must not generate coordinates, viewport values, formal IDs, or evidence fields.
- Do not add Evidence, Citation, Quote, Page, SourceSpan, paragraph provenance, cross-paper fusion, Query, Lint, Wiki, or full Knowledge Engine behavior to this skill's graph.
- The graph is a reading aid, not an independently verified scientific argument network. Navigation edges must not be presented as proof.

## Implementation rules

1. Prefer the existing `ForceGraph`, `ForceSimulation`, `graphLayout`, `graphTypes`, and `graphAppearance` modules. Do not reintroduce the removed full graph workspace or old Query/Lint/Wiki modules.
2. Keep physics bounded and local. Preserve deterministic initial seeds, finite-coordinate checks, link/repulsion forces, cooling, drag reheating, pinned nodes, and snapshot output. A change to force constants requires updating the design-contract test and the reference.
3. Preserve interaction semantics: drag nodes, double-click to pin/unpin, click nodes or edges for details, click the pane to clear selection, zoom/pan the canvas, fit only on a new view or explicit request, and save the viewport on move end.
4. Persist positions after drag/cooling, not on every animation frame. Polling, language changes, resize, hover, and sidebar changes must not silently restart physics or fit the view.
5. Keep node rendering measured and bounded so React Flow does not hide controlled nodes during rapid updates. Respect reduced-motion preferences and cancel animation frames on unmount.
6. Route all node colors and Chinese labels through `graphAppearance`. Keep a visible non-white fallback for unknown kinds. Distinguish `EXPLICIT` from `INFERRED` in edge styling or details without implying truth verification.
7. Keep selection, highlighting, and filtering as UI state. Filtering must not mutate knowledge data or invent replacement edges. A selected node may highlight first-order neighbors only.
8. Handle empty and small graphs without exceptions. Do not pad the graph with invented nodes or edges.

## Verification workflow

Before editing, run the graph design-contract test and the focused existing tests. After editing, run:

- the design-contract test;
- `ForceGraph.test.tsx`, `forceSimulation.test.ts`, and `graphAppearance.test.tsx`;
- TypeScript and ESLint checks for the desktop app;
- the real LightGraph browser test when renderer behavior, CSS, or React Flow wiring changes.

Tests must exercise behavior or pure-function output, not only headings or source text. Include assertions for stable layout reuse, cross-version isolation, finite coordinates, pin/drag semantics, camera persistence, fit behavior, unknown-kind fallback, and the absence of evidence fields in the LightGraph contract.

When changing the skill or its reference, run the bundled `skill-creator/scripts/quick_validate.py` validator. Keep generated `node_modules`, build output, browser reports, and TypeScript cache files out of the repository.
