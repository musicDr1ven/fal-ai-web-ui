# Proposal

## Why

Once the catalog in `add-image-model-catalog` lets users select any image model, the Generate form still sends a Flux-specific input shape (`num_inference_steps`, `guidance_scale`, `loras`, …) and the API route assumes a `images[0].url` result. Most catalog models will reject that input or return a different output shape, and image-to-image or background-removal models cannot run at all because the app has no way to supply an input image.

## What Changes

- Store each model's fal OpenAPI input/output schema locally, fetched lazily from `https://fal.ai/api/openapi/queue/openapi.json?endpoint_id=…` when a model is first selected and refreshed when stale.
- Add a schema API that serves the cached input schema (including referenced component schemas) and triggers a lazy fetch.
- Replace the fixed Flux form with a schema-driven form:
  - enums → selects, booleans → checkboxes, numbers/integers → bounded numeric inputs, strings → text inputs/areas
  - schema defaults prefilled
  - required fields marked and validated before submit
  - optional fields grouped under an "Advanced" disclosure
  - keep a dedicated repeatable editor for LoRA-style arrays
  - fall back to a raw JSON input editor for schemas the renderer cannot represent
- Add image input support: an upload endpoint that hosts files through fal storage and returns a URL, plus schema-driven image fields that accept an upload, a pasted URL, the image currently shown in the preview pane, or no image at all when the field is optional. Support multiple images where the schema or model capability allows it.
- Re-host a chosen previewed output through fal storage on demand so a generated image can be fed back in as input, without persisting an input-image library.
- No reusable upload library or input history: the previewed image is re-hosted per use.
- Rework generation execution to validate input against the cached schema, call `fal.subscribe(model, { input })`, and save every returned image regardless of output shape (`images[]`, `image`, `mask_image`).
- Preserve the existing response contract (`imageUrl`) so the current image history keeps working, while adding an `images[]` list for multi-output results.
- Show fal validation/API errors in the form instead of a generic failure message.

## Capabilities

### New Capabilities
- `model-parameter-schemas`: lazy fetch, caching, staleness handling, and serving of per-model OpenAPI input schemas.
- `dynamic-generation-form`: schema-driven rendering, defaults, required-field validation, advanced grouping, LoRA editor, and raw-JSON fallback.
- `image-input`: upload, URL, and previewed-image entry for schema image fields, including multiple-image models.
- `generation-execution`: server-side validation, fal dispatch, and output normalization/persistence for varied image result shapes.

### Modified Capabilities

<!-- None: `image-model-catalog` and `model-browser-ui` behavior is unchanged; this change consumes their data. -->

## Impact

- **Depends on**: `add-image-model-catalog` (SQLite connection, `models` table, selected-model data).
- **New files**: `lib/modelSchema.js`, `lib/inputValidation.js`, `lib/outputs.js`, `pages/api/models/[id]/schema.js`, `pages/api/upload.js`, `pages/api/upload/from-output.js`, dynamic form components under `app/components/`.
- **Modified files**: `pages/api/generateImage.js` (rewritten around `{ model, input }`), `app/components/GenerateForm.js`, `app/page.js` (schema/state wiring, current preview filename), `package.json` if a multipart helper is needed.
- **Database**: new `model_schemas` table keyed by endpoint ID with schema JSON and fetch timestamp; created idempotently alongside the existing schema.
- **External APIs**: fal queue OpenAPI endpoint (public) and fal storage upload (authenticated).
- **Environment**: `FAL_KEY` is required for uploads and generation.
- **Compatibility**: the Generate request payload changes from flat Flux fields to `{ model, input }`; the response keeps `imageUrl` and gains `images[]`.
- **Failure mode**: if schema retrieval fails, the form falls back to prompt-only plus raw JSON entry rather than blocking generation entirely.
