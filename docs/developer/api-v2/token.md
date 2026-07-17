# Token

The Token resource is a **whoami** endpoint: it returns metadata about the token
used for the request. It is the v2 equivalent of the v1 `auth` / `check_auth`
endpoints and is handy for verifying a token and discovering which scopes it holds.

- **Base path:** `/api/v2/token`
- **Scope:** none — any valid token may read its own metadata

!!! note
    Read the [API v2 overview](index.md) first for authentication, the response
    envelope and error codes.

## Endpoint

| Verb | Path | Scope | Purpose |
| --- | --- | --- | --- |
| `GET` | `/api/v2/token` | any valid token | Metadata of the current token |

The endpoint still requires a valid token (an invalid or expired one returns
`401`), but it needs no particular scope.

```bash
curl "https://<WAVELOG_URL>/index.php/api/v2/token" \
     -H "Authorization: Bearer wl2_your_token_here"
```

```json
{
  "data": {
    "id": 15,
    "name": "my logger",
    "owner": "N0CALL",
    "user_id": 1,
    "scopes": [ "qso:read", "qso:write", "lookup:read" ],
    "expires_at": "2026-08-16 12:05:50"
  }
}
```

`expires_at` is `null` for a token that never expires.

!!! tip
    Looking for the Wavelog version? That is available from the
    [Statistic](statistic.md) resource, not here.
