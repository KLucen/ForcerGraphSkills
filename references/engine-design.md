# Force-directed graph renderer design

This reference describes a reusable renderer architecture for small and medium interactive node-and-edge graphs. It is implementation guidance rather than a fixed API.

## Data boundary

Keep the graph projection small and serializable:

- nodes have a stable `id`, a `type` or `kind`, labels, and optional short display fields;
- edges have a stable `id`, `source`, `target`, a relation type, optional display labels, and optional semantic-basis flags;
- positions are separate records `{ node_id, x, y, pinned }`;
- the viewport is separate `{ pan_x, pan_y, zoom }`;
- long text, raw model output, provenance, and domain records stay outside the rendering projection.

The renderer should accept a graph that is already validated by its caller. It should not infer missing edges, create formal domain IDs, or silently turn display metadata into semantic relationships.

## Deterministic seeding and simulation

Seed positions from saved finite coordinates by stable node ID. Give new nodes deterministic initial positions so the same input is visually reproducible. One useful spiral seed is:

```text
x = cos(i * 2.399) * (180 + i * 9)
y = sin(i * 2.399) * (180 + i * 9)
pinned = false
```

The physics engine owns transient coordinates and never mutates domain data. A bounded local model can use:

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

These values are a practical starting point for a small graph. Tune them against real graph sizes and interaction latency, and keep the tick loop bounded so it cannot monopolize the UI thread.

## Rendering and interaction

Treat the graph surface as a controlled projection over React Flow, SVG, Canvas, or an equivalent renderer. Keep node dimensions known and bounded so rapid controlled updates do not make nodes disappear.

Interaction invariants:

- pointer dragging calls `move`; drag stop snapshots and persists positions;
- double-click or an explicit control toggles pinning and persists the snapshot;
- node click selects a node, edge click selects an edge, and pane click clears node selection;
- hover or selection highlights the active node and first-order neighbors; unrelated nodes and edges fade;
- hovering an incident edge may show its relation label;
- semantic basis values such as `EXPLICIT` and `INFERRED` remain distinguishable, with dashed styling available for inferred edges;
- the viewport is persisted on `onMoveEnd` and starts from the saved `defaultViewport`;
- `fitView` runs only for an initial empty layout, a changed graph scope, or an explicit fit request;
- polling, label-language changes, resize, hover, and unrelated sidebar changes do not restart physics or fit the view;
- set sensible minimum and maximum zoom values, disable accidental double-click zoom when double-click means pinning, and disable editing affordances when the graph is read-only;
- requestAnimationFrame callbacks are cancelled on unmount; reduced-motion mode advances a bounded batch and persists once rather than animating every frame.

## Persistence and identity

Semantic results and layout are separate versions. A layout may be reused only when the graph identity, graph scope, and stable node IDs match. A changed graph identity or incompatible topology must start with a fresh layout or an explicitly migrated subset. Resetting layout may preserve explicitly pinned positions, but it must never rewrite semantic graph data.

Persist after meaningful boundaries: drag stop, pin/unpin, cooled simulation, and viewport movement. Debounce or queue the outer store as needed, but do not write storage on every physics tick. If persistence uses optimistic versions, retain a local draft until the store acknowledges it and resolve conflicts by reloading the current stored layout instead of silently overwriting it.

## Visual language

Use one appearance table for node colors, labels, filters, and the legend. Unknown kinds need a visible fallback and must not render as an unintended white default. Color communicates type or category only; it does not communicate truth, review, confidence, or scientific quality unless the product explicitly defines that meaning.

Provide a legend, node details, relation details, and an explicit empty state. Keep labels accessible in the available languages without changing geometry when the display language changes. Important information must not be available only through hover.

## Scope and safety boundaries

The renderer is a visualization and interaction layer. Keep application-specific workflows, network calls, model prompts, evidence review, document publication, and domain-specific validation in the surrounding application. Do not make the renderer responsible for proving the meaning of an edge or deciding which domain records should exist.

## Change checklist

When modifying an implementation, inspect the equivalent of:

1. the graph renderer;
2. the force simulation;
3. the deterministic layout seeder;
4. the graph data types;
5. the shared appearance/legend table;
6. the design-contract test;
7. the browser or device test when real pointer, zoom, resize, or accessibility behavior changes.

Confirm that the change preserves finite coordinates, stable IDs, camera/layout separation, bounded physics, reduced-motion behavior, semantic/layout separation, and empty-graph handling before changing force constants or data contracts.
