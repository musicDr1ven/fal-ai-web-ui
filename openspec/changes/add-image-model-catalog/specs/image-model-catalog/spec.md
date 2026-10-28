# Spec Delta

## Purpose

Maintains a local, queryable catalog of fal image models so the application can offer model discovery, pricing, and persistent selection instead of a hard-coded list.

## ADDED Requirements

### Requirement: Catalog contains image-output models only
The catalog SHALL store only endpoints that declare image output. Endpoints whose discovery data lacks the `output:image` tag — including video, audio, 3D, and training endpoints — MUST NOT be stored as selectable models.

#### Scenario: Mixed catalog is filtered on refresh
- **WHEN** a refresh retrieves a page containing image, video, audio, and 3D endpoints
- **THEN** only the endpoints tagged `output:image` are persisted as selectable models

#### Scenario: Training endpoints are excluded
- **WHEN** a retrieved endpoint describes a model-training or adapter-training task
- **THEN** that endpoint is not persisted as a selectable model

### Requirement: Refresh imports the complete catalog
A refresh SHALL page through the entire public fal model catalog and upsert each model by endpoint ID. A failure on one page MUST NOT discard models already stored or prevent a later refresh from completing the import.

#### Scenario: Full refresh creates and updates models
- **WHEN** the user starts a refresh and all pages are retrieved
- **THEN** new endpoints are inserted, changed metadata is updated, and the result reports added and updated counts

#### Scenario: Partial failure preserves existing data
- **WHEN** a page request fails or is rate limited after earlier pages succeeded
- **THEN** the models from successful pages remain stored, the refresh reports a partial result, and the previously stored catalog is not deleted

### Requirement: Concurrent refreshes are serialized
Only one catalog import SHALL run at a time. A refresh request received while an import is active MUST NOT start a second import and SHALL report that an import is already in progress.

#### Scenario: Second refresh while running
- **WHEN** a refresh is requested while another refresh is still importing
- **THEN** the second request returns an in-progress status and no duplicate catalog fetch occurs

### Requirement: Refresh preserves selection state
A refresh MUST preserve the selected flag of every previously known model and MUST NOT automatically select newly discovered models.

#### Scenario: Selection survives refresh
- **WHEN** the user has selected two models and then runs a refresh
- **THEN** those two models remain selected and every newly imported model is unselected

### Requirement: First initialization seeds existing models
On first initialization of an empty catalog, the system SHALL mark `fal-ai/flux-lora`, `fal-ai/flux/dev`, and `fal-ai/flux-realism` as selected so the existing Generate behavior is preserved before any refresh.

#### Scenario: Empty database receives defaults
- **WHEN** the application starts with no catalog data
- **THEN** the three legacy Flux endpoints are present and selected

### Requirement: Removed models remain identifiable
Models that disappear from the catalog SHALL be marked unavailable rather than deleted, and unavailable models MUST NOT be offered for selection or generation.

#### Scenario: Endpoint disappears from fal
- **WHEN** a previously stored endpoint is absent from a completed refresh
- **THEN** it is marked unavailable, remains excluded from selection lists, and its historical record is retained

### Requirement: Pricing uses structured source when available
The system SHALL record a model's price from the structured pricing source when credentials allow it, otherwise from the catalog's prose pricing text when present, and otherwise record the price as unknown. No API credential SHALL be written to the catalog database.

#### Scenario: Structured pricing succeeds
- **WHEN** a valid `FAL_KEY` is configured and the pricing source returns a price for an endpoint
- **THEN** the stored price includes the numeric unit price, unit, and currency

#### Scenario: Pricing unavailable
- **WHEN** no key is configured or the pricing source rejects the request
- **THEN** the refresh still succeeds and the model's price falls back to prose text or unknown

### Requirement: Selection changes persist
Toggling a model's selection SHALL persist across application restarts, and the stored selection SHALL be the sole source for the Generate tab's model choices.

#### Scenario: Selection survives restart
- **WHEN** a user selects a model and restarts the application
- **THEN** the model is still selected and available in the Generate tab
