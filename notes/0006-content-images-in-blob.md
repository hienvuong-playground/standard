# 0006 – Content images in Blob, served via the API

## Context

All images lived in `frontend/public/assets/`. Product rows in Postgres pointed at them (`products.image = '/assets/headphones-1.jpg'`), so the data was in the backend but the files were in the frontend. The homepage banners (promos) and category tiles lived in `frontend/src/storefront.json`, bundled into the JS and never touching the API.

Two kinds of asset were mixed together:

- **UI assets** (font, icon sprite): part of the design, change with the code.
- **Content assets** (product, promo, category images): part of the data, change with the content.

## Decision

**Content images go to Blob Storage, the DB stores a key, the API returns full URLs. UI assets stay in the frontend.** See 0007 for the caching side.

- **Storage**: a dedicated account `stplaygroundassets` with public container `assets` (`infra/cloud/assets.tf`). It is separate from the observability account because public access is an account-level switch.
- **Keys are content hashes**: `images/<sha256[:12]>.jpg`, in one flat folder. Products 13–38 reuse the category/promo images, so per-type folders would have meant duplicate files. With hash keys, identical content lands on the same blob.
- **DB stores the key, not the URL**: the backend adds `ASSET_BASE_URL` (`backend/src/app.js`, `productFromRow` in `backend/src/catalog.js`). If the base changes later (Front Door, custom domain), only the env var changes and the data needs no migration. `ASSET_BASE_URL` comes from the Terraform output `assets_base_url` through a Flux substitution. The app refuses to start without it.
- **Promos/categories are content too**: they are now Postgres tables (`backend/src/db/migrations/003-storefront.sql`) behind `GET /api/storefront`, and `HomePage.jsx` fetches them. Before, they were frontend config only because nobody had questioned it.
- **No CDN for now**: on Azure, new CDN means Front Door Standard/Premium. Azure CDN from Microsoft (classic) stopped accepting new profiles on 2025-08-15 and retires on 2027-09-30. Plain Blob + `Cache-Control` is enough at this scale. Front Door can be added in front later by changing `ASSET_BASE_URL`.
- **Upload with Terraform, for now**: `azurerm_storage_blob` with `for_each` over `infra/cloud/assets/`, and CI apply uploads. The proper flow is an admin page: the backend issues a short-lived write SAS (user delegation, no account key), the client uploads straight to Blob, and the backend saves the key. That flow doesn't exist yet because there is no admin page.

Rejected:

- **Fixed names** (`headphones-1.jpg`): can't be cached as immutable (see 0007).
- **Seed upsert instead of migration 002**: would make `catalog.json` overwrite the DB on every pod start. The existing insert-only seed was kept.

## How data gets into the DB (no admin page yet)

- `backend/src/data/catalog.json` and `storefront.json` are **seed data, not the data store**. The API only reads Postgres.
- The seed runs in the backend initContainer (`db/bootstrap.js`, migrate then seed) on every pod start. It uses `ON CONFLICT (id) DO NOTHING`, so it only inserts **new ids**.
- Uploading a blob does **not** update the DB. Blob doesn't know which product an image belongs to. Adding or changing an image is two manual steps:
  1. Add the file to `infra/cloud/assets/`. Its key is `images/` + the first 12 chars of `shasum -a 256 <file>` + the extension.
  2. Point the DB at it:
     - New product/promo/category: add it to the seed JSON, and the seed inserts it.
     - **Existing row: write a migration** (`UPDATE ... SET image = '<key>' WHERE id = ...`), because the seed skips existing ids. `002-product-image-keys.sql` is the one-time conversion of the existing rows from paths to keys.
- Once an admin page exists, it writes the DB directly and these steps go away. The seed JSON stays as fixture data for local dev and tests (`backend/test/catalog.test.js`), and should no longer run in production.

## Consequences

- **Deleting an image deletes the blob on the next apply.** A tab opened before the change may 404 on a lazily loaded image until reload. This is accepted here. The cart in `localStorage` stores only ids, so URLs are always fresh from the API.
  - Lifecycle rules can't fix this. "Noncurrent version" applies to overwritten fixed-name blobs, and age-based rules count from upload, not from when an image stopped being used.
  - The real fix, once there is an admin page: a mark-and-sweep job that compares blobs against the keys in the DB, tags orphans (`orphaned-since`), and deletes them after N days.
- **Rollout order** across the three repos: infra apply → backend deploy → frontend deploy. The frontend no longer ships the images, so deploying it first breaks them.
- **Shared key is disabled**, so the provider needs `storage_use_azuread = true` (`infra/cloud/providers.tf`) to write blobs. Per the azurerm docs, *changing* `container_access_type` on an existing container always uses Shared Key, so that change would fail on this account.
- The CI identity only had Blob Data Contributor on the tfstate account, so `assets.tf` grants it on the `assets` container. A new role assignment can take minutes to propagate, so the first apply may hit 403 on upload; re-running fixes it.
- Verified locally: `terraform validate` on both stacks, `npm run build` (only hashed files in `dist/assets/`), and backend tests against local Postgres (11/11, including `/api/storefront` and image URLs). The cloud rollout is not verified yet.

## References

- https://learn.microsoft.com/en-us/azure/cdn/cdn-overview (classic CDN retirement)
- https://learn.microsoft.com/en-us/azure/storage/blobs/storage-lifecycle-management-concepts
- https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs#storage_use_azuread
- https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/storage_account (note under `shared_access_key_enabled`)
