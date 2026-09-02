# eBay OpenAPI 3 contract inventory

**Research date:** 2026-09-02. Context: wayfinder research ticket
[skrax/ebay-proj#5](https://github.com/skrax/ebay-proj/issues/5) (part of the
`ebay_client` implementation-spec map, [#4](https://github.com/skrax/ebay-proj/issues/4)).
Read [`docs/research/ebay-python-api-clients.md`](./ebay-python-api-clients.md) first.

**All `info.version` numbers, operation counts, and base paths below drift.** They are
what the cited contract document contained on the snapshot date given in the table's
"contract snapshot" column. eBay ships contract updates roughly monthly per API and does
not publish a changelog in the contract itself. Re-pull and diff before every codegen run.

**Sourcing.** `developer.ebay.com` returns **HTTP 403** to non-browser clients (verified
2026-09-02: `curl` and automated fetchers both get eBay's bot wall on the contract
`.json` URLs). Contract content below was retrieved from **Internet Archive Wayback
Machine** snapshots of the exact canonical `developer.ebay.com` URLs
(`https://web.archive.org/web/<timestamp>id_/<canonical-url>`), then parsed locally.
The canonical (non-Wayback) URL is what is cited so it can be opened in a normal browser.
Cross-checked against the contract-download/patch logic in
[`matecsaj/ebay_rest` `scripts/generate_code.py`](https://github.com/matecsaj/ebay_rest/blob/main/scripts/generate_code.py)
(a Swagger-Codegen pipeline that consumes these same contracts) and the
[Digital Signatures for APIs](https://developer.ebay.com/develop/guides/digital-signatures-for-apis)
guide.

---

## 0. Bottom line for the codegen pipeline

1. **39 OpenAPI 3.0 contract documents** are published across the Buy / Sell / Commerce /
   Developer families (38 verified against archived contract JSON + 1, Sell **Leads**,
   evidenced only via `ebay_rest` because eBay mislabels its filename and it is not
   archived). Collapsing beta/superseded duplicates (Buy Order v1β vs v2, Buy Feed
   "Item Feed Service" vs "Buy Feed API") that is **~36 distinct APIs**. Total
   operations across the 38 parsed contracts: **367** (300 paths).
2. **No contract contains a sandbox server or a sandbox `tokenUrl`.** Every contract
   hard-codes production only (`https://api.ebay.com/identity/v1/oauth2/token`,
   `servers[0].url` = `https://api.ebay.com{basePath}` or an `apix/apiz/apim/tppz`
   variant). The generated client **cannot** get its base URL or token URL from the
   spec — both must be overridden at runtime (`EBAY_ENV` → host + token-URL map).
3. **Five hosts, not one.** `api.ebay.com` (most), `apix.ebay.com` (Buy Order),
   `apiz.ebay.com` (Commerce Identity, Sell Finances, Developer Key Management),
   `apim.ebay.com` (Commerce Media), `tppz.ebay.com` (Developer Client Registration).
   Their sandbox equivalents are **all** `api.sandbox.ebay.com`, so the env switch must
   be a per-API host map, not a `api.` → `api.sandbox.` string replace.
4. **8 in-scope operations require digital signatures** and are out of scope for this
   client (§3): the **7** operations of the Sell **Finances** API + Sell **Fulfillment**
   `issueRefund`. Cleanest exclusion: drop the whole Finances contract from generation
   and path-filter `POST /order/{order_id}/issue_refund` out of Fulfillment.
5. **Six contracts are spec-broken around OAuth** (empty `securitySchemes.…flows` +
   operations referencing undeclared scopes): Commerce **Feedback**, Commerce
   **Message**, Commerce **Vero**, Sell **eDelivery International Shipping**, Sell
   **Stores**, Sell **Leads** (§4). `openapi-python-client` will emit an auth-less
   client for these and strict OAS validators reject them. They are all low-value APIs
   for this project — prefer excluding them; otherwise a pre-generation overlay/patch
   step is required (as `ebay_rest` does).
6. **`info.version` has no consistent format** (`v1.20.1`, `1.18.3`, `v1_beta.35.2`,
   bare `1`). Drift-detection/pinning must normalize.
7. **No API in these four families is OAS-2.0-only** (§6). eBay still publishes parallel
   OAS 2.0 documents for the older APIs, but 3.0 coverage is complete; the newer APIs
   (Commerce Message/Feedback/Vero, Sell eDelivery) are **3.0-only**. The legacy
   Trading/Finding/Shopping/Merchandising APIs and the **Post-Order API** have **no**
   OpenAPI contract at all (WSDL / hand-doc only) — relevant because Post-Order is where
   most signature-required dispute/refund operations live, and it is already out of scope.

---

## 1. Contract inventory

**Base URL** at runtime = `servers[0].url` with the `basePath` variable substituted,
e.g. Browse = `https://api.ebay.com/buy/browse/v1`. "Ops" counts operations
(`get/put/post/delete/patch` across all paths) in the archived contract.

**Download URL pattern** (open in a browser — 403 to bots):
`https://developer.ebay.com/api-docs/master/<path>/openapi/3/<file>.json` and the same
with `.yaml`. The `<file>.json` / `<path>` for each row is the last path segments shown
in the "canonical JSON URL" column; the YAML URL is identical with `.json` → `.yaml`.

### Buy family

| API (title) | `info.version` | Host | Base path | Paths | Ops | OAuth flow(s) | Contract snapshot | Canonical JSON URL (swap `.json`→`.yaml` for YAML) |
|---|---|---|---|---|---|---|---|---|
| Browse API | `v1.20.1` | api | `/buy/browse/v1` | 7 | 7 | clientCredentials | 2025-01-23 | `…/master/buy/browse/openapi/3/buy_browse_v1_oas3.json` |
| Deal API | `v1.3.0` | api | `/buy/deal/v1` | 4 | 4 | clientCredentials | 2024-07-25 | `…/master/buy/deal/openapi/3/buy_deal_v1_oas3.json` |
| Feed API — "Item Feed Service" (beta) | `v1_beta.35.2` | api | `/buy/feed/v1_beta` | 4 | 4 | clientCredentials | 2024-07-23 | `…/master/buy/feed/openapi/3/buy_feed_v1_beta_oas3.json` |
| Feed API — "Buy Feed API" (v1) | `v1.1.0` | api | `/buy/feed/v1` | 6 | 6 | clientCredentials | 2024-11-05 | `…/master/buy/feed/v1/openapi/3/buy_feed_v1_oas3.json` |
| Marketing API (beta) | `v1_beta.2.0` | api | `/buy/marketing/v1_beta` | 1 | 1 | clientCredentials | 2025-06-24 | `…/master/buy/marketing/openapi/3/buy_marketing_v1_beta_oas3.json` |
| Marketplace Insights API (beta, restricted) | `v1_beta.2.2` | api | `/buy/marketplace_insights/v1_beta` | 1 | 1 | clientCredentials | 2021-11-23 † | `…/master/buy/marketplace-insights/openapi/3/buy_marketplace_insights_v1_beta_oas3.json` |
| Offer API (beta) — proxy bidding | `v1_beta.0.1` | api | `/buy/offer/v1_beta` | 2 | 2 | authorizationCode | 2024-07-14 | `…/master/buy/offer/openapi/3/buy_offer_v1_beta_oas3.json` |
| Order API **v2** (current) | `v2.1.2` | **apix** | `/buy/order/v2` | 8 | 8 | clientCredentials | 2024-07-22 | `…/master/buy/order/openapi/3/buy_order_v2_oas3.json` |
| Order API **v1** (beta, superseded by v2) | `v1_beta.37.0` | **apix** | `/buy/order/v1` | 11 | 11 | authorizationCode | 2025-12-19 | `…/master/buy/order_v1/openapi/3/buy_order_v1_beta_oas3.json` |

### Commerce family

| API (title) | `info.version` | Host | Base path | Paths | Ops | OAuth flow(s) | Contract snapshot | Canonical JSON URL |
|---|---|---|---|---|---|---|---|---|
| Catalog API (beta, restricted) | `v1_beta.5.2` | api | `/commerce/catalog/v1_beta` | 2 | 2 | authorizationCode | 2024-10-14 | `…/master/commerce/catalog/openapi/3/commerce_catalog_v1_beta_oas3.json` |
| Charity API | `v1.2.1` | api | `/commerce/charity/v1` | 2 | 2 | clientCredentials | 2025-05-16 | `…/master/commerce/charity/openapi/3/commerce_charity_v1_oas3.json` |
| Feedback API (beta) | `v1.0.0` | api | `/commerce/feedback/v1` | 4 | 5 | **EMPTY** ⚠ | 2025-12-12 | `…/master/commerce/feedback/openapi/3/commerce_feedback_v1_beta_oas3.json` |
| Identity API | `v2.0.0` | **apiz** | `/commerce/identity/v1` | 1 | 1 | authorizationCode | 2024-10-13 | `…/master/commerce/identity/openapi/3/commerce_identity_v1_oas3.json` |
| Media API (beta) | `v1_beta.4.0` | **apim** | `/commerce/media/v1_beta` | 10 | 10 | authorizationCode | 2025-10-07 | `…/master/commerce/media/openapi/3/commerce_media_v1_beta_oas3.json` |
| Message API — title "M2M Public API Service" | `1.0.0` | api | `/commerce/message/v1` | 5 | 5 | **EMPTY** ⚠ | 2026-03-17 | `…/master/commerce/message/openapi/3/commerce_message_v1_oas3.json` |
| Notification API | `v1.5.1` | api | `/commerce/notification/v1` | 13 | 21 | clientCredentials + authorizationCode | 2024-06-23 † | `…/master/commerce/notification/openapi/3/commerce_notification_v1_oas3.json` |
| Taxonomy API | `v1.0.1` | api | `/commerce/taxonomy/v1` | 8 | 8 | clientCredentials | 2024-06-24 | `…/master/commerce/taxonomy/openapi/3/commerce_taxonomy_v1_oas3.json` |
| Translation API (beta) | `v1_beta.1.6` | api | `/commerce/translation/v1_beta` | 1 | 1 | clientCredentials | 2025-03-17 | `…/master/commerce/translation/openapi/3/commerce_translation_v1_beta_oas3.json` |
| Vero API — title "Vero Public API's" | `1.0.0` | api | `/commerce/vero/v1` | 5 | 5 | **EMPTY** ⚠ | 2025-08-08 | `…/master/commerce/vero/openapi/3/commerce_vero_v1_oas3.json` |

### Developer family

| API (title) | `info.version` | Host | Base path | Paths | Ops | OAuth flow(s) | Contract snapshot | Canonical JSON URL |
|---|---|---|---|---|---|---|---|---|
| Analytics API (beta) — rate-limit introspection | `v1_beta.0.0` | api | `/developer/analytics/v1_beta` | 2 | 2 | clientCredentials + authorizationCode | 2024-10-04 | `…/master/developer/analytics/openapi/3/developer_analytics_v1_beta_oas3.json` |
| Client Registration API — title "Developer Registration API" | `v1.0.0` | **tppz** | `/developer/registration/v1` | 1 | 1 | clientCredentials | 2025-12-15 | `…/master/developer/client-registration/openapi/3/developer_client_registration_v1_oas3.json` |
| Key Management API | `v1.0.0` | **apiz** | `/developer/key_management/v1` | 2 | 3 | clientCredentials | 2025-11-02 | `…/master/developer/key-management/openapi/3/developer_key_management_v1_oas3.json` |

### Sell family

| API (title) | `info.version` | Host | Base path | Paths | Ops | OAuth flow(s) | Contract snapshot | Canonical JSON URL |
|---|---|---|---|---|---|---|---|---|
| Account API **v1** | `v1.9.2` | api | `/sell/account/v1` | 24 | 36 | authorizationCode | 2024-06-14 | `…/master/sell/account/openapi/3/sell_account_v1_oas3.json` |
| Account API **v2** — title "Rate Table API" | `2.1.0` | api | `/sell/account/v2` | 4 | 4 | authorizationCode | 2024-06-24 | `…/master/sell/account/v2/openapi/3/sell_account_v2_oas3.json` |
| Analytics API | `1.3.2` | api | `/sell/analytics/v1` | 4 | 4 | authorizationCode | 2025-05-16 | `…/master/sell/analytics/openapi/3/sell_analytics_v1_oas3.json` |
| Compliance API | `1.4.3` | api | `/sell/compliance/v1` | 2 | 2 | authorizationCode | 2024-10-09 | `…/master/sell/compliance/openapi/3/sell_compliance_v1_oas3.json` |
| eDelivery International Shipping (EDIS) API | `1.0.0` | api | `/sell/edelivery_international_shipping/v1` | 20 | 23 | **EMPTY** ⚠ | 2025-08-08 | `…/master/sell/edelivery_international_shipping/openapi/3/sell_edelivery_international_shipping_oas3.json` |
| Feed API | `v1.3.1` | api | `/sell/feed/v1` | 16 | 23 | authorizationCode | 2024-09-14 | `…/master/sell/feed/openapi/3/sell_feed_v1_oas3.json` |
| **Finances API** — DIGITAL SIGNATURE, all ops (§3) | `v1.17.2` | **apiz** | `/sell/finances/v1` | 7 | 7 | authorizationCode | 2024-09-12 | `…/master/sell/finances/openapi/3/sell_finances_v1_oas3.json` |
| **Fulfillment API** — `issueRefund` needs DS (§3) | `v1.20.4` | api | `/sell/fulfillment/v1` | 14 | 15 | authorizationCode | 2024-06-18 | `…/master/sell/fulfillment/openapi/3/sell_fulfillment_v1_oas3.json` |
| Inventory API | `1.18.3` | api | `/sell/inventory/v1` | 23 | 36 | authorizationCode | 2025-07-01 | `…/master/sell/inventory/openapi/3/sell_inventory_v1_oas3.json` |
| Listing API (beta) | `v1_beta.3.0` | api | `/sell/listing/v1_beta` | 1 | 1 | authorizationCode | 2021-05-08 † | `…/master/sell/listing/openapi/3/sell_listing_v1_beta_oas3.json` |
| Logistics API (beta) | `v1_beta.0.0` | api | `/sell/logistics/v1_beta` | 6 | 6 | authorizationCode | 2025-03-23 | `…/master/sell/logistics/openapi/3/sell_logistics_v1_oas3.json` |
| Marketing API | `v1.21.1` | api | `/sell/marketing/v1` | 62 | 81 | clientCredentials + authorizationCode | 2024-06-14 | `…/master/sell/marketing/openapi/3/sell_marketing_v1_oas3.json` |
| Metadata API | `v1.7.1` | api | `/sell/metadata/v1` | 8 | 8 | clientCredentials + authorizationCode | 2024-06-20 | `…/master/sell/metadata/openapi/3/sell_metadata_v1_oas3.json` |
| Negotiation API | `v1.1.0` | api | `/sell/negotiation/v1` | 2 | 2 | authorizationCode | 2025-05-16 | `…/master/sell/negotiation/openapi/3/sell_negotiation_v1_oas3.json` |
| Recommendation API | `v1.1.0` | api | `/sell/recommendation/v1` | 1 | 1 | authorizationCode | 2025-04-23 | `…/master/sell/recommendation/openapi/3/sell_recommendation_v1_oas3.json` |
| Stores API — title "Store API" | `1` | api | `/sell/stores/v1` | 6 | 8 | **EMPTY** ⚠ | 2024-10-10 | `…/master/sell/stores/openapi/3/sell_stores_v1_oas3.json` |
| **Leads API** (see note) | n/a — not archived | api | `/sell/leads/v1` (est.) | ? | ? | **EMPTY** ⚠ (expected) | — | `…/master/sell/leads/openapi/3/sell_feed_v1_oas3.json` ‡ |

† Only an old Wayback snapshot exists; the live contract is almost certainly newer.
Re-pull from a browser before using.

‡ **eBay filename bug**: the Sell Leads OpenAPI 3 contract is served at a path ending
`…/sell/leads/openapi/3/**sell_feed_v1_oas3.json**` (says `feed`, should say `leads`).
Documented as `GOOFY_SELL_LEADS_URL` in
[`ebay_rest/scripts/generate_code.py`](https://github.com/matecsaj/ebay_rest/blob/main/scripts/generate_code.py),
which renames it to `sell_leads_v1_oas3.json` and patches in the
`…/oauth/api_scope/sell.leads` scope. No Wayback snapshot of this URL exists; its
existence and shape are attested only by that third-party script.

---

## 2. Fetchability and vendoring

### What is machine-fetchable

| Source | Fetchable from CI / scripts? | Notes |
|---|---|---|
| `developer.ebay.com/api-docs/master/**/openapi/3/*.json` (canonical) | **No** | HTTP 403 bot wall for `curl`/non-browser (verified 2026-09-02). Works from a real logged-out browser. |
| Same via `www.developer.ebay.com` | No | Same wall; redirects/serves identically. |
| `api.ebay.com/developer/mcp/v1/search` (eBay's spec-discovery API used by `@ebay/npm-public-api-mcp`) | **No (needs OAuth)** | Requires an application token; returns spec fragments per operation, not whole contracts. See [`openapi-helper.ts`](https://github.com/eBay/npm-public-api-mcp/blob/main/src/helper/openapi-helper.ts), [`constants.ts`](https://github.com/eBay/npm-public-api-mcp/blob/main/src/constant/constants.ts). |
| Wayback Machine `web.archive.org/web/<ts>id_/<canonical-url>` | **Yes**, mostly | Raw contract JSON/YAML is returned. Caveats: (a) snapshots lag the live contract by weeks–years (see snapshot dates in §1); (b) Internet Archive had intermittent multi-hour outages on 2026-09-02; (c) some snapshots are gzip-framed and need transparent decompression. Fine for a **one-time seed**, not for CI. |
| eBay GitHub org | **No contract repo** | eBay publishes SDKs (Feed, digital-signature, event-notification, OAuth, MCP) but **not** a repo of the OpenAPI contracts. Verified by enumerating `orgs/eBay/repos`. |
| Third-party mirrors: [`matecsaj/ebay_rest`](https://github.com/matecsaj/ebay_rest) (does not commit the specs, downloads them at build time), `michabbb/sdk-ebay-rest-*` (PHP, per-API, `openapi-generator` output — bundles the source spec) | Partial | Secondary sources; content is eBay's contract but versions are whatever the maintainer last regenerated. Use only for cross-checking. |

### Recommended vendoring approach

**Manual/browser download, committed to the repo, refreshed by a deliberate human step.**

1. A human opens each API's **Overview** page (`developer.ebay.com/api-docs/<family>/<api>/overview.html`)
   in a browser and downloads both the **"OpenAPI 3 JSON Contract"** and **"OpenAPI 3
   YAML Contract"** links. (eBay markets this download-and-generate path itself:
   [Take advantage of OpenAPI specifications of eBay APIs](https://developer.ebay.com/api-docs/static/gs_take-advantage-of-openapi.html),
   [openapi-coverage blog](https://developer.ebay.com/news/openapi-coverage).)
2. Commit under e.g. `contracts/ebay/<family>/<api>_v1_oas3.json` (+ `.yaml`), plus a
   small manifest recording, per file: canonical URL, `info.version`, SHA-256, and the
   retrieval date.
3. A **pre-generation overlay step** (JSON patch / `openapi-overlay`) fixes the
   spec-quality issues in §4 before `openapi-python-client` runs — mirroring what
   `ebay_rest`'s `generate_code.py` does with its `MISSING_OAUTH_SCOPES` table and its
   historical `patch_contract_sell_fulfillment`.
4. Refresh is a manual `just update-contracts` / documented runbook, **not** a CI job —
   generated client code is committed anyway, and eBay's bot wall + DoS protection
   (`ebay_rest` script comment: *"Wait a day if this script intermittently fails … eBay's
   DOS protection"*) make automated pulls unreliable. Optionally provide a local helper
   that scrapes the Overview pages from the developer's authenticated browser session,
   but treat it as convenience, not infrastructure.
5. Keep a **Wayback fallback** documented for when a page is temporarily unreachable.

There is **no** authenticated API that returns whole contracts, so "authenticated fetch"
is not an option beyond a browser session cookie.

---

## 3. Operations requiring digital signatures (OUT OF SCOPE — exclude/mark)

**Authoritative source:**
[Digital Signatures for APIs](https://developer.ebay.com/develop/guides/digital-signatures-for-apis)
(§ "APIs in scope"), verified via Wayback snapshot `20260516233905`. Quote:

> Signatures are required when the call is made for EU- or UK-domiciled sellers, and only
> for the following APIs/methods:
> - **All methods in the Finances API**
> - **`issueRefund` in the Fulfillment API**
> - `GetAccount` in the Trading API
> - The following methods in the Post-Order API: Issue Inquiry Refund, Issue case refund,
>   Issue return refund, Process Return Request, Create Cancellation Request, Approve
>   Cancellation Request

Headers added on in-scope calls: `x-ebay-signature-key`, `Content-Digest` (conditional —
omitted for GET/no-body), `Signature`, `Signature-Input` (RFC 9421 + RFC 9530). The
signing keypair comes from the **Developer Key Management API** (`createSigningKey`).
The requirement is conditional on the seller's domicile, but the client cannot know that
ahead of time, so these operations must be treated as **always** signature-required.

### In scope for this client (Buy/Sell/Commerce/Developer OAS 3): **8 operations**

| API / contract | Method + path | `operationId` | Note |
|---|---|---|---|
| Sell Finances (`sell_finances_v1_oas3.json`) | `GET /payout/{payout_Id}` | `getPayout` | all Finances ops |
| Sell Finances | `GET /payout` | `getPayouts` | |
| Sell Finances | `GET /payout_summary` | `getPayoutSummary` | |
| Sell Finances | `GET /seller_funds_summary` | `getSellerFundsSummary` | |
| Sell Finances | `GET /transaction` | `getTransactions` | |
| Sell Finances | `GET /transaction_summary` | `getTransactionSummary` | |
| Sell Finances | `GET /transfer/{transfer_Id}` | `getTransfer` | |
| Sell Fulfillment (`sell_fulfillment_v1_oas3.json`) | `POST /order/{order_id}/issue_refund` | `issueRefund` | the **only** Fulfillment op that needs DS; note its declared scope is `…/oauth/api_scope/sell.finances`, not a fulfillment scope |

Each contract's own operation `description` for these ops contains the string
*"Digital Signatures … is required for certain API calls made by EU/UK-domiciled sellers"*
(verified in the archived `sell_finances_v1_oas3.json` and `sell_fulfillment_v1_oas3.json`),
so the exclusion list can be regenerated by grepping operation descriptions for
`signature` and intersecting with the DS guide.

### NOT signature-required within OAS 3 scope (important — do not over-exclude)

- **Fulfillment payment-dispute operations** (`getPaymentDispute`,
  `getPaymentDisputeSummaries`, `getActivities`, `contestPaymentDispute`,
  `acceptPaymentDispute`, `addEvidence`, `updateEvidence`, `uploadEvidenceFile`,
  `fetchEvidenceContent`, `getPaymentDisputeActivities`) — 10 ops, scope
  `sell.payment.dispute`. The **Post-Order API** dispute/refund/cancellation operations
  require signatures, but Post-Order is a **separate legacy API with no OpenAPI 3
  contract**; its OAS-3 Fulfillment counterparts do **not**.
- **Trading `GetAccount`** — legacy XML API, no OpenAPI contract, not in this client.
- "payout … money-movement operations" from the map = the Finances API payout GETs
  (already covered). "dispute money-movement" = Post-Order (no OAS 3 contract). So there
  are **no additional OAS-3 operations** beyond the 8 above.

### Recommended exclusion mechanism

Drop the **entire Sell Finances contract** from generation (it is 7 GET reporting ops,
all DS-gated, low value without signing) and add a **path filter** removing
`POST /sell/fulfillment/v1/order/{order_id}/issue_refund` from the Fulfillment
generation. Maintain the list as data next to the overlay patches (§2.3); it is small
and stable.

---

## 4. Spec-quality issues (per family, with specifics)

### 4a. Empty `securitySchemes` OAuth flows + dangling scope references

`components.securitySchemes.api_auth.flows` is present but `{}` (empty) — **no
`tokenUrl`, no scope catalog** — while operations still carry `security` blocks that
reference scopes never declared anywhere in the document. Strict OAS 3 validators
(`openapi-spec-validator`, Spectral `oas3-*`) reject these; `openapi-python-client`
generates a client with **no auth wiring** and you cannot learn the token scope from the
contract.

| Contract | Empty `flows`? | Scopes referenced by ops but undeclared |
|---|---|---|
| `commerce_feedback_v1_beta_oas3.json` | yes | `…/oauth/api_scope/commerce.feedback`, `…/commerce.feedback.readonly` |
| `commerce_message_v1_oas3.json` | yes | `…/oauth/api_scope/commerce.message` |
| `commerce_vero_v1_oas3.json` | yes | `…/oauth/api_scope/commerce.vero` **and** `…/oauth/scope/commerce.vero` (two different URI shapes for the same scope) |
| `sell_edelivery_international_shipping_oas3.json` | yes | `…/oauth/scope/sell.edelivery` (non-standard `/oauth/scope/` path) |
| `sell_stores_v1_oas3.json` | yes | `…/oauth/api_scope/sell.stores` |
| `sell_leads` (`sell_feed_v1_oas3.json` at the leads path) | expected yes | `…/oauth/api_scope/sell.leads` |

`ebay_rest` carries exactly this patch table
([`generate_code.py` `MISSING_OAUTH_SCOPES`](https://github.com/matecsaj/ebay_rest/blob/main/scripts/generate_code.py))
for `commerce.message`, `commerce.vero`, `sell.edelivery`, `sell.stores`, `sell.leads`
(it does not yet list Feedback). It injects a `clientCredentials` flow with
`tokenUrl: https://api.ebay.com/identity/v1/oauth2/token` and the missing scope. This
corroborates the finding the prior research flagged
(`ebay_rest` issue *"V2 - Potential OAuth issues in some eBay Commerce and Sell OpenAPI
contracts"*).

### 4b. Non-standard scope URIs

`…/oauth/scope/sell.edelivery` and `…/oauth/scope/commerce.vero` use `/oauth/scope/`
where every other eBay scope uses `/oauth/api_scope/`. The Vero contract uses **both**
forms. Any scope-name allowlist/enum must tolerate this or normalize it.

### 4c. Inconsistent `info.version` format

`v1.20.1`, `v1.3.0` (prefixed dotted); `1.18.3`, `1.4.3`, `2.1.0` (unprefixed dotted);
`v1_beta.35.2`, `v1_beta.0.0` (beta-prefixed); bare `1` (Sell Stores). Cannot be parsed
as semver without normalization; pin/diff logic must handle all four shapes.

### 4d. Internal / unpolished `info.title` (affects generated package + client names)

| Contract | `info.title` | Should be |
|---|---|---|
| `commerce_message_v1_oas3.json` | `M2M Public API Service` | Message API |
| `commerce_vero_v1_oas3.json` | `Vero Public API's` (stray apostrophe) | VeRO API |
| `sell_account_v2_oas3.json` | `Rate Table API` | Account API v2 |
| `sell_edelivery_international_shipping_oas3.json` | `EDIS public shipping API` | eDelivery International Shipping API |
| `buy_feed_v1_beta_oas3.json` | `Item Feed Service` | Buy Feed API |
| `developer_client_registration_v1_oas3.json` | `Developer Registration API` | Client Registration API |

`openapi-python-client` derives the package name from `info.title` by default — every
one of these needs an explicit `package_name_override` / config entry.

### 4e. Fulfillment `issueRefund` scope oddity

`POST /order/{order_id}/issue_refund` declares scope `…/oauth/api_scope/sell.finances`
(not `sell.fulfillment`). Easy to miss when minting a token for a Fulfillment-only
integration.

### 4f. Historical (now fixed, keep on the radar)

`ebay_rest` used to patch the Sell Fulfillment contract: the `Address` schema declared a
`country` property while the API actually returns `countryCode`, so Swagger-Codegen
produced a wrong model
([`patch_contract_sell_fulfillment` in `generate_code.py`](https://github.com/matecsaj/ebay_rest/blob/main/scripts/generate_code.py),
now commented out — *"no longer needed, ebay fixed the problem"*). Evidence that eBay's
contracts have had schema-vs-behaviour drift; validate generated models against real
sandbox responses during the spike.

### 4g. `basePath` version segment vs `info.version` mismatch

Several contracts titled "…API" ship a `v1_beta` base path (`sell_logistics` →
`/sell/logistics/v1_beta`; `buy_marketing` → `/buy/marketing/v1_beta`) while carrying a
non-obvious version string. Trust the `servers[].variables.basePath.default`, not the
title, for the URL.

---

## 5. OAuth: scopes, token URLs, servers

### Token / auth endpoints (identical across every contract; production-only in the spec)

| | In every contract | Sandbox (NOT in any contract — from eBay docs / `@ebay/npm-public-api-mcp`) |
|---|---|---|
| `tokenUrl` | `https://api.ebay.com/identity/v1/oauth2/token` | `https://api.sandbox.ebay.com/identity/v1/oauth2/token` |
| `authorizationUrl` | `https://auth.ebay.com/oauth2/authorize` | `https://auth.sandbox.ebay.com/oauth2/authorize` |
| API host | `https://api.ebay.com` (+ `apix`/`apiz`/`apim`/`tppz` variants — see §1) | `https://api.sandbox.ebay.com` for all of them |

Sandbox values sourced from
[client-credentials grant guide](https://developer.ebay.com/api-docs/static/oauth-client-credentials-grant.html)
and [`@ebay/npm-public-api-mcp` `constants.ts`](https://github.com/eBay/npm-public-api-mcp/blob/main/src/constant/constants.ts)
(`TOKEN_ENDPOINTS`, `DOMAIN_NAME`). **Implication:** the generated client must take base
URL + token URL from config keyed on `EBAY_ENV`, ignoring the spec's `servers` /
`securitySchemes.*.tokenUrl`.

### Flow types per contract

- **`clientCredentials` only** (application token, public data): Browse, Deal, both Buy
  Feed, Buy Marketing, Marketplace Insights, Buy Order v2, Commerce Charity, Taxonomy,
  Translation, Developer Client Registration, Developer Key Management.
- **`authorizationCode` only** (user token): Buy Offer, Buy Order v1β, Commerce Catalog,
  Commerce Identity, Commerce Media, all of Sell except Marketing/Metadata (Account v1/v2,
  Analytics, Compliance, Feed, Finances, Fulfillment, Inventory, Listing, Logistics,
  Negotiation, Recommendation).
- **Both flows declared**: Commerce Notification, Developer Analytics, Sell Marketing,
  Sell Metadata.
- **Neither (empty)**: the six §4a contracts.

### Scopes referenced, by family (union across each family's operation `security` blocks)

**Buy** — `…/oauth/api_scope`, `…/buy.deal`, `…/buy.guest.order`, `…/buy.item.bulk`,
`…/buy.item.feed`, `…/buy.marketing`, `…/buy.marketplace.insights`, `…/buy.offer.auction`,
`…/buy.order`, `…/buy.order.readonly`.

**Commerce** — `…/oauth/api_scope`, `…/commerce.catalog.readonly`, `…/commerce.feedback`,
`…/commerce.feedback.readonly`, `…/commerce.identity.readonly` (+ `.address` `.email`
`.name` `.phone` `.readonly` variants), `…/commerce.message`,
`…/commerce.notification.subscription` (+ `.readonly`), `…/commerce.vero` (+ the
malformed `…/oauth/scope/commerce.vero`), `…/metadata.insights`, `…/sell.inventory`
(Media API borrows it).

**Developer** — `…/oauth/api_scope`, `…/commerce.catalog.readonly`, `…/sell.inventory`
(+ `.readonly`), `…/sell.marketing` (+ `.readonly`), `…/sell.marketplace.insights.readonly`
(the Analytics API's `getRateLimits`/`getUserRateLimits` enumerate other APIs' scopes).

**Sell** — `…/oauth/api_scope`, `…/sell.account` (+ `.readonly`), `…/sell.analytics.readonly`,
`…/sell.finances`, `…/sell.fulfillment` (+ `.readonly`), `…/sell.inventory` (+ `.readonly`),
`…/sell.item.draft`, `…/sell.logistics`, `…/sell.marketing` (+ `.readonly`),
`…/sell.payment.dispute`, `…/sell.stores`, `…/commerce.catalog.readonly`, plus the
malformed `…/oauth/scope/sell.edelivery`.

Canonical scope list & human descriptions:
[OAuth scopes](https://developer.ebay.com/api-docs/static/oauth-scopes.html); also mirrored
in `ebay_rest`'s
[`application_scopes.json`](https://github.com/matecsaj/ebay_rest/blob/main/src/ebay_rest/references/application_scopes.json)
and [`user_scopes.json`](https://github.com/matecsaj/ebay_rest/blob/main/src/ebay_rest/references/user_scopes.json).

---

## 6. OAS 2.0-only APIs

**None** in the Buy / Sell / Commerce / Developer families.

- eBay states OAS is available "for all of our RESTful public APIs"
  ([Take advantage of OpenAPI specifications…](https://developer.ebay.com/api-docs/static/gs_take-advantage-of-openapi.html),
  [openapi-coverage blog](https://developer.ebay.com/news/openapi-coverage)).
- Every API in §1 has an OpenAPI **3.0.0** contract (all 38 parsed documents report
  `"openapi": "3.0.0"`).
- eBay **also** publishes parallel **OAS 2.0** documents at `…/openapi/2/…` for the older
  APIs (Browse, Deal, Feed, Marketing, Order, Marketplace Insights, Catalog, Identity,
  Media, Notification, Taxonomy, Translation, all Developer, and Sell
  Account/Analytics/Compliance/Feed/Finances/Fulfillment/Inventory/…). The newer APIs —
  Commerce **Message**, **Feedback**, **Vero**, Sell **eDelivery** — have **no** OAS 2.0
  document (they are 3.0-only). eBay's guidance is to prefer the 3.0 contract; there is
  no reason for this pipeline to touch the 2.0 files.
- **No OpenAPI at all** (neither 2.0 nor 3.0): the legacy **Trading, Finding, Shopping,
  Merchandising** APIs (WSDL only) and the **Post-Order API** (hand-authored docs only).
  The Post-Order gap matters because that is where most signature-required
  refund/dispute/cancellation operations live (§3) — all already out of scope.

---

## Sources consulted

**Research date 2026-09-02.** `developer.ebay.com` pages verified via Internet Archive
Wayback Machine snapshots of the cited canonical URLs; the canonical URL is cited.
Contract JSON parsed from Wayback `id_` raw snapshots.

### eBay — developer docs (canonical URLs)
- [Take advantage of OpenAPI specifications of eBay APIs](https://developer.ebay.com/api-docs/static/gs_take-advantage-of-openapi.html) (Wayback `20260421003726`)
- [eBay's OpenAPI 3.0 Contracts Now Available (blog)](https://developer.ebay.com/updates/blog/ebays-openapi-3-0-contracts-now-available)
- [All eBay RESTful APIs Leverage OpenAPI Specification / openapi-coverage (blog)](https://developer.ebay.com/news/openapi-coverage)
- [Digital Signatures for APIs](https://developer.ebay.com/develop/guides/digital-signatures-for-apis) (Wayback `20260516233905`)
- [The client credentials grant flow](https://developer.ebay.com/api-docs/static/oauth-client-credentials-grant.html)
- [The authorization code grant flow](https://developer.ebay.com/api-docs/static/oauth-authorization-code-grant.html)
- [OAuth scopes](https://developer.ebay.com/api-docs/static/oauth-scopes.html)
- The 38 archived OpenAPI 3 contract JSON documents listed in §1 (Wayback `id_` snapshots, dates in the table), plus the parallel `…/openapi/2/…` URL listing (Wayback CDX, §6).

### eBay — GitHub
- [`eBay/npm-public-api-mcp`](https://github.com/eBay/npm-public-api-mcp) — `src/constant/constants.ts` (`TOKEN_ENDPOINTS`, `DOMAIN_NAME`, `RECALL_SPEC_*`), `src/helper/openapi-helper.ts`
- `orgs/eBay/repos` enumeration (GitHub API) — confirmed no OpenAPI-contract repo
- [`eBay/digital-signature-sdk-python`](https://github.com/eBay/digital-signature-sdk-python), [`eBay/key-management` docs] referenced via the DS guide

### Third-party (cross-check only)
- [`matecsaj/ebay_rest`](https://github.com/matecsaj/ebay_rest) — [`scripts/generate_code.py`](https://github.com/matecsaj/ebay_rest/blob/main/scripts/generate_code.py) (`GOOFY_SELL_LEADS_URL`, `MISSING_OAUTH_SCOPES`, `patch_missing_flows`, `patch_contract_sell_fulfillment`, `get_contract_urls`), `src/ebay_rest/references/application_scopes.json` + `user_scopes.json`
- `michabbb/sdk-ebay-rest-*` (per-API `openapi-generator` PHP SDKs) — existence noted, not parsed

### Tooling
- [`openapi-python-client`](https://github.com/openapi-generators/openapi-python-client) — target generator (behaviour re `info.title`, `securitySchemes`)
- Internet Archive Wayback Machine CDX API + `id_` raw-snapshot endpoint
