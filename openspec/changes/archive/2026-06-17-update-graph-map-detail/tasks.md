## 1. Concept search

- [x] 1.1 Add `ConceptSearch` component (`src/components/ConceptSearch.tsx`): debounced (~250ms) input that queries `/api/proxy/graph/concepts?search=<q>`, renders a results dropdown of name + state, exposes `onSelect(conceptId)`, and handles empty-results, loading, and close-on-select/blur/Escape
- [x] 1.2 Add `ConceptSearch.stories.tsx` and `ConceptSearch.test.tsx` (renders matches, fires `onSelect`, shows empty state)
- [x] 1.3 Mount `ConceptSearch` in the `GraphExplorer` toolbar; wire `onSelect` to `setSelectedId` + `setFocus` so a pick re-centers the graph and loads detail

## 2. Detail panel rewrite

- [x] 2.1 Add `NeighbourRow` component (`src/components/NeighbourRow.tsx`): relationship-type badge, peer name as a button calling `onNavigate(peer.concept_id)`, confidence, and compact evidence link(s) to `/entities/{id}`; no promote/demote/edit/delete controls
- [x] 2.2 Add `NeighbourRow.stories.tsx` and `NeighbourRow.test.tsx` (renders peer name not UUID, fires `onNavigate`, renders evidence links)
- [x] 2.3 Rewrite `ConceptDetail` (`src/components/ConceptDetail.tsx`) as the compact card: header (state indicator · name · `symbol_kind` · actions menu), description, metadata block (aliases, source-entity link when `source_entity_id` present, degree), segmented Confirmed/Candidate switch with counts, neighbour rows via `NeighbourRow`, footer (`extractor_version` · created date when present); accept an `onNavigate` prop
- [x] 2.4 Put concept-level actions (reuse `ConceptActions`, new `layout="menu"`) inside a header actions menu (shadcn `DropdownMenu`); remove `CandidateEdgeRow` usage from the panel
- [x] 2.5 Update `ConceptDetail.stories.tsx` and `ConceptDetail.test.tsx` for the new layout: neighbours show names not UUIDs, segmented switch toggles groups, `onNavigate` fires on neighbour select, no per-neighbour review buttons rendered

## 3. Wiring

- [x] 3.1 In `GraphExplorer`, pass the focus-setter as `onNavigate` to `ConceptDetail` so neighbour clicks re-focus the graph identically to node clicks; confirm `CandidateEdgeRow` and the Candidate review tab are untouched

## 4. Verification

- [x] 4.1 Run `pnpm --filter @cortex/admin-ui test` (53 pass). NOTE: `next lint` is interactive/unconfigured in this repo; type+lint checks run during `pnpm -r build` instead (4.2)
- [x] 4.2 Run `pnpm -r build`
- [x] 4.3 Deployed the rebuilt admin-ui image to the running stack; `/graph?tab=map` serves HTTP 200 with the new search box present in markup. Behaviours (search select, neighbour-name navigation, segmented Confirmed/Candidate, no per-neighbour review buttons) are asserted by the vitest component tests; final pixel/UX eyeball left to the operator at the live URL
- [x] 4.4 Run `openspec validate update-graph-map-detail --strict`, scan staged changes for secrets, commit, and push
