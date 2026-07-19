# Lookup

The Lookup resource looks up a single **callsign** or **gridsquare**, scoped to the
token owner's own logbook. The callsign lookup is the v2 equivalent of the v1
`lookup` / `private_lookup` endpoints (for rig-control overlays, DX cluster clients,
…); the grid lookup replaces v1 `logbook_check_grid`.

- **Base path:** `/api/v2/lookup`
- **Scope:** `lookup:read`

!!! note
    Read the [API v2 overview](index.md) first for authentication, the response
    envelope and error codes.

## Endpoints

| Verb | Path | Scope | Purpose |
| --- | --- | --- | --- |
| `GET` | `/api/v2/lookup?callsign=…` | `lookup:read` | Look up a callsign |
| `GET` | `/api/v2/lookup?grid=…` | `lookup:read` | Look up a gridsquare's worked/confirmed status |
| `GET` | `/api/v2/lookup?grid=all` | `lookup:read` | List all worked gridsquares |

The callsign is always passed as a query parameter, never as a path segment —
`/api/v2/lookup/DL1ABC` returns `404 not_found`, because Lookup has no
addressable single items. This keeps callsigns
containing a `/` (portable or DXCC-prefix calls like `DL1ABC/P` or `W/DL1ABC`)
working, which a path segment cannot: most webservers reject encoded slashes
(`%2F`).

`GET /api/v2/lookup` with neither `callsign` nor `grid` returns `400 validation_error`.

## Callsign lookup

### Query parameters

| Parameter | Default | Notes |
| --- | --- | --- |
| `callsign` | — | The callsign to look up (required for this form) |
| `detail` | `basic` | `full` or `basic` (see below) |
| `band` | — | Band for the per-band worked/confirmed flags (e.g. `20m`) |
| `mode` | — | Mode for the per-mode worked/confirmed flags (e.g. `FT8`) |
| `callbook` | — | `true` to include external callbook data (an extra HTTP lookup) |
| `station_ids` | all owned | Comma-separated station-location ids to scope the worked/confirmed status to; ids not owned are ignored |

## Detail levels

### `full`

Everything `basic` returns, plus the per-band/mode worked and confirmed flags, the
DXCC confirmation state and — when `callbook=true` — callbook data. This mirrors the
v1 `private_lookup` endpoint.

```bash
curl "https://<WAVELOG_URL>/index.php/api/v2/lookup?callsign=DL1ABC&detail=full&band=20m&mode=FT8" \
     -H "Authorization: Bearer wl2_your_token_here"
```

```json
{
  "data": {
    "callsign": "DL1ABC",
    "dxcc": "FEDERAL REPUBLIC OF GERMANY",
    "dxcc_id": "230",
    "dxcc_lat": "51",
    "dxcc_long": "10",
    "dxcc_cqz": "14",
    "dxcc_flag": "🇩🇪",
    "cont": "EU",
    "name": "",
    "gridsquare": "",
    "location": "",
    "iota_ref": "",
    "state": "",
    "us_county": "",
    "qsl_manager": "",
    "bearing": "",
    "call_worked": false,
    "call_worked_band": false,
    "call_worked_band_mode": false,
    "lotw_member": false,
    "dxcc_confirmed_on_band": false,
    "dxcc_confirmed_on_band_mode": false,
    "dxcc_confirmed": true,
    "call_confirmed": false,
    "call_confirmed_band": false,
    "call_confirmed_band_mode": false,
    "suffix_slash": "",
    "dxcc_ituz": 28
  },
  "meta": { "detail": "full" }
}
```

When the call has been worked before, the owner's stored `name`, `gridsquare`,
`location`, `iota_ref`, `state`, `us_county` and `qsl_manager` are filled in, a
`latlng` value is added for the grid, and the `call_worked*` / `call_confirmed*`
flags reflect the requested `band`/`mode`. `dxcc_ituz` is only present when the
entity has a known ITU zone; `callbook` is only present with `callbook=true`.

### `basic` (default)

DXCC derivation plus the owner's grid/name if the call was worked before — no
per-band/mode history. This mirrors the v1 `lookup` endpoint used by the
DXClusterAPI and is the cheaper call.

```bash
curl "https://<WAVELOG_URL>/index.php/api/v2/lookup?callsign=DL1ABC&detail=basic" \
     -H "Authorization: Bearer wl2_your_token_here"
```

```json
{
  "data": {
    "callsign": "DL1ABC",
    "dxcc": "FEDERAL REPUBLIC OF GERMANY",
    "dxcc_id": "230",
    "dxcc_lat": "51",
    "dxcc_long": "10",
    "dxcc_cqz": "14",
    "dxcc_flag": "🇩🇪",
    "cont": "EU",
    "name": "",
    "gridsquare": "",
    "location": "",
    "iota_ref": "",
    "state": "",
    "us_county": "",
    "qsl_manager": "",
    "bearing": "",
    "workedBefore": false,
    "lotw_member": false,
    "suffix_slash": ""
  },
  "meta": { "detail": "basic" }
}
```

## Grid lookup

`GET /api/v2/lookup?grid=<grid>` — is the gridsquare worked or confirmed in your
logbook? (Replaces the v1 `logbook_check_grid` endpoint.)

| Parameter | Notes |
| --- | --- |
| `grid` | The gridsquare to look up (required for this form) |
| `band` | Optional band filter, e.g. `20m` |
| `cnfm` | Optional confirmation source: `qsl`, `lotw` or `eqsl` — anything else returns `400 validation_error` |
| `logbook_id` | Optional: restrict to one owned logbook (`403 forbidden` for a foreign id) |

```bash
curl "https://<WAVELOG_URL>/index.php/api/v2/lookup?grid=JN47&cnfm=lotw" \
     -H "Authorization: Bearer wl2_your_token_here"
```

```json
{ "data": { "gridsquare": "JN47", "result": "Confirmed" }, "meta": { "type": "grid" } }
```

`result` is one of:

| Value | Meaning |
| --- | --- |
| `Not Found` | The grid has not been worked |
| `Found` | Worked (returned when no `cnfm` was requested) |
| `Worked` | Worked but not confirmed via the requested `cnfm` source |
| `Confirmed` | Confirmed via the requested `cnfm` source |

## All worked grids

`GET /api/v2/lookup?grid=all` — instead of checking a single gridsquare, return
every gridsquare worked in your logbook. (Replaces the v1
`logbook_get_worked_grids` endpoint.)

| Parameter | Notes |
| --- | --- |
| `grid` | The literal value `all` (required for this form) |
| `band` | Optional band filter, e.g. `20m`; `SAT` restricts to satellite QSOs |
| `cnfm` | Optional confirmation source: `qsl`, `lotw` or `eqsl` — anything else returns `400 validation_error` |
| `logbook_id` | Optional: restrict to one owned logbook (`403 forbidden` for a foreign id) |

```bash
curl "https://<WAVELOG_URL>/index.php/api/v2/lookup?grid=all&band=20m" \
     -H "Authorization: Bearer wl2_your_token_here"
```

```json
{
  "data": { "grids": ["JN47", "JO30"], "count": 2 },
  "meta": { "type": "worked_grids", "band": "20m", "cnfm": null }
}
```

Grids from `VUCC_GRIDS` fields are included. All values are uppercased and
truncated to the first four characters, so the list contains each worked field
exactly once.

## Clubstation tokens

A lookup answers "have I worked this before" out of the logbook, so it follows
the same boundary the [QSO](qso.md) resource does. For a
[clubstation](clubstation.md) token below officer level only the acting member's
own QSOs are considered:

- `?callsign=` reports `workedBefore: false` (and `call_worked: false` with
  `detail=full`) when the only match belongs to another operator, and returns no
  name, QTH or locator from it.
- `?grid=` and `?grid=all` ignore other operators' QSOs.

An officer sees the whole club logbook. DXCC data is derived from the callsign
itself and is never restricted.
