# Statistic

The Statistic resource is a **read-only** endpoint that returns flat, numeric
counters. It is designed for dashboards and monitoring/alerting systems (for
example Zabbix): each value has a stable JSONPath, so you can map it directly onto
a monitoring item or trigger.

- **Base path:** `/api/v2/statistic`
- **Scope:** `statistic:read`

!!! note
    Read the [API v2 overview](index.md) first for authentication, the response
    envelope and error codes.

## Endpoint

| Verb | Path | Scope | Purpose |
| --- | --- | --- | --- |
| `GET` | `/api/v2/statistic` | `statistic:read` | Return one or more statistics topics |

Query parameter:

| Parameter | Default | Values |
| --- | --- | --- |
| `profile` | `qso` | `qso`, `system`, `full` |

Each topic is nested under its own key, so a value's JSONPath is identical whether
you request the topic on its own or as part of `full`
(e.g. `$.data.qso.total`). The response `meta` reports the requested `profile` and
whether the token owner is an `admin`.

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
      "confirmations": { "lotw": 4, "eqsl": 3, "qsl": 6 },
      "dxcc": { "worked": 15, "confirmed": 9, "available": 340 }
    }
  },
  "meta": { "profile": "qso", "admin": false }
}
```

`by_band` and `by_mode` return the top entries by count.

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
        "client_url": null,
        "nodes": [],
        "nodes_alive": 0,
        "nodes_total": 0,
        "active_topics": 0,
        "connected_clients": 0
      }
    }
  },
  "meta": { "profile": "system", "admin": true }
}
```

### `full`

Returns every topic the token is permitted to see. For a regular token that is
just `qso`; for an administrator token it also includes `system`.

```bash
curl "https://<WAVELOG_URL>/index.php/api/v2/statistic?profile=full" \
     -H "Authorization: Bearer wl2_your_token_here"
```

## Monitoring tips

- Poll a single cheap topic (`profile=qso`) rather than `full` when you only need
  a few values.
- Because JSONPaths are stable, you can point a Zabbix (or similar) item straight
  at, for example, `$.data.qso.total` or `$.data.qso.activity.today`.
- Respect any [rate limits](index.md#rate-limiting) the instance owner has set.
