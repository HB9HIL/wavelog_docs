# API v2

Wavelog's **REST API v2** is a modern, resource-oriented HTTP API for third-party
tools — radios, external loggers, monitoring systems and your own scripts. It runs
side by side with the [legacy v1 API](../api.md): v2 does not replace v1, and both
can be used at the same time.

!!! note
    API v2 requires a dedicated **v2 token** (prefixed `wl2_`). Legacy v1 API keys
    are **not** accepted on any `/api/v2/…` endpoint.

## What's different from v1

| Aspect | v1 (`/api/…`) | v2 (`/api/v2/…`) |
| --- | --- | --- |
| URL style | RPC — the operation is the method name (`/api/qso`) | REST — a resource path plus an HTTP verb (`POST /api/v2/qso`) |
| HTTP verbs | mostly `POST` | `GET`, `POST`, `PATCH`, `DELETE` |
| Authentication | key inside the JSON body / URL | `Authorization: Bearer <token>` header |
| Token format | `wl…` | `wl2_…` |
| Token storage | plaintext in the database | only a SHA-256 **hash** is stored |
| Permissions | a single `r` / `rw` right | granular per-resource scopes (`qso:read`, `qso:write`, …) |
| Response | varies per endpoint | consistent envelope, HTTP status carries the meaning |

## Base URL

Every endpoint lives under the `api/v2` prefix:

```text
https://<your-wavelog-host>/index.php/api/v2/<resource>[/<id>]
```

- `<resource>` is a singular, lowercase name (`qso`, `station`, `radio`, `statistic`,
  `lookup`, `club`, `token`).
- `<id>` is the numeric primary key of a single item, where applicable.

That is the **complete** URL space: a path with more than those two segments is
not a URL that exists and returns `404 not_found`, rather than being answered by
ignoring the surplus. A trailing slash is fine.

```text
/api/v2/qso          ✅ list
/api/v2/qso/42       ✅ single item
/api/v2/qso/         ✅ same as /api/v2/qso
/api/v2/qso/42/notes ❌ 404 not_found
```

Depending on your webserver configuration you may be able to omit the `index.php/`
segment. All examples in this documentation include it for maximum compatibility.

## Available resources

| Resource | Path | Scope | Description |
| --- | --- | --- | --- |
| [QSO](qso.md) | `/api/v2/qso` | `qso:*` | Read, create, update and delete log contacts; ADIF import/export |
| [Station](station.md) | `/api/v2/station` | `station:*` | Manage station locations (logbook profiles) |
| [Radio](radio.md) | `/api/v2/radio` | `radio:*` | Push and read live CAT radio state |
| [Statistic](statistic.md) | `/api/v2/statistic` | `statistic:read` | Read-only counters for dashboards & monitoring |
| [Lookup](lookup.md) | `/api/v2/lookup` | `lookup:read` | Look up a callsign (DXCC + worked/confirmed) or a gridsquare |
| [Club](club.md) | `/api/v2/club` | `club:read` | List a clubstation's members (officers only) |
| [Token](token.md) | `/api/v2/token` | — | Metadata about the current token (whoami) |

## Authentication

### Creating a token

1. Log in to Wavelog and open the **API** section from the user menu.
2. Create a new **API v2 token**. You choose:
    - a **name** (to recognise the token later),
    - one or more **scopes** (see [Scopes](#scopes)),
    - an **expiry**: 30, 90 or 365 days, or *never*.
3. The full token (`wl2_…`) is shown **exactly once**, right after creation. Copy
   it immediately — Wavelog only stores a hash and can never show it again.

You can revoke a token at any time from the same screen; revocation is immediate.

!!! warning
    Treat a token like a password. Anyone holding it can act on your data within
    the granted scopes. If a token leaks, revoke it and create a new one.

### Sending the token

Send the token in the `Authorization` header using the `Bearer` scheme:

```http
Authorization: Bearer wl2_your_token_here
```

As a fallback, an `X-API-Key` header is also accepted:

```http
X-API-Key: wl2_your_token_here
```

!!! note
    The API is **sessionless**. It never uses your browser login — all access and
    ownership is derived from the token's owner. A token can only ever see and
    modify data belonging to the user it was created for.

    A token created while switched into a clubstation is owned by the *club*,
    and what it may do additionally depends on the creator's permission level
    there. See [Clubstations](clubstation.md).

### Scopes

Permissions are **granular** and follow the pattern `<resource>:<action>`:

| Action | Granted for | Required by |
| --- | --- | --- |
| `read` | reading data | `GET` requests |
| `write` | creating and updating data | `POST`, `PATCH` requests |
| `delete` | deleting data | `DELETE` requests |

Scopes are independent, so you can mint a read-only token, a write-only token, or
any combination. A request whose verb needs a scope the token does not carry is
rejected with `403 insufficient_scope`.

The full set of scopes offered when creating a token is:

| Scope | Allows |
| --- | --- |
| `qso:read` / `qso:write` / `qso:delete` | Read / create+update / delete QSOs |
| `station:read` / `station:write` / `station:delete` | Read / create+update / delete station locations |
| `radio:read` / `radio:write` / `radio:delete` | Read / create+update / delete radios |
| `statistic:read` | Read statistics |
| `lookup:read` | Look up callsigns and grids |
| `club:read` | Read club members |

The [Token](token.md) resource (whoami) needs no scope — any valid token may read
its own metadata.

Scopes are the *only* permission layer for a personal token. A
[clubstation](clubstation.md) token passes a second one on top: the member's
permission level on the club. Holding `station:write` does not help if the level
behind the token is below officer.

## Request and response conventions

### HTTP verbs

| Verb | Path | Meaning |
| --- | --- | --- |
| `GET` | `/api/v2/<resource>` | List items |
| `GET` | `/api/v2/<resource>/<id>` | Fetch a single item |
| `POST` | `/api/v2/<resource>` | Create an item |
| `PATCH` | `/api/v2/<resource>/<id>` | **Partial** update — only the fields you send are changed |
| `DELETE` | `/api/v2/<resource>/<id>` | Delete an item |

!!! note "The API has no `PUT`"
    Updates are always partial. Wavelog is the source of truth for your data, and
    a full replace would let a client blank fields it never knew existed — every
    field added in a future release would silently be wiped by older clients. To
    overwrite an item completely, send every field explicitly in a `PATCH`.

Not every resource implements every verb (for example, [Radio](radio.md) has no
`PATCH`, and [Statistic](statistic.md) is read-only). An unsupported verb
returns `405 method_not_allowed` together with an `Allow` header listing exactly
the verbs that resource does implement:

```http
HTTP/1.1 405 Method Not Allowed
Allow: GET, POST, PATCH, DELETE
```

The verb is checked **before** the scope, so a `POST` to a read-only resource is
a `405` and not a `403`: the scope it would need (`lookup:write`) does not exist
and no token can ever hold it, so reporting a missing scope would point at the
wrong problem.

### Request bodies

- Send write payloads as JSON with `Content-Type: application/json`.
- Body fields must be **scalar** values (string, number, boolean or `null`).
  Nested objects or arrays are rejected with `400 validation_error`.
- Malformed JSON returns `400 invalid_json`.

### Success responses

Successful responses use a slim envelope. The payload is always under `data`.
Every JSON success response also includes a `meta` block with common request
context:

| Field | Meaning |
| --- | --- |
| `timestamp` | UTC response timestamp (ISO-8601, e.g. `2026-07-18T12:34:56+00:00`) |
| `resource` | API v2 resource segment (`qso`, `station`, `statistic`, ...; `meta` for `/api/v2` and `/api/v2/status`) |
| `method` | HTTP method (`GET`, `POST`, `PATCH`, `DELETE`) |

Resource-specific metadata (pagination, lookup detail, statistic profile, ...)
is added to the same `meta` object.

Examples:

```json
{
  "data": { "id": 42, "call": "N9EAT" },
  "meta": {
    "timestamp": "2026-07-18T12:34:56+00:00",
    "resource": "qso",
    "method": "GET"
  }
}
```

```json
{
  "data": [ { "id": 42 }, { "id": 43 } ],
  "meta": {
    "timestamp": "2026-07-18T12:34:56+00:00",
    "resource": "qso",
    "method": "GET",
    "page": 1,
    "per_page": 50,
    "count": 2
  }
}
```

A successful `DELETE` returns **`204 No Content`** with an empty body.

### Error responses

Errors use an `error` envelope and a matching HTTP status code:

```json
{
  "error": {
    "code": "insufficient_scope",
    "message": "Token is missing the required scope: qso:write",
    "details": { "required_scope": "qso:write" }
  }
}
```

| HTTP | `code` | When |
| --- | --- | --- |
| 400 | `validation_error` | Missing/invalid field, or a non-scalar value |
| 400 | `invalid_json` | The request body is not valid JSON |
| 401 | `unauthorized` | No token supplied |
| 401 | `invalid_token` | Unknown, revoked, or non-`wl2_` token (e.g. a legacy v1 key) |
| 401 | `token_expired` | The token's expiry date has passed |
| 403 | `insufficient_scope` | The token lacks the scope the verb requires |
| 403 | `forbidden` | The referenced item does not belong to the token owner |
| 403 | `insufficient_club_permission` | [Clubstation](clubstation.md) token whose permission level is too low (see `details.required_level`) |
| 403 | `club_access_revoked` | The [clubstation](clubstation.md) membership behind the token has been withdrawn |
| 404 | `not_found` | Unknown resource, path or item id — see the note below |
| 405 | `method_not_allowed` | Verb not supported by this resource (see `Allow` header) |
| 409 | `conflict` | Duplicate, or a state that forbids the action |
| 429 | `rate_limited` | [Rate limit](#rate-limiting) exceeded (see `details.retry_after`, in seconds) |
| 500 | `internal_error` | Unexpected server-side error |

!!! note "`404` also covers items you may not see"
    An item that exists but is out of reach for your token is reported as
    `404 not_found`, not `403`. Confirming that a row exists would already leak
    something about another user's data. This is why a
    [clubstation](clubstation.md) member gets a `404` for a QSO logged by a
    different operator.

### Pagination

List endpoints that can return many rows (currently [QSO](qso.md)) support
pagination through query parameters:

| Parameter | Default | Notes |
| --- | --- | --- |
| `page` | `1` | 1-based page number |
| `per_page` | `50` | Maximum `5000` (see the [QSO](qso.md#list-qsos) resource for per-format defaults) |

The pagination fields are part of `meta` and report the page state:

| Field | Meaning |
| --- | --- |
| `page` | The page you requested |
| `per_page` | Items per page |
| `count` | Items on **this** page |
| `total` | Items across **all** pages for the current filter |
| `total_pages` | `ceil(total / per_page)` |
| `has_more` | `true` while `page < total_pages` |

```json
{
  "data": [ { "id": 42 } ],
  "meta": {
    "timestamp": "2026-07-18T12:34:56+00:00",
    "resource": "qso",
    "method": "GET",
    "page": 1,
    "per_page": 50,
    "count": 50,
    "total": 237,
    "total_pages": 5,
    "has_more": true
  }
}
```

To iterate the full set, request pages until `has_more` is `false` (equivalently,
until `page` reaches `total_pages`). You never need to probe for an empty page.

### Rate limiting

Rate limiting is **opt-in**: it is inactive until the instance owner configures it,
and it uses a sliding window. When a limit is exceeded, the API returns
`429 rate_limited` with `details.retry_after` (seconds to wait). If no limit is
configured, no throttling is applied — but please still call the API only when you
actually have new data to send or fetch.

#### Buckets

Every v2 request passes through up to two independent counters:

| Bucket | Checked | Counted per | Purpose |
| --- | --- | --- | --- |
| `api_v2_auth` | before authentication | client IP address | Slows down brute-force attempts against the token keyspace |
| `api_v2_<resource>` | after successful authentication | token | Normal usage limit, e.g. `api_v2_qso`, `api_v2_station` |

A client with a valid token passes through both. A client with an invalid token
never gets past the first — which is the point: a rejected token never reaches the
per-token bucket, so without `api_v2_auth` the token keyspace could be probed at
full speed.

`api_v2_auth` is deliberately **not** per resource. If it were, the limit could be
sidestepped by rotating the resource in the URL (`/qso`, `/station`, `/radio`, …).

#### Configuration

Limits live in `application/config/config.php` (Docker installations:
`application/config/docker/config.php`). Each entry sets a maximum number of
`requests` within a `window` in seconds:

```php
$config['api_rate_limits'] = [
    // API v2 — bucket names carry the "api_v2_" prefix
    'api_v2_auth'      => ['requests' => 10,  'window' => 120],
    'api_v2_qso'       => ['requests' => 120, 'window' => 60],
    'api_v2_lookup'    => ['requests' => 60,  'window' => 60],
    'api_v2_statistic' => ['requests' => 30,  'window' => 60],

    // v1 endpoints keep their bare method names
    'private_lookup'   => ['requests' => 60,  'window' => 60],

    // applies to every bucket not listed above
    'default'          => ['requests' => 60,  'window' => 60],
];
```

!!! warning "Without `default`, unlisted buckets are unlimited"
    Buckets are looked up by exact name, then fall back to `default`. If neither
    matches, **no limit is applied at all** to that bucket.

    This is easy to get wrong: an instance that configures only
    `'private_lookup'` has rate limiting switched on, yet every single v2
    endpoint — including `api_v2_auth` — remains unthrottled.

    Either set a `default`, or list `api_v2_auth` explicitly.

v1 and v2 share the same config array but never collide: v1 buckets are named
after the API method (`lookup`, `private_lookup`, `qso`), v2 buckets always carry
the `api_v2_` prefix. Both are configured independently.

To disable rate limiting entirely, set `$config['api_rate_limits'] = null;` or
leave it commented out.

### CORS

The API answers `OPTIONS` preflight requests with `204 No Content` and permits
cross-origin calls (`Access-Control-Allow-Origin: *`) for the `GET`, `POST`,
`PUT`, `PATCH`, `DELETE` and `OPTIONS` methods, allowing the `Authorization`,
`Content-Type` and `X-API-Key` headers.

### Status endpoint

`GET /api/v2` (and `GET /api/v2/status`) is **public** — no token required — and
is handy for a quick reachability check:

```bash
curl https://<WAVELOG_URL>/index.php/api/v2/status
```

```json
{
  "data": { "name": "Wavelog API", "status": "ok" },
  "meta": {
    "timestamp": "2026-07-18T12:34:56+00:00",
    "resource": "status",
    "method": "GET"
  }
}
```

## Quickstart

List your station locations:

```bash
curl https://<WAVELOG_URL>/index.php/api/v2/station \
     -H "Authorization: Bearer wl2_your_token_here"
```

Create a QSO on station location `1`:

```bash
curl -X POST https://<WAVELOG_URL>/index.php/api/v2/qso \
     -H "Authorization: Bearer wl2_your_token_here" \
     -H "Content-Type: application/json" \
     -d '{
           "station_profile_id": 1,
           "call": "N9EAT",
           "band": "20m",
           "mode": "SSB",
           "freq": 14075000,
           "qso_date": "2026-06-16",
           "time_on": "1706",
           "rst_sent": "59",
           "rst_rcvd": "57"
         }'
```

See the per-resource pages for the full field reference:
[QSO](qso.md) · [Station](station.md) · [Radio](radio.md) ·
[Statistic](statistic.md) · [Lookup](lookup.md) ·
[Club](club.md) · [Token](token.md).

If your token belongs to a clubstation rather than to your own account, read
[Clubstations](clubstation.md) as well — the permission level changes what
several of these resources return.
