# Python client options for the eBay REST APIs (for an MCP server)

**Research date:** 2026-09-02. All version numbers, release dates, commit dates, and
issue counts are as-of that date and will drift.

**Scope note:** this is about the modern eBay REST APIs — the
`api.ebay.com/{buy,sell,commerce,developer}/...` family (Browse, Feed, Inventory,
Fulfillment, Account, Analytics, Taxonomy, Identity, etc.) — not the legacy
Trading/Finding/Shopping XML APIs except for contrast.

**Note on sourcing:** `developer.ebay.com` blocks automated fetches (HTTP 403 with a
bot wall) from this research environment. Where a `developer.ebay.com` page is cited,
its content was verified through the Internet Archive Wayback Machine snapshot of that
same canonical URL; the canonical URL is the one cited so you can open it in a normal
browser. GitHub and PyPI were queried directly via their APIs.

---

## Bottom line

For an MCP server that needs a **focused, reliable, type-safe subset** of eBay REST
endpoints with minimal dependencies, **generate a thin client from eBay's official
OpenAPI 3 specs** (per-API JSON/YAML contracts published on `developer.ebay.com`), or
hand-roll an even thinner client and keep the specs only as a reference. Do **not**
build on `ebaysdk` — it targets only the legacy XML APIs, its last release was
April 2020, and it advertises Python 2.6/2.7 support
([PyPI](https://pypi.org/project/ebaysdk/), [repo](https://github.com/timotheus/ebaysdk-python)).
The one real REST wrapper on PyPI, **`ebay-rest`**, is genuinely maintained (v1.1.4,
Feb 2026), MIT-licensed, supports Python 3.10–3.14, and is itself Swagger-Codegen-generated
from eBay's specs
([pyproject.toml](https://github.com/matecsaj/ebay_rest/blob/main/pyproject.toml)) —
but it is synchronous-only, thinly type-hinted, returns untyped `dict`s, pulls in
`playwright`/`selenium` for its user-token flow, ships *all* eBay APIs as one ~1,500-file
package, and its maintainer has explicitly **frozen the v1 line** "due to upstream
spec/tooling" while v2 is redesigned
([issue](https://github.com/matecsaj/ebay_rest/issues), see "V1 - ** frozen due to upstream spec/tooling **").
The OAuth2 piece is **not** the hard part: eBay uses a standard OAuth2
`client_credentials` grant (application token) and `authorization_code` + `refresh_token`
grant (user token) against a single token endpoint
(`https://api.ebay.com/identity/v1/oauth2/token`), HTTP Basic auth with
`base64(client_id:client_secret)`, ~30 lines of code with `httpx` or `authlib`
([client credentials grant](https://developer.ebay.com/api-docs/static/oauth-client-credentials-grant.html),
[eBay's own MCP token manager](https://github.com/eBay/npm-public-api-mcp/blob/main/src/helper/token-manager.ts)).
eBay ships an **official MCP server** (`@ebay/npm-public-api-mcp`, TypeScript, Apache-2.0,
actively developed) that you may want to use directly or study rather than reimplement
([repo](https://github.com/eBay/npm-public-api-mcp)).

**Recommendation:** hand-roll a thin async client (httpx + a small OAuth2 token
manager) for the 5–15 endpoints the MCP server actually needs, using eBay's OpenAPI
specs to generate Pydantic models for just those request/response shapes (or to drive a
scoped `openapi-python-client` generation). This gives type safety, `async`, 3.12+, and
~2 direct dependencies, and avoids taking a dependency on a frozen or heavyweight
wrapper. Keep `ebay-rest` in mind only as a fast prototyping shortcut or a reference
implementation of the token/refresh dance.

---

## 1. Python client libraries for the eBay APIs

### Summary table

| Library (PyPI) | Maintainer | eBay APIs covered | Last release | Last commit | Py 3.12+ | Async | Types | License |
|---|---|---|---|---|---|---|---|---|
| [`ebay-rest`](https://pypi.org/project/ebay-rest/) | Third-party (Peter Matecsa, `matecsaj`) — **not** eBay | Modern REST (Buy, Sell, Commerce, Developer) | v1.1.4, 2026-02-15 | 2026-03-03 | Yes (3.10–3.14) | No | Minimal (Swagger-Codegen docstrings; returns `dict`) | MIT |
| [`ebaysdk`](https://pypi.org/project/ebaysdk/) | Third-party (Tim Keefer, `timotheus`); repo also mirrored at [`eBay/ebaysdk-python`](https://github.com/eBay/ebaysdk-python) | **Legacy XML only** (Trading, Finding, Shopping, Merchandising) | v2.2.0, 2020-04-20 | 2021-11-23 | Unofficially (classifiers say Py2.6–3.5) | No | No | CDDL-1.0 |
| [`eBay/ebay-oauth-python-client`](https://github.com/eBay/ebay-oauth-python-client) | **Official eBay** | OAuth2 token minting only (not API calls) | Not on PyPI | 2022-08-30 | No (declares "Python 2.7 project") | No | No | Apache-2.0 (repo LICENSE) |
| [`eBay/digital-signature-sdk-python`](https://github.com/eBay/digital-signature-sdk-python) | **Official eBay** | Message-signing helper for the subset of Sell/Finances APIs that require [digital signatures](https://developer.ebay.com/develop/guides/digital-signatures-for-apis) | pyproject present, not confirmed on PyPI | 2026-01-28 | Likely | No | Partial | Apache-2.0 |
| [`eBay/FeedSDK-Python`](https://github.com/eBay/FeedSDK-Python) | **Official eBay** | Buy **Feed API** only (download/filter TSV feed files) | — | 2022-04-08 | Py3 port exists | No | No | Apache-2.0 |
| [`GustavoBelaunde2004/ebay_python_sdk`](https://github.com/GustavoBelaunde2004/ebay_python_sdk) | Community (single author) | Modern REST subset (Browse, Orders, Inventory, Account) | created 2025-11-25 | 2025-12-05 | Yes | — | "type-safe models" claimed | MIT |

There is **no official, full eBay Python SDK for the REST APIs.** eBay's official
first-party code artifacts in this space are: the OAuth client libraries (per language),
the digital-signature SDKs (per language), the Feed SDKs, event-notification SDKs, and —
newest — the TypeScript **MCP server** (`@ebay/npm-public-api-mcp`). eBay's stated
strategy for REST API clients is "download our OpenAPI contract and generate a client in
your language"
([Take advantage of OpenAPI specifications of eBay APIs](https://developer.ebay.com/api-docs/static/gs_take-advantage-of-openapi.html);
[eBay's OpenAPI 3.0 Contracts Now Available](https://developer.ebay.com/updates/blog/ebays-openapi-3-0-contracts-now-available)).

### 1a. `ebay-rest` (the one to actually consider)

- **PyPI:** [`ebay-rest`](https://pypi.org/project/ebay-rest/) — current **1.1.4**, uploaded
  **2026-02-15**
  ([PyPI JSON](https://pypi.org/pypi/ebay-rest/json)). Prior cadence: 1.0.12 (2025-06),
  1.0.13 (2025-07), 1.0.14 (2025-08), 1.1.0 (2025-11), 1.1.1–1.1.4 through Feb 2026 —
  i.e. active, roughly monthly releases.
- **Source repo:** [`github.com/matecsaj/ebay_rest`](https://github.com/matecsaj/ebay_rest).
  71 stars. Not affiliated with eBay (README "Legal" section:
  *"This project is not affiliated with or endorsed by eBay Inc."*
  [README.md](https://github.com/matecsaj/ebay_rest/blob/main/README.md)).
- **Last commit:** `pushed_at` **2026-03-03**; recent commits (Jan–Feb 2026) include
  *"Refactor OAuth2 logic to use Authlib"*, *"Improved thread safety … token expiry"*,
  and *"Removed deprecated models and API for eBay Marketing"*
  (GitHub commits API for `matecsaj/ebay_rest`).
- **APIs covered:** the modern REST family only. The generated client code lives in
  `src/ebay_rest/api/` and is produced with **Swagger Codegen** from eBay's OpenAPI
  contracts — `pyproject.toml` excludes `src/ebay_rest/api/` from Black with the comment
  *"Exclude directories that contain Swagger-generated code"* and the `dev` extra notes
  *"Swagger Codegen must be installed separately"*
  ([pyproject.toml](https://github.com/matecsaj/ebay_rest/blob/main/pyproject.toml)).
  README claims *"Over a hundred methods are available"*
  ([README.md](https://github.com/matecsaj/ebay_rest/blob/main/README.md)).
- **Open issues / PRs:** 15 open issues, **0 open PRs**
  (GitHub repo metadata for `matecsaj/ebay_rest`). Maintainer *is* responsive — issues
  have maintainer comments and there is an organized roadmap. **Important:** the open
  issues include *"V1 - ** frozen due to upstream spec/tooling **"* and
  *"V2 - Establish 1.x Maintenance Branch, Start 2.x Development"*, and the v2 wishlist
  contains *"Add asynchronous execution support"*, *"Make Selenium optional"*,
  *"Replace custom HTTP signing code with RFC 9421 library"*, and
  *"migrate from authlib.jose to joserfc"*
  ([issues list](https://github.com/matecsaj/ebay_rest/issues)). So v1 is in
  maintenance-only mode and a breaking redesign is pending.
- **Python support:** `requires-python = ">=3.10"`; classifiers list 3.10, 3.11, **3.12**,
  3.13, 3.14; "Development Status :: 5 - Production/Stable"
  ([pyproject.toml](https://github.com/matecsaj/ebay_rest/blob/main/pyproject.toml)).
- **Type hints:** minimal. The Swagger-Codegen output uses docstring-style type info and
  the public surface returns plain `dict`s (README FAQ: results are
  *"dict (objects) … list (arrays) … Optional elements may be omitted"*
  [README.md](https://github.com/matecsaj/ebay_rest/blob/main/README.md)). Some
  hand-written modules were recently touched for *"comments, type hints and black"*
  (commit 2026-02-11).
- **Async:** none in the shipped package (`requests`-based). `aiohttp`/`aiofiles` appear
  only in the `dev` extra for the code-generation scripts. Async is a v2 wishlist item
  ([pyproject.toml](https://github.com/matecsaj/ebay_rest/blob/main/pyproject.toml),
  [issues](https://github.com/matecsaj/ebay_rest/issues)).
- **License:** MIT
  ([LICENSE](https://github.com/matecsaj/ebay_rest/blob/main/LICENSE)).
- **Runtime dependencies:** `authlib`, `certifi`, `cryptography`, `requests`,
  `python-dateutil`, `six`, `urllib3`; plus optional `playwright` via the `[complete]`
  extra for browser-driven user-token capture
  ([pyproject.toml](https://github.com/matecsaj/ebay_rest/blob/main/pyproject.toml)).

### 1b. `ebaysdk` (legacy — do not use for REST)

- **PyPI:** [`ebaysdk`](https://pypi.org/project/ebaysdk/) — current **2.2.0**, uploaded
  **2020-04-20** ([PyPI JSON](https://pypi.org/pypi/ebaysdk/json)). Classifiers:
  Python 2, 2.6, 2.7, 3, 3.5. `requires_python` is unset. License:
  **CDDL-1.0**.
- **Repo:** [`timotheus/ebaysdk-python`](https://github.com/timotheus/ebaysdk-python) —
  856 stars, but **166 open issues**, last commit **2021-11-23**
  (GitHub metadata). eBay keeps a mirror at
  [`eBay/ebaysdk-python`](https://github.com/eBay/ebaysdk-python) (created 2024-09, but
  the code is the same stale 2021 tree, 0 issues because they're not tracked there).
- **APIs covered:** legacy only. README: *"standardizing calls … across the Finding,
  Shopping, Merchandising & Trading APIs"*, with a generic `HTTP` class and a `Parallel`
  class ([README.rst](https://github.com/timotheus/ebaysdk-python/blob/master/README.rst)).
  Trading/Shopping/Merchandising/Finding are the legacy XML/SOAP APIs; several are
  deprecated or on eBay's decommission track
  ([API Deprecation Status](https://developer.ebay.com/develop/apis/api-deprecation-status)).
- **Async / types:** none.
- **Verdict:** unmaintained, wrong API family, Python-2-era. Only relevant if you need a
  legacy call the REST APIs don't cover yet (e.g. some Trading-only listing operations).

### 1c. `eBay/ebay-oauth-python-client` (official, OAuth-only, stale)

- **Repo:** [`github.com/eBay/ebay-oauth-python-client`](https://github.com/eBay/ebay-oauth-python-client)
  — 102 stars, 11 open issues, **last commit 2022-08-30** (a dependabot PyYAML bump),
  substantive code last touched **2019** (GitHub metadata).
- **Not on PyPI** under `ebay-oauth-python-client`, `ebay-oauth`, `ebay_oauth`, or
  `ebay-oauth-client` (checked PyPI JSON API for each). There is an unrelated
  squatted `ebayoauth 0.0.1`.
- **What it does:** only OAuth token acquisition — `oauth2api.get_application_token`
  (client_credentials), `generate_user_authorization_url` +
  `exchange_code_for_access_token` (authorization_code / refresh_token). It does **not**
  make API calls
  ([README.adoc](https://github.com/eBay/ebay-oauth-python-client/blob/master/README.adoc)).
- **Python support:** README states *"This is created as a Python 2.7 project"*;
  `requirements.txt` pins `selenium==3.141.0`, `requests==2.21.0`, `PyYAML==5.4`
  ([requirements.txt](https://github.com/eBay/ebay-oauth-python-client/blob/master/requirements.txt)).
- **License:** repo LICENSE.txt (Apache-2.0; GitHub reports "NOASSERTION" because the
  file header is non-standard).
- **Verdict:** the canonical reference for *what* the eBay OAuth flows are, but not
  something to depend on in 2026. Its Java/Node siblings are more current
  ([ebay-oauth-java-client](https://github.com/eBay/ebay-oauth-java-client),
  [ebay-oauth-nodejs-client](https://github.com/eBay/ebay-oauth-nodejs-client)).

### 1d. Other official eBay Python repos (niche)

- [`eBay/digital-signature-sdk-python`](https://github.com/eBay/digital-signature-sdk-python)
  — Apache-2.0, last commit 2026-01-28. Implements the
  [message-signing scheme](https://developer.ebay.com/develop/guides/digital-signatures-for-apis)
  now **required** for certain Sell/Finances/Fulfillment write operations involving
  payment data. If the MCP server touches those endpoints you will need this logic
  (or an RFC 9421 library) regardless of the rest of your client.
- [`eBay/FeedSDK-Python`](https://github.com/eBay/FeedSDK-Python) — Apache-2.0, last real
  commit 2022-04-08. Buy **Feed API** only.
- [`eBay/event-notification-*-sdk`](https://github.com/eBay) — no Python variant (Java,
  Node, PHP, Go).

### 1e. Community "modern" attempts

- [`GustavoBelaunde2004/ebay_python_sdk`](https://github.com/GustavoBelaunde2004/ebay_python_sdk)
  — MIT, created 2025-11, 3 stars, single contributor, "Browse, Orders, Inventory,
  Account". Too new/small to depend on but shows the shape of a hand-rolled typed client.
- Many "eBay MCP" repos bundle their own tiny Browse-API client (see §6).

---

## 2. Are they outdated?

| Library | Targets current REST APIs? | Maintained? | Works on modern Python (3.12+)? |
|---|---|---|---|
| `ebay-rest` | **Yes** (generated from eBay OpenAPI specs) | **Yes**, but v1 is officially "frozen … due to upstream spec/tooling" with v2 redesign pending ([issues](https://github.com/matecsaj/ebay_rest/issues)) | **Yes** — classifiers 3.10–3.14 ([pyproject.toml](https://github.com/matecsaj/ebay_rest/blob/main/pyproject.toml)) |
| `ebaysdk` | **No** — legacy Finding/Shopping/Merchandising/Trading only ([README.rst](https://github.com/timotheus/ebaysdk-python/blob/master/README.rst)) | **No** — last release 2020-04, last commit 2021-11, 166 open issues ([repo](https://github.com/timotheus/ebaysdk-python)) | Not officially — classifiers stop at 3.5 ([PyPI](https://pypi.org/project/ebaysdk/)); community patches exist |
| `ebay-oauth-python-client` | OAuth only | **No** — 2019-era code, "Python 2.7 project" ([README.adoc](https://github.com/eBay/ebay-oauth-python-client/blob/master/README.adoc)) | **No** as-shipped |
| `FeedSDK-Python` | Feed API only | **No** — 2022 | partial |

**Net:** `ebay-rest` is the only maintained option that targets the modern REST APIs on
current Python. Everything else is either the wrong API family or effectively abandoned.

---

## 3. How hard is each to use?

### eBay's OAuth2 model (applies to any client)

- **Token endpoint:** `https://api.ebay.com/identity/v1/oauth2/token` (production),
  `https://api.sandbox.ebay.com/identity/v1/oauth2/token` (sandbox)
  ([client credentials grant](https://developer.ebay.com/api-docs/static/oauth-client-credentials-grant.html);
  confirmed in eBay's OpenAPI contracts, e.g. the Browse spec's
  `securitySchemes.api_auth.flows.clientCredentials.tokenUrl`, and in
  [eBay's MCP `constants.ts`](https://github.com/eBay/npm-public-api-mcp/blob/main/src/constant/constants.ts)).
- **Application token** (client-credentials grant): POST
  `grant_type=client_credentials&scope=<space-separated scopes>`, with
  `Authorization: Basic base64(client_id:client_secret)` and
  `Content-Type: application/x-www-form-urlencoded`. Response:
  `{"access_token": "...", "expires_in": 7200, "token_type": "Application Access Token"}`
  — valid **2 hours**, no refresh token; just mint a new one
  ([client credentials grant](https://developer.ebay.com/api-docs/static/oauth-client-credentials-grant.html)).
  Used for public data (Browse search, Taxonomy, Catalog, etc.).
- **User token** (authorization-code grant): redirect the user to a consent URL, receive
  a `code`, POST `grant_type=authorization_code&code=...&redirect_uri=<RuName>`. Response:
  `{"access_token": "...", "expires_in": 7200, "refresh_token": "...",
  "refresh_token_expires_in": 47304000, "token_type": "User Access Token"}` — access
  token **2 hours**, refresh token **~547 days (~18 months)**. Refresh with
  `grant_type=refresh_token&refresh_token=...` (scopes are inherited, don't re-send them)
  ([authorization code grant](https://developer.ebay.com/api-docs/static/oauth-authorization-code-grant.html)).
  Used for private data (a seller's inventory/orders, a buyer's cart).
- **Scopes:** every method lists its required OAuth scope(s); the token must be minted
  with a superset ([OAuth scopes](https://developer.ebay.com/api-docs/static/oauth-scopes.html)).
- **This is standard OAuth2.** eBay's own MCP implements the whole thing —
  in-memory token cache, proactive refresh 5 min before expiry, 401-retry, and a
  concurrency guard — in a **single ~350-line file** with just `axios`
  ([token-manager.ts](https://github.com/eBay/npm-public-api-mcp/blob/main/src/helper/token-manager.ts)).
  In Python that's `httpx` + ~40 lines, or `authlib`'s `OAuth2Client` with almost no
  custom code.

### `ebay-rest`

- **OAuth:** handled. You put credentials (and optionally a pre-obtained
  `refresh_token` + `refresh_token_expiry`) in an `ebay_rest.json` config; the library
  mints/caches/refreshes application and user tokens (now via `authlib`). For the
  *first* user-token acquisition it will drive a headless browser (Playwright, `[complete]`
  extra) to complete the consent screen unless you supply the refresh token yourself
  ([README.md](https://github.com/matecsaj/ebay_rest/blob/main/README.md), FAQ
  *"Can the browser automation be avoided?"*). Token handling functions are **not** in
  the public API yet (open issue *"V2 - Expose token handling functions"*).
- **Rate limiting:** not actively managed; README only warns that repeated identical
  calls can trigger eBay "Internal Error"
  ([README.md](https://github.com/matecsaj/ebay_rest/blob/main/README.md)).
- **Pagination:** handled — paged endpoints are exposed as Python **generators**; you
  omit `offset`, optionally set `limit`, or omit `limit` to iterate everything (subject
  to eBay's 10,000-record ceiling)
  ([README.md](https://github.com/matecsaj/ebay_rest/blob/main/README.md), FAQ
  *"How are paged API results handled?"*).
- **Sandbox vs production:** selected by which named credential set you pass
  (`application='production_1'` vs a sandbox set) in the config
  ([ebay_rest_EXAMPLE.json](https://github.com/matecsaj/ebay_rest/blob/main/ebay_rest_EXAMPLE.json)).
- **Threading:** documented as safe; multiprocessing untested
  ([README.md](https://github.com/matecsaj/ebay_rest/blob/main/README.md)).
- **Minimal authenticated REST call (from the project README):**

  ```python
  from ebay_rest import API, DateTime, Error, Reference

  try:
      api = API(application='production_1', user='production_1', header='US')
  except Error as error:
      print(f'Error {error.number} is {error.reason} {error.detail}.')
  else:
      for record in api.buy_browse_search(q='iPhone', sort='price', limit=5):
          if 'record' in record:
              item = record['record']
              print(f"item id: {item['item_id']} {item['item_web_url']}")
  ```
  ([README.md](https://github.com/matecsaj/ebay_rest/blob/main/README.md))

### `ebaysdk` (legacy, for contrast)

```python
from ebaysdk.finding import Connection
api = Connection(appid='YOUR_APPID_HERE', config_file=None)
response = api.execute('findItemsAdvanced', {'keywords': 'legos'})
print(response.dict())
```
([README.rst](https://github.com/timotheus/ebaysdk-python/blob/master/README.rst)) — note
this is the legacy Finding API (`svcs.ebay.com`), not a REST call, and `appid`-only
(no OAuth).

### `ebay-oauth-python-client`

```python
from ebay_oauth_python_client import oauth2api, credentialutil, model
credentialutil.load('ebay-config.yaml')
app_token = oauth2api().get_application_token(model.environment.PRODUCTION,
                                              ['https://api.ebay.com/oauth/api_scope'])
```
([README.adoc](https://github.com/eBay/ebay-oauth-python-client/blob/master/README.adoc))
— gets you a token string; you still make all HTTP calls yourself.

---

## 4. eBay's own OpenAPI specs

**Yes, eBay publishes machine-readable OpenAPI contracts per API, per version, in both
JSON and YAML.** eBay joined the OpenAPI Initiative in Aug 2017 and was an early
publisher of OpenAPI **3.0** contracts
([eBay's OpenAPI 3.0 Contracts Now Available](https://developer.ebay.com/updates/blog/ebays-openapi-3-0-contracts-now-available);
[Take advantage of OpenAPI specifications of eBay APIs](https://developer.ebay.com/api-docs/static/gs_take-advantage-of-openapi.html);
Linux Foundation / OpenAPI blog:
[eBay Provides OpenAPI Specification (OAS) for All its RESTful Public APIs](https://www.openapis.org/blog/2018/08/14/ebay-provides-openapi-specification-oas-for-all-its-restful-public-apis)).

### Where to get them

Each API's **Overview** page in the API docs has download links labelled
**"OpenAPI 3 JSON Contract"** and **"OpenAPI 3 YAML Contract"** (some still marked
"(beta)"). The URL pattern is:

```
https://developer.ebay.com/api-docs/master/{category}/{api}/openapi/3/{category}_{api}_v1_oas3.json
https://developer.ebay.com/api-docs/master/{category}/{api}/openapi/3/{category}_{api}_v1_oas3.yaml
```

Example (verified — fetched and parsed):
`https://developer.ebay.com/api-docs/master/buy/browse/openapi/3/buy_browse_v1_oas3.json`
returned a valid **`"openapi": "3.0.0"`** document, `info.title = "Browse API v1.20.4"`,
`servers[0].url = "https://api.ebay.com{basePath}"` with `basePath` default
`/buy/browse/v1`, 7 paths, and:

```json
"securitySchemes": {
  "api_auth": {
    "type": "oauth2",
    "flows": {
      "clientCredentials": {
        "tokenUrl": "https://api.ebay.com/identity/v1/oauth2/token",
        "scopes": {
          "https://api.ebay.com/oauth/api_scope": "View public data from eBay",
          "https://api.ebay.com/oauth/api_scope/buy.item.bulk": "Retrieve eBay items in bulk."
        }
      }
    }
  }
}
```

with per-operation `security` such as
`{"api_auth": ["https://api.ebay.com/oauth/api_scope"]}` on `GET /item_summary/search`
(link from the [Browse API overview](https://developer.ebay.com/api-docs/buy/browse/overview.html);
verified via Wayback snapshot of the `.json` contract).

Older **OAS 2.0 (Swagger)** contracts also exist under an `.../openapi/2/...` path for
some APIs; eBay's guidance is to prefer the 3.0 contracts
([Take advantage of OpenAPI specifications of eBay APIs](https://developer.ebay.com/api-docs/static/gs_take-advantage-of-openapi.html)).

### Coverage

eBay states OAS contracts are available for **all its RESTful public APIs** — the Buy,
Sell, Commerce, and Developer families
([Take advantage of OpenAPI specifications of eBay APIs](https://developer.ebay.com/api-docs/static/gs_take-advantage-of-openapi.html);
[All eBay RESTful APIs Leverage OpenAPI Specification](https://developer.ebay.com/updates/blog/openapi-coverage)).
The legacy Trading/Finding/Shopping APIs are **not** OpenAPI (they have WSDLs instead).

### Usability for code generation

- eBay explicitly markets the specs as usable with generators: *"download OpenAPI
  contract … generate clients in one of 40+ supported programming languages, and
  successfully invoke an eBay API in minutes"*
  ([Take advantage of OpenAPI specifications of eBay APIs](https://developer.ebay.com/api-docs/static/gs_take-advantage-of-openapi.html)).
- Real-world confirmation: `ebay-rest` generates its entire client layer with
  **Swagger Codegen** from these contracts
  ([pyproject.toml](https://github.com/matecsaj/ebay_rest/blob/main/pyproject.toml)),
  and there is a large ecosystem of per-API generated PHP SDKs
  (`michabbb/sdk-ebay-rest-*`, generated with `openapi-generator`,
  [example](https://github.com/michabbb/sdk-ebay-rest-account)).
- eBay's **own MCP server** consumes the specs at runtime: it parses them with
  `@apidevtools/swagger-parser`, builds tool schemas from each operation, and even
  offers a hosted **spec-discovery API**
  (`https://api.ebay.com/developer/mcp/v1/search?query=...`) so a client can find the
  right contract/operation by natural-language query
  ([openapi-helper.ts](https://github.com/eBay/npm-public-api-mcp/blob/main/src/helper/openapi-helper.ts),
  [constants.ts](https://github.com/eBay/npm-public-api-mcp/blob/main/src/constant/constants.ts)).
- Known friction points (from the `ebay-rest` maintainer and PHP-SDK authors): eBay's
  contracts sometimes lag the HTML docs, a few `securityDefinitions`/`oauth` blocks in
  the Commerce/Sell contracts have been reported as inaccurate (`ebay-rest` open issue
  *"V2 - Potential OAuth issues in some eBay Commerce and Sell OpenAPI contracts"*
  [issues](https://github.com/matecsaj/ebay_rest/issues)), and the sheer number of
  models (the Browse contract alone is ~380 KB) makes "generate everything" produce a
  large package. Generating **per-API** or **per-operation** keeps it manageable.
- For Python specifically, `openapi-python-client` (Pydantic v2 models, `httpx`, async
  support) is a better fit than Swagger Codegen's dated Python template, and can be
  pointed at a single eBay contract.

---

## 5. Recommendation: library vs. generated client vs. hand-rolled

### Context that shapes the answer

- The consumer is an **MCP server** that will expose a **small, curated set of tools**
  (likely: Browse search + get item; maybe Taxonomy; maybe a few Sell/Inventory/
  Fulfillment reads). It does **not** need all ~100 eBay operations.
- MCP servers benefit from **reliability, explicit typed schemas** (tool
  input/output schemas map naturally onto Pydantic models), **`async`** (MCP Python SDK
  is async), and **few dependencies** (easy `uvx`/`pipx` distribution).

### Options

| Option | Pros | Cons |
|---|---|---|
| **Use `ebay-rest`** | Fastest to a working call; OAuth + refresh + pagination already solved; covers every API; MIT; maintained; Py3.12+ | Sync only (blocks the MCP event loop or forces a thread pool); returns untyped `dict`s — you'd re-wrap for tool schemas anyway; heavy transitive deps (`authlib`, `cryptography`, `six`, and `playwright` if you need first-time user tokens); ships ~1,500 files for the whole API surface; **v1 explicitly frozen**, v2 will be breaking; not eBay-official |
| **Generate a client from eBay's OpenAPI specs** (per-API, via `openapi-python-client`) | Exact typed models (Pydantic v2) for just the APIs you use; async `httpx`; regenerate when eBay updates the contract; no third-party wrapper to bit-rot; you control the surface | You still write the OAuth2 token manager + auth injection (generators produce the calls, not the token dance); generated code can be verbose; occasional spec inaccuracies need patching; multiple APIs = multiple generated packages or a merge step |
| **Hand-roll a thin client** (httpx + small token manager + a few Pydantic models) | Smallest dependency footprint (httpx + pydantic); async native; you model only the ~10 fields per endpoint the MCP tools expose; trivial to test/mock; no upstream freeze risk | You own everything: pagination helper, retry/429 handling, sandbox switch, and — if you touch payment-related Sell endpoints — digital-signature signing; more upfront code than `pip install` |

### The OAuth2 question specifically

**OAuth2 is not the hard part and not a strong reason to take a library.** It's a
textbook client-credentials + refresh-token flow against one endpoint with HTTP Basic
auth. eBay's own MCP does it in one small file with only `axios`
([token-manager.ts](https://github.com/eBay/npm-public-api-mcp/blob/main/src/helper/token-manager.ts));
`ebay-rest` recently *simplified* its implementation by delegating to `authlib`
(commit *"Refactor OAuth2 logic to use Authlib"*, 2026-01-31). In Python you can either:

- use **`authlib`**'s `OAuth2Client` / `AsyncOAuth2Client` (handles token caching +
  auto-refresh with an `update_token` hook), or
- write ~40 lines: cache `(access_token, expires_at)`, refresh at `expires_at - 300s`,
  on `401` clear and retry once.

The genuinely fiddly eBay-specific bits are **(a)** the first-time user-consent redirect
(only needed if the MCP server acts on behalf of a signed-in eBay user — a pure Browse
server uses only the application token and never touches this), and **(b)** digital
signatures for a handful of payment-related Sell operations. A library only saves you
(a), and only partially (it automates it with a headless browser, which you likely don't
want in an MCP server anyway).

### Recommended approach

1. **Hand-roll a thin async client**: `httpx.AsyncClient` + a small `TokenManager`
   (application-token by default; optional refresh-token mode) + a `paginate()` helper +
   429/`Retry-After` handling + a `EBAY_ENV` sandbox/production switch.
2. **Use eBay's OpenAPI 3 contracts as the source of truth** for the endpoints you
   expose — either run `openapi-python-client` against the specific contract(s) and keep
   only the models/endpoints you need, or copy the relevant request/response shapes into
   hand-written Pydantic models. Pin the `info.version` you generated against and
   diff on eBay contract updates.
3. Dependencies stay at roughly **`httpx`, `pydantic`** (+ the MCP SDK). Optionally
   `authlib` if you'd rather not hand-write refresh logic.
4. Keep `ebay-rest` installed in a spike branch as a **reference** for tricky
   endpoints and as a sanity check on request shapes.

Use `ebay-rest` directly instead only if: you need broad, changing API coverage fast;
you're fine with sync calls behind a thread pool; and you accept the pending v2
breakage.

---

## 6. Other things that matter for the decision

### eBay Developers Program registration / keysets

- You must **join the eBay Developers Program** and create an **application keyset**
  (App ID / Client ID, Dev ID, Cert ID / Client Secret) at
  [developer.ebay.com/my/keys](https://developer.ebay.com/my/keys); separate keysets for
  Sandbox and Production
  ([Creating an eBay developer account](https://developer.ebay.com/api-docs/static/creating-edp-account.html)).
- *"After you join the eBay Developers Program and get your application keyset, you can
  start using eBay APIs immediately"* with default call limits
  ([API call limits](https://developer.ebay.com/develop/apis/api-call-limits)).
- **Restricted APIs need approval.** Most **Buy APIs (including Browse, Feed, Marketing,
  Marketplace Insights, Deal, Order)** are *limited/restricted release* — usable with
  default limits for testing, but production access at scale (and some at all) requires
  passing the **Application Growth Check**, where eBay reviews your app for License
  Agreement compliance
  ([Use the application growth check to get access to restricted APIs](https://developer.ebay.com/api-docs/static/gs_use-the-application-growth.html);
  [Get started on a buying application](https://developer.ebay.com/develop/get-started/get-started-on-a-buying-application)).
  This is a real gate for a Browse-based MCP server intended for third-party use.

### Rate limits (default tier, per calendar day, resets 00:00 UTC-7)

From [API call limits](https://developer.ebay.com/develop/apis/api-call-limits):

| API | Default daily limit |
|---|---|
| Buy → **Browse** | **5,000 / day** |
| Buy → Feed (beta) | 10,000/day (item/item_group); 75,000/day (item_snapshot) |
| Buy → Marketing, Marketplace Insights, Deal, Offer, Order | 5,000 / day each |
| Commerce → Taxonomy, Charity, Identity, Translation | 5,000 / day each |
| Commerce → Catalog | 10,000 / day |
| Sell → Account | 25,000 / day |
| Sell → Inventory | 2,000,000 / day |
| Sell → Fulfillment (Order) | 100,000 / day |
| Sell → Feed | 100,000 / day |
| Sell → Analytics | 100–400 / day (very low) |
| Developer → Analytics (rate-limit introspection) | 5,000 / day |

The **Developer Analytics API** `getRateLimits` / `getUserRateLimits` lets your client
query its own remaining quota programmatically
([Analytics API](https://developer.ebay.com/api-docs/developer/analytics/overview.html)).
The Browse 5,000/day default is low — an MCP server used interactively by several people
can hit it; plan caching and/or an Application Growth Check.

### Sandbox

- Full sandbox at `api.sandbox.ebay.com` with its own keyset, OAuth endpoint
  (`https://api.sandbox.ebay.com/identity/v1/oauth2/token`), and test users
  ([sandbox test users](https://developer.ebay.com/tools/sandbox);
  [constants.ts](https://github.com/eBay/npm-public-api-mcp/blob/main/src/constant/constants.ts)).
- Sandbox data is sparse/synthetic — Browse search often returns little. eBay's own MCP
  server **does not officially support sandbox** yet
  ([README](https://github.com/eBay/npm-public-api-mcp/blob/main/README.md),
  "Important Limitations").

### Existing eBay MCP servers

- **Official:** [`eBay/npm-public-api-mcp`](https://github.com/eBay/npm-public-api-mcp)
  (`npm i @ebay/npm-public-api-mcp`) — TypeScript, **Apache-2.0**, created 2026-01,
  last commit **2026-08-14**, actively developed (31+ merged PRs), Node 22+. Takes
  `EBAY_CLIENT_ID` / `EBAY_CLIENT_SECRET` (+ optional `EBAY_REFRESH_TOKEN` for user
  mode); auto-manages tokens. **Production is read-only (GET only)** and **sandbox
  unsupported** in the current release. It's a *generic* server: it discovers eBay REST
  operations from the OpenAPI specs and exposes them as tools, rather than curating a
  set ([README](https://github.com/eBay/npm-public-api-mcp/blob/main/README.md)).
- **PyPI:** [`ebay-mcp` 0.1.0](https://pypi.org/project/ebay-mcp/) — *"eBay Browse API
  MCP server — buy-side listing search and price intelligence"*, `requires_python
  >=3.12`. Community, brand-new, single release.
- **GitHub (community, dozens):** e.g.
  [`cunicopia-dev/ebay-mcp`](https://github.com/cunicopia-dev/ebay-mcp) (Python, Browse,
  read-only), [`markddavidoff/ebay-mcp`](https://github.com/markddavidoff/ebay-mcp)
  (FastMCP + Browse), [`taka392/ebay-mcp`](https://github.com/taka392/ebay-mcp)
  (OAuth2 + Browse + Commerce Identity),
  [`hanku4u/ebay-mcp-server`](https://github.com/hanku4u/ebay-mcp-server) (Python 3.10+),
  [`KalGuinn/ebay-mcp`](https://github.com/KalGuinn/ebay-mcp) (read-only Browse),
  [`YosefHayim/ebay-mcp`](https://github.com/YosefHayim/ebay-mcp) (TypeScript, claims
  full Sell-API coverage, ~299 tools), plus `strangeco0l/ebay-mcp`, `acato/ebay-mcp`,
  `luke-nielsen/ebay-mcp`, `mrnajiboy/ebay-mcp-remote-edition`, and more. Quality
  varies widely; several are explicitly "Claude-generated". None is an obvious
  authoritative Python reference. The
  [`modelcontextprotocol/servers`](https://github.com/modelcontextprotocol/servers)
  official list does **not** include an eBay server.
- **Takeaway:** the space is crowded but shallow. If you only need read-only Browse,
  eBay's official TS server or a small community Python one may already suffice; if you
  need a curated, typed, write-capable Python server, that niche is open.

### API terms that matter for a third-party tool

- The **[eBay API License Agreement](https://developer.ebay.com/join/api-license-agreement)**
  governs all use. Points that bite third-party/AI tools: data caching limits and
  required refresh intervals; restrictions on storing/redistributing eBay item data;
  prohibition on using the APIs to build a competing marketplace or to scrape;
  attribution / "Powered by eBay" style requirements for buyer-facing surfaces; and the
  Application Growth Check as the compliance checkpoint before production scale
  ([API License Agreement](https://developer.ebay.com/join/api-license-agreement);
  [program policies](https://developer.ebay.com/develop/get-started)).
- The **Buy APIs** additionally require you to be an approved buying partner /
  affiliate for many use cases
  ([Get started on a buying application](https://developer.ebay.com/develop/get-started/get-started-on-a-buying-application)).
- **Marketplace Account Deletion / Closure notifications:** production apps that store
  eBay user data must implement an endpoint to receive account-deletion notifications
  ([Marketplace Account Deletion](https://developer.ebay.com/develop/guides-v2/marketplace-user-account-deletion/marketplace-user-account-deletion)).
  An MCP server that caches user data needs to account for this.
- **Digital Signatures:** required for certain Sell/Finances operations that move money
  ([Digital Signatures for APIs](https://developer.ebay.com/develop/guides/digital-signatures-for-apis)).

---

## Sources consulted

### eBay official — developer docs (canonical URLs; content verified via Wayback Machine)
- [Take advantage of OpenAPI specifications of eBay APIs](https://developer.ebay.com/api-docs/static/gs_take-advantage-of-openapi.html)
- [eBay's OpenAPI 3.0 Contracts Now Available (blog)](https://developer.ebay.com/updates/blog/ebays-openapi-3-0-contracts-now-available)
- [All eBay RESTful APIs Leverage OpenAPI Specification (blog)](https://developer.ebay.com/updates/blog/openapi-coverage)
- [Authorization / OAuth tokens overview](https://developer.ebay.com/api-docs/static/oauth-tokens.html)
- [The client credentials grant flow](https://developer.ebay.com/api-docs/static/oauth-client-credentials-grant.html)
- [The authorization code grant flow](https://developer.ebay.com/api-docs/static/oauth-authorization-code-grant.html)
- [OAuth scopes](https://developer.ebay.com/api-docs/static/oauth-scopes.html)
- [Browse API Overview](https://developer.ebay.com/api-docs/buy/browse/overview.html) and its OpenAPI 3 JSON contract `https://developer.ebay.com/api-docs/master/buy/browse/openapi/3/buy_browse_v1_oas3.json` (fetched & parsed)
- [API call limits](https://developer.ebay.com/develop/apis/api-call-limits)
- [Use the application growth check to get access to restricted APIs](https://developer.ebay.com/api-docs/static/gs_use-the-application-growth.html)
- [Get started on a buying application](https://developer.ebay.com/develop/get-started/get-started-on-a-buying-application)
- [Creating an eBay developer account](https://developer.ebay.com/api-docs/static/creating-edp-account.html)
- [Application keys page](https://developer.ebay.com/my/keys)
- [eBay API License Agreement](https://developer.ebay.com/join/api-license-agreement)
- [Digital Signatures for APIs](https://developer.ebay.com/develop/guides/digital-signatures-for-apis)
- [Marketplace User Account Deletion](https://developer.ebay.com/develop/guides-v2/marketplace-user-account-deletion/marketplace-user-account-deletion)
- [API deprecation status](https://developer.ebay.com/develop/apis/api-deprecation-status)

### eBay official — GitHub
- [`eBay/npm-public-api-mcp`](https://github.com/eBay/npm-public-api-mcp) — README, `src/constant/constants.ts`, `src/helper/token-manager.ts`, `src/helper/openapi-helper.ts`
- [`eBay/ebay-oauth-python-client`](https://github.com/eBay/ebay-oauth-python-client) — README.adoc, requirements.txt
- [`eBay/digital-signature-sdk-python`](https://github.com/eBay/digital-signature-sdk-python)
- [`eBay/FeedSDK-Python`](https://github.com/eBay/FeedSDK-Python)
- [`eBay/ebaysdk-python`](https://github.com/eBay/ebaysdk-python) (mirror)
- eBay org repo listing via GitHub API (218 public repos enumerated)

### Third-party libraries
- [`ebay-rest` on PyPI](https://pypi.org/project/ebay-rest/) + [PyPI JSON](https://pypi.org/pypi/ebay-rest/json)
- [`matecsaj/ebay_rest`](https://github.com/matecsaj/ebay_rest) — README.md, pyproject.toml, LICENSE, issues list, commits API, repo metadata
- [`ebaysdk` on PyPI](https://pypi.org/project/ebaysdk/) + [PyPI JSON](https://pypi.org/pypi/ebaysdk/json)
- [`timotheus/ebaysdk-python`](https://github.com/timotheus/ebaysdk-python) — README.rst, repo metadata
- [`GustavoBelaunde2004/ebay_python_sdk`](https://github.com/GustavoBelaunde2004/ebay_python_sdk)
- [`michabbb/sdk-ebay-rest-account`](https://github.com/michabbb/sdk-ebay-rest-account) (example of openapi-generator output)

### MCP ecosystem
- [`ebay-mcp` on PyPI](https://pypi.org/project/ebay-mcp/)
- [`modelcontextprotocol/servers`](https://github.com/modelcontextprotocol/servers)
- GitHub search for `ebay mcp` (community servers: cunicopia-dev, markddavidoff, taka392, hanku4u, KalGuinn, YosefHayim, strangeco0l, acato, luke-nielsen, mrnajiboy, and others)

### Standards / background
- [OpenAPI Initiative: eBay Provides OAS for All its RESTful Public APIs](https://www.openapis.org/blog/2018/08/14/ebay-provides-openapi-specification-oas-for-all-its-restful-public-apis)
- [RFC 6749 — OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)
- [`openapi-python-client`](https://github.com/openapi-generators/openapi-python-client)
