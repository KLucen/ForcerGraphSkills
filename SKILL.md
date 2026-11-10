---
name: force-directed-graph-renderer
description: Design, implement, or review a force-directed graph renderer with bounded physics, direct manipulation, visual semantics, and persisted layouts.
---

# Force-directed graph renderer

Use this skill when building or modifying an interactive node-and-edge graph renderer. It applies to React Flow, SVG, Canvas, or another retained-mode graph surface.

Read [references/engine-design.md](references/engine-design.md) before changing force parameters, graph projection, layout persistence, or interaction behavior.

## Core boundary

- Keep semantic graph data separate from layout data. Stable node IDs may restore positions only when the graph identity and scope match; never reuse a layout for an unrelated graph.
- Coordinates, viewport values, and formal IDs belong to the renderer or persistence layer. An upstream model or data source may provide labels, descriptions, relation types, and semantic basis, but it must not control layout.
- Keep provenance, citations, long text, and application-specific workflows outside the renderer unless the product explicitly defines them as display metadata.
- Treat the graph as an interactive projection. Navigation or organizational edges must not be presented as proof of a domain claim unless the application separately defines and validates that meaning.

## Implementation rules

1. Separate the renderer into data projection, layout seeding, physics simulation, interaction state, and persistence. Do not let UI callbacks mutate domain records directly.
2. Keep physics bounded and local. Preserve deterministic initial seeds, finite-coordinate checks, link/repulsion forces, cooling, drag reheating, pinned nodes, and serializable snapshots. A change to force constants requires updating the design-contract test and the reference.
3. Preserve interaction semantics: drag nodes, double-click to pin/unpin, click nodes or edges for details, click the pane to clear selection, zoom/pan the canvas, fit only on a new view or explicit request, and save the viewport on move end.
4. Persist positions after drag/cooling, not on every animation frame. Polling, language changes, resize, hover, and sidebar changes must not silently restart physics or fit the view.
5. Keep node rendering measured and bounded so a controlled surface does not hide nodes during rapid updates. Respect reduced-motion preferences and cancel animation frames on unmount.
6. Route node colors, labels, and legend entries through one appearance table. Keep a visible fallback for unknown kinds. Distinguish semantic basis such as `EXPLICIT` and `INFERRED` in edge styling or details without implying truth verification.
7. Keep selection, highlighting, and filtering as UI state. Filtering must not mutate knowledge data or invent replacement edges. A selected node may highlight first-order neighbors only.
8. Handle empty and small graphs without exceptions. Do not pad the graph with invented nodes or edges.

## Verification workflow

Before editing, run the renderer's design-contract test and focused unit tests. After editing, run:

- the design-contract test;
- layout, simulation, appearance, and interaction tests;
- the project's type checker, linter, and formatter;
- a real browser or device test when renderer behavior, CSS, or input wiring changes.

Tests must exercise behavior or pure-function output, not only headings or source text. Include assertions for stable layout reuse, graph-identity isolation, finite coordinates, pin/drag semantics, camera persistence, fit behavior, unknown-kind fallback, and the separation between semantic data and layout state.

When changing the skill or its reference, run the bundled `skill-creator/scripts/quick_validate.py` validator. Keep generated `node_modules`, build output, browser reports, and TypeScript cache files out of the repository.
