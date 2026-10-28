# Spec Delta

## Purpose

Executes generation against the selected model's own input contract and normalizes the varied image result shapes into the application's history.

## ADDED Requirements

### Requirement: Server validates input against the model schema
The generation endpoint SHALL reject requests whose input is missing required fields, has wrong types, violates enum values or numeric bounds, or targets an unknown/unavailable model, and SHALL return field-level errors. The endpoint MUST NOT forward invalid input to fal.

#### Scenario: Missing required field
- **WHEN** a request omits a field the schema marks required
- **THEN** the endpoint returns a 4xx response naming the missing field and makes no fal call

#### Scenario: Unavailable model
- **WHEN** a request targets a model marked unavailable
- **THEN** the endpoint rejects it without calling fal

### Requirement: Only model-supported parameters are sent
The endpoint SHALL send the model's declared fields, including schema defaults the client omitted, and MUST NOT send form fields that are absent from the model's schema.

#### Scenario: Switching from Flux to another model
- **WHEN** the client submits a payload containing a parameter the selected model does not declare
- **THEN** that parameter is excluded from the fal input

#### Scenario: Optional image omitted
- **WHEN** the submitted input omits an optional image field
- **THEN** the fal request omits that field entirely and the model runs without image input

### Requirement: fal errors are surfaced
When fal rejects the request, the endpoint SHALL return the provider's error message and status category so the form can display an actionable error.

#### Scenario: Provider validation error
- **WHEN** fal responds with a validation error
- **THEN** the UI receives the provider message rather than a generic failure

### Requirement: All image outputs are persisted
The endpoint SHALL detect every image output in the fal result — including `images[]`, a single `image`, and `mask_image` — download each, save it under `public/outputs`, and return their local URLs.

#### Scenario: Multi-image result
- **WHEN** a model returns two images in `images[]`
- **THEN** both are saved and returned in an `images[]` response field

#### Scenario: Single image and mask
- **WHEN** a background-removal model returns `image` and `mask_image`
- **THEN** both are saved and both URLs are returned

### Requirement: Existing response contract is preserved
The response SHALL continue to include `imageUrl` pointing at the primary saved image so the existing history and preview behavior keeps working.

#### Scenario: History still updates
- **WHEN** a generation succeeds
- **THEN** `imageUrl` is present and the image appears in the existing history list after refresh

### Requirement: Partial save failures are reported
If an image output cannot be downloaded or written, the endpoint SHALL keep the images that succeeded, return them, and report which outputs failed instead of discarding the whole result.

#### Scenario: One download fails
- **WHEN** two outputs are returned and one download fails
- **THEN** the successful image is returned and the response includes a warning about the failed output
