## Why

The `/graph` Map view can only be navigated by clicking nodes, and its detail panel reuses the candidate-review row (`CandidateEdgeRow`) — so neighbours render as raw concept UUIDs (`f8555ee2… → 8b9e997a…`), every row carries Promote/Demote/Edit/Delete buttons, and the layout overflows the narrow side panel. An operator can't jump to a known concept, can't read what a neighbour actually is, and can't navigate by neighbour. Everything needed to fix this is already in the API responses (`GET /graph/concepts?search=` and the `ConceptDetail` payload, whose `Neighbour.peer` already carries the neighbour's name) — this is purely an admin-UI rework.

## What Changes

- Add a **concept search** box to the Map toolbar: debounced queries to `GET /graph/concepts?search=`, a results dropdown (name + state), and selecting a result re-focuses the graph and loads its detail.
- Rewrite the Map **detail panel** into a compact, information-rich card: header (state indicator · name · kind · actions menu), description, a metadata block (aliases, source-entity link, degree), a segmented Confirmed/Candidate switch, neighbour rows, and a footer (extractor version · created date).
- Render neighbours as **readable, clickable names** (from `Neighbour.peer`) with their relationship type, confidence, and evidence links; clicking a neighbour **re-focuses the graph** on that concept.
- Introduce a `NeighbourRow` component for the panel; stop using `CandidateEdgeRow` there. `CandidateEdgeRow` and the Candidate review tab are unchanged.
- Move concept-level actions (Promote / Demote / Merge / Tombstone) into the header **actions menu**; remove per-neighbour review buttons and raw-UUID rendering from the Map panel.
- Add/update `*.stories.tsx` for every new or changed shared component.

## Capabilities

### New Capabilities

<!-- none -->

### Modified Capabilities

- `admin-ui`: adds Map concept search and a redesigned, navigable, detail-rich Map detail panel to the existing "Interactive graph exploration view"; no change to backend routes or wire types.

## Impact

- **Code**: `services/admin-ui` only — `src/app/graph/GraphExplorer.tsx` (toolbar search + neighbour/search → focus wiring), `src/components/ConceptDetail.tsx` (rewrite), new `src/components/ConceptSearch.tsx` and `src/components/NeighbourRow.tsx`, plus their `*.stories.tsx`. `ConceptActions` reused for the menu.
- **No** backend, pipeline route, wire-type (`@cortex/shared`), schema, or compose change. Consumes existing `GET /graph/concepts?search=`, `GET /graph/concepts/{id}`, and `GET /graph/traverse`.
- **Tests**: vitest component tests for `ConceptSearch`, `NeighbourRow`, and the rewritten `ConceptDetail` (names not UUIDs, neighbour-click focus callback, segmented tabs, no per-edge review buttons).
- The existing Concepts / Candidate review / Vocabulary tabs and all current Map scenarios (default view, traverse render, node-click re-center, candidate toggle, labelling, empty/error states) remain unchanged.
