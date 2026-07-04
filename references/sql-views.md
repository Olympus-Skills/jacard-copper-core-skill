# Copper LLM SQL Schema

Use only `llm_*` views through `/sql_store` and `/sql_global`. Store-scoped views are automatically filtered by `slug` in `/sql_store`.

Rules:

- Use only `llm_*` views. Do not query raw tables.
- Never include semicolons.
- Prefer explicit column lists. Do not use `select *` except a one-row schema probe.
- Never invent `id`, `name`, `is_ready`, `barcode`, `product_name`, `collection_name`, or `ready` columns.
- Use `product_id` for products, `slug` for stores, `title` for product/collection titles, `status` for readiness/state, `collection_handle` for collection filters, `family_key` for embedding family, and `estimated_cost_usd` for cost.

## Store-Scoped Views

`llm_products`

Purpose: product catalog rows for the active store.

Key columns: `product_id`, `external_product_id`, `canonical_url`, `title`, `brand`, `raw_product_type`, `currency`, `min_price`, `max_price`, `available_for_sale`, `in_stock`, `status`, `created_at`, `updated_at`.

`llm_variants`

Purpose: variant prices, inventory, and normalized option fields.

Key columns: `product_id`, `product_title`, `variant_id`, `external_variant_id`, `sku`, `color`, `size`, `material`, `unit_price`, `compare_at_price`, `inventory_qty`, `inventory_status`, `option_values_json`, `status`, `updated_at`.

`llm_assets`

Purpose: product images/assets and caption metadata.

Key columns: `product_id`, `product_title`, `asset_id`, `variant_id`, `asset_type`, `role`, `asset_url`, `width`, `height`, `sort_order`, `alt_text`, `caption_text`, `caption_provider`, `caption_model`, `captioned_at`, `status`, `updated_at`.

`llm_collections`

Purpose: first-class store collections.

Key columns: `collection_id`, `external_collection_id`, `handle`, `title`, `description`, `canonical_url`, `image_url`, `status`, `created_at`, `updated_at`.

`llm_product_collections`

Purpose: product-to-collection membership.

Key columns: `product_collection_id`, `product_id`, `product_title`, `collection_id`, `collection_handle`, `collection_title`, `status`, `created_at`, `updated_at`.

`llm_text_chunks`

Purpose: active/searchable product text chunks and captions.

Key columns: `product_id`, `product_title`, `doc_id`, `doc_type`, `chunk_id`, `chunk_type`, `section_path`, `chunk_seq`, `chunk_text`, `retrieval_text`, `display_snippet`, `token_count`, `weight`, `status`, `updated_at`.

`llm_embedding_bindings`

Purpose: SQL-side embedding coverage and errors for chunks/assets.

Key columns: `product_id`, `product_title`, `family_id`, `family_key`, `entity_type`, `entity_id`, `status`, `generated_at`, `last_attempted_at`, `updated_at`, `error_json`.

## Global Views

`llm_stores`

Purpose: store discovery and readiness.

Key columns: `store_id`, `slug`, `domain`, `connector_type`, `locale`, `currency`, `status`, `created_at`, `updated_at`.

`llm_store_metrics`

Purpose: cached dashboard counts, embedding coverage, and cost summaries.

Key columns: `store_slug`, `store_domain`, `counts_json`, `embedding_coverage_json`, `embedding_cost_json`, `source_crawl_run_id`, `refreshed_at`, `updated_at`.

`llm_latest_runs`

Purpose: latest crawl/update/caption/embed run per store/run type.

Key columns: `store_slug`, `crawl_run_id`, `run_type`, `trigger_type`, `status`, `created_at`, `started_at`, `finished_at`, `claimed_at`, `heartbeat_at`, `worker_id`, `stats_json`, `error_json`.

`llm_embedding_costs`

Purpose: ledgered estimated embedding spend grouped by store/family/modality/entity.

Key columns: `store_slug`, `family_id`, `family_key`, `modality`, `text_doc_type`, `entity_type`, `ledger_rows`, `entity_count`, `estimated_tokens`, `estimated_cost_usd`, `latest_ledger_at`.

`llm_product_counts_by_store`

Purpose: product count and price-bound summary per store.

Key columns: `store_slug`, `products_total`, `products_active`, `products_available`, `products_in_stock`, `min_price`, `max_price`.

`llm_collection_counts_by_store`

Purpose: collection count summary per store.

Key columns: `store_slug`, `collections_total`, `collections_active`, `active_product_collection_memberships`.

## Common Joins

Products to collections:

```sql
select p.product_id, p.title, pc.collection_handle
from llm_products p
join llm_product_collections pc on pc.product_id = p.product_id
where p.status = 'active' and pc.status = 'active'
limit 20
```

Products to variants:

```sql
select p.product_id, p.title, v.variant_id, v.sku, v.unit_price, v.inventory_status
from llm_products p
join llm_variants v on v.product_id = p.product_id
where p.status = 'active'
limit 20
```

Products to assets:

```sql
select p.product_id, p.title, a.asset_url, a.alt_text, a.caption_text
from llm_products p
join llm_assets a on a.product_id = p.product_id
where p.status = 'active' and a.status = 'active'
limit 20
```

Product embedding coverage:

```sql
select p.product_id, p.title, b.family_key, b.entity_type, b.status
from llm_products p
join llm_embedding_bindings b on b.product_id = p.product_id
where p.status = 'active'
limit 20
```

Global store metrics/counts:

```sql
select s.slug, s.status, pc.products_active, cc.collections_active
from llm_stores s
left join llm_product_counts_by_store pc on pc.store_slug = s.slug
left join llm_collection_counts_by_store cc on cc.store_slug = s.slug
order by s.slug
limit 20
```

## Safe Examples

Active products:

```sql
select product_id, title, min_price, max_price, currency
from llm_products
where status = 'active'
order by title
limit 10
```

Top collections:

```sql
select collection_handle, count(*) as product_count
from llm_product_collections
where status = 'active'
group by collection_handle
order by product_count desc
limit 10
```

Availability and price aggregate:

```sql
select count(*) as products_active,
       count(*) filter (where available_for_sale) as products_available,
       count(*) filter (where in_stock) as products_in_stock,
       min(min_price) as min_price,
       max(max_price) as max_price
from llm_products
where status = 'active'
```

Embedding coverage:

```sql
select family_key, entity_type, status, count(*) as bindings
from llm_embedding_bindings
group by family_key, entity_type, status
order by family_key, entity_type, status
limit 50
```

Ready stores:

```sql
select slug, domain, status
from llm_stores
where status = 'ready'
order by slug
limit 20
```

Failed latest runs:

```sql
select store_slug, run_type, status, created_at, error_json
from llm_latest_runs
where status = 'failed'
order by created_at desc
limit 20
```

Product counts by store:

```sql
select store_slug, products_active, products_available, products_in_stock, min_price, max_price
from llm_product_counts_by_store
order by products_active desc
limit 20
```

Embedding spend:

```sql
select store_slug, family_key, modality, entity_type, sum(estimated_cost_usd) as estimated_cost_usd
from llm_embedding_costs
group by store_slug, family_key, modality, entity_type
order by estimated_cost_usd desc
limit 20
```
