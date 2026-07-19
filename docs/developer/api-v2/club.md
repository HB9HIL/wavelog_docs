# Club

The Club resource lists the members of a clubstation. It is the v2 equivalent of the
v1 `list_clubmembers` endpoint.

- **Base path:** `/api/v2/club`
- **Scope:** `club:read`

!!! note
    Read the [API v2 overview](index.md) first for authentication, the response
    envelope and error codes.

## Endpoint

| Verb | Path | Scope | Purpose |
| --- | --- | --- | --- |
| `GET` | `/api/v2/club` | `club:read` | List the clubstation's members |

## Who may call this

The endpoint is restricted to **club officers**. In clubstation terms the token
owner is the clubstation and the token creator is the member acting on its behalf,
so the caller must:

- use a token created for the clubstation by a member (owner ≠ creator), and
- that member must hold permission level **9** (officer) on the clubstation.

Any other token — including a personal token — is refused: a member below
officer level gets `403 insufficient_club_permission`, a personal token
`403 forbidden`. See [Clubstations](clubstation.md) for the permission levels.

!!! warning
    The response contains every member's **email address** alongside their name
    and locator. That is why the endpoint stays officer-only, and why a token
    with `club:read` should not be handed to a third-party tool lightly.

```bash
curl "https://<WAVELOG_URL>/index.php/api/v2/club" \
     -H "Authorization: Bearer wl2_your_token_here"
```

```json
{
  "data": [
    {
      "user_firstname": "Alex",
      "user_lastname": "Example",
      "user_locator": "JN47",
      "callsign": "N0CALL",
      "user_name": "alex",
      "user_email": "alex@example.org",
      "permission_level": 9,
      "user_language": "english"
    }
  ],
  "meta": { "count": 1 }
}
```

Each member object contains:

| Field | Notes |
| --- | --- |
| `user_firstname` / `user_lastname` | Member's real name |
| `user_locator` | Maidenhead locator |
| `callsign` | Member's callsign |
| `user_name` | Login/username |
| `user_email` | Member's email address |
| `permission_level` | Clubstation permission level (e.g. `9` = officer) |
| `user_language` | Member's UI language |

!!! note
    Because the list includes member contact details (email), it is restricted to
    club officers as described above.
