# Design

## Context

See `proposal.md` for motivation and `specs/image-model-catalog/spec.md` / `specs/model-browser-ui/spec.md` for required behavior.

Current state that shapes the approach:

- Next.js 14 App Router page (`app/page.js`, client component) with three Pages Router API routes under `pages/api/`. There is no database, auth, or test suite.
- `app/page.js` already holds all generation state, and `pages/api/generateImage.js` forwards a fixed Flux-shaped input object to `fal.subscribe(model, { input })`.
- The public fal catalog exposes 1,447 endpoints; 576 declare `output:image`. Pagination is capped at 40 per page, so a complete image-catalog refresh is ~37 HTTP calls.
- Fine-grained purpose lives in `discovery.tasks` / `discovery.tags` (`image-editing`, `remove-background`, `task:*`, `output:image`), not in the coarse `category` field.
- Structured pricing requires `Authorization: Key $FAL_KEY`; the public catalog only carries inconsistent prose in `pricingInfoOverride`.
- Rapid pagination triggers HTTP 429, so refresh must be throttled and resilient.

## Goals / Non-Goals

**Goals:**

- One durable local catalog that both the Models tab and the Generate drop-down read from.
- Refresh that is safe to interrupt, repeat, and partially fail without corrupting selection.
- Price data that is honest about its source and confident only when structured data exists.
- UI that stays responsive with ~600 cards and does not require a page reload to sync selection.

**Non-Goals:**

- Dynamic per-model parameter forms, schema caching, and image upload — deferred to `add-dynamic-model-parameters`.
- Multi-user accounts, remote/shared catalog, or background job infrastructure.
- Video/audio/3D/training endpoints and non-image pricing.

## Decisions

### SQLite with `better-sqlite3`

Use a single SQLite file at `data/models.db` via `better-sqlite3`. It is synchronous, zero-configuration, and adequate for a single-process Next server; it avoids ORM codegen and migration tooling for a two-table schema.

Alternatives considered:
- `sqlite3`: async callback API adds noise without benefit at this scale.
- Prisma/Drizzle: adds schema/migration machinery disproportionate to two tables.
- JSON file: cannot express selection updates, filtering, or partial refresh safely.

The connection is created once and cached on `globalThis` so Next.js dev hot-reload does not leak file handles. `PRAGMA journal_mode = WAL` and `PRAGMA busy_timeout` are set at open time.

### Catalog source: public `fal.ai/api/models`

Refresh reads `https://fal.ai/api/models?page=N&size=40` because it returns `discovery.tags` (needed to filter `output:image` and to retain fine-grained tasks), facets, prose pricing, and a reliable `total`/`pages` pair. The platform API `https://api.fal.ai/v1/models` was rejected as the primary source because its `category` is coarse and it exposes no output-modality tag; it cannot distinguish background removal from generic image-to-image.

Refresh is sequential with a short delay between pages (~250 ms) plus exponential backoff on 429. It is single-flight: a module-level promise plus a `refresh_runs` marker in SQLite prevents a second import in the same or another request.

### Image-only filter and availability lifecycle

A model is eligible when `discovery.tags` includes `output:image`. Endpoints whose `kind` or task indicates training, or whose status is not public, are skipped. Rows are never deleted by refresh: models missing from a completed import get `is_available = 0`, so previously selected rows keep their identity and simply stop being offered.

### Pricing resolution order

1. Structured: paginate `https://api.fal.ai/v1/models/pricing` with `Authorization: Key ${FAL_KEY}`, map by `endpoint_id`, store `unit_price`, `unit`, `currency`.
2. Prose fallback: `pricingInfoOverride` is stored as display text with no numeric price.
3. Unknown: both sources absent; UI shows an explicit unknown indicator.

The key is read from the request-time environment only and never written to SQLite or returned to the client.

### Schema

```sql
CREATE TABLE models (
  endpoint_id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  description TEXT,
  category TEXT,
  tasks TEXT,                -- JSON array from discovery.tasks
  tags TEXT,                 -- JSON array from discovery.tags
  output_modalities TEXT,    -- JSON array derived from tags
  lab TEXT,
  family TEXT,
  hosting TEXT,
  thumbnail_url TEXT,
  model_url TEXT,
  license_type TEXT,
  status TEXT,
  unavailable INTEGER NOT NULL DEFAULT 0,
  is_selected INTEGER NOT NULL DEFAULT 0,
  unit_price REAL,
  price_unit TEXT,
  currency TEXT,
  pricing_text TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  last_seen_at TEXT
);

CREATE TABLE meta (
  key TEXT PRIMARY KEY,
  value TEXT
);
```

Indexes on `category`, `is_selected`, and `unavailable` support the list/filter queries.

### API surface

- `GET /api/models?category=&q=&selected=1&page=&pageSize=` — paginated list, default `pageSize=60`, returns `items`, `total`, and filter facets derived from stored rows.
- `POST /api/models/refresh` — runs or reports the single-flight import; returns `{ status, added, updated, unavailable, pricing, error? }`.
- `PATCH /api/models/[id]/selection` — body `{ selected: boolean }`; returns the updated row.
- `GET /api/models/selection` — lightweight list for the Generate drop-down.

All SQL uses bound parameters. The endpoint ID is URL-encoded in the path segment.

### UI structure

`app/page.js` becomes a client shell that owns tab state and selected-model state. Generate UI moves to `app/components/GenerateForm.js`; model discovery to `app/components/ModelsTab.js`; shared card and price formatting in small components. Selection is lifted to the shell so the drop-down updates without refetching on every switch.

The card grid uses incremental "load more" pagination (60 rows per request) plus `loading="lazy"` thumbnails rather than a virtualization dependency. With 576 image models this keeps DOM size and network bounded while staying simple; virtualization can replace it later without changing the API.

### Seeding

On first database creation, insert the three legacy endpoints (`fal-ai/flux-lora`, `fal-ai/flux/dev`, `fal-ai/flux-realism`) with `is_selected = 1` and minimal metadata. Refresh overwrites their metadata but never clears selection. This makes the change invisible until the user opts into discovering more models.

## Risks / Trade-offs

- **Rate limiting (429) during refresh** → sequential pages, 250 ms spacing, exponential backoff with a cap, and partial-success reporting that keeps prior rows.
- **Refresh latency (~37 requests)** → synchronous request with an in-progress state; the UI must not block other tabs. If this proves too slow, a streaming/background variant can be added later without changing the catalog contract.
- **`better-sqlite3` native build failures** → pin a compatible major, document Node >= 18, and keep the data layer isolated in `lib/db.js` so a driver swap is contained.
- **Ephemeral filesystem on serverless deployment** → the README already targets local/Vercel use; document that the catalog is instance-local and requires a persistent volume for self-hosting beyond a single process.
- **Catalog drift and renamed endpoints** → upsert by endpoint ID, keep unavailable rows, and never auto-select.
- **Prose pricing is not comparable across models** → store it as display-only text; never derive numeric costs from it.
- **Next.js dev hot reload opening duplicate DB handles** → `globalThis` singleton plus WAL mode.

## Migration Plan

1. Add `better-sqlite3`, create `data/` and add `data/*.db*` to `.gitignore`.
2. Lazy-initialize schema and seed on first API call; no manual migration step.
3. Deploy with existing Generate behavior unchanged (three seeded models).
4. Rollback: revert the application files and delete `data/models.db*`; the legacy drop-down list returns with the prior code.

No existing API or database is modified, so rollback has no data-loss impact beyond the new catalog file.

## Open Questions

None at this stage. Whether refresh should run automatically on first visit was decided against: it is user-triggered only, and the seeded defaults cover first use.
