# MCP Events — Design Sketch

**Status:** Draft proposal
**Authors:** Peter Alexander
**Date:** 2026-02-19

## Summary

Add an Events primitive to MCP that allows clients to subscribe to things happening in the outside world (new email, Slack message, devops alert, scheduled reminder, etc.) and react to them. The design supports three delivery mechanisms — client-side polling, server-side push over an existing connection, and webhook delivery to an external endpoint — with a unified subscription API that abstracts the difference from application code.

Key design principles:

- **Servers advertise delivery modes; no mode is mandatory.** Each event type lists the delivery modes it supports. The client picks the best mode it can use. If there is no overlap between what the server offers and what the client can consume, that event type is simply unavailable to that client — the protocol does not require a universal fallback. (Reference SDKs are still encouraged to make poll cheap to support so that overlap is common in practice.)
- **SDK-level polling, not LLM-level polling.** The client SDK drives the polling loop without burning LLM inference tokens. The LLM is only invoked when events actually arrive.
- **No durable subscription state.** Poll holds no subscription state at all. Push scopes subscription state to connection lifetime. Webhook uses soft state with mandatory TTL — the server holds subscriptions in memory, but they expire automatically if the client stops refreshing. The client is always the source of truth across all three modes. (This principle is about *subscription* state. Servers that relay from upstream sources will still hold other state — upstream credentials, webhook registrations with the upstream, an in-memory event buffer — but none of it is owed to any particular MCP client and all of it is outside the protocol's concern.)
- **Client owns subscription state.** In all modes, the client holds the canonical list of subscriptions. For poll and push, the server has no subscription state at all. For webhook, the server holds TTL-scoped subscription records, but the client drives their lifecycle via periodic refresh.
- **Event payloads are untrusted data.** The spec must be explicit that event payloads carry the same injection risks as tool results.
- **Servers own delivery.** Unlike traditional pub/sub (Kafka, SNS) where a shared broker decouples producers from subscribers, MCP servers run as independent processes with no central infrastructure. Each server handles event delivery itself, with the SDK absorbing the delivery complexity. In the common case, the server is a relay — ingesting events from an upstream system (GitHub webhooks, a Slack socket, a message bus) and routing them to subscribed clients — rather than the originating producer. Deployments that want a decoupled model can use a dedicated MCP server as a broker — no protocol changes required.

## Capability Declaration

Servers advertise event support in their capabilities:

```jsonc
{
  "capabilities": {
    "events": {
      "subscribe": true
    }
  }
}
```

## Listing Available Events

### Request: `events/list`

```jsonc
// params (optional)
{ "cursor": "..." }  // pagination
```

### Response

```jsonc
{
  "events": [
    {
      "name": "email.received",
      "description": "Fires when a new email arrives in the inbox",
      "delivery": ["poll"],
      "inputSchema": {
        "type": "object",
        "properties": {
          "from": { "type": "string", "description": "Glob pattern for sender address" },
          "subject_contains": { "type": "string" },
          "redact_pii": { "type": "boolean", "default": false, "description": "Strip PII from event payloads" },
          "include_body_preview": { "type": "boolean", "default": true, "description": "Include a snippet of the email body" }
        }
      },
      "payloadSchema": {
        "type": "object",
        "properties": {
          "messageId": { "type": "string" },
          "from": { "type": "string" },
          "subject": { "type": "string" },
          "receivedAt": { "type": "string", "format": "date-time" }
        }
      }
    },
    {
      "name": "incident.created",
      "description": "Fires when a new PagerDuty incident is created",
      "delivery": ["webhook", "push", "poll"],
      "inputSchema": {
        "type": "object",
        "properties": {
          "severity": { "type": "string", "enum": ["P1", "P2", "P3", "P4"] },
          "service": { "type": "string" },
          "deduplicate_window_seconds": { "type": "integer", "default": 0, "description": "Suppress duplicate alerts within this window" }
        }
      },
      "payloadSchema": { "..." : "..." }
    }
  ]
}
```

**Notes:**

- `delivery` lists the delivery modes this event type supports — any non-empty subset of `"poll"`, `"push"`, `"webhook"`. No mode is mandatory. A client that cannot use any of the listed modes cannot subscribe to this event type.
- `inputSchema` is a JSON Schema describing valid subscription parameters — these may include filters (which narrow the event stream), transforms (which modify payloads), or other server-defined configuration. This mirrors the `inputSchema` on tools for consistency.
- `payloadSchema` describes the shape of `data` in delivered events.

### Dynamic Event Types: `notifications/events/list_changed`

If the set of available event types changes at runtime (e.g., a plugin is loaded, a data source is connected), the server sends a `notifications/events/list_changed` notification. The client SHOULD re-call `events/list` to refresh its event type registry. This is consistent with `notifications/tools/list_changed` and `notifications/resources/list_changed`.

## Subscribing and Event Delivery

There are three delivery modes with different subscription mechanisms:

- **Poll mode:** Client calls `events/poll` with event name, params, and cursor. No separate subscribe step needed — the first poll with a null cursor bootstraps the subscription. Server is fully stateless.
- **Push mode:** Client opens a long-lived POST (`events/stream`) carrying all desired subscriptions. Events are delivered on the SSE response stream (HTTP) or as notifications on stdout (stdio). Connection close terminates all subscriptions. Server state is scoped to connection lifetime.
- **Webhook mode:** Client calls `events/subscribe` to register a callback URL. The server POSTs events to that URL as they occur. Subscriptions have a mandatory TTL — the client must periodically refresh by re-calling `events/subscribe` before the TTL expires. If the client stops refreshing, the subscription expires and the server reclaims resources. Designed for remote servers where maintaining a long-lived connection is impractical.

### Error Codes (applicable to all modes)

| Code | Message | Meaning |
|------|---------|---------|
| `-32602` | `InvalidParams` | Params don't match inputSchema (standard JSON-RPC invalid params) |
| `-32001` | `EventNotFound` | Unknown event name |
| `-32002` | `Unauthorized` | User lacks permission for this event/params combination |
| `-32003` | `TooManySubscriptions` | Server-imposed subscription limit reached |
| `-32005` | `InvalidCallbackUrl` | Webhook URL is unreachable or rejected by the server |
| `-32006` | `SubscriptionNotFound` | Unknown subscription ID (for `events/unsubscribe`) |

### Poll-Based Delivery

The client SDK calls `events/poll` at the server-recommended interval. This is a protocol-level operation, NOT an LLM tool call.

#### Request: `events/poll`

```jsonc
{
  "maxEvents": 50,                    // optional; cap events per subscription
  "subscriptions": [
    {
      "id": "sub_email",              // client-provided identifier
      "name": "email.received",
      "params": {
        "from": "*@anthropic.com",
        "redact_pii": true
      },
      "cursor": null                  // null = start from now
    },
    {
      "id": "sub_incidents",
      "name": "incident.created",
      "params": { "severity": "P1" },
      "cursor": "cursor_abc"          // resume from previous position
    }
  ]
}
```

#### Response

```jsonc
{
  "results": [
    {
      "id": "sub_email",
      "events": [
        {
          "eventId": "evt_001",
          "name": "email.received",
          "timestamp": "2026-02-19T15:30:00Z",
          "data": {
            "messageId": "msg_xyz",
            "from": "dsp@anthropic.com",
            "subject": "MCP spec review",
            "receivedAt": "2026-02-19T15:29:58Z"
          }
        }
      ],
      "cursor": "historyId_99842",
      "hasMore": false,
      "nextPollSeconds": 30
    },
    {
      "id": "sub_incidents",
      "events": [],
      "cursor": "cursor_abc",
      "hasMore": false,
      "nextPollSeconds": 60
    }
  ]
}
```

**Notes:**

- `id` is a client-provided identifier for each subscription. It is opaque to the server and echoed back in responses to allow the client to correlate results with subscriptions. It must be unique within a single `events/poll` request.
- `cursor` is opaque to the client. The client stores it and passes it back on the next poll. A `null` cursor means "start from now" — the server returns no events and provides a fresh cursor for subsequent polls.
- `eventId` enables client-side deduplication across polls (e.g., after a crash/restart).
- `maxEvents` is an optional top-level field that caps the number of events returned per subscription. If more events are available than the limit, the server returns a partial batch with an intermediate cursor and sets `hasMore: true`. The client SHOULD poll again immediately (ignoring `nextPollSeconds`) to drain the backlog. If omitted, the server uses its own default limit.
- `hasMore` indicates whether additional events are available beyond the returned batch. When `true`, the client should poll again immediately with the updated cursor. When `false`, the client waits `nextPollSeconds` before the next poll.
- `nextPollSeconds` allows the server to dynamically adjust polling frequency per subscription (e.g., back off when rate-limited upstream, speed up when activity is detected). Ignored when `hasMore` is `true`.
- Empty `events` array means nothing happened — this is the common case and should be cheap.
- The server holds no per-client subscription state. Each poll request is self-contained: the client provides the event name, params, and cursor. The server does not need to "remember" previous poll requests. (For emit-only event types, the server does hold a ring buffer of recent events — see *Emit-only event types* under Server SDK Guidance — but that buffer is shared infrastructure, not per-subscription state.)

#### Error Handling

If an individual subscription within a poll request is invalid, the server returns an error for that subscription without failing the entire request:

```jsonc
{
  "results": [
    {
      "id": "sub_email",
      "events": [ "..." ],
      "cursor": "historyId_99842",
      "nextPollSeconds": 30
    },
    {
      "id": "sub_bogus",
      "error": {
        "code": -32001,
        "message": "EventNotFound"
      }
    }
  ]
}
```

### Push-Based Delivery

Push delivery uses a long-lived `events/stream` request. The client sends all desired subscriptions, and the server delivers events as they occur. The request is a standard JSON-RPC request with an `id`, which enables cancellation via `notifications/cancelled`.

The transport mechanism differs by transport type:

- **Streamable HTTP:** The `events/stream` request is a POST that returns an SSE response stream. This replaces the GET-based SSE stream — `events/stream` is the sole server→client push channel. Connection close implicitly cancels — no explicit cancellation message is needed.
- **stdio:** The `events/stream` request is sent on stdin. The server delivers events as JSON-RPC notifications on stdout. Since there is no connection to close, the client cancels by sending `notifications/cancelled` with the request's `id`.

#### Request: `events/stream`

```jsonc
// Streamable HTTP: POST /mcp
// stdio: written to stdin
{
  "jsonrpc": "2.0",
  "method": "events/stream",
  "id": 1,
  "params": {
    "subscriptions": [
      {
        "id": "sub_email",
        "name": "email.received",
        "params": { "from": "*@anthropic.com", "redact_pii": true },
        "cursor": null
      },
      {
        "id": "sub_incidents",
        "name": "incident.created",
        "params": { "severity": "P1" },
        "cursor": "cursor_abc"
      }
    ]
  }
}
```

#### Event Delivery

The server confirms each subscription, reports errors for invalid ones, and then delivers events as notifications. When the stream ends, the server sends a `StreamEventsResult` as the final frame:

```jsonc
// Confirmation (one per valid subscription)
{"jsonrpc":"2.0","method":"notifications/events/active","params":{"id":"sub_email","cursor":"historyId_99840"}}
{"jsonrpc":"2.0","method":"notifications/events/active","params":{"id":"sub_incidents","cursor":"cursor_abc"}}

// Error for invalid subscription (stream remains open for valid ones)
{"jsonrpc":"2.0","method":"notifications/events/error","params":{"id":"sub_bogus","code":-32001,"message":"EventNotFound"}}

// Events as they occur
{"jsonrpc":"2.0","method":"notifications/event","params":{"id":"sub_email","eventId":"evt_001","name":"email.received","timestamp":"2026-02-19T15:30:00Z","data":{"messageId":"msg_xyz","from":"dsp@anthropic.com","subject":"MCP spec review"},"cursor":"historyId_99842"}}

// Final frame when stream closes (StreamEventsResult)
{"jsonrpc":"2.0","id":1,"result":{"_meta":{}}}
```

On Streamable HTTP, notifications are SSE `data:` frames, and the `StreamEventsResult` is the final `data:` frame that closes the request. On stdio, notifications are newline-delimited JSON messages on stdout, and the result is sent when the stream is cancelled.

Non-event MCP notifications (e.g., `notifications/tools/list_changed`, `notifications/resources/updated`) also flow alongside event notifications.

#### Lifecycle

- **Stream termination.** The `StreamEventsResult` is an empty typed result (`{"_meta": {}}`) sent as the final message when the stream closes. It carries no information — it satisfies JSON-RPC's requirement that every request gets a response. All meaningful content is in the preceding notifications.
- **Heartbeat.** The server MUST send periodic keepalive messages on the push stream so the client can distinguish "nothing to send" from "connection is dead." On Streamable HTTP, this is an SSE comment (`: keepalive\n\n`). On stdio, this is a `notifications/events/heartbeat` message. The server SHOULD send a heartbeat at least every 30 seconds. The client SHOULD treat absence of any data (events or heartbeats) beyond a threshold (e.g., 60 seconds) as connection failure and reconnect with cursors.
- **Cancellation.** On Streamable HTTP, the client closes the connection. On stdio, the client sends `notifications/cancelled` with the `requestId` matching the `events/stream` request's `id`. In both cases, the server MUST stop delivering events, send the `StreamEventsResult`, and release any associated resources.
- **Updating subscriptions.** To add, remove, or change subscriptions, the client cancels the current stream (connection close on HTTP, `notifications/cancelled` on stdio) and sends a new `events/stream` request with the updated subscription list and current cursors. The cursor-based replay ensures no events are missed during the brief transition.
- **Reconnection after failure.** If the connection drops (HTTP) or the server stops sending (stdio), the client sends a new `events/stream` with the same subscriptions and their last-known cursors.
- **Empty subscriptions.** A client may open `events/stream` with an empty `subscriptions` array solely to receive non-event notifications (e.g., `notifications/tools/list_changed`).

#### Cursor Advancement

Each event notification on the push stream includes a `cursor` field. This cursor represents the subscription's position *after* this event. The client tracks the latest cursor per subscription for use during reconnection.

### Webhook-Based Delivery

Webhook delivery is for remote MCP servers where maintaining a long-lived connection (push) or frequent polling is impractical. The server POSTs events to a callback URL provided by the client. Webhook subscriptions use soft state with mandatory TTL — the server holds them in memory, but they expire automatically if the client stops refreshing.

The callback URL does not need to be the client itself. A common deployment is a forward proxy that receives webhooks and serves events to clients via poll or push:

```
Upstream → MCP Server → webhook POST → Forward Proxy ← Client (poll or push)
```

#### Subscribing: `events/subscribe`

Unlike poll and push, webhook delivery requires an explicit subscribe step because the server needs to know where to POST events. `events/subscribe` is idempotent — calling it again with the same subscription key (see *Subscription Identity* below) refreshes the TTL and updates mutable fields. This is the mechanism clients use to keep subscriptions alive.

```jsonc
{
  "jsonrpc": "2.0",
  "method": "events/subscribe",
  "id": 2,
  "params": {
    "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "name": "incident.created",
    "params": { "severity": "P1" },
    "delivery": {
      "mode": "webhook",
      "url": "https://proxy.example.com/hooks/client123"
    },
    "cursor": null
  }
}
```

```jsonc
// Response
{
  "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "secret": "whsec_5c8f...",          // present when subscription is (re)created
  "cursor": "cursor_start_001",
  "refreshBefore": "2026-02-19T16:30:00Z"
}
```

**Notes:**

- `events/subscribe` is ONLY used for webhook delivery. Poll and push do not need it.
- `id` is a client-generated, high-entropy identifier for the logical subscription. It MUST contain at least 122 bits of entropy (e.g., a UUIDv4). See *Subscription Identity* below.
- `secret` is a shared secret for HMAC-SHA256 signature verification of webhook deliveries. The server generates it when the subscription is created and returns it in the response; it is not returned on subsequent refreshes of an existing subscription. If the server has lost the subscription (restart, TTL expiry) the refresh creates a new one, and a fresh `secret` appears in the response — the client MUST check for its presence on every refresh and update the verifier accordingly. The client MAY supply `delivery.secret` in the request to override server generation (e.g., when the secret is provisioned out-of-band in a vault); servers SHOULD accept this but are not required to.
- `cursor` follows the same semantics as other modes: `null` means "start from now" on a new subscription. When refreshing an existing subscription, `null` means "keep the server's current position"; a non-null value resets the subscription's replay position.
- `refreshBefore` is mandatory. It is an ISO 8601 timestamp indicating when the subscription will expire. The client MUST re-call `events/subscribe` with the same subscription key before this time to keep the subscription alive. The server resets the TTL on each refresh.
- `events/subscribe` is idempotent within the caller's subscription scope (see *Subscription Identity*). If a subscription with the same scoped key exists, the server resets the TTL and updates mutable fields in place. If the subscription has expired — or the server has restarted and lost it — the server creates a fresh subscription using the provided cursor.
- The server holds subscription state (id, event name, params, callback URL, secret, cursor) in memory with TTL. No durable storage is required — if the server restarts, clients will re-subscribe on their next refresh cycle, and cursors ensure no events are missed.

#### Subscription Identity

Webhook subscriptions are keyed by a compound **subscription key** that determines which `events/subscribe` calls refer to the same logical subscription. The server uses this key for idempotent upsert on subscribe, for lookup on unsubscribe, and to isolate tenants from one another.

**Key composition.** The subscription key is:

- On servers that authenticate the caller: `(principal, id)`.
- On servers without caller authentication: `(delivery.url, id)`.

where `principal` is the server's canonical identifier for the authenticated subject (e.g., OAuth `sub`, API key ID, service-account name) and `delivery.url` is the callback URL exactly as supplied by the client. Servers that support both authenticated and anonymous access MUST select the scoping mode per-request: `(principal, id)` whenever a principal is present, `(delivery.url, id)` otherwise. A subscription created under one scope is not visible under the other.

**`id` requirements.** The `id` field MUST be a high-entropy value containing at least 122 bits of randomness; a UUIDv4 is RECOMMENDED. The client MUST generate `id` once per logical subscription and persist it for the subscription's lifetime — a fresh `id` on each subscribe call creates a new subscription rather than refreshing the existing one. SDKs SHOULD generate and persist `id` on the client's behalf and SHOULD NOT expose an interface that encourages hand-picked low-entropy values.

**Capability semantics (unauthenticated).** When the server does not authenticate callers, knowledge of `(delivery.url, id)` is sufficient to refresh, modify, or delete the subscription. The `id` therefore functions as a bearer capability and SHOULD be treated as confidential: clients SHOULD NOT log it at default verbosity, embed it in URLs, or expose it to untrusted intermediaries. Servers MAY reject `id` values that are obviously low-entropy (shorter than 16 bytes, dictionary words, sequential integers) to guard against misconfigured clients.

**Mutable vs. key fields.** On an idempotent subscribe against an existing key, the server updates fields as follows:

| Field | Behavior on existing subscription |
|---|---|
| `name`, `params` | Replaced. Changing these redefines what the subscription listens for. |
| `delivery.url` | **Authenticated scope:** replaced — the key is `(principal, id)`, so the URL is mutable. **Unauthenticated scope:** part of the key — a different URL addresses a different subscription. |
| `delivery.secret` | Replaced if supplied (client-driven rotation / override). If omitted, the existing secret is unchanged and is not returned. |
| `cursor` | Replaced if non-null; ignored if null (the server keeps its current position — a refresh loop typically passes `null`). |
| TTL | Reset. |

**Cross-tenant isolation.** Because the subscription key always includes either `principal` or `delivery.url`, two distinct tenants cannot collide on `id` alone. A malicious caller who learns another tenant's `id` but not their principal credentials or callback URL cannot refresh, modify, or delete that tenant's subscription, nor redirect its deliveries.

#### Webhook Event Delivery

The server POSTs events to the callback URL as they occur:

```
POST https://proxy.example.com/hooks/client123
Content-Type: application/json
X-MCP-Signature: sha256=<HMAC-SHA256(secret, timestamp + "." + body)>
X-MCP-Timestamp: 1739980800

{
  "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "eventId": "evt_789",
  "name": "incident.created",
  "timestamp": "2026-02-19T16:00:00Z",
  "data": {
    "incidentId": "INC-1234",
    "title": "Database connection pool exhausted",
    "severity": "P1"
  },
  "cursor": "cursor_xyz"
}
```

**Notes:**

- `X-MCP-Signature` contains an HMAC-SHA256 signature computed over `timestamp + "." + body` using the shared secret from subscription. The receiver MUST verify this before processing. The `X-MCP-Timestamp` header contains the Unix timestamp (seconds) of the request; the receiver SHOULD reject deliveries older than 5 minutes to prevent replay attacks.
- `eventId` in the body enables idempotent processing. The receiver SHOULD deduplicate by this value.
- The server SHOULD retry on non-2xx responses with exponential backoff.
- **Subscribe/delivery race.** Because the server may begin delivering as soon as the subscription is persisted, the first webhook POST can arrive before the `events/subscribe` response (and thus the `secret`) reaches the client. The recommended handling for now is retry tolerance: a receiver that gets a delivery for an `id` it does not yet recognise, or that it cannot yet verify, SHOULD return a retryable status (e.g., `503` or `425 Too Early`) rather than dropping it; the server's normal retry/backoff machinery will redeliver once the client is ready. `eventId` deduplication and cursor replay make this safe. A two-phase activate handshake was considered and may be added later if this proves insufficient in practice.
- After repeated failures (server-defined threshold), the server MAY stop retrying for the remainder of the TTL. The subscription will expire naturally, and the client's next refresh re-establishes it with cursor-based replay.

#### Webhook Delivery Status

The `events/subscribe` response MAY include a `deliveryStatus` object when refreshing an existing subscription. This lets the client detect delivery problems without a separate monitoring channel:

```jsonc
// Healthy subscription refresh
{
  "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "cursor": "cursor_xyz",
  "refreshBefore": "2026-02-19T17:00:00Z",
  "deliveryStatus": {
    "active": true,
    "lastDeliveryAt": "2026-02-19T16:28:00Z",
    "lastError": null
  }
}
```

```jsonc
// Subscription with delivery failures
{
  "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "cursor": "cursor_xyz",
  "refreshBefore": "2026-02-19T17:00:00Z",
  "deliveryStatus": {
    "active": false,
    "lastDeliveryAt": "2026-02-19T15:45:00Z",
    "lastError": "Webhook endpoint returned 403 Forbidden",
    "failedSince": "2026-02-19T15:50:00Z"
  }
}
```

`deliveryStatus` is OPTIONAL — servers MAY omit it entirely. When present, `active` indicates whether the server is currently delivering events (`false` means it has stopped retrying after repeated failures). `lastError` provides a human-readable description of the most recent failure. The client can use this information to diagnose connectivity or authentication issues with the webhook endpoint.

#### Webhook Security

**SSRF prevention.** The server MUST validate callback URLs at subscribe time. Servers SHOULD reject URLs pointing to private or loopback address ranges (`127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`, `::1`, `fc00::/7`) unless explicitly configured to allow them. The server SHOULD resolve the hostname and validate the resolved IP before accepting the subscription to prevent DNS rebinding attacks. Servers MAY maintain an allowlist of permitted callback URL patterns.

**TLS requirement.** Callback URLs SHOULD use `https://`. Event payloads transit in cleartext over plain `http://`, exposing them to interception. Servers MAY reject non-TLS callback URLs.

**Replay attack prevention.** The webhook POST SHOULD include an `X-MCP-Timestamp` header containing the Unix timestamp (seconds) at which the request was generated. The HMAC signature MUST cover both the timestamp and the body: `HMAC-SHA256(secret, timestamp + "." + body)`. The receiver SHOULD reject deliveries where the timestamp is more than 5 minutes old. This prevents captured webhook payloads from being replayed.

**Secret generation.** The signing secret is generated by the server. This matches the dominant webhook convention (Stripe, Slack, Shopify, the Standard Webhooks spec): the party that signs deliveries owns the key, which guarantees adequate entropy, per-subscription uniqueness, and unilateral rotation. Client-supplied secrets (the GitHub model) are supported as an optional override via `delivery.secret` for deployments that pre-provision secrets in a vault, but are not the default.

**Secret rotation.** A client that needs to force rotation calls `events/subscribe` with the same subscription key and supplies a new `delivery.secret`; the idempotent upsert replaces it atomically alongside the TTL refresh. Server-initiated rotation happens implicitly whenever the server (re)creates the subscription — the new secret appears in the response and the client adopts it. On unauthenticated servers, the ability to rotate is gated solely by knowledge of `(delivery.url, id)` — see *Subscription Identity* for the capability implications.

**Authentication to the callback endpoint.** HMAC is the only authentication mechanism the protocol defines between the MCP server and the callback endpoint. This matches the dominant pattern for product webhooks (Stripe, GitHub, Slack, Twilio all authenticate deliveries with HMAC alone) and keeps the server's per-subscription credential surface to a single secret.

HMAC is verified by application code after the request has been routed. It does not help if the callback URL sits behind infrastructure that requires transport-level auth before routing — an API gateway that demands `Authorization: Bearer`, a cloud endpoint requiring a platform-issued identity token, an mTLS-only mesh. For those deployments, point the callback at a forward proxy under the tenant's control: the proxy presents a public endpoint to the MCP server, terminates HMAC, and re-authenticates to downstream services using whatever mechanism the internal network requires. The protocol does not provide a way to pass through bearer tokens, perform an OAuth client-credentials grant, or attach OIDC identity tokens on the tenant's behalf; these may be considered for a future revision if deployment experience warrants it.

#### Unsubscribing: `events/unsubscribe`

`events/unsubscribe` is an eager cleanup mechanism. It is not required for correctness — subscriptions will expire naturally when the client stops refreshing. However, calling it immediately frees server resources and stops webhook deliveries without waiting for TTL expiry.

```jsonc
{
  "jsonrpc": "2.0",
  "method": "events/unsubscribe",
  "id": 3,
  "params": {
    "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479"
  }
}
```

`events/unsubscribe` resolves the subscription using the same scoped key as `events/subscribe`. For authenticated callers, `(principal, id)` is sufficient and the payload above is complete. For unauthenticated callers, the client MUST also include `delivery.url` so the server can form the `(delivery.url, id)` key:

```jsonc
// Unauthenticated server
{
  "jsonrpc": "2.0",
  "method": "events/unsubscribe",
  "id": 3,
  "params": {
    "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "delivery": { "url": "https://proxy.example.com/hooks/client123" }
  }
}
```

`events/unsubscribe` is ONLY used for webhook delivery. Poll subscriptions are implicit (stop polling to unsubscribe). Push subscriptions are scoped to the `events/stream` connection (close to unsubscribe).

## Listing Active Subscriptions

The client owns all subscription state across all three delivery modes. There is no server-side subscription listing method. The client maintains its own subscription registry and can reconstruct server-side state at any time via idempotent `events/subscribe` (for webhook mode) or by resuming poll/push with stored cursors.

Enterprise governance tools can inspect the client SDK's subscription registry for audit purposes. For webhook mode, orphaned subscriptions (e.g., from a crashed client) are cleaned up automatically by TTL expiry — no server-side listing is needed.

## Cursor Lifecycle

Cursors are opaque strings managed by the server. They represent a position in the event stream.

**Initial cursor:** Passing `cursor: null` in a subscription (either in `events/poll` or `events/stream`) means "start from now." The server returns a fresh cursor representing the current position. No historical events are replayed.

**Cursor expiry:** Cursors may become invalid (upstream data compacted, server state reset). The server returns a `CursorExpired` error for the affected subscription. The client SDK SHOULD handle this by re-subscribing with `cursor: null` to get a fresh cursor.

For poll mode, this appears in the per-subscription results:

```jsonc
{
  "id": "sub_email",
  "error": {
    "code": -32004,
    "message": "CursorExpired",
    "data": { "reason": "Upstream history compacted" }
  }
}
```

For push mode, this appears as a notification on the stream:

```
data: {"jsonrpc":"2.0","method":"notifications/events/error","params":{"id":"sub_email","code":-32004,"message":"CursorExpired","data":{"reason":"Upstream history compacted"}}}
```

On `CursorExpired`, the client should reconnect the affected subscription with `cursor: null`.

## Server SDK Guidance

The server SDK should make implementing events as simple as implementing tools. The server author writes one function — a "check for changes since cursor" function — and the SDK handles delivery mechanics for all three modes (poll, push, and webhook). The same check function backs every delivery mode; the SDK decides how to deliver based on what the client requested.

By default the server author does not declare which delivery modes an event type supports. The SDK infers them from the server's configuration and transport:

- **Poll** is available by default — backed by the check function when one is provided, or by the SDK's emit-fed ring buffer otherwise (see *Emit-only event types* below). Authors MAY opt an event type out of poll (e.g., when buffering is undesirable for a high-volume push-only source).
- **Push** is available when the transport supports streaming (Streamable HTTP or stdio).
- **Webhook** is available when the server has `webhook_ttl` configured.

The SDK computes the `delivery` array in `events/list` responses from the above, minus any modes the author has explicitly disabled. The protocol does not require any particular mode to be present, so the SDK MUST also accept an explicit `delivery=[...]` override per event type for authors who want full control. Servers that call `server.emit()` for an event type additionally enable real-time push and webhook delivery without the SDK needing to run an internal polling loop.

```python
@server.event(
    name="email.received",
    description="New email arrives",
    input_schema={ ... },
    payload_schema={ ... },
)
async def check_email(context, params, cursor):
    """Called by the SDK for all delivery modes."""
    if cursor is None:
        # Bootstrap: return current position, no events
        current = await gmail.history().get_current_id()
        return EventResult(events=[], cursor=current)

    history = await gmail.history().list(startHistoryId=cursor)
    events = []
    for msg in history.messages:
        if matches_params(params, msg):
            data = {"messageId": msg.id, "from": msg.sender, ...}
            if params.get("redact_pii"):
                data = redact(data)
            events.append(Event(name="email.received", data=data))
    return EventResult(events=events, cursor=history.historyId)
```

The SDK calls this function in different contexts depending on the delivery mode, but the server author does not need to distinguish between them:

**For poll mode:** The SDK calls the check function for each subscription in an incoming `events/poll` request and returns the results. The server is stateless.

**For push mode:** The SDK supports two patterns:

- **Poll-driven push (default).** The SDK maintains an internal polling loop per push subscription, calling the check function periodically and writing events to the stream. This is the simplest path for servers wrapping poll-only APIs (e.g., Gmail's history endpoint).
- **Direct emit.** Servers with true push sources (webhooks, WebSocket listeners, change streams) can emit events directly via `server.emit()`. The SDK routes emitted events to all active push streams with matching subscriptions.

Push state is scoped to the lifetime of the `events/stream` connection — when the connection closes, all loops and listeners stop.

**For webhook mode:** The SDK uses the same two patterns (poll-driven or direct emit), but instead of writing to an SSE stream, it POSTs events to the subscriber's callback URL with HMAC signatures. The SDK holds webhook subscriptions in memory with TTL — no external storage is required. If the server restarts, all webhook subscriptions are lost. Clients will re-subscribe on their next refresh cycle, passing their last-known cursor to resume without missing events. This is by design: the mandatory TTL + refresh mechanism eliminates the need for durable subscription storage.

```python
# Webhook mode uses in-memory subscription state with TTL.
# No external storage needed — clients refresh before expiry.
server = MCPServer(
    webhook_ttl=timedelta(minutes=30),  # subscriptions expire after 30 min
)
```

**Direct emit works across all modes.** When `server.emit()` is called, the SDK routes the event to all active subscriptions with matching event name. The SDK supports two emit patterns:

- **Broadcast emit.** The server emits an event without specifying a subscription. The SDK matches the event against all active subscriptions' params and delivers to those that match. This is appropriate when the upstream source delivers all events regardless of subscription params (e.g., a PagerDuty webhook that fires for all incidents — the SDK filters by `severity`).
- **Targeted emit.** The server emits an event to a specific subscription by ID. This is appropriate when the server has set up a per-subscription upstream listener and already knows which subscription the event belongs to.

```python
# Broadcast emit — SDK filters by subscription params.
async def on_pagerduty_webhook(payload):
    server.emit(Event(
        name="incident.created",
        data={"incidentId": payload["id"], "severity": payload["severity"], ...}
    ))

# Targeted emit — server already knows the subscription.
async def on_slack_message(subscription_id, message):
    server.emit(Event(
        name="slack.message",
        data={"text": message.text, "channel": message.channel, ...}
    ), subscription_id=subscription_id)
```

**Emit-only event types.** Many upstream sources are push-only with no cursor-addressable change feed — the server author receives upstream events (via webhook, WebSocket, message bus, etc.) and has no meaningful `check()` function to write. For these event types, the server author declares the event with `emit_only=True` and omits the check function entirely:

```python
@server.event(
    name="incident.created",
    description="Fires when a new PagerDuty incident is created",
    emit_only=True,  # no check function; poll served from SDK ring buffer
    input_schema={ ... },
    payload_schema={ ... },
)
class IncidentCreated:
    pass

async def on_pagerduty_webhook(payload):
    server.emit(Event(name="incident.created", data={...}))
```

SDKs SHOULD provide an in-memory ring buffer that retains a bounded window of emitted events per event type (configurable by time and/or count). `events/poll` for an emit-only event type reads from this buffer. The cursor is a buffer-local sequence number; it is process-scoped — a server restart invalidates all cursors and clients receive `CursorExpired`, resetting to "now." Events emitted during server downtime are not recoverable. This matches the upstream's own guarantees for push-only sources and keeps the server author's implementation to a single `emit()` call per upstream event.

```python
server = MCPServer(
    emit_buffer=EmitBuffer(
        max_age=timedelta(minutes=10),   # retain 10 min of events
        max_events=10_000,               # cap per event type
    ),
)
```

For servers wrapping upstreams that *do* offer a durable cursor (Gmail historyId, Kubernetes resourceVersion, Stripe `/v1/events`, Kafka offset), authors SHOULD prefer a `check()` function that queries the upstream directly, so the MCP cursor is the upstream cursor and poll survives server restart without gaps.

**Subscription lifecycle hooks.** For events where the upstream source must be configured per subscription (e.g., "watch this Slack channel"), the server needs to know when subscriptions are added and removed. The SDK provides lifecycle hooks:

```python
@server.on_subscribe("slack.message")
async def on_subscribe(context, params, subscription_id):
    """Set up upstream listener for this subscription's params."""
    await slack.join_channel(params["channel"])

@server.on_unsubscribe("slack.message")
async def on_unsubscribe(context, params, subscription_id):
    """Tear down upstream listener."""
    await slack.leave_channel(params["channel"])
```

These hooks are called by the SDK across all delivery modes. The server author writes the upstream setup/teardown logic once; the SDK handles the delivery mechanics.

**Unsubscribe timing by mode.** Push and webhook have explicit end-of-life signals — push fires `on_unsubscribe` when the stream closes, webhook when `events/unsubscribe` is called or the TTL lapses. Poll does not: the client simply stops calling `events/poll`, and the server never sees a goodbye. To prevent poll-provisioned upstream resources from leaking, the SDK treats poll subscriptions as leased. `on_subscribe` fires the first time a given `(principal, subscription id)` appears in a poll request; each subsequent poll for that id renews the lease; `on_unsubscribe` fires when the lease expires without renewal. The lease window is SDK-configurable and SHOULD default to a small multiple of the server's typical `nextPollSeconds` so that a well-behaved client never lapses between polls:

```python
server = MCPServer(
    poll_subscription_ttl=timedelta(minutes=5),
)
```

This lease tracking is soft state the SDK maintains on the server author's behalf. It does not survive restart — on restart the next poll is "first sight" again and `on_subscribe` re-fires, which is the correct behavior since the upstream listener also needs re-establishing. Server authors SHOULD write `on_subscribe` to be idempotent for this reason.

## Client SDK Guidance

Event names and params are not known at code time — they are discovered at runtime via `events/list`, typically driven by the LLM. The client SDK provides a dynamic subscription API:

```python
client = MCPClient(
    server_url="...",
    webhook=WebhookConfig(                              # optional, global
        url="https://proxy.example.com/hooks/client123",
        # secret is server-generated by default; pass secret=... to override
    ),
)

# Subscribe dynamically (e.g., driven by LLM decision after events/list)
sub = await client.subscribe(
    name="email.received",
    params={"from": "*@anthropic.com", "redact_pii": True},
)

# Receive events
async for event in sub:
    # Called regardless of delivery mode (poll, push, or webhook)
    details = await client.call_tool("get_email", {"id": event.data["messageId"]})
    # ... process the email

# Unsubscribe
await sub.cancel()
```

The SDK manages delivery mode selection, poll loops, push stream lifecycle, webhook refresh cycles, cursor tracking, deduplication by `eventId`, cursor expiry recovery, and reconnection — all transparently.

### Delivery Mode Selection

The SDK intersects the event type's advertised `delivery` list with the modes the client is configured to use, and picks in this preference order:

1. **Webhook** if the client has `WebhookConfig` set and the server lists it — low latency without tying up a persistent connection.
2. **Push** if the transport supports streaming (stdio, or HTTP when webhook isn't configured) and the server lists it — low latency, but requires holding a connection open.
3. **Poll** if the server lists it — no server state, no persistent connection.

If the intersection is empty, `client.subscribe()` raises `NoCompatibleDeliveryMode`. The protocol does not guarantee a universal fallback, so client SDKs MUST surface this as an error rather than silently degrading.

The caller can override per subscription:

```python
# Force poll
sub = await client.subscribe("email.received", params={...}, delivery="poll")

# Force push (error if server doesn't support it)
sub = await client.subscribe("email.received", params={...}, delivery="push")

# Force webhook (error if WebhookConfig not set or server doesn't support it)
sub = await client.subscribe("incident.created", params={...}, delivery="webhook")
```

### How Each Mode Works

**Poll mode:** The SDK runs a background task that calls `events/poll` at the server-recommended interval, batching all subscriptions to the same server into a single request. Events are yielded to the subscription's async iterator. The caller does not implement or manage the polling loop.

**Push mode:** The SDK sends an `events/stream` request and listens for notifications. When subscriptions are added or removed, the SDK cancels the current stream (via `notifications/cancelled` or connection close) and sends a new `events/stream` with the updated subscription list and current cursors. The transition gap is covered by cursor-based replay.

**Webhook mode:** The SDK calls `events/subscribe` to register the callback URL with the server, which returns the signing secret. It runs a background refresh loop, re-calling `events/subscribe` before `refreshBefore` expires; if a refresh response includes a `secret`, the SDK updates the verifier. The SDK monitors `deliveryStatus` on each refresh to detect delivery failures. The webhook receiver (which may be a forward proxy) accepts incoming POSTs, verifies HMAC signatures, checks timestamp freshness, and routes events to the appropriate subscription iterator. On `sub.cancel()`, the SDK calls `events/unsubscribe` for eager cleanup.

### Delivering Events to the LLM

When an event arrives (via any delivery mode), the client SDK surfaces it to the agent runtime. How this works depends on the agent framework — it might inject the event into the conversation context, trigger a new agent turn, or queue it for processing. The MCP spec does not prescribe this; it is an application-level concern.

## Security Considerations

### Event Payloads Are Untrusted Data

Event payloads MUST be treated with the same caution as tool results. They originate from external systems and may contain:

- Prompt injection attempts (e.g., an email subject line containing "ignore all previous instructions")
- Malformed or malicious data
- PII that should not be forwarded or logged

Clients SHOULD sanitize or sandbox event payloads before presenting them to an LLM. The spec should include guidance similar to the existing tool result security model.

### Payload Minimality

Servers SHOULD keep event payloads minimal — enough to identify and triage the event, not the full content. For example, an email event should include sender, subject, and messageId, but not the full email body. The client can use tools to fetch full content when needed.

This reduces PII exposure in event infrastructure (logs, queues, buffers) and limits the injection surface area.

### Authorization

**Subscribe-time:** The server MUST verify that the authenticated user has permission to subscribe to the requested event type with the given params. For example, a Slack server must verify the user has access to the channel specified in the params.

**Delivery-time:** The server SHOULD periodically re-verify permissions. If the user's access is revoked (e.g., removed from a Slack channel), the server notifies the client. For push mode, this is sent on the SSE stream:

```
data: {"jsonrpc":"2.0","method":"notifications/events/terminated","params":{"id":"sub_slack","reason":"Access revoked"}}
```

For poll mode, this appears as an error in the poll response for that subscription. The client SDK SHOULD remove the subscription and notify the application.

**Action-time:** Event receipt does NOT constitute authorization to act. The agent's response to an event (e.g., calling a tool) goes through normal MCP authorization. The spec should be explicit about this.

## Enterprise Governance

The spec does not mandate specific governance mechanisms but is designed to enable them:

- **Audit:** The client owns all subscription state. Enterprise agent runtimes can inspect the client SDK's subscription registry and log all event-triggered actions.
- **Policy:** Clients SHOULD support policy evaluation between event receipt and action execution. The policy language is an application concern, not a protocol concern.
- **Kill switch:** For poll mode, stop polling. For push mode, close the connection (HTTP) or send `notifications/cancelled` (stdio). For webhook mode, call `events/unsubscribe` for immediate termination, or simply stop refreshing and the subscription will expire at TTL.
- **Rate limiting:** `nextPollSeconds` allows servers to control polling frequency dynamically. Clients SHOULD respect it. Enterprise deployments may impose additional rate limits.

## Relationship to Existing Primitives

**Resources:** `notifications/resources/updated` remains unchanged. Events are a separate primitive for domain-specific occurrences that don't map to a specific resource. Servers MAY implement resource watching via events, but the two mechanisms are independent.

**Tools:** Events and tools compose at the application layer. An event arrives, the agent reasons about it, and may call tools in response. The MCP spec does not prescribe event-to-tool wiring.

**Prompts:** Future work may add event-bound prompts (prompt templates designed to be instantiated when a specific event fires). Not in scope for v1.

**Sampling:** Events may trigger LLM reasoning. The client decides whether to invoke sampling in response to an event — the server does not control this.

## What Is NOT in v1

- **Durable subscriptions with replay.** No mechanism to replay events from before subscription time. Cursors represent "now" forward.
- **Message queue bindings.** The protocol does not define bindings for specific message queue systems (SQS, MQTT, Kafka, Pub/Sub, etc.). For high-availability delivery through message queues, deploy an MCP proxy that receives events via poll, push, or webhook and writes to your organization's queue infrastructure. The client reads from the queue using the queue's native client. This keeps the protocol transport-agnostic while enabling any intermediary.
- **Rich query language for params.** Params are simple key-value. No CEL, JSONPath, or complex predicates. Servers define their own param semantics via `inputSchema`.
- **Cross-server event routing.** No mechanism for one server's events to trigger another server's tools. This is an application/orchestration concern.
- **Event-bound prompts.** Prompt templates that auto-instantiate on events. Deferred to a future version.
- **Guaranteed delivery.** All three modes provide at-least-once delivery when cursors are used correctly (the client replays from the last known cursor on reconnect/restart). Exactly-once requires application-level deduplication via `eventId`.

## Open Questions

1. **How should the client SDK expose events to agent frameworks?** As injected context? As a special message type? As a tool-call-like callback? This is an SDK design question, not a protocol question, but guidance would be useful.

2. **Should resource subscriptions and `list_changed` notifications be re-expressed as events?** The existing `resources/subscribe` → `notifications/resources/updated` flow and the `notifications/{tools,resources,prompts}/list_changed` family are effectively a special-cased push-only event channel. Folding them into this primitive (e.g., a reserved `mcp.resource.updated` event type whose params are `{uri}`, and `mcp.{tools,resources,prompts}.list_changed` event types) would give them cursors, poll/webhook delivery, and replay for free, and remove a parallel mechanism from the spec. The cost is a migration path for existing clients and the question of whether the protocol should reserve an `mcp.*` event-name prefix at all.

3. **How should servers publish their egress IP ranges for webhook delivery?** Enterprises deploying webhook mode will typically need to allowlist the source IPs that webhook requests originate from. Should the spec define a discovery mechanism (e.g., a well-known endpoint like `/.well-known/mcp-egress-ips`, or a field in the server's capabilities/metadata)? Or leave it to out-of-band documentation per server? A standard mechanism would simplify enterprise onboarding, but adds burden on server operators who may use dynamic egress (cloud NAT, serverless) where stable IP ranges are hard to guarantee.

4. **Should webhook delivery adopt the Standard Webhooks format instead of defining its own?** [Standard Webhooks](https://www.standardwebhooks.com/) specifies a delivery envelope very close to what this doc defines: `webhook-id`, `webhook-timestamp`, and `webhook-signature` headers, `HMAC-SHA256(secret, id + "." + timestamp + "." + body)` with a `whsec_` prefix on secrets, base64 `v1,<sig>` signature encoding with multi-signature support for rotation, and a versioned scheme that already accommodates asymmetric signing (`v1a,`). Adopting it would let MCP receivers reuse off-the-shelf verifiers (Svix and others ship libraries in most languages) and would resolve open question 5 below for free. The cost is losing the `X-MCP-*` header namespace and binding the spec to an external document we don't control. A middle path is to declare MCP webhook delivery a Standard Webhooks profile — same wire format, with MCP-specific body schema and the subscription `id` carried as `webhook-id`.

5. **Should webhook deliveries support multiple signatures for zero-downtime secret rotation?** The current design has a single `X-MCP-Signature` header, and rotation happens via an atomic upsert of `delivery.secret`. But there's a race: webhooks in flight when the upsert lands were signed with the old secret, and a receiver that has already switched to validating against the new secret will reject them. Stripe-style multi-signature (e.g., `X-MCP-Signature: t=<ts>,v1=<sig_old>,v1=<sig_new>`) lets the server sign with both secrets during a grace window so the receiver can verify against either. Is the in-flight window small enough to ignore, or should the spec allow `delivery.secret` to be an array (or require servers to dual-sign for N seconds after rotation)?

## Note on Consistency and Ordering

This design intentionally does not provide protocol-level guarantees around event ordering, exactly-once delivery, or transactional consistency. There are no logical clocks, sequence numbers, or cross-subscription ordering constraints.

The rationale is that the protocol connects to many different upstream systems (Gmail, Slack, PagerDuty, etc.), each with their own consistency models. Imposing a unified consistency model at the MCP layer would either be too weak to be useful or too strong to be implementable across diverse backends.

What the protocol *does* provide:

- **Cursors** for resumability — the client can pick up where it left off after a disconnect or crash.
- **`eventId`** for client-side deduplication — the client can detect and discard duplicates that arise during reconnection.
- **Per-subscription ordering** — for poll and push, events within a single subscription are delivered in the order the server produces them. For webhook, delivery order is best-effort — concurrent HTTP requests may arrive out of order, and retries can reorder deliveries. Clients that need ordering can use the `timestamp` field in event payloads. No ordering is guaranteed across subscriptions in any mode.

Servers that need stronger guarantees can implement them. For example, a server wrapping a Kafka topic could expose partition offsets as cursors and provide exactly-once semantics within a partition. A database change-data-capture server could use LSNs as cursors and guarantee causal ordering. The protocol's cursor mechanism is flexible enough to support these — the cursor is opaque, so it can encode whatever the server needs (sequence numbers, timestamps, composite positions).

The client and agent layer should be designed to tolerate out-of-order and duplicate events. For most AI agent use cases — reacting to new emails, triaging alerts, responding to messages — eventual consistency with deduplication is sufficient. Agents that need stronger guarantees should use tools to read authoritative state rather than relying solely on event payloads.

## Note on Flow Control

This design does not include a protocol-level flow control mechanism (e.g., credit-based or token-based backpressure). This is intentional for v1.

In practice, the dominant bottleneck in an MCP event pipeline is LLM inference — processing a single event may take seconds to minutes of model time. Compared to this, event delivery rates from typical upstream sources (email, Slack, PagerDuty) are negligible. The system is inherently consumer-bound, not producer-bound.

What exists today is sufficient:

- **Transport-level backpressure.** TCP flow control naturally throttles the server when the client stops reading from the connection. On stdio, OS pipe buffers provide the same effect.
- **Server-controlled push frequency.** For push mode, the server controls how often it checks upstream sources and delivers events. For poll mode, `nextPollSeconds` lets the server throttle the client explicitly.
- **Client-side prioritization.** When multiple events arrive while the LLM is busy, the client SDK can prioritize by event type, severity, or recency. This is an SDK concern, not a protocol concern.

Head-of-line blocking — where a slow-to-process event delays delivery of subsequent events — is likewise handled at the client SDK layer. The SDK can maintain per-subscription queues and let the agent framework process events concurrently or by priority, rather than strictly in arrival order.

If future use cases involve high-throughput event streams where producer-side backpressure becomes necessary, a credit-based flow control extension can be added without breaking the existing protocol.
