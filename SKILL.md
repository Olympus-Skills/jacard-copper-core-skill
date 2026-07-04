---
name: jacard-copper-core-skill
description: Use when an agent needs retail catalog answers through the Jacard Copper Core MCP — keyword/semantic/hybrid product search, deterministic inventory lists, product detail, collection facets, and readonly SQL over llm_* views. Assumes the MCP is already connected with a valid key.
---

# Jacard Copper Core MCP — usage guide

How to get good results from the **Jacard Copper Core MCP** (retail catalog retrieval). The MCP is already connected and authenticated for you; this skill is about using its tools well. Calls are metered per API key (allowed store slugs, a per-minute rate limit, and a daily request budget), so behavior below follows from your key.

## Start with get_key_info

Call `get_key_info` first — it's **free** (never debits your budget). It returns your allowed store slugs, whether you have all-store access, your rate/day limits, and today's remaining budget. What you can do follows from it, so check it before planning a sequence of calls.

## Scoping

- Store tools take a `slug`. With a single-store key you may omit it (it defaults to your one store); **all-store (`*`) and multi-store keys must pass `slug` explicitly** — omitting it errors with `invalid_arguments`. A slug outside your key's allowlist fails with `store_not_allowed`.
- All-store keys (`*`) additionally get the global tools (`global_keyword_search_products`, `global_semantic_search_products`, `global_hybrid_search_products`, `global_list_inventory_products`, `readonly_sql_global`). Store-scoped keys don't see them at all.
- A multi-store key (no wildcard) queries one slug at a time.

## Choosing a tool

- `keyword_search_products` — fast lexical match. Exact terms, SKUs, product names.
- `semantic_search_products` — embedding similarity. Vague/descriptive queries ("minimal office earrings") and image queries (`image_url` or `image_base64` + `image_mime_type`).
- `hybrid_search_products` — keyword + semantic fused. Best default for natural-language product discovery.
- `list_inventory_products` — deterministic listing with exact `total_count` and stable cursors. Use for "all / how many products" requests, never for relevance ranking.
- `get_product_detail` — full record (variants, images, text chunks) by `product_id`.
- `list_store_collections` — valid collection handles + price bounds before filtering a search by collection.
- `readonly_sql_store` — SELECT over store-scoped `llm_*` views (max 500 rows) for counts, aggregates, audits, exact checks.
- `readonly_sql_global` (all-store keys) — cross-store views like `llm_stores`, `llm_global_products` (max 1000 rows).

Rules of thumb: prefer the search tools for recommendations and product picks; prefer SQL for counts and exact facts. Discover valid collection handles with `list_store_collections` before filtering by `collection`. Paginate with the returned `cursor` / `next_cursor` rather than inflating `limit`.

## Operating within budgets and errors

- Every tool call except `get_key_info` debits 1 request from the daily budget. Budgets reset at UTC midnight.
- Exceeding requests-per-minute → HTTP 429 with `Retry-After` seconds. Back off, then retry once the window passes.
- Tool-level failures come back with `isError` and a JSON body `{ "error": { "code", "message", "details?" } }`:

| Code | Meaning | What to do |
|---|---|---|
| `invalid_arguments` | e.g. `slug` omitted on a multi-store key | Fix arguments |
| `store_not_allowed` | Slug outside the key's allowlist | Use an allowed slug from `get_key_info` |
| `scope_not_allowed` | Global tool used by a store-scoped key | Not available on your key |
| `daily_limit_exceeded` | Day budget exhausted (includes `resets_at`) | Stop; retry after the UTC reset |
| `upstream_error` | Copper Core rejected the call (bad SQL, etc.) | Read `message`, fix, retry |

Transport-level: HTTP 401 (bad/revoked key), 403 (key not enabled for this service), 429 (rate limit — honor `Retry-After`).

---

# Tool reference

All results come back as JSON text. Product payloads are compacted (primary image, price range, variant count) and capped at 200 entities / ~60k chars, with `truncated: true` when clipped.

## get_key_info

Free. No arguments.

```json
{
  "service": "copper-core",
  "owner": "alice@example.com",
  "allowed_store_slugs": ["giva"],
  "all_store_access": false,
  "limits": { "requests_per_minute": 60, "requests_per_day": 2000 },
  "usage_today": { "unit": "requests", "used": 12, "remaining": 1988, "resets_at": "2026-07-04T00:00:00.000Z" }
}
```

## Search tools (store-scoped)

`keyword_search_products`, `semantic_search_products`, `hybrid_search_products`. Arguments (all optional unless noted):

| Field | Type | Notes |
|---|---|---|
| `slug` | string | Required unless your key has exactly one store |
| `query` | string | Text query |
| `image_url` / `image_base64` + `image_mime_type` | string | Semantic/hybrid only; image similarity |
| `limit` | int 1-100 | Default 20 |
| `cursor` | string | Opaque pagination cursor from the previous response |
| `collection` | string[] | Collection handles (see `list_store_collections`) |
| `min_price` / `max_price` | number | Price filter |
| `availability` | `all\|available\|in_stock` | |
| `sort` | `updated_desc\|price_asc\|price_desc\|title_asc\|relevance` | |
| `min_semantic_score` | 0-1 | Default 0.4; semantic/hybrid |
| `per_embedding_limit` | int | Semantic/hybrid; per-embedding candidate cap before fusion |
| `family_keys` | string[] | Embedding families; omit to use all active |
| `target_modalities` | ("text"\|"image")[] | |
| `text_doc_type` | ("seo"\|"metadata"\|"description"\|"image_caption")[] | |

Response: `{ products: [...], pagination: { cursor, has_more }, returned_count, total_count?, truncated }` — each product has `product_id`, `title`, `vendor`, `product_type`, `price_range`, `primary_image`, `store_slug`, `store_url`, `collection_handles`, `variant_count`, availability flags.

```json
{ "name": "hybrid_search_products", "arguments": { "slug": "giva", "query": "rose gold pendant under 3000", "max_price": 3000, "limit": 10 } }
```

## list_inventory_products

Deterministic inventory listing (exact counts, stable ordering). Arguments: `slug`, `query`, `collection`, `min_price`, `max_price`, `availability`, `sort`, `limit` (1-200, default 50), `cursor`. Response includes `applied_filters`, `pagination.total_count`, `definition.total_count_basis`.

## get_product_detail

Arguments: `slug`, `product_id` (required). Response: `{ product: { ...variants[≤20], images[≤8] }, chunks[≤8], embedding_status, store }`.

## list_store_collections

Arguments: `slug`. Response: `{ collections: [{ handle, title, product_count, ... }], price_bounds }`. Use before filtering searches by `collection`.

## readonly_sql_store

Arguments: `slug`, `sql` (required), `params?` (positional `$1`… values), `max_rows?` (≤500), `timeout_ms?`. SELECT only, single statement, no semicolons, `llm_*` views only.
Response: `{ columns: [{name, data_type}], rows: [...], row_count, execution_ms, truncated }`.

```json
{ "name": "readonly_sql_store", "arguments": { "slug": "giva", "sql": "select count(*) as n from llm_products where in_stock" } }
```

## Global tools (all-store keys only)

`global_keyword_search_products`, `global_semantic_search_products`, `global_hybrid_search_products`, `global_list_inventory_products` — same args as the store variants but no `slug`; optional `slugs: string[]` filters to a subset. `readonly_sql_global` — like `readonly_sql_store` without `slug`; max 1000 rows.

---

# SQL over llm_* views

Use only `llm_*` views. Never query raw tables, never use semicolons, prefer explicit column lists (`select *` only for a one-row schema probe). Do not invent columns like `id`, `name`, `product_name`, `is_ready`. Canonical columns: `product_id` (products), `slug` (stores), `title` (titles), `status` (readiness/state), `collection_handle` (collection filters), `family_key` (embedding family), `estimated_cost_usd` (cost).

`raw_product_type` is the product-type column in SQL (search **responses** call it `product_type`). It's free-form and inconsistent — `Ring`, `Rings`, `Gold Ring`, `Demi Fine Rings` coexist, and `Earrings` contains "ring". Run `select distinct raw_product_type from llm_products` before filtering, then use an explicit `in (...)` list or an anchored pattern; `ilike '%ring%'` also matches earrings, so exclude them with `and raw_product_type not ilike '%earring%'`.

## Store-scoped views (`readonly_sql_store`)

- `llm_products` — `product_id`, `external_product_id`, `canonical_url`, `title`, `brand`, `raw_product_type`, `currency`, `min_price`, `max_price`, `available_for_sale`, `in_stock`, `status`, `created_at`, `updated_at`.
- `llm_variants` — `product_id`, `product_title`, `variant_id`, `sku`, `color`, `size`, `material`, `unit_price`, `compare_at_price`, `inventory_qty`, `inventory_status`, `option_values_json`, `status`, `updated_at`.
- `llm_assets` — `product_id`, `asset_id`, `variant_id`, `asset_type`, `role`, `asset_url`, `width`, `height`, `sort_order`, `alt_text`, `caption_text`, `status`, `updated_at`.
- `llm_collections` — `collection_id`, `handle`, `title`, `description`, `canonical_url`, `image_url`, `status`, `created_at`, `updated_at`.
- `llm_product_collections` — `product_id`, `product_title`, `collection_id`, `collection_handle`, `collection_title`, `status`.
- `llm_text_chunks` — `product_id`, `doc_type`, `chunk_id`, `chunk_type`, `section_path`, `chunk_seq`, `chunk_text`, `display_snippet`, `token_count`, `weight`, `status`.
- `llm_embedding_bindings` — `product_id`, `family_key`, `entity_type`, `entity_id`, `status`, `generated_at`, `error_json`.

## Global views (`readonly_sql_global`)

- `llm_stores` — `store_id`, `slug`, `domain`, `connector_type`, `locale`, `currency`, `status`.
- `llm_store_metrics` — `store_slug`, `counts_json`, `embedding_coverage_json`, `embedding_cost_json`, `refreshed_at`.
- `llm_latest_runs` — `store_slug`, `crawl_run_id`, `run_type`, `status`, `created_at`, `finished_at`, `error_json`.
- `llm_product_counts_by_store` — `store_slug`, `products_total`, `products_active`, `products_available`, `products_in_stock`, `min_price`, `max_price`.
- `llm_collection_counts_by_store` — `store_slug`, `collections_total`, `collections_active`.
- `llm_embedding_costs` — `store_slug`, `family_key`, `modality`, `entity_type`, `estimated_tokens`, `estimated_cost_usd`.
- `llm_global_products` — cross-store product rows; mirrors `llm_products` columns plus `store_slug`.

## Example queries

```sql
-- active products
select product_id, title, min_price, max_price, currency
from llm_products where status = 'active' order by title limit 10
```
```sql
-- availability + price aggregate
select count(*) as products_active,
       count(*) filter (where available_for_sale) as products_available,
       count(*) filter (where in_stock) as products_in_stock,
       min(min_price) as min_price, max(max_price) as max_price
from llm_products where status = 'active'
```
```sql
-- products joined to collections
select p.product_id, p.title, pc.collection_handle
from llm_products p
join llm_product_collections pc on pc.product_id = p.product_id
where p.status = 'active' and pc.status = 'active' limit 20
```
```sql
-- (global) ready stores with counts
select s.slug, s.status, pc.products_active, cc.collections_active
from llm_stores s
left join llm_product_counts_by_store pc on pc.store_slug = s.slug
left join llm_collection_counts_by_store cc on cc.store_slug = s.slug
order by s.slug limit 20
```
