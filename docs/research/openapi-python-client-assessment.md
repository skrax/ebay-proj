# `openapi-python-client` capability assessment (for `ebay_client`)

**Research date:** 2026-09-02. Version numbers, dates and issue counts are as-of that
date and will drift.

**Ticket:** [skrax/ebay-proj#6](https://github.com/skrax/ebay-proj/issues/6) ·
**Map:** [skrax/ebay-proj#4](https://github.com/skrax/ebay-proj/issues/4) ·
**Prior research:** [`docs/research/ebay-python-api-clients.md`](./ebay-python-api-clients.md)

**Method:** primary sources (the tool's own README/CHANGELOG/templates on GitHub,
its CLI `--help`, PyPI + GitHub API metadata) plus a real hands-on trial run:
`openapi-python-client` **0.29.1** run under Python 3.12 via
`uvx openapi-python-client@0.29.1` against eBay's real **Browse API v1.20.4** OAS3
contract (`buy_browse_v1_oas3.json`, 7 paths / 75 schemas, retrieved from the
Internet Archive snapshot `20260210161815` of the canonical
`developer.ebay.com/api-docs/master/buy/browse/openapi/3/buy_browse_v1_oas3.json`
because `developer.ebay.com` bot-walls automated fetches), and against a small
hand-written synthetic OAS3 spec exercising `allOf` / `oneOf` / `anyOf` / `nullable`
/ `additionalProperties` / typed error bodies (the Browse contract contains none of
these). Trial artifacts were generated under `.scratch/opc-trial/` and are **not**
committed.

---

## Bottom line

**Fit: yes, with one significant caveat and one structural decision.**

`openapi-python-client` (opc) does exactly what the map wants at the transport layer:
one deterministic, `ruff`-formatted, fully type-annotated httpx client per spec, sync
**and** async, generated code that is meant to be committed, MIT-licensed, actively
maintained, Python 3.12/3.13/3.14 supported. The trial client imported and
round-tripped real eBay JSON on the first try.

**Caveat — models are `attrs`, not Pydantic v2.** opc has generated `@attrs.define`
classes with hand-written `to_dict()` / `from_dict()` since v0.15.0; there is no
Pydantic option and the maintainer has repeatedly declined to commit to one
([discussion #837](https://github.com/openapi-generators/openapi-python-client/discussions/837),
last activity 2025-03-08). The map (#4) and the downstream spec/tickets currently
say "**httpx + Pydantic v2**". That phrase has to change to "**httpx + attrs**", or
the project has to switch generators. See *Alternatives* and *Impact on other
tickets* below.

**Structural decision — one package per spec, composed by path, not merged.** opc
has no multi-spec mode. The clean approach is to generate each of the ~19 eBay
contracts as its own self-contained sub-package under
`src/ebay_client/_generated/<family>/<api>/` with `--meta none`, and have the
hand-written core import from them. Do **not** try to merge them into one model
namespace. Details below.

---

## 1. Multi-spec handling / composing ~19 packages into one `ebay_client`

opc generates **one package per OpenAPI document**. There is no CLI option to feed
multiple specs or to merge outputs (confirmed against `generate --help` and the
README — "*The content contains no information about … multiple OpenAPI documents*").
`--url` / `--path` each take exactly one document.

Each generated package is **fully self-contained**: it ships its own `client.py`,
`types.py`, `errors.py`, `models/` and `api/`, and every internal import is
**relative** (`from ...client import AuthenticatedClient`, `from ..types import
UNSET`). (trial run — verified across `browse_api_client/api/item_summary/search.py`
and every model file.) Nothing is imported by absolute package path, so a generated
package keeps working no matter how deeply it is nested inside another package.

### Recommended composition

Generate each spec with `--meta none` (emit just the module directory, no
`pyproject.toml`/README) into its own directory:

```
openapi-python-client generate \
  --path specs/buy_browse_v1_oas3.json \
  --meta none \
  --config config/buy_browse.yml \
  --output-path src/ebay_client/_generated/buy/browse
```

Result layout:

```
src/ebay_client/
  __init__.py                # hand-written public API
  auth.py, config.py, retry.py, pagination.py, errors.py   # hand-written core
  _generated/
    buy/
      browse/                # = the opc package, "browse_api_client" renamed away
        __init__.py  client.py  errors.py  types.py
        api/item_summary/search.py ...
        models/item.py ...
      feed/ ...
    sell/
      account/ ...
      inventory/ ...
    commerce/ ...
    developer/ ...
```

- **Namespacing:** by Python package path (`ebay_client._generated.sell.account`).
  The generated `Client` / `AuthenticatedClient` in each package are *identical
  code* (the `client.py.jinja` template is spec-independent) but are distinct
  classes; the hand-written core builds its own `httpx` client and injects it into
  each (see §4), so the duplication is inert.
- **Model-name collisions across specs:** a non-issue by construction. eBay reuses
  type names heavily (`Error`, `ErrorParameter`, `Amount`, `Address`,
  `ConvertedAmount`…); each becomes `..._generated.buy.browse.models.error.Error`
  vs `..._generated.sell.account.models.error.Error` — separate modules, separate
  classes, no clash. The cost is **duplication**: the same `Error` shape is
  regenerated ~19 times. For an eBay-sized surface this is tens of MB of committed
  Python but no correctness problem.
- **Model-name collisions *within* one spec** are real and only partly handled. opc
  disambiguates by prefixing the schema's path into the class name
  (`use_path_prefixes_for_title_model_names`, default `true`). Where that still
  collides after `snake_case`, opc has a known bug —
  [#1163 "Conflicting module names for models silently generate invalid code"](https://github.com/openapi-generators/openapi-python-client/issues/1163)
  and [#1117](https://github.com/openapi-generators/openapi-python-client/issues/1117).
  Mitigation: run with `--fail-on-warning` in CI and keep a per-spec
  `class_overrides` map for any spec that trips it.
- **Shared components / `$ref` across files:** eBay ships each API as a single
  self-contained document (no external `$ref`), so this doesn't arise. If a spec
  ever used remote refs, opc would need them bundled first (e.g. with
  `redocly bundle`).
- **`__init__.py` surface:** each package's top-level `__init__.py` exports only
  `Client` and `AuthenticatedClient`; `models/__init__.py` re-exports every model
  in an `__all__`. The endpoint functions are **not** re-exported — callers import
  `from ..._generated.buy.browse.api.item_summary import search` and call
  `search.sync(...)` / `search.asyncio(...)`. The hand-written core is what gives
  consumers an ergonomic facade over that.

### Renaming the package

`--output-path` sets the directory but the inner module name comes from the spec
title (`browse_api_client`). Use `package_name_override` in the per-spec config to
force it (`package_name_override: browse`) so the directory and module agree.

---

## 2. Exact output layout & module structure (trial run)

`--meta none`, synthetic spec:

```
<output-path>/
  __init__.py              # """A client library for accessing <title>""" ; exports Client, AuthenticatedClient
  client.py                # attrs @define Client + AuthenticatedClient
  errors.py                # UnexpectedStatus(Exception) only
  types.py                 # UNSET / Unset sentinel, File, FileTypes, RequestFiles, Response[T] (Generic)
  py.typed
  api/
    __init__.py
    <tag>/                 # one dir per OpenAPI tag; untagged -> "default"
      __init__.py
      <operation_id_snake_case>.py   # one module per operation
  models/
    __init__.py            # re-exports all models in __all__
    <schema_name_snake_case>.py      # one module per schema
```

With `--meta poetry|uv|pdm|setup` opc additionally emits a project root
(`pyproject.toml` + `README.md` + `.gitignore`) and nests the module dir one level
down. `--meta uv` produces a PEP 621 `pyproject.toml` with `uv_build` backend,
`requires-python = ">=3.11"`, deps `httpx>=0.23.1,<0.29.0` and `attrs>=22.2.0`, and
a `[tool.ruff]` block. For committed-into-a-monorepo use, **`--meta none` is the
right choice** — we don't want 19 nested `pyproject.toml`s.

Per-operation module (e.g. `api/item_summary/search.py`) contains exactly four
public callables plus three private helpers:

| callable | returns |
|---|---|
| `sync_detailed(*, client, ...)` | `Response[Union[...]]` (status, headers, content, parsed) |
| `sync(*, client, ...)` | the parsed body or `None` |
| `asyncio_detailed(*, client, ...)` | `Response[Union[...]]` |
| `asyncio(*, client, ...)` | parsed body or `None` |
| `_get_kwargs(...)` | dict of httpx request kwargs |
| `_parse_response(...)` | dispatch on `response.status_code` |
| `_build_response(...)` | wrap into `Response` |

`client` is a **required keyword-only argument on every call** — there is no
bound-method style. The full endpoint docstring (with eBay's raw HTML) is copied
into every one of the four functions. The Browse client is 76 model modules + 7
endpoint modules; `models/item.py` alone is 1332 lines.

---

## 3. Model shape; `oneOf` / `allOf` / `anyOf` / nullable / additionalProperties (trial run)

**Model shape.** Not Pydantic. Each model is:

```python
@_attrs_define
class Error:
    category: str | Unset = UNSET
    error_id: int | Unset = UNSET
    parameters: list["ErrorParameter"] | Unset = UNSET
    additional_properties: dict[str, Any] = _attrs_field(init=False, factory=dict)

    def to_dict(self) -> dict[str, Any]: ...          # hand-rolled, camelCase keys
    @classmethod
    def from_dict(cls, src_dict: Mapping[str, Any]) -> "Error": ...
```

- No runtime validation, no coercion beyond what `from_dict` explicitly codes
  (`datetime.fromisoformat`, nested `X.from_dict`). Unknown JSON keys are **kept**
  in `additional_properties` and re-emitted by `to_dict()` — models are "open" by
  default even when the schema omits `additionalProperties` (trial run: eBay
  schemas set no `additionalProperties`, models still got the dict + `__getitem__`
  / `__setitem__` / `__contains__` mapping protocol).
- JSON wire names are camelCase, Python attrs are `snake_case`; `next` →
  `next_`, `filter` → `filter_` (keyword-collision suffix, configurable via
  `field_prefix`).
- Optional vs required: optional → `T | Unset` defaulting to the `UNSET` sentinel
  (distinct from `None`); required → bare `T` with no default.
- Cross-model references use `from __future__ import annotations` + `TYPE_CHECKING`
  imports + deferred local imports inside `from_dict` to break import cycles.
- Enums → `str`/`int` `Enum` subclasses by default, or `Literal` unions with
  `literal_enums: true`.

**`allOf`** (synthetic `Thing = allOf[Base, {inline props}]`): **flattened**. `Thing`
is a single class carrying Base's `id` + `created_at` *and* the inline `name` /
`kind` / `tags`. It does **not** subclass `Base`; `Base` is still emitted as its own
model. Property merge, not inheritance.

**`oneOf`** (`kind: oneOf[Cat, Dog]`): a **union** `Cat | Dog | Unset`. `from_dict`
generates a `_parse_kind(data)` that tries each variant in declaration order inside
`try/except (TypeError, ValueError, AttributeError, KeyError)` and returns the first
that constructs. No discriminator support unless the spec supplies
`discriminator` (then it keys on the property). Ambiguous overlapping variants
resolve to whichever is listed first.

**`anyOf`** (`anyOf[string, integer]`): union of scalars `int | str | Unset`, with a
`cast()` and no real disambiguation for primitives.

**`nullable: true`** (OAS 3.0 style; `name` was also `required`): rendered as
`None | str` with a `_parse_name` helper that passes `None` through. A nullable
*and* optional field becomes `None | T | Unset`. Note open bug
[#679 "from_dict raises when nullable ref is set to None"](https://github.com/openapi-generators/openapi-python-client/issues/679)
for the nullable-`$ref` case — eBay's specs don't use `nullable` at all (trial run:
zero occurrences in Browse), so low risk, but worth a regression check on other
families.

**`additionalProperties: {type: string}`** on an inline object (`tags`): opc emits a
**dedicated model** `ThingTags` whose only member is
`additional_properties: dict[str, str]`. A free-form `additionalProperties: true`
maps to `dict[str, Any]` inline.

---

## 4. Custom httpx client injection & auth

**Yes — a caller-supplied client is a first-class, supported path.** Generated
`Client` / `AuthenticatedClient` (both `@attrs.define`) expose (trial run,
`client.py`):

```python
def set_httpx_client(self, client: httpx.Client) -> Self: ...          # "override ANY other settings"
def get_httpx_client(self) -> httpx.Client: ...                        # lazily builds one if unset
def set_async_httpx_client(self, async_client: httpx.AsyncClient) -> Self: ...
def get_async_httpx_client(self) -> httpx.AsyncClient: ...
```

Every endpoint function calls `client.get_httpx_client().request(**kwargs)` (or the
async equivalent). So the hand-written core can build **one** `httpx.Client` (and
one `httpx.AsyncClient`) with our auth + retry + base-URL wiring and push it into
whichever generated package(s) a call needs via `set_httpx_client()`. Because the
generated `Client` is an attrs class we can also subclass or wrap it.

Other injection seams:
- `httpx_args: dict` constructor kwarg — forwarded verbatim to the `httpx.Client` /
  `AsyncClient` constructor (transport, limits, mounts, event hooks, `auth=`).
- `base_url`, `headers`, `cookies`, `timeout`, `verify_ssl`, `follow_redirects`
  constructor kwargs, plus immutable-update helpers `with_headers()`,
  `with_cookies()`, `with_timeout()` returning an evolved copy.

**How auth is meant to be attached.** Out of the box `AuthenticatedClient` just does
static-header auth: on first `get_httpx_client()` it sets
`headers["Authorization"] = f"Bearer {self.token}"` once and never refreshes it
(trial run). That is **not enough** for eBay (client-credentials token expiry +
user-token refresh + sandbox/prod switch). The intended pattern for anything
dynamic is to **own the `httpx` client** and attach auth there:

- **`httpx.Auth` flow (recommended):** implement a custom `httpx.Auth` subclass
  whose `auth_flow()` injects `Authorization` and, on a `401`, refreshes and retries
  — this is also the natural home for OAuth2. Pass it as
  `httpx.Client(auth=..., ...)` then `set_httpx_client()`.
- **Event hooks** (`event_hooks={"request": [...], "response": [...]}`): fine for
  logging / 429-`Retry-After` observation, but `request` hooks can't do a
  token-refresh round-trip cleanly, so keep token logic in the `Auth` flow.
- **Per-call:** every endpoint also accepts arbitrary header kwargs generated from
  the spec (`x_ebay_c_marketplace_id`, `accept_language`, …), but `Authorization`
  is not one of them — it must come from the client/transport layer.

Net: the generated code does not fight us; our auth layer sits entirely in the
`httpx.Client` we construct and inject.

---

## 5. Async

**Both, always, unconditionally.** Every operation module emits `sync`,
`sync_detailed`, `asyncio`, `asyncio_detailed` (trial run). No flag to suppress
either. `asyncio*` uses `client.get_async_httpx_client()`. Async support has existed
since v0.6.0.

---

## 6. Config, hooks, templates, meta

**Config file:** `--config <path>` takes YAML or JSON. Relevant keys (from README):

| key | use for eBay |
|---|---|
| `class_overrides` | rename colliding / ugly schema classes per spec |
| `package_name_override` / `project_name_override` | force `browse` instead of `browse_api_client` |
| `package_version_override` | pin generated version instead of spec's `info.version` |
| `field_prefix` (default `field_`) | control the prefix for invalid identifiers |
| `use_path_prefixes_for_title_model_names` (default `true`) | keep to avoid within-spec name clashes |
| `literal_enums` (`true`) | emit `Literal` instead of `Enum` when enum values collide |
| `generate_all_tags` (default `false`) | eBay ops usually have one tag; leave off — otherwise duplicate functions per tag |
| `docstrings_on_attributes` | per-attr docstrings instead of one class docstring |
| `content_type_overrides` | map odd content types (e.g. treat some `*+json` as `application/json`) |
| `post_hooks` | commands run after generation; **default** `["ruff check . --fix-only", "ruff format ."]` |
| `http_timeout` | spec-retrieval timeout only (default 5s) |

There is **no** `pyproject.toml`-embedded config for opc itself — config is a
separate file passed per invocation. Good for us: one `config/<api>.yml` per spec,
all under version control, driven by a wrapper script / `Makefile` target.

**Post-generation hooks:** `post_hooks` (above). This is the seam for spec-quirk
patching — e.g. append a `ruff` isolation step, a `pyright` check, or a project
script that applies a `.patch` to work around an upstream spec bug. Hooks run with
CWD = generated project root.

**Custom templates:** `--custom-template-path <dir>`. Copy any of opc's ~40 Jinja
templates (`model.py.jinja`, `endpoint_module.py.jinja`, `client.py.jinja`,
`endpoint_macros.py.jinja`, `property_templates/*.py.jinja`, `README.md.jinja`, …)
into that dir — filenames must match exactly — and opc uses yours. README warns:
"*this is a beta-level feature … the API exposed in the templates is undocumented
and unstable*." Practical: a small `endpoint_module.py.jinja` / `models_init.py.jinja`
override to trim the copied eBay HTML docstrings or add a common import is low-risk;
rewriting `client.py.jinja` to bake in our auth is possible but couples us to opc
internals across upgrades — prefer injection (§4).

**Meta:** `--meta <none|poetry|setup|pdm|uv>`; use `none`.

---

## 7. Error responses / non-2xx (trial run — important)

`_parse_response` switches on `response.status_code`:

- For each status **that has a response body schema in the spec**, opc parses it
  into the declared model and includes it in the return **union**. Synthetic spec
  (`400 -> ErrorEnvelope`): `sync()` returns `ErrorEnvelope | Thing | None` and the
  `400` branch does `ErrorEnvelope.from_dict(response.json())`.
- For a status documented **without** a content schema, opc emits
  `response_400 = cast(Any, None); return response_400` — **the body is silently
  discarded**.
- For an undocumented status: if `client.raise_on_unexpected_status` is `True`
  (constructor kwarg, **default `False`**) it raises
  `errors.UnexpectedStatus(status_code, content)`; otherwise `_parse_response`
  returns `None`.
- `errors.py` contains **only** `UnexpectedStatus`. There is no exception hierarchy,
  no "raise on 4xx/5xx". A `404` on a spec'd-but-bodyless error is indistinguishable
  from a `200` with an empty body at the `sync()` level — you must call
  `sync_detailed()` and inspect `Response.status_code` / `.content`.

**The eBay-specific problem:** eBay's Browse contract declares `400`, `409`, `500`
with **`description` only and no `content`** (trial run — verified in the raw spec;
error detail lives in a non-standard `x-response-codes` vendor extension opc
ignores). So the generated Browse client throws away every error body. eBay actually
returns its standard envelope (`{"errors":[{"errorId","domain","category","message",
"longMessage","parameters":[...]}]}`) — the `Error` / `ErrorParameter` models even
get generated (they're referenced by `warnings` fields) — but no endpoint maps a
non-2xx status onto them.

**Consequence for the error/exception ticket:** the hand-written core **must**
own error handling. Options: (a) a response event hook / wrapper on the injected
`httpx.Client` that inspects `response.status_code`, and on non-2xx parses the JSON
body into a hand-written `EbayError` model and raises a typed exception before opc's
`_parse_response` ever runs; or (b) a `post_hooks` / custom-template pass that
rewrites bodyless error branches. (a) is far simpler and keeps generated code
pristine. Either way, **do not rely on opc for typed eBay errors.** This sharpens
the map's open "Error / exception model" item: the answer is "opc gives us nothing
here; build it in the core."

---

## 8. Regeneration ergonomics (trial run)

- **Determinism:** generating the Browse client twice produced **byte-identical**
  output (`diff -rq` clean). Model/endpoint file order is stable; `post_hooks` run
  `ruff format` so whitespace is pinned.
- **Speed:** Browse (7 paths / 75 schemas) generated in **~0.5 s** wall (excluding
  the one-time `uvx` tool download). ~19 specs of this size ≈ a few seconds total;
  eBay's larger specs (Fulfillment, Inventory) will be bigger but this is still a
  sub-minute full regen.
- **Diff noise:** low. Because output is deterministic and `ruff`-formatted, a spec
  bump produces a diff that tracks the spec change. The docstrings copy eBay's raw
  HTML verbatim, so eBay reworying a description churns large docstring hunks —
  a custom `endpoint_module.py.jinja` that truncates descriptions would cut that if
  it becomes annoying.
- `--overwrite` is required to regen in place; since v0.23.0 it only deletes
  directories opc itself created (`api/`, `models/`), so hand-written files placed
  alongside are safe — but cleanest is to keep `_generated/` free of hand-written
  code.
- **Recommended workflow:** a committed `specs/` dir (vendored eBay JSON, updated
  deliberately), a `config/<api>.yml` per spec, a regenerate script iterating specs
  → `--meta none --overwrite`, and `--fail-on-warning` in CI so spec drift that
  breaks generation (e.g. #1163) fails loudly.

---

## 9. Version, maintenance, license, Python support

| | |
|---|---|
| Latest release | **0.29.1**, 2026-08-30 ([PyPI](https://pypi.org/project/openapi-python-client/), [GH release](https://github.com/openapi-generators/openapi-python-client/releases/tag/v0.29.1)) |
| Cadence | ~every 1–3 months; 0.29.0 2026-05-30, 0.28.0 2025-12-03, 0.26.0 2025-08-26, 0.21.0 2024-06-08 |
| Maintenance | active — 116 open issues, ~1 985 stars, last push 2026-08-30, not archived ([GH API](https://api.github.com/repos/openapi-generators/openapi-python-client)) |
| License | **MIT** |
| Python (tool) | `>=3.11,<4.0`; runs fine under 3.12 (trial run) |
| Python (generated code) | `>=3.11` in emitted `pyproject.toml`; uses `X | Y` unions, `Self`, `datetime.fromisoformat` — clean on 3.12+. 0.27.0 dropped 3.9, 0.29.0 dropped 3.10 in generated output |
| Runtime deps of generated code | `httpx>=0.23.1,<0.29.0`, `attrs>=22.2.0` — **no Pydantic, no other deps** |
| Classifiers | Python 3.11 / 3.12 / 3.13 / 3.14 |
| Note | 0.29.1 was a **security fix** ("arbitrary code generation") + a template breaking change — pin the exact version used and bump deliberately |

---

## 10. Alternatives (brief, sourced)

The only materially different option is **`openapi-generator`** (the Java
OpenAPITools project) with its **`python`** generator, which since ~7.x produces
**Pydantic v2** models + `urllib3` (or `python-pydantic-v1`; an httpx-based
`python-nextgen` was folded into `python`). If "Pydantic v2 models" is a hard
requirement, this is the mainstream choice: huge user base, Pydantic-native
validation, discriminator support. Costs: a JVM/Docker build dependency, no async in
the default `python` generator (community reports), heavier generated runtime
(`urllib3`, `pydantic`, `python-dateutil`, `aenum`), historically noisier diffs, and
a templating story that is powerful but sprawling
([Speakeasy OSS comparison](https://www.speakeasy.com/docs/sdks/languages/python/oss-comparison-python/),
[libhunt comparison](https://www.libhunt.com/compare-openapi-generator-vs-openapi-python-client),
[OpenAPITools/openapi-generator#16888](https://github.com/OpenAPITools/openapi-generator/issues/16888)).
Smaller Pydantic-first generators exist —
[`MarcoMuellner/openapi-python-generator`](https://github.com/MarcoMuellner/openapi-python-generator)
(pydantic + httpx/requests/aiohttp, sync+async) and
[`sennder/python-client-generator`](https://github.com/sennder/python-client-generator)
(httpx + pydantic, async/sync) — but both are far less maintained and less
battle-tested than opc, and neither solves multi-spec composition. There is also
[`openapi-python-client-pydantic`](https://github.com/izeberg/openapi-python-client-pydantic),
a thin wrapper that post-processes opc output, but it lags opc versions.

**Recommendation:** stay with `openapi-python-client` and accept `attrs` models. The
generated `attrs` classes are plain, dependency-light, fully typed, and — because
our core already has to own auth, retry, pagination and error parsing — the lack of
Pydantic validation at the transport boundary is a small loss. If the team decides
Pydantic v2 models are non-negotiable for the public surface, that is a
generator switch to `openapi-generator`, and it should be decided **now**, before
the codegen-pipeline ticket, not retrofitted.

---

## Impact on other tickets (reshapes)

- **Map #4 wording:** "generated from eBay's OpenAPI 3 contracts with
  `openapi-python-client` (httpx + **Pydantic v2**…)" is factually wrong — opc emits
  **httpx + attrs**. The map/spec must either change the wording or change the
  generator. (This ticket does not touch the map body; flag for the map owner.)
- **Codegen-pipeline ticket ([#9](https://github.com/skrax/ebay-proj/issues/9)):**
  - one `--meta none` package per spec under `src/ebay_client/_generated/<family>/<api>/`;
  - per-spec `config/<api>.yml` with `package_name_override` and a `class_overrides`
    escape hatch; committed vendored `specs/`; a regen script; `--fail-on-warning`
    in CI to catch #1163-class breakage;
  - pin `openapi-python-client==0.29.1` exactly;
  - decide the digital-signature route exclusion by trimming `specs/` inputs or a
    `post_hooks` filter — opc has no per-operation exclude.
- **Auth-core ticket ([#10](https://github.com/skrax/ebay-proj/issues/10)):**
  - auth lives entirely in a hand-built `httpx.Client` / `AsyncClient` injected via
    `set_httpx_client()` / `set_async_httpx_client()` into each generated package;
  - implement token acquisition/refresh + sandbox/prod base-URL switch as a custom
    `httpx.Auth` flow (handles `401` retry); generated `AuthenticatedClient`
    static-token behavior is unused;
  - the core must construct one client and fan it out to ~19 generated packages —
    the auth object and the httpx client should be singletons the facade shares.
- **Error/exception-model (map "Not yet specified"):** opc discards eBay error
  bodies (eBay specs declare error statuses with no schema). The typed exception
  hierarchy over eBay's `errorId`/`domain`/`category`/`parameters` envelope is
  **100% hand-written core**, best implemented as a response hook on the injected
  httpx client that raises before opc's `_parse_response`. Generated `sync()` return
  unions will still be `SomeModel | None` — callers should mostly use the facade,
  not raw `sync()`.
- **Transport/pagination (map "Not yet specified"):** the injected httpx client is
  also where retry / 429-`Retry-After` belongs. `SearchPagedCollection` exposes
  `href`, `next_`, `offset`, `limit`, `total`, `warnings` (trial run) — the
  pagination helper can walk `next_` generically across Browse-style responses.

---

## Sources consulted

- Trial run: `openapi-python-client` 0.29.1 via `uvx`, Python 3.12, against
  eBay Browse API v1.20.4 OAS3 (`buy_browse_v1_oas3.json`, Wayback snapshot
  `20260210161815` of
  `https://developer.ebay.com/api-docs/master/buy/browse/openapi/3/buy_browse_v1_oas3.json`)
  and a hand-written synthetic OAS3 spec. Artifacts under `.scratch/opc-trial/`
  (uncommitted).
- README: <https://github.com/openapi-generators/openapi-python-client/blob/main/README.md>
- CHANGELOG: <https://github.com/openapi-generators/openapi-python-client/blob/main/CHANGELOG.md>
- Templates dir: <https://github.com/openapi-generators/openapi-python-client/tree/main/openapi_python_client/templates>
- `generate --help` (v0.29.1, local)
- PyPI JSON API: <https://pypi.org/pypi/openapi-python-client/json>
- GitHub repo API: <https://api.github.com/repos/openapi-generators/openapi-python-client>
- Issue #1163 (module-name collisions): <https://github.com/openapi-generators/openapi-python-client/issues/1163>
- Issue #1117 (snake_case property collisions): <https://github.com/openapi-generators/openapi-python-client/issues/1117>
- Issue #679 (nullable `$ref` → `None`): <https://github.com/openapi-generators/openapi-python-client/issues/679>
- Discussion #837 (Pydantic models): <https://github.com/openapi-generators/openapi-python-client/discussions/837>
- Speakeasy OSS comparison: <https://www.speakeasy.com/docs/sdks/languages/python/oss-comparison-python/>
- libhunt comparison: <https://www.libhunt.com/compare-openapi-generator-vs-openapi-python-client>
- OpenAPITools/openapi-generator#16888 (python-pydantic generators): <https://github.com/OpenAPITools/openapi-generator/issues/16888>
- `MarcoMuellner/openapi-python-generator`: <https://github.com/MarcoMuellner/openapi-python-generator>
- `sennder/python-client-generator`: <https://github.com/sennder/python-client-generator>
- Prior research: [`docs/research/ebay-python-api-clients.md`](./ebay-python-api-clients.md)
