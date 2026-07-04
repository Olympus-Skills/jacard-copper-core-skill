---
name: jacard-copper-core-skill
description: Use when an agent needs retail catalog answers through the Jacard Copper Core MCP — keyword/semantic/hybrid product search, deterministic inventory lists, product detail, collection facets, and readonly SQL over llm_* views. Assumes the MCP is already connected with a valid key.
---

# Jacard Copper Core MCP — usage guide

How to get good results from the **Jacard Copper Core MCP** (retail catalog retrieval). The MCP is already connected and authenticated for you; this skill is about using its tools well. Calls are metered per API key (allowed store slugs, a per-minute rate limit, and a daily request budget), so behavior below follows from your key.

## Start with get_key_info

Call `get_key_info` first — it's **free** (never debits your budget). It returns your allowed store slugs, whether you have all-store access, your rate/day limits, and today's remaining budget. What you can do follows from it, so check it before planning a sequence of calls.

## Scoping

- Store tools take a `slug`. With a single-store key you may omit `slug` (it defaults to your one store). A slug outside your key's allowlist fails with `store_not_allowed`.
- All-store keys (`*`) additionally get the global tools (`global_keyword_search_products`, `global_semantic_search_products`, `global_hybrid_search_products`, `global_list_inventory_products`, `readonly_sql_global`). Store-scoped keys don't see them at all.
- A multi-store key (no wildcard) queries one slug at a time.

## Choosing a tool

- `keyword_search_products` — fast lexical match. Exact terms, SKUs, product names.
- `semantic_search_products` — embedding similarity. Vague/descriptive queries ("minimal office earrings") and image queries (`image_url` or `image_base64` + `image_mime_type`).
- `hybrid_search_products` — keyword + semantic fused. Best default for natural-language product discovery.
- `list_inventory_products` — deterministic listing with exact `total_count` and stable cursors. Use for "all / how many products" requests, never for relevance ranking.
- `get_product_detail` — full record (variants, images, text chunks) by `product_id`.
- `list_store_collections` — valid collection handles + price bounds before filtering a search by collection.
- `readonly_sql_store` — SELECT over store-scoped `llm_*` views (max 500 rows) for counts, aggregates, audits, exact checks. Columns and joins: `references/sql-views.md`.
- `readonly_sql_global` (all-store keys) — cross-store views like `llm_stores`, `llm_global_products` (max 1000 rows).

Rules of thumb: prefer the search tools for recommendations and product picks; prefer SQL for counts and exact facts. Discover valid collection handles with `list_store_collections` before filtering by `collection`. Paginate with the returned `cursor` / `next_cursor` rather than inflating `limit`.

## Operating within budgets and errors

- Every tool call except `get_key_info` debits 1 request from the daily budget. Budgets reset at UTC midnight.
- Exceeding requests-per-minute → HTTP 429 with `Retry-After` seconds. Back off, then retry once the window passes.
- Tool-level failures come back with `isError` and a JSON body `{ "error": { "code", "message", "details?" } }`. Codes: `store_not_allowed`, `scope_not_allowed`, `invalid_arguments`, `daily_limit_exceeded` (includes `resets_at`), `upstream_error`.
- On `daily_limit_exceeded`, stop retrying — the budget is hard until the UTC reset.

## References

- Full tool contracts and example arguments: `references/tools.md`
- `llm_*` view columns and safe SQL patterns: `references/sql-views.md`
