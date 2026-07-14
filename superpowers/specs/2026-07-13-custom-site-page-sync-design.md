# Page-identity sync for custom websites

**Date:** 2026-07-13
**Branch:** `api-wrapper`

## Problem

The "Sync pages from WordPress" button backfills page identity onto crawled
`site_nodes` rows: it pages through the CMS's page list, matches each page to a
crawled URL, and stamps `page_id`, `slug`, `product_name`, `category`, `tags`,
`page_type` and `platform` onto the node. That identity is what lets bulk update
and the agent know which CMS entity a crawled URL actually is.

The `custom` platform (the API wrapper for customer-owned sites) has no such
sync, so custom-connected brands get crawled nodes with no entity identity.

## Approach: generalize, don't duplicate

`server/controllers/integrations/wordpress/syncSiteNodes.js` is WordPress-only
in exactly three places — the integration resolver, the page-list call, and the
literal `platform = 'wordpress'` it writes. Everything else (chunked streaming,
indexed `href_hash` match, one-pass slug fallback, recrawl of unmatched
published pages) is platform-neutral.

The module moves to `server/controllers/integrations/platform/syncSitePages.js`
and dispatches on the resolved integration's platform:

| Concern | Before | After |
| --- | --- | --- |
| Which integration | `getConnectedWooIntegration()` | `getConnectedBlogIntegration()` (already exists in `platform/blogCategories.js`) |
| How to list pages | `wpGetPages()` | `listPages()` switch → `wpGetPages` \| `customGetPages` |
| Platform stamped | literal `'wordpress'` | temp-table column carrying the integration's platform |

One code path, one worker job, one button. WordPress becomes one case of it.

## Contract change (additive)

`GET /pages` gains two **optional** fields:

- `categories`: `string[]`
- `tags`: `string[]`

Sites that omit them still sync; those `site_nodes` columns stay null.

No `status` field is added — the endpoint is already specified as "list
**published** pages", so everything it returns is published and any unmatched
URL is a valid recrawl candidate.

The custom `type` label maps to `page_type` the same way WordPress's
`post_type`/`taxonomy` does:

| `type` | `page_type` |
| --- | --- |
| `product` | Ecommerce Product Page |
| `product_cat` / `product_category` | Ecommerce Category Page |
| `product_tag` / `category` | Category Page |
| `post` / `posts` | Blog Page |
| anything else (e.g. `page`) | `null` — keep the crawler's classification |

Touches: `docs/openapi/contract.yaml`, `docs/guides/*`, `dev/custom-receiver/server.js`,
`api-test-frontend/src/app/api/castro/pages/route.js`.

## Capability gating

`customGetPages()` already returns 501 when the site did not declare
`pages.list` at handshake. Building on that:

- the trigger endpoint pre-checks the capability and returns a clear error
  rather than enqueuing a job that silently no-ops;
- the UI renders the Sync button for a custom integration only when its stored
  capabilities include `pages.list`.

## Trigger points

All three existing triggers become platform-aware:

1. **Manual button** — `POST /integrations/sync-wp-pages` → `/sync-pages`.
   `client/src/store/credentials.js` is the only caller.
2. **On connect** — `integrations/index.js` enqueues the job when the WP plugin
   connects; `connectCustom()` gets the same enqueue after a successful
   handshake.
3. **On crawl completion** — `scraper/crawlee/crawl.js` already fires the job
   per brand unconditionally, so it starts working for custom brands for free
   once the job is platform-agnostic.

## Naming / compatibility

The worker route and `tool_name` stay `wp-entity-sync` even though the job is no
longer WordPress-only. Renaming would orphan jobs already queued at deploy time
and break continuity in existing job history, for a cosmetic gain. Only the
internal queue key keeps the old name; the file, functions, and all user-facing
strings become neutral.

`site_nodes.wp_synced_at` is likewise reused as the generic "last synced"
timestamp rather than renamed.

## Verification

Against `api-test-frontend`: crawl the test site, press Sync, then confirm the
`site_nodes` rows come back with `platform='custom'`, real `page_id`s, titles
and categories — and that a published page absent from the crawl gets queued for
one.
