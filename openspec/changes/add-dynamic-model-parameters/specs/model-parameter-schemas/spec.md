# Spec Delta

## Purpose

Provides cached, resolved input and output schemas for catalog models so the form and the server can render and validate each model's own parameters.

## ADDED Requirements

### Requirement: Schemas are fetched lazily per model
When a catalog model's schema is requested and none is cached, the system SHALL fetch it from the model's fal OpenAPI document and persist the input schema, output schema, and referenced component schemas before responding.

#### Scenario: First request for a model
- **WHEN** the UI requests the schema of a model that has never been fetched
- **THEN** the system fetches the OpenAPI document, stores it, and returns the input schema and its referenced components

#### Scenario: Subsequent request uses cache
- **WHEN** the UI requests a schema that is already cached and fresh
- **THEN** no outbound fal request is made and the cached schema is returned

### Requirement: Stale schemas refresh without blocking
A cached schema older than seven days SHALL be refreshed on request. If the refresh fails, the system MUST serve the stale schema and mark it stale rather than failing the request.

#### Scenario: Successful refresh
- **WHEN** a cached schema is older than seven days and fal responds
- **THEN** the stored schema is replaced and the response is marked fresh

#### Scenario: Refresh failure
- **WHEN** a stale schema cannot be refreshed because fal is unavailable
- **THEN** the stale schema is returned with a stale indicator

### Requirement: Schema failures are explicit
If no schema is cached and the fetch fails, the system SHALL return a failure status that callers can distinguish from a model with no parameters, and MUST NOT fabricate a schema.

#### Scenario: Fetch failure with empty cache
- **WHEN** the OpenAPI request fails and no schema is cached for that model
- **THEN** the request returns an error status and the UI can offer prompt-only or raw JSON input

### Requirement: Schema requests are deduplicated
Concurrent requests for the same uncached model MUST share one outbound fetch, and a failed fetch MUST NOT leave a partial schema cached.

#### Scenario: Simultaneous requests
- **WHEN** two requests for the same uncached model arrive together
- **THEN** only one fal OpenAPI request is made and both responses receive the same schema

### Requirement: Schemas are scoped to catalog models
Schema retrieval SHALL be limited to endpoint IDs present in the local catalog. Unknown or unavailable endpoint IDs MUST be rejected.

#### Scenario: Unknown endpoint requested
- **WHEN** a schema is requested for an endpoint that is not in the catalog
- **THEN** the request returns not-found and no fal request is made
