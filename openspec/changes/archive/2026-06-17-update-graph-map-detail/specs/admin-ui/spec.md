## ADDED Requirements

### Requirement: Map concept search

The Map view SHALL provide a concept search control in its toolbar so an operator can jump to any concept without clicking through the graph. The control MUST query the existing `GET /graph/concepts?search=` endpoint (debounced) via the graph proxy and MUST NOT require any new pipeline route or wire-type change. Selecting a result SHALL make that concept the focus — refetching the traversal, re-centering the graph, and loading its detail — exactly as clicking a node does.

#### Scenario: Searching lists matching concepts

- **WHEN** an operator types into the Map search box
- **THEN** the client issues a debounced `GET /graph/concepts?search=<query>` request and renders the matches as a results list showing each concept's name and state

#### Scenario: Selecting a search result focuses the graph

- **WHEN** an operator chooses a concept from the search results
- **THEN** that concept becomes the focus (reflected in the URL query), the traversal is refetched and re-centered around it, and its detail panel loads

#### Scenario: No matches degrades gracefully

- **WHEN** the search query returns no concepts
- **THEN** the control shows a clear empty-results state and the current graph and detail panel are left unchanged

### Requirement: Map detail panel content and navigation

The Map detail panel SHALL present the selected concept with a readable, information-rich layout sourced from the existing `GET /graph/concepts/{id}` (`ConceptDetail`) payload, and MUST NOT require a new pipeline route or wire-type change. The panel MUST show the concept name, state, and description, a metadata block (aliases, a link to the source entity when `source_entity_id` is present, and neighbour degree), and a footer with `extractor_version` and creation date when present. Confirmed and candidate neighbours MUST be presented under a segmented Confirmed/Candidate switch with per-group counts.

Each neighbour MUST be rendered by its **peer concept name** (from `Neighbour.peer`), never a raw concept UUID, alongside its relationship type, confidence, and evidence link(s) to the supporting entities. Selecting a neighbour SHALL re-focus the graph on that peer concept, so the operator can walk the graph from the panel.

Concept-level review actions (promote, demote, merge, tombstone) SHALL be available from an actions menu in the panel header. The panel MUST NOT render per-neighbour promote/demote/edit/delete controls; that candidate-edge review workflow remains on the Candidate review tab and is unchanged.

#### Scenario: Neighbours show names, not UUIDs

- **WHEN** the detail panel renders a concept that has confirmed or candidate neighbours
- **THEN** each neighbour row displays the peer concept's name, its relationship type, its confidence, and evidence link(s), and no raw concept UUID is shown

#### Scenario: Selecting a neighbour navigates the graph

- **WHEN** an operator selects a neighbour in the detail panel
- **THEN** the peer concept becomes the new focus, the traversal is refetched and re-centered around it, and the detail panel updates to that peer

#### Scenario: Confirmed and candidate neighbours are segmented

- **WHEN** a concept has both confirmed and candidate neighbours
- **THEN** the panel shows a Confirmed/Candidate switch with each group's count, and the selected group's neighbours are listed

#### Scenario: Concept actions are available without per-neighbour controls

- **WHEN** an operator opens the detail panel's actions menu
- **THEN** promote, demote, merge, and tombstone actions for the selected concept are available, AND no neighbour row exposes promote/demote/edit/delete controls
