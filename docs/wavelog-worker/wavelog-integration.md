# Wavelog Integration

Once the Worker is running, you need to tell Wavelog where to find it. This page covers the Wavelog-side configuration.

## Prerequisites

- The Worker is running and reachable (see [Installation](installation.md))
- You have the `worker_secret` value you set in the Worker's `config.yaml`
- You know the Worker's internal API URL (e.g. `http://localhost:9001` or `http://wavelog-worker:9001`). For a [cluster](clustering.md) this is the URL of your load balancer or Kubernetes service in front of the workers, one URL is enough.
- You know the Worker's public WebSocket URL (e.g. `wss://your-domain.example/worker`)

## Wavelog Configuration

Create a new file `application/config/worker.php` based on `application/config/worker.sample.php` and fill in the required values:

```php
<?php

/*
| -------------------------------------------------------------------------
| Wavelog Worker Configuration
| -------------------------------------------------------------------------
|
| Optional WebSocket gateway for real-time updates in the browser. Requires
| the separate wavelog_worker service. Set worker_enabled = true to activate.
| Wavelog falls back to the classic AJAX heartbeat when disabled.
|
*/

// Enable or disable the Worker integration entirely.
$config['worker_enabled'] = true;

// Internal URL of the Worker (PHP -> Worker, HTTP).
// Single instance: the URL of your worker.
// Cluster: the URL of your load balancer / k8s service in front of the workers.
// The cluster nodes are discovered automatically (Worker 0.3.0 or newer), so
// one URL is enough. A cluster needs a Redis / Valkey instance, see the
// wavelog_worker sample config.yaml.
$config['worker_url'] = 'http://127.0.0.1:9001';

// Shared secret — must match worker_secret in the worker's config.yaml.
// Generate with: openssl rand -hex 32
$config['worker_secret'] = '';

// Timeout for publish calls in seconds (float). Keep it short:
// a slow worker must not block QSO saves.
$config['worker_timeout'] = 1.0;

// Public WebSocket URL for the browser (Browser -> Worker).
// May differ from worker_url when behind a reverse proxy or in Docker.
// Format: ws://host:port or wss://host:port. Empty = no WebSocket in browser.
$config['worker_client_url'] = 'ws://log.example.org:9000';

```

!!! tip "Docker setups"
    If Wavelog and the Worker run in the same Docker Compose stack, use the
    service name as the hostname:

    ```php
    $config['worker_url'] = 'http://wavelog-worker:9001';
    ```

## Upgrading to `worker_url`

Older Wavelog versions used two keys for the worker address: `worker_vip` (a load balancer URL) and `worker_urls` (a list with one entry per worker node, which the debug page polled one by one). Since Worker 0.3.0 the nodes announce themselves in Redis and any node reports the whole cluster, so Wavelog only needs **one** URL. That URL is the new key `worker_url`.

What to change in `application/config/worker.php`:

| You have | Set |
|---|---|
| a single worker, `worker_urls = ['http://host:9001']` | `$config['worker_url'] = 'http://host:9001';` |
| a cluster with `worker_vip` set | `$config['worker_url'] = <your worker_vip>;` |
| a cluster without `worker_vip`, several `worker_urls` | `$config['worker_url'] = <load balancer / service URL, or any one node>;` |

Then remove `worker_vip` and `worker_urls`.

- **Nothing breaks if you do not change anything.** `worker_vip` and `worker_urls` are still read when `worker_url` is empty, but they are deprecated and will be removed in **Wavelog Worker Version 1.0.0**. Until then the debug page shows a reminder.
- With a Worker **older than 0.3.0** the per-node overview on the debug page still relies on `worker_urls` listing every node. Update the Worker first, then switch to `worker_url`.
- The debug page keeps showing the node count and a "Degraded" badge as before; it now gets that information from the Worker instead of polling every node. See [Clustering → Node Lifecycle](clustering.md#node-lifecycle) for what "Degraded" means.

## How the Integration Works

The following diagram shows the full lifecycle of a real-time session (e.g. a contest, a live dashboard, or any feature using the Worker):

```mermaid
sequenceDiagram
    participant PHP as Wavelog PHP
    participant W as Worker :9001
    participant B1 as Client A (Browser)
    participant B2 as Client B (Browser)

    Note over PHP,W: Session starts
    PHP->>W: POST /internal/register<br/>{topic, require_token: true}
    W-->>PHP: 200 OK

    Note over B1,W: Clients connect
    PHP-->>B1: Page load (includes auth token + WS URL)
    B1->>W: WebSocket /ws?topic=...
    B1->>W: {type: "auth", token: "..."}
    W-->>B1: {type: "auth_ok"}

    PHP-->>B2: Page load (includes auth token + WS URL)
    B2->>W: WebSocket /ws?topic=...
    B2->>W: {type: "auth", token: "..."}
    W-->>B2: {type: "auth_ok"}

    Note over B1,PHP: Client A triggers an update
    B1->>PHP: POST (action via AJAX)
    PHP->>W: POST /internal/publish<br/>{topic, payload}
    W-->>B1: {type: "push", payload: ...}
    W-->>B2: {type: "push", payload: ...}
```

### Topics

Each session gets its own **topic** — an opaque string identifier used to group connected browsers. Wavelog creates the topic when a feature is activated and registers it with the Worker. Other clients joining the same session receive the same topic and auth token on their page load.

### Authentication

When the Worker topic is registered with `require_token: true`, every browser must present a valid **HMAC token** as the first WebSocket frame. This token is generated by PHP using the shared secret, includes an expiry timestamp, and is embedded in the page when it loads.

The token is short-lived by design: if it expires while the connection is open, the next AJAX heartbeat will return a 401, and the Worker closes the connection with an `auth_expired` error. The page then reloads and obtains a fresh token automatically.

### Internal API Endpoints

These endpoints are used exclusively by Wavelog's PHP backend. They are **not** meant to be called manually during normal operation.

All requests must include the `X-Worker-Secret` header.

#### `POST /internal/register`

Registers a topic before any browser can connect to it.

```json
{
  "topic": "session:abc123",
  "meta": {
    "require_token": true
  }
}
```

#### `POST /internal/unregister`

Removes a topic when the session ends.

```json
{
  "topic": "session:abc123"
}
```

#### `POST /internal/publish`

Broadcasts a payload to all browsers subscribed to a topic.

```json
{
  "topic": "session:abc123",
  "payload": { ... }
}
```

Returns `404` if the topic is not registered. Wavelog handles this by re-registering and retrying.

#### `GET /internal/status`

Returns a JSON status object with uptime, connected client count, and registered topics. Useful for monitoring.

```bash
curl -s -H "X-Worker-Secret: your-secret" http://localhost:9001/internal/status
```

```json
{
  "status": "ok",
  "version": "0.3.0",
  "uptime": "14m22s",
  "registered_topics": 2,
  "active_topics": 2,
  "connected_clients": 5,
  "connected_sockets": 6,
  "cluster_nodes": -1,
  "nodes": [
    {
      "id": "3f9c2a1b7d4e6f80",
      "name": "logbook-host",
      "version": "0.3.0",
      "started_at": "2026-09-21T08:00:00Z",
      "seen_at": "2026-09-21T08:14:22Z",
      "active_topics": 2,
      "connected_clients": 5,
      "connected_sockets": 6,
      "alive": true,
      "uptime": "14m22s",
      "uptime_seconds": 862
    }
  ]
}
```

Add `?topics=1` to include `topic_list` and `active_topic_list`. `nodes` lists every cluster member (just this worker in single-instance mode), see [Clustering](clustering.md#verifying-cluster-mode).

## Troubleshooting

### Browsers cannot connect / WebSocket fails

- Check that the WebSocket URL (`worker_client_url`) is correct and uses `wss://` for HTTPS sites.
- Verify your reverse proxy passes `Upgrade: websocket` headers (see [Installation → Reverse Proxy](installation.md#reverse-proxy-https-wss)).

### PHP cannot reach the internal API

- Check that `worker_url` points to port 9001, not 9000.
- Verify firewall or Docker network rules allow PHP → Worker on port 9001.

### `topic not registered` (HTTP 404 from `/internal/publish`)

- The Worker may have restarted and lost its in-memory registry. Wavelog handles this automatically: it catches the 404, re-registers the topic, and retries the publish.
- In Redis cluster mode, topics are stored in Redis and survive restarts.

### `auth_expired` WebSocket error

- The browser's auth token expired. The page will reload automatically and obtain a fresh token. This is expected behaviour after long idle periods.
