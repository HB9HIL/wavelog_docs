# Radio

The Radio resource exposes your **CAT radios** — the live rig state (frequency,
mode, power, …) that CAT clients such as WSJT-X, rigctl or GridTracker push to
Wavelog. It is the v2 equivalent of the legacy [`api/radio`](../api.md#apiradio)
endpoint.

- **Base path:** `/api/v2/radio`
- **Scopes:** `radio:read`, `radio:write`, `radio:delete`

All operations are scoped to the token owner. A radio belonging to another user is
treated as *not found*.

!!! note
    Read the [API v2 overview](index.md) first for authentication, the response
    envelope and error codes.

## The upsert model

Unlike other resources, a radio is identified by its **name**, not created by id.
CAT clients continuously push a fresh snapshot of the current rig state, so there
is **no `PATCH` or `PUT`** — every new snapshot is simply a `POST`:

- if no radio with that name exists yet for you, it is created (`201 Created`);
- otherwise the existing radio is updated in place (`200 OK`).

## Endpoints

| Verb | Path | Scope | Purpose |
| --- | --- | --- | --- |
| `GET` | `/api/v2/radio` | `radio:read` | List all your radios |
| `GET` | `/api/v2/radio/{id}` | `radio:read` | Fetch a single radio |
| `POST` | `/api/v2/radio` | `radio:write` | Create or update a radio (upsert by name) |
| `DELETE` | `/api/v2/radio/{id}` | `radio:delete` | Remove a radio |

List endpoints are not paginated.

## The radio object

Frequencies are in **Hz**.

```json
{
  "id": 3,
  "radio": "FT-950",
  "frequency": 14075000,
  "frequency_rx": null,
  "mode": "SSB",
  "mode_rx": null,
  "power": 100,
  "prop_mode": null,
  "sat_name": null,
  "updated_at": "2026-06-16 17:06:00"
}
```

An empty or `non` mode is normalised to `null`.

## Push a radio state

`POST /api/v2/radio`

**Required field:** `radio` (the device name).

| Field | Notes |
| --- | --- |
| `radio` | Arbitrary name identifying the device (required) |
| `frequency` | TX frequency in Hz |
| `frequency_rx` | RX frequency in Hz (split / satellite) |
| `mode` | TX mode |
| `mode_rx` | RX mode |
| `power` | TX power in watts |
| `prop_mode` | Propagation mode, e.g. `SAT` |
| `sat_name` | Satellite name, e.g. `QO-100` |

```bash
curl -X POST https://<WAVELOG_URL>/index.php/api/v2/radio \
     -H "Authorization: Bearer wl2_your_token_here" \
     -H "Content-Type: application/json" \
     -d '{
           "radio": "FT-950",
           "frequency": 14075000,
           "mode": "SSB",
           "power": 100
         }'
```

The response contains the stored radio object in `data`. A newly created radio
returns `201 Created` with a `Location` header; an update returns `200 OK`.

Satellite / split example:

```bash
curl -X POST https://<WAVELOG_URL>/index.php/api/v2/radio \
     -H "Authorization: Bearer wl2_your_token_here" \
     -H "Content-Type: application/json" \
     -d '{
           "radio": "QO-100 Station",
           "frequency": 2400170000,
           "frequency_rx": 10489670000,
           "mode": "SSB",
           "mode_rx": "SSB",
           "prop_mode": "SAT",
           "sat_name": "QO-100",
           "power": 5
         }'
```

!!! note
    The callback URL (`cat_url`) is **not** writable through the API. If you need a
    custom CAT callback URL, use the legacy [`api/radio`](../api.md#apiradio)
    endpoint.

!!! tip
    Push a radio state when the frequency or mode actually changes — not blindly
    every second.

### Clubstation operators

When the token belongs to a club member (the token owner differs from its creator),
the radio state is recorded under the club account with the acting member as
operator, mirroring the legacy radio endpoint.

Each member therefore has their own set of rigs inside the shared club account.
Below officer level, listing, reading and deleting are scoped to those: another
member's radio returns `404 not_found`. See [Clubstations](clubstation.md).

## Delete a radio

`DELETE /api/v2/radio/{id}`

```bash
curl -X DELETE https://<WAVELOG_URL>/index.php/api/v2/radio/3 \
     -H "Authorization: Bearer wl2_your_token_here"
```

On success the API returns `204 No Content`.
