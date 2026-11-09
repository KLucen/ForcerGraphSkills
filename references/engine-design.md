# ResearchGraph force graph engine design

This reference captures the behavior that was implemented and tested in the earlier ResearchGraph graph work. It is guidance for reuse, not a new API contract.

## Data boundary

The renderer consumes a small `GraphData` projection:

- nodes have a stable `id`, a `type` or legacy `kind`, bilingual labels, and optional short display fields;
- edges have stable `id`, `source`, `target`, a relation type, optional Chinese label, and optional dashed/basis flags;
- positions are separate records `{ node_id, x, y, pinned }`;
- the viewport is separate `{ pan_x, pan_y, zoom }`.

For Lite, the semantic graph is a single-paper LightGraph containing `RESEARCH_QUESTION`, `CONCEPT`, `METHOD`, `CONTRIBUTION`, and `FINDING` nodes, normally 6–12 and never padded with invented content. Relations are directed and use the most specific available type (`INVESTIGATES`, `USES`, `ANALYZES`, `YIELDS`, `CONTRIBUTES_TO`, or `RELATED_TO`). Node and relation `basis` distinguishes `EXPLICIT` from `INFERRED`. Evidence, citation, page, quote, source span, and paragraph provenance do not belong in this projection.

## Deterministic seeding and simulation

`placeNodes` first reuses saved finite coordinates by stable node ID. New nodes receive deterministic seeds:

```text
x = cos(i * 2.399) * (180 + i * 9)
y = sin(i * 2.399) * (180 + i * 9)
pinned = false
```

The local `ForceSimulation` owns transient coordinates and never mutates knowledge data. Its current bounded model is:

- `wake()` sets `alpha = 0.75`;
- pairwise repulsion uses `min(12, 2600 / distance² + max(0, 50 - distance) * 0.12)` with a minimum distance of 4;
- each edge is a spring toward distance 115 with coefficient `0.035`;
- velocity is damped by `0.7`, and one tick moves a point by at most 14 pixels per axis;
- alpha decays by `0.965` while cooling and `0.99` during dragging; while dragging it never falls below `0.25`;
- a point marked `pinned` or currently dragged is not advanced by physics;
- the simulation is settled when `alpha < 0.005` and no node is being dragged;
- `move` accepts only finite coordinates, clears velocity, marks the node as dragging, and reheats;
- `pin` toggles the flag, clears velocity, and reheats;
- `snapshot` emits only serializable position records.

These values keep the graph responsive for the small LightGraph projection. Do not turn this into a global graph layout service or feed full paper text into the physics loop.

## React Flow behavior

The renderer is a controlled projection over React Flow. Nodes remain measured at 18×18 pixels so rapid controlled updates do not make React Flow hide them. The renderer maps positions from the simulation into node positions and maps graph edges into straight, non-connectable edges.

Interaction invariants:

- pointer dragging calls `move`; drag stop snapshots and persists positions;
- double-click toggles pinning and persists the snapshot;
- an external pin request toggles the requested node;
- node click selects a node, edge click selects an edge, and pane click clears node selection;
- hover or selection highlights the active node and first-order neighbors; unrelated nodes and edges fade;
- hovering an incident edge may show its relation label;
- `EXPLICIT` and `INFERRED` relations remain distinguishable, with dashed styling available for inferred edges;
- the viewport is persisted on `onMoveEnd` and starts from the saved `defaultViewport`;
- `fitView` runs only for an initial version-0 layout, a changed view scope, or an explicit fit request;
- polling, language changes, resize, hover, and sidebar changes do not restart physics or fit the view;
- `minZoom` is `0.1`, `maxZoom` is `4`, double-click zoom is disabled, and deletion/connection affordances are disabled;
- requestAnimationFrame callbacks are cancelled on unmount; reduced-motion mode advances a bounded batch and persists once rather than animating every frame.

## Persistence and versioning

Semantic results and layout are separate versions. A layout may be reused when the paper identity, literature version, graph scope, and stable node IDs match. A new literature version must start without the old positions and camera. Resetting layout keeps only explicitly pinned positions when the product asks for a rearrangement; it must not rewrite the semantic graph.

Persist after meaningful boundaries: drag stop, pin/unpin, cooled simulation, and viewport movement. Debounce or queue the outer store as needed, but do not write the database on every physics tick. If the persistence layer has optimistic versions, retain a local draft until the server acknowledges it and resolve conflicts by reloading the current server layout rather than silently overwriting it.

## Visual language

Use the shared `graphAppearance` table for node color and Chinese labels. The current Lite palette covers the five node kinds above and an `UNKNOWN` fallback (`#74847b`). Unknown values must never render as a white default node. Color communicates type only; it does not communicate truth, review, or scientific quality.

The graph UI should identify itself as an AI-generated single-paper concept graph and provide a legend, node details, relation details, and a clear empty state. Keep bilingual labels available: Chinese is the primary display, English preserves the paper's terminology, and `both` shows both when present.

## Historical full-graph lessons that still apply

The earlier full graph workspace established useful shell behavior: search and type/paper filters precede the canvas, details follow the canvas, a focused node highlights a local neighborhood, full-screen keeps the same canvas instance, and resize does not unexpectedly refit. It also established that shared support nodes should be represented once and that navigation edges must not be mistaken for scientific relations.

Those behaviors are reusable UI lessons only. Do not bring back the old cross-paper graph, Evidence nodes, Query, Lint, Wiki, comparator, or automatic knowledge-evolution workflows into Lite.

## Change checklist

When modifying this engine, inspect:

1. `apps/desktop/src/ForceGraph.tsx`
2. `apps/desktop/src/forceSimulation.ts`
3. `apps/desktop/src/graphLayout.ts`
4. `apps/desktop/src/graphTypes.ts`
5. `apps/desktop/src/graphAppearance.tsx`
6. `apps/desktop/src/ForceGraph.design.test.tsx`
7. `apps/desktop/e2e/graph.spec.ts` when browser behavior changes.

Confirm that the change preserves finite coordinates, stable IDs, camera/layout separation, bounded physics, reduced-motion behavior, and the Lite semantic boundary before changing force constants or data contracts.
