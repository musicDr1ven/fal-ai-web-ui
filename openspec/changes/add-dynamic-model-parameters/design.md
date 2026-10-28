# Design

## Context

See `proposal.md` for motivation and the four capability specs for required behavior. This change builds on `add-image-model-catalog`: it assumes the SQLite connection, `models` table, and selected-model state exist.

Constraints discovered while investigating:

- `https://fal.ai/api/openapi/queue/openapi.json?endpoint_id=…` is public, returns OpenAPI 3.1, and embeds category/about plus named `*Input`/`*Output` schemas and reusable components (`ImageSize`, `ImageFile`).
- Input schemas vary substantially: Flux requires only `prompt`; birefnet requires `image_url`; `image_size` is sometimes an enum, sometimes `anyOf` a component object. Optional arrays such as `loras` are arrays of `{path, scale}`.
- Output shapes vary: `{images: []}`, `{image}`, `{image, mask_image}`. The existing route hard-codes `result.images[0].url`.
- `fal.storage.upload(blob)` is available in the installed `@fal-ai/serverless-client@0.14.3` and returns a hosted URL.
- `pages/api/generateImage.js` imports `node-fetch`, which is **not installed**; the route is already broken at import time. Node 24 provides global `fetch` and `Blob`.
- `getGeneratedImages` only lists `jpg|jpeg|png`, so WebP outputs would silently disappear from history.

## Goals / Non-Goals

**Goals:**

- A form that works for the common JSON-Schema shapes in the fal image catalog without per-model code.
- A server that is authoritative on validation and does not forward garbage to fal.
- Image input for image-to-image and background-removal models using fal-hosted URLs.
- Output persistence that adapts to the result shape.

**Non-Goals:**

- Non-image schemas (video/audio/3D) and their output handling.
- Reusable user-uploaded assets or an input-image history.
- Full JSON-Schema UI coverage (we deliberately fall back to raw JSON for exotic constructs).
- Offline schema generation.

## Decisions

### Store the full OpenAPI document per model

Add a `model_schemas` table rather than columns on `models`:

```sql
CREATE TABLE model_schemas (
  endpoint_id TEXT PRIMARY KEY REFERENCES models(endpoint_id),
  document TEXT NOT NULL,          -- full OpenAPI JSON
  input_schema_name TEXT NOT NULL, -- e.g. FluxDevInput
  output_schema_name TEXT,
  fetched_at TEXT NOT NULL
);
```

Keeping the whole document makes `$ref: "#/components/schemas/…"` resolvable on both client and server without copying or dereferencing fragments, and keeps schema churn independent of catalog refresh. Storing only the input schema would require inlining components and handling cycles.

TTL is seven days. Freshness is checked on read; stale reads trigger a refresh but still return the cached document if the refresh fails.

### In-flight deduplication in-process

Schema fetches are deduplicated with a module-level `Map<endpointId, Promise>`. This is sufficient for the single-process target; a multi-process deployment would need a DB lock, which is out of scope. Failed fetches are not cached.

### Schema endpoint returns the document, not dereferenced fields

`GET /api/models/[id]/schema` returns:

```json
{
  "endpointId": "fal-ai/flux/dev",
  "document": { "...": "OpenAPI 3.1 document" },
  "inputSchemaName": "FluxDevInput",
  "outputSchemaName": "FluxDevOutput",
  "fetchedAt": "2026-10-28T...",
  "stale": false
}
```

Dereferencing server-side would either duplicate a large document or risk cycles; the renderer resolves refs lazily as it walks properties. Only catalog endpoints are accepted, and endpoint IDs must match `^[a-z0-9][a-z0-9._-]*(\/[a-z0-9._-]+)+$` before being interpolated into the fal URL.

### Custom renderer, `ajv` on the server only

The UI renderer walks the input schema properties and supports:

- `enum` → select
- `type: boolean` → checkbox
- `type: number|integer` → numeric input with `minimum`/`maximum`/`multipleOf`
- `type: string` → single-line, or textarea for `prompt`/`description`-like names
- `anyOf` containing one `$ref` and one `enum` (e.g. `ImageSize`) → enum select plus a "custom width/height" option
- arrays of `{path, scale}` → dedicated LoRA rows
- arrays of strings/numbers → repeatable inputs
- nested objects whose properties are all scalar → grouped sub-fields

Anything else (`oneOf`, `allOf`, deep objects, unrecognized `$ref` cycles) switches the form to **raw JSON mode** for the whole input, with parse validation.

Server-side validation uses `ajv` (draft 2020-12, `strict: false`, OpenAPI keywords ignored) compiled against the stored document and the named input schema. `ajv` stays server-side to avoid shipping a validator to the browser; the client performs immediate required/type/bounds checks for UX, and the server remains authoritative. This split avoids the bundle cost while keeping validation correct for raw JSON input.

### Model switching preserves only compatible values

The form keeps values in a keyed object and, on schema change, retains a value when the new schema has a field with the same name and compatible type; everything else is dropped. `prompt` is therefore preserved across Flux variants, while `num_inference_steps` disappears for a model that does not declare it.

### Defaults are materialized at submit time

The UI shows defaults, but the server also injects schema defaults for missing fields before validation and before calling fal. This keeps raw-JSON submissions and API clients consistent with the form.

### Uploads use base64 JSON, not multipart

`POST /api/upload` accepts `{ filename, contentType, data }` where `data` is base64, with `bodyParser.sizeLimit = '12mb'`. The Pages Router has no built-in multipart parser; adding one is unnecessary for a single-file field. The route validates decoded size (8 MB), declared content type, and magic bytes (JPEG/PNG/WebP), converts the buffer to a `Blob`, calls `fal.storage.upload(blob)`, and returns `{ url }`. No bytes are written to `public/`.

Alternatives considered: `formidable` (extra dependency, streaming not needed at 8 MB), direct browser-to-fal upload (would expose the API key).

### Image fields are identified by name and shape

A field is rendered as an image control when its name matches `(^|_)(image|images|image_url|image_urls|init_image|mask|reference_image|reference_images)($|_)` and its type is string or array of strings, or when the model's catalog tags include `capability:multiple-image-inputs`. Fields named `*_urls` or array-typed accept multiple images; everything else accepts one.

Each slot offers exclusive sources: upload, pasted URL, the currently previewed image, and — for optional fields only — no image at all. Choosing one clears the others so a stale URL or file can never be submitted alongside the active choice. When a field is required, the no-image choice is not rendered.

### Reusing the previewed output as input

The Generate shell already tracks `imageUrl`, the image shown in the center preview: either the newest generation result or an older history thumbnail the user clicked. It passes that filename to the form so any image slot can adopt it.

History is derived from filenames in `public/outputs`, and the original fal media URL is not persisted, so reuse performs a server-side re-upload:

- `POST /api/upload/from-output` with `{ filename }`.
- Accept only a basename matching the output naming pattern with an image extension; resolve it under `public/outputs` and reject if the resolved path escapes that directory.
- Read the bytes, call `fal.storage.upload`, and return `{ url }`.

Why re-upload instead of reusing the original fal URL: `sourceUrl` is not persisted with history and fal media URLs are subject to expiry settings. Re-uploading is deterministic across reloads, works for any output format, and requires no schema change. Alternative considered: a `generations` table storing endpoint and source URL, deferred because it duplicates the filename-based history and adds migration work.

Re-hosting reuses the same blocked-submission behavior as uploads: while it is in progress or after it fails, the form identifies the field and prevents submit. The option is disabled when nothing is previewed or the preview is not a local output.

### Generation contract and output normalization

`POST /api/generateImage` now accepts `{ model, input }`:

1. Load the model row; reject unknown/unavailable.
2. Load the schema (fetching if needed); reject if unavailable.
3. Inject defaults, drop fields not declared by the schema, strip empty optional image fields, and validate with `ajv`.
4. Call `fal.subscribe(model, { input })`.
5. Extract image outputs in this order: `images[]`, `image`, `mask_image`, then a recursive scan for objects whose `content_type` starts with `image/` or whose URL ends in a known image extension. Deduplicate by URL.
6. Download each with global `fetch`, save to `public/outputs` with a model-slug + timestamp + index filename and the detected extension.
7. Return `{ imageUrl, images: [{ url, sourceUrl, type }], warnings }`.

`imageUrl` remains the primary (first) output so the existing preview and history keep working. `getGeneratedImages` is extended to list WebP/AVIF as well.

Partial failures keep successful outputs and return `warnings`; a failed primary output returns a 502 with the provider message when available.

### Error mapping

- Schema fetch failure → 502 `schema_unavailable`; the form offers prompt-only/raw JSON.
- Validation failure → 400 with `{ errors: [{ field, message }] }`.
- fal API error → provider status/message passed through with a stable `provider_error` type.
- Upload failure → 4xx/502 with a specific reason (type, size, credentials, storage).

## Risks / Trade-offs

- **Schema variety outpaces the renderer** → raw JSON fallback guarantees every model is usable; unsupported constructs are logged with the endpoint ID for future renderer work.
- **`anyOf`/nullable handling can mis-render** → treat `anyOf` with `null` as optional; anything more complex goes to raw mode.
- **8 MB base64 payloads can hit proxy limits** → documented limit and explicit error; a multipart path can replace it later without changing the client contract.
- **fal storage upload requires the key and network** → upload errors are surfaced before generation; no silent fallback to data URIs.
- **`ajv` strictness against OpenAPI keywords** → `strict: false` plus keyword filtering; validation only covers the input schema, not the OpenAPI envelope.
- **One fal call can exceed serverless timeouts** → this remains a local/self-hosted feature; document that long-running models need a longer function timeout.
- **Output scan may pick up non-image URLs** → restrict by `content_type` or extension and dedupe; mask/secondary outputs are explicitly named.
- **Form state grows complex** → keep values in a single keyed object and derive controls from the schema on each render; no per-model bespoke state.

## Migration Plan

1. Create `model_schemas` idempotently in `lib/db.js`; no migration of existing rows.
2. Ship the schema endpoint and validation before switching the form, so the legacy flat payload keeps working during development.
3. Switch `generateImage` to `{ model, input }` and update the form in the same release; no external API consumers exist.
4. Extend `getGeneratedImages` to additional extensions.
5. Rollback: revert code and drop `model_schemas`; catalog and selection data are unaffected.

## Open Questions

None that affect scope or tasks. Schema TTL (seven days) and upload size (8 MB) are defaults that can be tuned without changing the specs.
