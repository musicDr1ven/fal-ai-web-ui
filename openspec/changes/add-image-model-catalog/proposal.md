# Proposal

## Why

The Generate form only offers three hard-coded Flux variants (`app/page.js:177-179`), even though fal.ai exposes 576 image-output models. Users cannot discover, filter, or choose from the catalog without editing source code, and the app stores no model metadata, so there is no place to keep cost, purpose, or selection state.

## What Changes

- Add a local SQLite database for image-model metadata: endpoint ID, title, description, category, fine-grained tasks/tags, lab/family/hosting, thumbnail, license, pricing (unit price, unit, currency), selected flag, and timestamps.
- Add a **Refresh models** API that pages the public fal models catalog, keeps only models whose discovery tags include `output:image`, and upserts them while preserving existing selections. Throttled and resumable on rate limits.
- Populate pricing from fal's structured pricing API when `FAL_KEY` is present; fall back to the catalog's prose `pricingInfoOverride`; otherwise show no price. No secrets are stored in the database.
- Seed the three currently hard-coded models as selected on first initialization so existing behavior is preserved.
- Add a two-tab layout: **Generate** (existing form) and **Models** (new).
- Add a Models tab with a refresh button, category filter, text search, scrollable model cards (thumbnail, title, lab, category, price), and a checkbox per card.
- Checkbox state persists in SQLite and drives the model drop-down on the Generate tab. The drop-down is built from selected models instead of a literal list.
- Add API endpoints for listing/filtering models, toggling selection, and reading refresh status.
- Keep the catalog scoped to image endpoints only; video, audio, 3D, and training endpoints are excluded from refresh.

## Capabilities

### New Capabilities
- `image-model-catalog`: catalog synchronization from fal, SQLite persistence, image-only filtering, pricing resolution, and selection state.
- `model-browser-ui`: the Models tab experience — tab navigation, refresh control, filtering/search, model cards, and checkbox-driven drop-down synchronization.

### Modified Capabilities

<!-- None: no existing specs exist in this project. -->

## Impact

- **New dependency**: `better-sqlite3` (native SQLite driver) plus a typed/validated data-access module.
- **New files**: `lib/db.js`, `lib/falModels.js`, `pages/api/models/index.js`, `pages/api/models/refresh.js`, `pages/api/models/[id]/selection.js`, `app/models/` UI components, `data/` database directory.
- **Modified files**: `app/page.js` (tab shell, drop-down from DB), `.gitignore` (database file), `package.json`.
- **External APIs**: `https://fal.ai/api/models` (public catalog, page size 40, throttled), `https://api.fal.ai/v1/models/pricing` (authenticated, optional), all server-side.
- **Environment**: `FAL_KEY` enables structured pricing only; refresh works without it.
- **Persistence**: SQLite file under `data/` is local to the running instance. Serverless deployments (e.g. Vercel) have ephemeral filesystems, so this feature targets local/self-hosted use.
- **Compatibility**: initial seed keeps the existing three models selected, so current generate behavior is unchanged until the user refreshes and changes selections.
