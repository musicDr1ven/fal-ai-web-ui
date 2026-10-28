# Tasks

## 1. Schema storage and API

- [ ] 1.1 Add `ajv` to dependencies and verify `node -e "require('ajv')"` exits 0
- [ ] 1.2 Add the `model_schemas` table and its creation guard to `lib/db.js`; verify an existing database opens unchanged and lists the new table
- [ ] 1.3 Implement `lib/modelSchema.js` with lazy fetch from the fal OpenAPI endpoint, document persistence, seven-day TTL, and in-flight deduplication; verify two simultaneous requests produce one outbound fetch
- [ ] 1.4 Add `GET /api/models/[id]/schema` returning the document and input schema name, with `stale` state; verify a cached model makes no outbound call and a stale failure returns the cached document
- [ ] 1.5 Reject schema requests for unknown/unavailable endpoints and invalid endpoint ID formats; verify both return 4xx without contacting fal

## 2. Schema-driven form

- [ ] 2.1 Implement a descriptor builder that resolves `$ref`, `anyOf`, enums, bounds, arrays, and required flags from the document; verify it produces the expected field list for `fal-ai/flux/dev` and `fal-ai/birefnet`
- [ ] 2.2 Render primitives (enum select, boolean checkbox, bounded numeric inputs, string inputs/areas) with schema titles and descriptions; verify each control type appears for a Flux schema
- [ ] 2.3 Apply schema defaults on load and group optional fields under a collapsed Advanced section; verify defaults are submitted without touching Advanced
- [ ] 2.4 Add client-side required/type/bounds validation that blocks submission and identifies the offending field; verify an empty required prompt blocks submit
- [ ] 2.5 Render the dedicated LoRA array editor when the schema declares an array of `{path, scale}`; verify add/remove produces the expected array
- [ ] 2.6 Add raw JSON mode for unsupported constructs with parse validation before submit; verify an unsupported fixture switches modes and invalid JSON cannot be submitted
- [ ] 2.7 Preserve compatible values across model switches and drop incompatible ones; verify a typed prompt survives Flux→Flux and Flux-only parameters disappear for birefnet
- [ ] 2.8 Show schema loading state and keep submit disabled until the current model's schema or fallback is ready; verify switching to an uncached model disables submit while loading

## 3. Image input

- [ ] 3.1 Implement `POST /api/upload` accepting base64 JSON with an 8 MB decoded limit, type/magic-byte validation, and fal storage upload; verify a valid PNG returns a hosted URL
- [ ] 3.2 Reject non-image, oversized, and credential-missing uploads with specific messages; verify each case returns the documented status
- [ ] 3.3 Render image fields with exclusive upload, URL, previewed-image, and no-image sources, driven by field name/type and `capability:multiple-image-inputs`; verify an optional image field shows all four choices and a required field hides no-image
- [ ] 3.4 Support multiple images for array image fields and submit them in the declared shape; verify two uploads produce a two-element array
- [ ] 3.5 Block submission while an upload is in progress or failed and identify the affected field; verify submit stays disabled during upload
- [ ] 3.6 Implement `POST /api/upload/from-output` that accepts a basename, validates extension and path containment under `public/outputs`, and re-hosts the file through fal storage; verify an existing output returns a hosted URL and traversal or unknown filenames are rejected
- [ ] 3.7 Wire the previewed-image option to the current center-preview filename, show its preview, and disable it with an explanation when nothing is previewed; verify clicking an older history thumbnail changes which image is adopted
- [ ] 3.8 Surface re-host failures on the field and block submission until resolved; verify a deleted output file produces a field error and no generation request
- [ ] 3.9 Submit a cleared optional image by omitting the field; verify a field that held an image and is then set to No image input produces no `image_url`/`image_urls` key, while a required field still blocks submission

## 4. Generation execution

- [ ] 4.1 Rewrite `pages/api/generateImage.js` to accept `{ model, input }`, load the schema, inject defaults, drop undeclared fields, strip empty optional image fields, and validate with `ajv`; verify missing required fields and unknown parameters produce the expected input and an omitted optional image reaches fal as no field
- [ ] 4.2 Replace the missing `node-fetch` import with global `fetch` and verify the route imports cleanly under Node 24
- [ ] 4.3 Map validation, schema, and provider failures to the documented statuses and messages; verify a provider error surfaces its message
- [ ] 4.4 Implement output extraction for `images[]`, `image`, `mask_image`, and nested image objects with URL deduplication; verify each fixture shape yields the expected list
- [ ] 4.5 Save every extracted image under `public/outputs` with the detected extension and return `{ imageUrl, images, warnings }`; verify a two-image fixture saves two files and a one-failure fixture returns the survivor plus a warning
- [ ] 4.6 Extend `getGeneratedImages` to include WebP/AVIF and verify a saved WebP appears in the history response

## 5. Integration and documentation

- [ ] 5.1 End-to-end text-to-image: select `fal-ai/flux/dev`, use generated controls, generate, and verify the image saves and appears in history
- [ ] 5.2 End-to-end image-to-image: select `fal-ai/birefnet`, generate once, then feed the previewed output back in as input and verify both background-removed output and mask are saved
- [ ] 5.3 Run `npm run build` and verify it succeeds with `ajv`, the new routes, and no missing imports
- [ ] 5.4 Document schema fetching, raw JSON mode, upload limits, and the `{ model, input }` API contract in `README.md`; verify the documented examples match the running API

## Workflow follow-up

- Apply `add-dynamic-model-parameters` only after `add-image-model-catalog` is implemented and archived.
- Archive this change after the project's review requirements are satisfied.
