# Tasks

## 1. Data foundation

- [ ] 1.1 Add `better-sqlite3` to `package.json` dependencies, install it, and verify `node -e "require('better-sqlite3')"` exits 0
- [ ] 1.2 Create `data/` with a `.gitkeep`, add `data/*.db*` to `.gitignore`, and verify `git status` shows no database file
- [ ] 1.3 Implement `lib/db.js` with a `globalThis`-cached connection, WAL mode, and schema creation for `models` and `meta`; verify by running a Node script that opens the DB and lists the created tables
- [ ] 1.4 Seed the three legacy endpoints as selected during first initialization and verify a fresh DB lists exactly them with `is_selected = 1`

## 2. fal catalog client

- [ ] 2.1 Implement paged catalog fetching in `lib/falModels.js` (`page`, `size=40`, `total`/`pages` handling) and verify a script fetches page 1 and reports 40 items with the correct total
- [ ] 2.2 Add the `output:image` filter plus exclusion of non-public/training endpoints, and verify a fixture page yields fewer rows than it contains and contains no video/audio endpoints
- [ ] 2.3 Add throttling and 429 backoff with a bounded retry count, and verify a simulated 429 response is retried and does not throw out of the fetcher
- [ ] 2.4 Implement upsert that preserves `is_selected`, sets `last_seen_at`, and marks absent rows `unavailable`; verify re-running an import changes no selected flags and marks a removed fixture endpoint unavailable

## 3. Pricing

- [ ] 3.1 Implement structured pricing retrieval from `https://api.fal.ai/v1/models/pricing` using `Authorization: Key ${FAL_KEY}` with cursor pagination; verify with a key that a known endpoint receives `unit_price`, `unit`, and `currency`
- [ ] 3.2 Implement fallback to `pricingInfoOverride` prose and to unknown, and verify a refresh without `FAL_KEY` still completes with prices marked unknown or prose
- [ ] 3.3 Confirm no credential is persisted or serialized to the client by grepping the DB file and an API response for the configured key value

## 4. Catalog API

- [ ] 4.1 Add `POST /api/models/refresh` with single-flight protection and a summary payload; verify it returns `added`/`updated` counts and a second concurrent call returns an in-progress status
- [ ] 4.2 Add `GET /api/models` with category filter, text search, selection filter, and pagination; verify `?selected=1` returns only selected rows and `?q=` narrows results
- [ ] 4.3 Add `PATCH /api/models/[id]/selection`; verify toggling on and off persists and that an unknown ID returns 404
- [ ] 4.4 Add `GET /api/models/selection` for the drop-down; verify it returns the exact selected set after a toggle

## 5. Models tab UI

- [ ] 5.1 Refactor `app/page.js` into a tab shell and extract the existing form into `app/components/GenerateForm.js`; verify both tabs render and the form still generates with a seeded model
- [ ] 5.2 Build `app/components/ModelsTab.js` with a refresh button, loading state, and added/updated result summary; verify a refresh updates the summary and a failed refresh leaves the list visible
- [ ] 5.3 Build model cards showing thumbnail, title, category, lab/family when present, and a formatted price or unknown indicator; verify cards render both priced and unknown-price fixtures
- [ ] 5.4 Add category filter, search, result count, and empty state; verify each filter changes the count and a nonsense query shows the empty state
- [ ] 5.5 Wire card checkboxes to `PATCH /api/models/[id]/selection` with optimistic update and rollback on failure; verify the checkbox reflects persistence after a reload
- [ ] 5.6 Add incremental "load more" pagination (60 per page) and lazy thumbnails; verify the initial view fetches one page and scrolling/loading adds the next page

## 6. Generate integration

- [ ] 6.1 Populate the Generate model drop-down from selected models and keep it in sync after toggles; verify selecting a model in Models makes it immediately available in Generate
- [ ] 6.2 Disable generation with an explanatory message when no models are selected; verify deselecting the last model disables submission and reselecting re-enables it
- [ ] 6.3 Verify an end-to-end generation still works with a seeded model and that the selected endpoint ID reaches `fal.subscribe`

## 7. Documentation and integration

- [ ] 7.1 Document the catalog, SQLite file location, refresh behavior, and pricing requirements in `README.md`; verify the documented steps run as written
- [ ] 7.2 Run `npm run build` and verify it succeeds with the new dependency and API routes
- [ ] 7.3 Smoke-test the full flow: fresh DB → refresh → filter → select a non-Flux model → confirm the drop-down and selection persist across a dev-server restart

## Workflow follow-up

- Archive the change after the project's review requirements are satisfied.
- Verify the archived spec reflects the implemented behavior.
