# Statistic

The Statistic resource is a **read-only** endpoint that returns flat, numeric
counters. It is designed for dashboards and monitoring/alerting systems (for
example Zabbix): each value has a stable JSONPath, so you can map it directly onto
a monitoring item or trigger.

- **Base path:** `/api/v2/statistic`
- **Scope:** `statistic:read`
- **Since version:** <span class="wl-since">3.1.0</span>

!!! note
    Read the [API v2 overview](index.md) first for authentication, the response
    envelope and error codes.

## Endpoint

| Verb | Path | Scope | Purpose | Since version |
| --- | --- | --- | --- | --- |
| `GET` | `/api/v2/statistic` | `statistic:read` | Return one or more statistics topics | <span class="wl-since">3.1.0</span> |

Query parameter:

| Parameter | Default | Values |
| --- | --- | --- |
| `profile` | `qso` | `qso`, `confirmations`, `system`, `full` |

Each topic is nested under its own key, so a value's JSONPath is identical whether
you request the topic on its own or as part of `full`
(e.g. `$.data.qso.total`). The response `meta` reports the requested `profile` and
whether the token owner is an `admin`.

!!! note "Clubstation tokens report club-wide figures"
    Unlike the [QSO](qso.md) resource, the counters are **not** narrowed to the
    acting member: every [clubstation](clubstation.md) token sees the whole
    club's numbers, at any permission level. The values are aggregates rather
    than QSO data.

    A consequence worth knowing: for a member below officer level
    `$.data.qso.total` will be larger than the `meta.total` of
    `GET /api/v2/qso`.

## Topics

### `qso` (default)

QSO analytics **scoped to the token owner's** station locations — these are your
own numbers, not the whole instance.

```bash
curl "https://<WAVELOG_URL>/index.php/api/v2/statistic?profile=qso" \
     -H "Authorization: Bearer wl2_your_token_here"
```

```json
{
  "data": {
    "qso": {
      "total": 28,
      "activity": { "today": 2, "month": 5, "year": 7 },
      "breakdown": {
        "by_band": [ { "band": "20m", "count": 12 } ],
        "by_mode": [ { "mode": "CW", "count": 7 } ]
      },
      "dxcc": {
        "worked": 15,
        "confirmed": 9,
        "confirmed_paper": 6,
        "confirmed_lotw": 7,
        "available": 340,
        "deleted": { "worked": 1, "confirmed": 1, "confirmed_paper": 1, "confirmed_lotw": 0 }
      }
    }
  },
  "meta": { "profile": "qso", "admin": false }
}
```

!!! note "New in Wavelog 3.1.1"
    The `dxcc` block gained the paper/LoTW confirmation split and a `deleted`
    sub-object, matching the dashboard DXCC card (PR #3663, mirroring #3656 / #3615).

    - `confirmed_paper`, `confirmed_lotw`: split of `confirmed` by paper QSL
      and by LoTW (overlap allowed, so each is `<= confirmed`).
    - `deleted`: same three keys for DXCC entities worked but since removed
      from the active list — shown in brackets on the card.
    - `available` now counts only **currently active** DXCC entities (deleted
      excluded), aligning it with the dashboard's Needed denominator.

`dxcc` mirrors the dashboard DXCC card. `confirmed` counts entities confirmed
by **paper QSL or LoTW** (combined and deduped — the card's Needed numerator);
`confirmed_paper` and `confirmed_lotw` are the split the card shows in its
Confirmed row, so each is `<= confirmed` while their sum may exceed it.
`available` is the count of currently active DXCC entities (deleted ones
excluded). `deleted` carries the same shape for entities worked but since
removed from the active list — the bracketed figure on the card. eQSL is
intentionally excluded from `dxcc.confirmed` even though it appears in the
[confirmations](#confirmations) topic below.

`by_band` and `by_mode` return the top entries by count.

### `confirmations`

QSL confirmation counts per confirmation type, broken down by band and by mode
alongside the grand totals. Like `qso`, it is scoped to the token owner's station
locations.

Unlike the other topics, `confirmations` accepts filters. They are all optional
and can be combined:

| Parameter | Default | Values |
| --- | --- | --- |
| `type` | all | Comma-separated list of `lotw`, `eqsl`, `qsl`, `qrz`, `clublog` |
| `since` | none | `YYYY-MM-DD` — only confirmations **received** on or after this date |
| `qso_since` | none | `YYYY-MM-DD` — only QSOs **made** on or after this date |
| `qso_until` | none | `YYYY-MM-DD` — only QSOs **made** on or before this date |
| `band` | all | A band such as `20m`, or `SAT` for satellite QSOs |
| `mode` | all | A mode or submode, e.g. `CW` or `FT8` |

`since` and `qso_since` answer different questions. `since` filters on the date a
confirmation arrived — "how many confirmations came in this week" — and is applied
per type against that type's own received-date column. `qso_since`/`qso_until` filter
on the date of the QSO itself — "how many of my QSOs since January are confirmed", or
paired, "how did last year's contest QSOs end up being confirmed". Both ends are
inclusive and can be used on their own. They are the same filters as on the
[QSO list](qso.md#list-qsos).

An invalid `type` value or a malformed date returns `400 validation_error`. Note
that HRDLog is **not** a confirmation type: Wavelog only uploads to it, there is
no received status to count.

Every row carries two framing numbers next to the per-type counts:

| Field | Meaning |
| --- | --- |
| `qsos` | QSOs the counters were drawn from, after all filters |
| `confirmed` | QSOs confirmed by **at least one** of the requested types |

!!! note "Clarification about `confirmed`"
    `confirmed` is deliberately **not** the sum of the per-type counts: a QSO confirmed via both
    LoTW and eQSL is one confirmed QSO, not two. So `confirmed <= lotw + eqsl + …` while
    `confirmed <= qsos` always holds. Reading a row as "68 QSOs on 17m, 50 of them confirmed,
    36 of those via LoTW" is the intended interpretation.

Groups without a single confirmation are still listed — a band with QSOs and no
confirmations is usually the interesting one. The `filters` object echoes back the
resolved filters, so a poller can interpret the numbers without tracking the query
it sent.

```bash
curl "https://<WAVELOG_URL>/index.php/api/v2/statistic?profile=confirmations&type=lotw,eqsl&since=2026-07-01" \
     -H "Authorization: Bearer wl2_your_token_here"
```

```json
{
  "data": {
    "confirmations": {
      "counts": { "qsos": 2480, "lotw": 812, "eqsl": 430, "confirmed": 1024 },
      "by_band": [
        { "band": "20m", "qsos": 1120, "lotw": 402, "eqsl": 210, "confirmed": 501 },
        { "band": "40m", "qsos": 780, "lotw": 236, "eqsl": 118, "confirmed": 297 }
      ],
      "by_mode": [
        { "mode": "FT8", "qsos": 1503, "lotw": 610, "eqsl": 300, "confirmed": 700 },
        { "mode": "CW", "qsos": 640, "lotw": 202, "eqsl": 130, "confirmed": 324 }
      ],
      "filters": {
        "type": ["lotw", "eqsl"],
        "since": "2026-07-01",
        "qso_since": null,
        "qso_until": null,
        "band": null,
        "mode": null
      }
    }
  },
  "meta": { "profile": "confirmations", "admin": false }
}
```

Only the requested types appear in `counts` and in the breakdown rows. Summing a
breakdown column always reproduces the matching value in `counts`.

Satellite QSOs are included and appear in `by_band` under their band; use
`band=SAT` to look at them on their own.

### `system` (admin only)

Version, build and instance internals — users, database and PHP versions, worker
status, cache and process statistics.

!!! warning
    The `system` topic is restricted to tokens owned by a Wavelog **administrator**.
    For a non-admin token the topic is hidden entirely: it is absent from `full`,
    and requesting `profile=system` returns `400 validation_error` (unknown
    profile), so its existence is never disclosed.

```json
{
  "data": {
    "system": {
      "wavelog": "2.0",
      "adif": "3.1.4",
      "migration_db": 287,
      "migration_config": 287,
      "database": "8.0.36",
      "php": "8.3.0",
      "environment": "production",
      "time": "2026-06-16 17:06:00",
      "wavelog_stats": { "users": 3, "stations": 5, "logbooks": 4, "radios": 2 },
      "system_stats": { "memory_usage": 8388608, "memory_peak": 10485760, "cpu_time": 0 },
      "cache": {},
      "worker": {
        "enabled": true,
        "client_url": "wss://log.example.org/ws",
        "nodes": [
          { "name": "worker-1", "url": null, "alive": true,  "version": "0.3.0", "active_topics": 2, "connected_clients": 5, "uptime": "3h22m0s" },
          { "name": "worker-2", "url": null, "alive": false, "version": "0.3.0", "active_topics": 1, "connected_clients": 3, "uptime": "1h02m11s" }
        ],
        "nodes_alive": 1,
        "nodes_total": 2,
        "active_topics": 2,
        "connected_clients": 5
      }
    }
  },
  "meta": { "profile": "system", "admin": true }
}
```

`worker` is `{ "enabled": false }` when the Worker integration is off. Otherwise:

| Field | Meaning |
|---|---|
| `nodes[]` | One entry per Worker node. With Worker 0.3.0+ the list is the cluster roster reported by the Worker itself (`name` = hostname, `url` = `null`). With older Workers, or while Redis is unreachable, Wavelog polls every configured URL instead (`name` = `url` = the polled URL). |
| `nodes[].alive` | Roster: heartbeat younger than 15 s. Polling: HTTP 200 from `/internal/status`. A crashed node stays listed as `alive: false` for 15 minutes. |
| `nodes_total` / `nodes_alive` | Size of `nodes[]` and how many are alive. `nodes_alive < nodes_total` is what the debug page shows as "Degraded". |
| `active_topics` / `connected_clients` | Sum over the alive nodes, `null` if no node answered. |

### `full`

Returns every topic the token is permitted to see. For a regular token that is
`qso` and `confirmations`; for an administrator token it also includes `system`.
The `confirmations` filters apply here too.

```bash
curl "https://<WAVELOG_URL>/index.php/api/v2/statistic?profile=full" \
     -H "Authorization: Bearer wl2_your_token_here"
```

## Monitoring tips

- Poll a single cheap topic (`profile=qso`) rather than `full` when you only need
  a few values.
- Because JSONPaths are stable, you can point a Zabbix (or similar) item straight
  at, for example, `$.data.qso.total` or `$.data.qso.activity.today`. The same
  holds for the confirmation counters, e.g. `$.data.confirmations.counts.lotw`.
- To alert on incoming confirmations, poll
  `profile=confirmations&since=<yesterday>` and trigger on
  `$.data.confirmations.counts.confirmed`.
- Respect any [rate limits](index.md#rate-limiting) the instance owner has set.
