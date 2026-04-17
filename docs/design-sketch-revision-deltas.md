# Design Sketch Revision — Implementation Deltas

This document lists every change between the previous version of `design-sketch-proposal.md` (commit `4f8a112`) and the current revision, written for an implementation that was built against the previous version. Each delta states what changed, why, and what code needs to change.

Read alongside `git diff 4f8a112 -- docs/design-sketch-proposal.md` for exact wording.

---

## 1. Webhook cursor model — server no longer tracks a watermark **(breaking)**

**Was:** Server stored a per-subscription cursor and advanced it on each `2xx` ack. `events/subscribe` response included `"cursor"`. Refresh with `cursor: null` meant "keep the server's current position."

**Now:** Cursor is **client-owned**. Server does not track a delivery position at all.
- `events/subscribe` **request** still accepts `cursor` (`null` = start from now; non-null = replay from there if upstream is durable).
- `events/subscribe` **response** no longer contains `cursor`. Remove the field from `SubscribeEventsResult`.
- Webhook delivery **payload** body still carries `cursor` (position after this event). Unchanged on the wire, but now the *only* place the client gets a cursor.
- Server retries each event independently with exponential backoff on non-`2xx`. No watermark advancement, no head-of-line blocking logic.
- Refresh with `cursor: null` now means "leave delivery uninterrupted" (no server position to keep).

**Why:** The previous model was unsound under out-of-order acks (event B acks while A retries → cursor=5 but A undelivered → gap on restart). Per-event retry without a server watermark matches Stripe/GitHub/Shopify/Standard Webhooks.

**Code changes:**
- Delete server-side per-subscription `cursor` field and all "advance on 2xx" logic.
- Remove `cursor` from the subscribe response struct and both response examples.
- Remove `cursor` from `deliveryStatus`.
- Delivery worker: drop any serialization/ordering on cursor; fire each event with its own retry schedule.
- Client SDK: persist `cursor` from each delivered payload; pass it back on resubscribe.

---

## 2. Webhook subscription key — `delivery.url` is always part of the key **(breaking)**

**Was:** Authenticated scope keyed on `(principal, id)` with `delivery.url` mutable on refresh. Unauthenticated scope keyed on `(delivery.url, id)`.

**Now:** Both scopes include `delivery.url` in the key:
- Authenticated: `(principal, delivery.url, id)`
- Unauthenticated: `(delivery.url, id)`

`delivery.url` is therefore **immutable** for a subscription's lifetime in all scopes. To change endpoint: `events/unsubscribe` then `events/subscribe`.

**Why:** Previously, the same client code (refresh with new URL) mutated in-place when authenticated but silently forked a second subscription when unauthenticated. Making URL always part of the key gives consistent behavior.

**Code changes:**
- Change the subscription map key type to include `delivery.url` in the authenticated path.
- Remove the "update URL in place on refresh" branch.
- Update the Mutable Fields handling: a refresh with a different URL now addresses a *different* subscription (i.e., creates one).

---

## 3. `events/stream` no longer replaces the GET SSE stream **(breaking if you removed GET SSE)**

**Was:** `events/stream` was "the sole server→client push channel," replacing the transport's GET-based SSE stream. Non-event notifications (`tools/list_changed`, progress, logging) rode on `events/stream`. Clients could open it with empty `subscriptions` just to receive those.

**Now:** `events/stream` carries **only** `notifications/events/*`. The existing GET SSE stream is retained unchanged for all non-event server-initiated notifications. The "empty subscriptions" use-case is removed.

**Why:** The previous framing was a sweeping base-transport change buried in a sub-bullet. Walking it back keeps this proposal additive.

**Code changes:**
- If you removed/redirected the GET SSE endpoint: restore it.
- Stop routing `tools/list_changed`, `resources/updated`, progress, and logging notifications onto `events/stream` responses — route them as before.
- Reject or no-op `events/stream` with an empty `subscriptions` array (it has no purpose now); or accept it and just heartbeat.

---

## 4. Poll lease key — now `(principal, eventName, canonicalHash(params))` **(breaking for SDK internals)**

**Was:** Lease table keyed on `(principal, subscription id)` from the poll request.

**Now:** Keyed on `(principal, eventName, canonicalHash(params))`. The request `id` is **not** used — it remains opaque and request-scoped per the wire contract.

**Why:** The `id` is only required to be unique within a single poll request. A compliant client can reuse `id: "1"` for different subscriptions across requests, which would corrupt an id-keyed lease table.

**Code changes:**
- Change the lease map key. Implement `canonicalHash(params)` (stable JSON canonicalization → SHA-256, or equivalent).
- `on_subscribe` fires on first appearance of the new key; renewal and expiry logic unchanged otherwise.
- Add a note in your SDK that lease state is ephemeral: never persisted; restart re-fires `on_subscribe` on next poll.

---

## 5. Broadcast emit — requires author-supplied `match()` and `transform()` hooks **(breaking for SDK API)**

**Was:** "The SDK matches the event against all active subscriptions' params."

**Now:** Event registration takes two optional callbacks:
- `match(event, params) -> bool` — decides whether a subscription receives the event. If absent, all subscriptions for that event name receive it.
- `transform(event, params) -> event` — shapes the payload per subscription (e.g., apply `redact_pii`, honour an `expand` param). If absent, the event is delivered as emitted.

On broadcast emit, the SDK iterates active subscriptions for that event name, calls `match`, and on `True` delivers `transform(event, params)`.

**Why:** Params are author-defined (globs, transforms, dedup windows). The SDK cannot generically evaluate or apply them.

**Code changes:**
- Add `match: Callable[[Event, dict], bool] | None` and `transform: Callable[[Event, dict], Event] | None` to the event registration API.
- Broadcast fan-out: `if match is None or match(event, sub.params): deliver(sub, transform(event, sub.params) if transform else event)`.

---

## 6. DNS rebinding mitigation — validate at delivery time, not subscribe time **(security)**

**Was:** Resolve hostname and validate IP once, at subscribe time.

**Now:** On **every delivery**: resolve hostname → validate resolved IP against private/loopback blocklist → connect directly to that validated IP, sending the original hostname in `Host` header / TLS SNI. Subscribe-time validation MAY remain as an early-reject convenience but is not the security boundary.

**Why:** Subscribe-time-only resolution is exactly what DNS rebinding defeats (public IP at validate time, private IP at delivery time).

**Code changes:**
- Move IP validation into the delivery path.
- Use a resolve-then-connect-to-IP pattern (most HTTP clients support a custom resolver or socket-level connect with `Host`/SNI override).

---

## 7. `X-MCP-Timestamp` header — SHOULD → MUST

**Was:** Header was SHOULD; signature formula required it anyway.

**Now:** Header is MUST. Signature formula unchanged: `HMAC-SHA256(secret, timestamp + "." + body)`.

**Code changes:**
- Always send the header. Receiver: reject deliveries missing it.

---

## 8. `StreamEventsResult` — only sent when the server can write a final frame

**Was:** "Server MUST send `StreamEventsResult`" on cancellation, unconditionally.

**Now:**
- stdio: server MUST send `StreamEventsResult` when the stream ends (either side).
- Streamable HTTP, **server-initiated** close: send `StreamEventsResult` as the final SSE `data:` frame.
- Streamable HTTP, **client-initiated** close (connection drop): no result is sent (physically can't).

**Code changes:**
- HTTP handler: on client disconnect, just clean up — don't attempt to write the result. On server-initiated shutdown, write result then close.
- stdio handler: unchanged.

---

## 9. Webhook `CursorExpired` and `terminated` — now POSTed to the endpoint

**Was:** These error paths were specified for poll and push only; webhook was silent.

**Now:** When a webhook subscription hits `CursorExpired` (`-32014`) or is terminated (e.g., `-32012 Unauthorized`), the server POSTs a signed error envelope to the callback URL:

```
POST <delivery.url>
X-MCP-Signature: sha256=<...>
X-MCP-Timestamp: <unix>

{"id":"<subscription id>","error":{"code":-32014,"message":"CursorExpired","data":{"reason":"..."}}}
```

It also surfaces the error in `deliveryStatus.lastError` on the next refresh response.

**Code changes:**
- Add an error-delivery path that builds this envelope, signs it like a normal delivery, and POSTs it.
- Endpoint/receiver: handle a body with top-level `error` instead of `data`.

---

## 10. Endpoint forwarding requirement — `cursor` and `eventId` MUST reach the client

New normative requirement on the webhook **endpoint** (the receiver you POST to): it MUST make `cursor` and `eventId` available to the consuming client by whatever channel it forwards events.

**Code changes (endpoint/relay implementations):** ensure both fields are passed through, not dropped.

---

## 11. "No subscription state" wording softened — no behavior change

The doc now says poll holds "no **protocol-required** subscription state" and explicitly permits the SDK to hold ephemeral derived state (lease table, emit ring buffer). If your SDK already holds these, you're now explicitly compliant. **No code change** — documentation alignment only.

---

## 12. At-least-once guarantee qualified — no behavior change

The guarantee now reads: at-least-once **when the cursor is backed by a durable upstream**; emit-only event types are **at-most-once across server restarts**. If you advertised unconditional at-least-once anywhere (docs, capability metadata), qualify it. **No runtime code change.**

---

## 13. Push-mode reconnect backlog — acknowledged as a v1 gap

Flow Control section now notes that push reconnect with a stale cursor can dump a large backlog with no protocol-level bound (poll has `maxEvents`/`hasMore`; push does not). Left to TCP backpressure / server-side pacing for v1.

**Code changes:** none required. Optionally, add server-side pacing on backlog replay.

---

## 14. `eventId` is server-assignable from upstream identifiers **(SDK API)**

**Was:** Unstated; SDKs typically auto-generated `eventId`.

**Now:** Spec clarifies `eventId` SHOULD be the upstream's stable event identifier (Stripe `evt_*`, GitHub delivery GUID, Kafka offset, Gmail message ID) when one exists. SDK auto-generates only when the author supplies none.

**Why:** Dual-path sources (e.g., Stripe webhook emit + poll backfill) otherwise surface the same upstream event with two different SDK-generated IDs, breaking client dedup.

**Code changes:**
- `Event` constructor / `emit()` / `check()` return: accept an optional `eventId` from the author. Only generate a UUID when it's omitted.

---

## 15. Wire-format corrections **(breaking on the wire)**

| Item | Was | Now | Action |
|---|---|---|---|
| Event notification method | `notifications/event` (in one example) | `notifications/events/event` everywhere | Rename the method constant if you copied from the JSON example |
| `nextPollSeconds` placement | Top-level in one diagram | Per-result entry only | Ensure your `PollEventsResult` has it inside each `results[i]`, not at top level |
| Error codes | `-32001..-32006` | `-32011..-32016` (shifted to avoid collision with base MCP `-32002 ResourceNotFound`); `CursorExpired` (`-32014`) now listed in the table | Renumber your error-code enum: `EventNotFound -32011`, `Unauthorized -32012`, `TooManySubscriptions -32013`, `CursorExpired -32014`, `InvalidCallbackUrl -32015`, `SubscriptionNotFound -32016` |
| `notifications/events/heartbeat` | Named, shape undefined | `{"jsonrpc":"2.0","method":"notifications/events/heartbeat","params":{}}` | Emit with empty `params` object |

---

## 16. `events/unsubscribe` requires `delivery.url` **(breaking on the wire)**

Consequence of #2. `delivery.url` is now part of the subscription key in all scopes, so `events/unsubscribe` MUST include it alongside `id`. Previously it was optional for authenticated callers.

**Code changes:** Add `delivery: {url}` to the unsubscribe request schema (required). Server forms the full key `(principal?, delivery.url, id)`. If you keep a Redis-style store, key it on `sub:{principal}:{urlHash}:{id}`.

---

## 17. Webhook `name`/`params` are immutable **(breaking)**

`name` and `params` join `delivery.url` as immutable identity fields. A refresh that supplies different values addresses a different subscription (treated as create). To change what a subscription listens for: unsubscribe + resubscribe.

**Why:** Mutating them in place left the upstream listener (provisioned by `on_subscribe`) bound to old params, silently delivering nothing.

**Code changes:** Remove the "replace `name`/`params` on existing key" branch. Only `delivery.secret`, `cursor`, TTL, and `active` are mutable on refresh.

---

## 18. Webhook refresh behavior — always pass cursor; refresh reactivates delivery

Two clarifications to #1:
- Clients pass their **last-persisted cursor on every refresh** (not `null`). Server is idempotent: if the sub is live and the cursor is at/behind in-flight position, no-op; if lapsed/restarted, replay from there.
- A successful refresh **sets `active: true`**. If delivery was suspended after repeated failures, refresh resumes it.

**Code changes:** Client SDK refresh loop sends persisted cursor, not `null`. Server: on refresh, clear the suspended flag and resume retry of pending events.

---

## 19. Unified error/terminated shape across modes **(wire change)**

`notifications/events/error` (push) now nests under `params.error` instead of flat `params.code`/`params.message`:
```
{"method":"notifications/events/error","params":{"id":"...","error":{"code":-32011,"message":"..."}}}
```
`notifications/events/terminated` is now defined with the same shape (`{id, error: {code, message, data?}}`). Poll error entries and webhook error POSTs already use this nested shape.

**Code changes:** Update push error notification builder. Add `notifications/events/terminated` to your push message types.

---

## 20. Webhook delivery headers + signature encoding **(wire change)**

- Add `X-MCP-Subscription-Id: <id>` header so receivers select the secret before parsing the body.
- Signature header is exactly `X-MCP-Signature: sha256=<hex-lowercase>`.
- Each retry regenerates `X-MCP-Timestamp` and the signature.

**Code changes:** Add the header; lowercase-hex encode the HMAC; move timestamp+sign into the per-attempt path.

---

## 21. SSRF hardening — `fe80::/10` + no redirects **(security)**

Extends #6. Add `fe80::/10` (IPv6 link-local) to the blocklist. Webhook delivery requests MUST NOT follow HTTP redirects.

---

## 22. `match`/`transform` signature + scope **(SDK API change)**

Extends #5. Signatures are now `match(ctx, event, params)` / `transform(ctx, event, params)` (`ctx` carries principal + request metadata). The SDK applies these hooks not only on broadcast emit but also when an `events/poll` request reads from the emit-only ring buffer.

---

## 23. Capability object — `listChanged` instead of `subscribe` **(wire change)**

Events capability is `{"events": {"listChanged": true}}`. Delivery modes are advertised per-event in `deliveryModes`, not at the capability level.

---

## 24. Cancellation wording

HTTP cancellation = abort request stream (TCP close on HTTP/1.1, `RST_STREAM` on HTTP/2). stdio `StreamEventsResult` after cancel is now SHOULD (was MUST in #8's first pass) to align with base MCP's SHOULD-NOT-respond-after-cancel.

---

## Quick triage

If you need to prioritize:

1. **Security:** #6 (DNS rebinding), #7 (timestamp MUST), #21 (`fe80::/10`, no redirects)
2. **Wire-breaking:** #1, #2, #3, #15, #16 (unsubscribe needs url), #17 (name/params immutable), #19 (error shape), #20 (headers/encoding), #23 (capability)
3. **SDK-internal breaking:** #4 (lease key), #5/#22 (`match`/`transform` hooks + ctx), #14 (`eventId` passthrough), #18 (refresh cursor/reactivate)
4. **Additive:** #8, #9, #10
5. **Docs-only/minor:** #11, #12, #13, #24
