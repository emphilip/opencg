## Context

The Map (`services/admin-ui/src/app/graph/GraphExplorer.tsx`) renders a force graph (left) and a detail `aside` (right). The aside currently renders `ConceptDetail`, which reuses `CandidateEdgeRow` for neighbours — a 4-column candidate-review row built for the Candidate review tab. In the narrow (`lg:w-80`, 320px) aside it overflows, shows `from_concept_id → to_concept_id` UUIDs, and carries Promote/Demote/Edit/Delete on every row. There is no way to search for a concept; navigation is click-only.

The backend already supplies everything:
- `GET /graph/concepts?search=` returns matching concepts (used by the Concepts tab).
- `GET /graph/concepts/{id}` returns `ConceptDetail`, whose `neighbours_confirmed` / `neighbours_candidate` are `Neighbour { edge, peer: ConceptListItem, evidence_entity_ids }` — the peer name is already present and currently ignored.

So this is an admin-UI-only change with no new routes or `@cortex/shared` wire types.

## Goals / Non-Goals

**Goals:**

- Add a concept search box to the Map toolbar that re-focuses the graph on a chosen concept.
- Replace the detail panel with a compact, detail-rich, navigable card (the approved "Option C") that shows real neighbour names and supports clicking a neighbour to navigate.
- Keep the graph-navigation surface (depth, candidates, node click, hover spotlight, labelling, empty/error states) exactly as-is.
- Keep `CandidateEdgeRow` and the Candidate review tab untouched.

**Non-Goals:**

- No backend, pipeline route, schema, or wire-type changes.
- No per-neighbour edge review in the Map panel (that stays on the Candidate review tab).
- No change to the force-graph rendering/physics itself.
- No new concept metadata that the API doesn't already return (e.g. a true symbol "kind" field — see Decisions).

## Decisions

### Decision 1: Search reuses `GET /graph/concepts?search=`, owned by a new `ConceptSearch` component

A self-contained `ConceptSearch` renders the toolbar input, debounces (~250ms) queries through the existing `/api/proxy/graph/concepts?search=` route, shows a results dropdown (name + state), and calls an `onSelect(conceptId)` prop. `GraphExplorer` passes `onSelect={(id) => { setSelectedId(id); setFocus(id); }}` — the same state setters `onNodeClick` already uses, so selection re-centers the graph and loads detail through the existing effects with zero new data flow.

*Alternative considered:* fetch the full concept list once and filter client-side. Rejected — the endpoint already does server-side search, and the catalogue can grow unbounded.

### Decision 2: Rewrite `ConceptDetail`; extract a `NeighbourRow`; reuse `ConceptActions`

`ConceptDetail` becomes the compact card. A new `NeighbourRow` renders one neighbour: relationship-type badge (`RelationshipTypeBadge`), peer name as a button calling `onNavigate(peer.concept_id)`, confidence, and evidence links (`/entities/{id}`). The Confirmed/Candidate segmented switch is local component state defaulting to whichever group is non-empty (Confirmed first). Concept-level actions move into a header menu that renders the existing `ConceptActions`. `CandidateEdgeRow` is left as-is for the Candidate review tab.

`GraphExplorer` passes the same focus-setter as `onNavigate` to `ConceptDetail`, so neighbour clicks navigate identically to node clicks.

*Alternative considered:* parameterise `CandidateEdgeRow` with a `readOnly`/`showActions` flag instead of a new component. Rejected — the two rows have diverging layouts and responsibilities; a focused `NeighbourRow` is clearer to read and test than a flag-laden shared row.

### Decision 3: Show `symbol_kind` directly (it already exists)

The mockup shows a "function" kind. `ConceptListItem` (and therefore `ConceptDetail`) already carries `symbol_kind?: string | null`, so the panel renders it directly when present and omits the kind suffix otherwise — no derivation and no backend change needed.

### Decision 4: Menu component

Use a minimal accessible dropdown for the header actions menu. If the shadcn/ui kit in `src/components/ui` already provides a dropdown-menu primitive, use it; otherwise a small popover built from existing `Button` primitives is acceptable. The menu's contents are the existing `ConceptActions`.

## Risks / Trade-offs

- **Evidence/degree presentation in 320px.** The panel is narrow. → Keep rows single-purpose, truncate long names with ellipsis + title attribute, and show evidence as a compact "evidence ×N" link rather than repeated links.
- **Search dropdown vs. canvas focus/keyboard.** A toolbar dropdown over a canvas can clash with focus handling. → Standard listbox semantics, close on select/blur/Escape; this is ordinary form UI, low risk.
- **"Kind" derivation may be wrong for non-code concepts.** → Treat it as a best-effort hint and omit when unsure; nothing depends on it.

## Migration Plan

Pure additive UI change. No data, schema, or route migration. Rollback is reverting the commit; the previous panel returns. Verified via `pnpm --filter @cortex/admin-ui test`, `pnpm -r build`, and a manual pass on `/graph?tab=map` against the running stack.

## Open Questions

- None blocking. A real concept "kind" field (vs. the derived hint) could be a follow-up if operators want it surfaced reliably.
