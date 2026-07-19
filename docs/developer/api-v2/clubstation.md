# Clubstations

A [clubstation](../../admin-guide/administration/clubstations.md) lets several operators
share one club or special callsign. API v2 supports that, but a token created
inside a clubstation session behaves differently from a personal one — it is
bound to the **permission level** the member holds on the club.

!!! note
    None of this applies to a personal token, and none of it applies at all when
    the instance has the clubstation feature (`special_callsign`) switched off.
    If you only ever use your own account, you can skip this page.

## Personal tokens vs. club tokens

Every token records two users:

| | Personal token | Club token |
| --- | --- | --- |
| Owner (whose data it reaches) | you | the **clubstation** |
| Creator (who acts through it) | you | the **member** who created it |

A club token is created by switching into the clubstation in the web UI and
then minting a token from the API page as usual. Everything the token reaches
belongs to the club; everything it *writes* is attributed to the member.

Use the [Token](token.md) resource to tell them apart — `owner` is the
clubstation's callsign for a club token:

```bash
curl https://<WAVELOG_URL>/index.php/api/v2/token \
     -H "Authorization: Bearer wl2_your_token_here"
```

## Permission levels

The level is set per member on the club's permission page and is re-read on
**every** API request — a change takes effect immediately, with no need to
recreate the token.

| Level | Name | Through the API |
| --- | --- | --- |
| 3 | Club Member | Log and manage your own QSOs; read station locations |
| 6 | Club Member ADIF | Same as level 3 (the extra ADIF rights are a UI distinction) |
| 9 | Club Officer | Unrestricted access to the whole club logbook |

Levels 3 and 6 are treated identically by the API. The difference between them
in the web UI is about which ADIF screens a member may open; over the API both
are simply "not an officer".

## What a member below officer level may do

### Their own QSOs only

For levels 3 and 6 every QSO operation is scoped to the member's own callsign
(`OPERATOR`):

- `GET /api/v2/qso` lists only their QSOs. `meta.total` counts only those, too.
- `GET /api/v2/qso?format=adif` exports exactly the same set — the format does
  not widen the result.
- `GET`, `PATCH` and `DELETE` on `/api/v2/qso/{id}` of another operator's QSO
  return **`404 not_found`**, not `403`. A QSO the token may not see is reported
  as if it did not exist, rather than confirming its existence.

An officer sees and edits every operator's QSO.

### The operator is assigned, not chosen

When a member creates a QSO, `operator` is always set to their own callsign.
Sending a different one has no effect — the value is overwritten, on all three
create paths (single JSON, bulk `qsos[]` and `import_type=adif`):

```bash
# Sent by a level-3 member of HB9CLUB whose own callsign is HB9ABC
curl -X POST https://<WAVELOG_URL>/index.php/api/v2/qso \
     -H "Authorization: Bearer wl2_club_token" \
     -H "Content-Type: application/json" \
     -d '{"station_profile_id": 4, "call": "N9EAT", "band": "20m",
          "mode": "SSB", "qso_date": "2026-06-16", "time_on": "1706",
          "operator": "HB9XYZ"}'
# -> the QSO is stored with OPERATOR = HB9ABC
```

An officer may set `operator` freely, which is how QSOs are logged on behalf of
another operator.

### Station locations are officer-only

Station locations are shared club infrastructure, and deleting one removes all
of its QSOs along with it. Creating, updating and deleting them therefore
requires level 9:

| Request | Level 3 / 6 | Level 9 |
| --- | --- | --- |
| `GET /api/v2/station` | ✅ | ✅ |
| `GET /api/v2/station/{id}` | ✅ | ✅ |
| `POST /api/v2/station` | `403` | ✅ |
| `PATCH /api/v2/station/{id}` | `403` | ✅ |
| `DELETE /api/v2/station/{id}` | `403` | ✅ |

The refusal carries the codes and levels involved:

```json
{
  "error": {
    "code": "insufficient_club_permission",
    "message": "This operation requires clubstation permission level 9",
    "details": { "required_level": 9, "granted_level": 3 }
  }
}
```

### Radios belong to their operator

Each member registers their own rigs into the shared club account, so
[Radio](radio.md) is scoped the same way: a member below officer level lists,
reads and deletes only the radios they registered themselves. Another member's
radio returns `404 not_found`.

### Lookups follow the same boundary

A [lookup](lookup.md) answers "have I worked this before" out of the logbook, so
it honours the same restriction — otherwise it would hand back the name, QTH and
locator recorded in a QSO the member is not allowed to list:

- `?callsign=` reports `workedBefore: false` (and `call_worked: false` with
  `detail=full`) when the only matching QSO belongs to another operator.
- `?grid=` and `?grid=all` only consider the member's own QSOs.

### Club member list

`GET /api/v2/club` stays officer-only. No disclosure of member information is provided to non-officers. See [Club](club.md).

## Statistics are club-wide

One deliberate exception: [Statistic](statistic.md) reports **club-wide**
figures for every member, regardless of level. The counters are aggregates, not
QSO data, and they are shared with the dashboard.

This means the numbers can legitimately disagree with what the QSO resource
returns to the same token:

```text
GET /api/v2/statistic?profile=qso   ->  total: 412   (the whole club)
GET /api/v2/qso                     ->  total: 57    (this member's QSOs)
```

Do not use the statistic totals to paginate or reconcile a QSO list.

## Losing membership

The permission is checked on every request, so removing a member from the club
takes effect at once — including for tokens that were set to never expire. Any
call made with such a token afterwards is refused:

```json
{
  "error": {
    "code": "club_access_revoked",
    "message": "The clubstation membership behind this token has been revoked"
  }
}
```

The status is `403`, not `401`: the token itself is still valid, only the
permission behind it is gone. Re-adding the member restores access immediately.

!!! tip "Handling this in a client"
    Treat `club_access_revoked` as permanent and stop retrying — unlike
    `rate_limited` it will not resolve on its own. Surface it to the user: the
    fix is on the club's side, not the client's.

## Summary

| | Personal | Level 3 / 6 | Level 9 |
| --- | --- | --- | --- |
| Read own QSOs | ✅ | ✅ | ✅ |
| Read other operators' QSOs | — | `404` | ✅ |
| Choose the `operator` field | ✅ | overwritten | ✅ |
| Read station locations | ✅ | ✅ | ✅ |
| Write/delete station locations | ✅ | `403` | ✅ |
| Own radios | ✅ | ✅ | ✅ |
| Other operators' radios | — | `404` | ✅ |
| Lookups across all operators | ✅ | own only | ✅ |
| Club member list | — | `403` | ✅ |
| Statistics | own | club-wide | club-wide |
