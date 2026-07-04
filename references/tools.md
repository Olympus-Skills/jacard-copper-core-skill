# Copper Core MCP — tool contracts

All results come back as JSON text in the tool result. Product payloads are compacted (primary image, price range, variant count) and capped at 200 entities / ~60k chars with `truncated: true` when clipped.

## get_key_info

Free. No arguments.

```json
{
  "service": "copper-core",
  "owner": "alice@example.com",
  "key_prefix": "hmk_56788e",
  "allowed_store_slugs": ["giva"],
  "all_store_access": false,
  "limits": { "requests_per_minute": 60, "requests_per_day": 2000 },
  "usage_today": { "unit": "requests", "used": 12, "remaining": 1988, "resets_at": "2026-07-04T00:00:00.000Z" }
}
```

## Search tools (store-scoped)

`keyword_search_products`, `semantic_search_products`, `hybrid_search_products`

Arguments (all optional unless noted):

| Field | Type | Notes |
|---|---|---|
| `slug` | string | Required unless your key has exactly one store |
| `query` | string | Text query |
| `image_url` / `image_base64` + `image_mime_type` | string | Semantic/hybrid only; image similarity |
| `limit` | int 1-100 | Default 20 |
| `cursor` | string | Opaque pagination cursor from previous response |
| `collection` | string[] | Collection handles (see `list_store_collections`) |
| `min_price` / `max_price` | number | Price filter |
| `availability` | `all\|available\|in_stock` | |
| `sort` | `updated_desc\|price_asc\|price_desc\|title_asc\|relevance` | |
| `min_semantic_score` | 0-1 | Default 0.4; semantic/hybrid |
| `family_keys` | string[] | Embedding families; omit to use all active |
| `target_modalities` | ("text"\|"image")[] | |
| `text_doc_type` | ("seo"\|"metadata"\|"description"\|"image_caption")[] | |

Response: `{ products: [...], pagination: { cursor, has_more }, returned_count, total_count?, truncated }` — each product has `product_id`, `title`, `vendor`, `product_type`, `price_range`, `primary_image`, `store_slug`, `store_url`, `collection_handles`, `variant_count`, availability flags.

Example:

```json
{ "name": "hybrid_search_products", "arguments": { "slug": "giva", "query": "rose gold pendant under 3000", "max_price": 3000, "limit": 10 } }
```

## list_inventory_products

Deterministic inventory listing (exact counts, stable ordering). Arguments: `slug`, `query`, `collection`, `min_price`, `max_price`, `availability`, `sort`, `limit` (1-200, default 50), `cursor`.

Response includes `applied_filters`, `pagination.total_count`, and `definition.total_count_basis`.

## get_product_detail

Arguments: `slug` (same rule as above), `product_id` (required).
Response: `{ product: { ...variants[≤20], images[≤8] }, chunks[≤8], embedding_status, store }`.

## list_store_collections

Arguments: `slug`. Response: `{ collections: [{ handle, title, product_count, ... }], price_bounds }`. Use before filtering searches by `collection`.

## readonly_sql_store

Arguments: `slug`, `sql` (required), `params?` (positional `$1`... values), `max_rows?` (≤500), `timeout_ms?`.

- SELECT only, single statement, no semicolons, only `llm_*` views (store-scoped: `llm_products`, `llm_variants`, `llm_assets`, `llm_collections`, `llm_product_collections`, `llm_text_chunks`, `llm_embedding_bindings`).
- Response: `{ columns: [{name, data_type}], rows: [...], row_count, execution_ms, truncated }`.

```json
{ "name": "readonly_sql_store", "arguments": { "slug": "giva", "sql": "select count(*) as n from llm_products where in_stock" } }
```

## Global tools (all-store keys only)

`global_keyword_search_products`, `global_semantic_search_products`, `global_hybrid_search_products`, `global_list_inventory_products` — same arguments as their store variants but no `slug`; optional `slugs: string[]` filters to a store subset.

`readonly_sql_global` — same contract as `readonly_sql_store` without `slug`; queries global views (`llm_stores`, `llm_store_metrics`, `llm_latest_runs`, `llm_product_counts_by_store`, `llm_collection_counts_by_store`, `llm_global_products`); max 1000 rows.

## Errors

Tool errors return `isError: true` with body `{ "error": { "code", "message", "details?" } }`:

| Code | Meaning | What to do |
|---|---|---|
| `invalid_arguments` | e.g. slug omitted on a multi-store key | Fix arguments |
| `store_not_allowed` | Slug outside the key's allowlist | Use an allowed slug from get_key_info |
| `daily_limit_exceeded` | Day budget exhausted | Stop; retry after `details.resets_at` |
| `upstream_error` | Copper Core rejected the call (bad SQL, etc.) | Read `message`, fix, retry |

Transport-level: HTTP 401 (bad/revoked key), 403 (key not enabled for this service), 429 (rate limit; honor `Retry-After`).
