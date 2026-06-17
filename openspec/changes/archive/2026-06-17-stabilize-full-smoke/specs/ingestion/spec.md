## ADDED Requirements

### Requirement: Bounded Git ingestion

The Git ingestion CLI SHALL accept optional positive integer limits for maximum documents and maximum chunks. With neither limit supplied, ingestion MUST retain its existing unlimited behavior.

Document selection MUST follow deterministic connector discovery order. Once the document limit is reached, no additional parent entity may be inserted. Once the chunk limit is reached, no additional chunk entity may be inserted, embedded, vector-upserted, or graph-extracted. The CLI summary MUST report the applied limits and whether the run was truncated.

#### Scenario: Default ingestion remains unlimited

- **WHEN** the operator runs the Git ingestion CLI without document or chunk limits
- **THEN** every discovered document and chunk is processed according to the existing connector contract
- **AND** the summary does not report truncation

#### Scenario: Document limit stops new files

- **WHEN** the operator ingests a repository with `--max-documents 1`
- **THEN** exactly one parent document is processed
- **AND** no parent or chunk from a second document is inserted
- **AND** the summary reports document truncation

#### Scenario: Chunk limit stops all downstream work

- **WHEN** the operator ingests a repository with `--max-chunks 2`
- **THEN** at most two chunk entities are inserted
- **AND** at most two embedding, vector-upsert, and text-extraction operations occur
- **AND** the summary reports chunk truncation

#### Scenario: Invalid limits are rejected

- **WHEN** the operator supplies zero or a negative document or chunk limit
- **THEN** the CLI exits before cloning with a validation error naming the invalid option
