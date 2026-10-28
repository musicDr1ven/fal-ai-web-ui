# Spec Delta

## Purpose

Lets users supply input images to models that require them, using an uploaded file, a pasted URL, or the image currently being previewed, and lets optional image fields be cleared.

## ADDED Requirements

### Requirement: Uploads are hosted through fal storage
The system SHALL accept an image upload from the form, transmit it to fal storage using the configured credentials, and return the hosted URL for use in generation input. The uploaded file's bytes MUST NOT be stored in the application's public directories.

#### Scenario: Successful upload
- **WHEN** the user uploads a valid image
- **THEN** the API returns a fal-hosted URL that the generation request can reference

#### Scenario: Missing credentials
- **WHEN** no `FAL_KEY` is configured
- **THEN** upload fails with a clear credential error and the form remains usable

### Requirement: Uploads are validated
The system SHALL reject files that are not images or that exceed the size limit, and MUST report which constraint failed.

#### Scenario: Non-image file
- **WHEN** the user selects a PDF
- **THEN** the upload is rejected with a message naming the accepted image types

#### Scenario: Oversized file
- **WHEN** the selected image exceeds the configured size limit
- **THEN** the upload is rejected and the limit is shown

### Requirement: Image fields accept upload, URL, previewed image, or none
For a schema field that expects an image URL, the form SHALL accept an upload, a pasted URL, the image currently shown in the preview pane, or no image when the field is optional, SHALL preview the chosen image source, and MUST validate that a value is present when the field is required.

#### Scenario: Paste a URL
- **WHEN** the user pastes an image URL into a required image field
- **THEN** the form previews it and allows submission without an upload

#### Scenario: Choose the previewed image
- **WHEN** the user chooses the previewed image for an image field
- **THEN** the field previews that image and uses its re-hosted URL for generation

#### Scenario: Required image missing
- **WHEN** a model requires `image_url` and no source is provided
- **THEN** submission is blocked and the field is marked required

### Requirement: Image source choice is exclusive
Each image slot SHALL have exactly one active source. Choosing upload, URL, previewed image, or no image MUST clear or replace the previously active source's value so a stale image is never submitted.

#### Scenario: Switch from URL to preview
- **WHEN** a URL is active in an image field and the user switches to the previewed image
- **THEN** the URL value is removed and only the re-hosted preview is submitted

### Requirement: Optional image fields can be left empty
When an image field is not required, the form SHALL offer a "No image input" choice that clears the active image, omits the field from the submitted input, and starts no upload or re-host. Required image fields MUST NOT offer this choice.

#### Scenario: Clear an image used previously
- **WHEN** an optional image field holds an uploaded, pasted, or previewed image and the user chooses No image input
- **THEN** the value is cleared, the field is absent from the submitted input, and submission is allowed with text only

#### Scenario: Required field cannot be cleared
- **WHEN** the schema marks the image field required
- **THEN** No image input is unavailable and an empty value still blocks submission

### Requirement: Previewed image is re-hosted before use
The system SHALL make the previewed image reachable by fal before generation by re-hosting the corresponding local output. If re-hosting fails, the form MUST show the failure and block submission. When no image is being previewed, the option MUST be unavailable and explain why.

#### Scenario: Re-host succeeds
- **WHEN** the preview shows a generated output and the user selects it as input
- **THEN** the system returns a fal-hosted URL, the field previews it, and generation can proceed

#### Scenario: Nothing previewed
- **WHEN** no generated image is being previewed
- **THEN** the previewed-image option is disabled with an explanation

#### Scenario: Re-host fails
- **WHEN** the local output cannot be re-hosted
- **THEN** an error is shown for that field and submission stays blocked

### Requirement: Multiple images are supported where declared
When a model declares an array of images or advertises multiple-image input, the form SHALL allow more than one image and submit them in the field shape the schema expects.

#### Scenario: Two reference images
- **WHEN** the schema declares `image_urls` as an array and the user provides two images
- **THEN** the submitted input contains both URLs in the array

### Requirement: Upload state blocks premature submission
While an upload or re-host is in progress or has failed, the form MUST prevent submission and indicate the affected field.

#### Scenario: Submit during upload
- **WHEN** the user tries to submit while an image is uploading
- **THEN** submission is blocked until the upload completes or is cancelled
