# Station

The Station resource manages your **station locations** (logbook profiles) — the
callsign, grid, DXCC and other location details a QSO is logged against. It reuses
the same data layer as the web UI, applying the same normalisation and validation.

- **Base path:** `/api/v2/station`
- **Scopes:** `station:read`, `station:write`, `station:delete`

All operations are scoped to the token owner. A station location belonging to
another user is treated as *not found*.

!!! note
    Read the [API v2 overview](index.md) first for authentication, the response
    envelope and error codes.

## Endpoints

| Verb | Path | Scope | Purpose |
| --- | --- | --- | --- |
| `GET` | `/api/v2/station` | `station:read` | List all your station locations |
| `GET` | `/api/v2/station/{id}` | `station:read` | Fetch a single station location |
| `POST` | `/api/v2/station` | `station:write` | Create a station location |
| `PATCH` | `/api/v2/station/{id}` | `station:write` | Partial update |
| `DELETE` | `/api/v2/station/{id}` | `station:delete` | Delete a station location and all its QSOs |

!!! note "There is no `PUT`"
    Updates are always partial, for the same reason as on [QSO](qso.md): Wavelog
    is the source of truth, and a full replace would let a client blank fields it
    never knew existed. To overwrite a station location completely, send every
    field explicitly in a `PATCH`.

List endpoints are not paginated — users typically have only a handful of station
locations.

## The station object

```json
{
  "id": 1,
  "name": "JO30oo / DJ7NT",
  "callsign": "DJ7NT",
  "gridsquare": "JO30OO",
  "city": "Bonn",
  "dxcc": 230,
  "country": "FEDERAL REPUBLIC OF GERMANY",
  "cq": 14,
  "itu": 28,
  "state": "",
  "cnty": "",
  "iota": "",
  "sota": "",
  "wwff": "",
  "pota": "",
  "sig": "",
  "sig_info": "",
  "power": 100,
  "active": true
}
```

`country` and `active` are read-only. The remaining fields correspond to the
[writable fields](#writable-fields) below, so a `GET` result can be sent straight
back through `PATCH`.

## Create a station location

`POST /api/v2/station`

**Required fields:** `name`, `callsign`, `dxcc`, `cq`, `itu`.

```bash
curl -X POST https://<WAVELOG_URL>/index.php/api/v2/station \
     -H "Authorization: Bearer wl2_your_token_here" \
     -H "Content-Type: application/json" \
     -d '{
           "name": "DJ7NT at Home",
           "callsign": "DJ7NT",
           "gridsquare": "JO30oo",
           "city": "Unkel",
           "dxcc": 230,
           "cq": 14,
           "itu": 28,
           "power": 100
         }'
```

- Your **first** station location automatically becomes the active one.
- On success the API returns `201 Created`, a `Location` header, and the created
  object in `data`.
- An identical existing location returns `409 conflict`.
- An invalid grid locator returns `400 validation_error`.
- A missing required field returns `400 validation_error`, listing what is
  missing in `details.missing`.

### Writable fields

| Field | Notes |
| --- | --- |
| `name` | Profile name (required on create) |
| `callsign` | Station callsign (required on create) |
| `gridsquare` | Maidenhead locator, validated |
| `city` | QTH / city |
| `dxcc` | DXCC entity number (required on create) |
| `cq` | CQ zone (required on create) |
| `itu` | ITU zone (required on create) |
| `state` | Primary administrative subdivision |
| `cnty` | Secondary subdivision (county) — see note below |
| `iota` | IOTA reference |
| `sota` | SOTA reference |
| `wwff` | WWFF reference |
| `pota` | POTA reference |
| `sig` | Special interest group |
| `sig_info` | Special interest group info |
| `power` | Default TX power in watts |

!!! note
    External-service credentials (QRZ, HRDLog, ClubLog, OQRS, webADIF, eQSL) are
    **not** exposed through the API and keep their defaults on create.

!!! warning "Clubstation tokens need officer level to write"
    Station locations are shared club infrastructure, and deleting one removes
    all of its QSOs with it. Creating, updating and deleting them therefore
    requires permission level 9; reading stays open to every member. A lower
    level is refused with `403 insufficient_club_permission`. See
    [Clubstations](clubstation.md).

!!! note
    `cnty` (county) is only stored for DXCC entities that have ADIF secondary
    subdivisions. For any other DXCC entity the county is cleared automatically,
    mirroring the web UI.

## Update a station location

`PATCH /api/v2/station/{id}` — partial update, only the fields you send change.
Anything you omit keeps its stored value.

```bash
curl -X PATCH https://<WAVELOG_URL>/index.php/api/v2/station/1 \
     -H "Authorization: Bearer wl2_your_token_here" \
     -H "Content-Type: application/json" \
     -d '{ "power": 50, "city": "Bonn" }'
```

The updated station object is returned in `data`.

!!! note "Required fields cannot be blanked out"
    The fields required on create stay required for the lifetime of the
    location. Sending one of them as `null` or `""` is refused with
    `400 validation_error` and the offending names in `details.fields`.
    Leave a field out of the body to keep its stored value.

## Delete a station location

`DELETE /api/v2/station/{id}`

!!! warning
    This is **destructive**. Deleting a station location also deletes **all of its
    QSOs** and their associated QSL/eQSL data, exactly like the web UI.

- The **active** station location cannot be deleted and returns `409 conflict`.
  Activate another location first.
- On success the API returns `204 No Content`.

```bash
curl -X DELETE https://<WAVELOG_URL>/index.php/api/v2/station/2 \
     -H "Authorization: Bearer wl2_your_token_here"
```
