# SEP-XXXX: Resource Events

- **Status**: Draft
- **Type**: Extensions Track
- **Extension Identifier**: `io.modelcontextprotocol/resource-events`
- **Created**: 2026-10-31
- **Author(s)**: Larry Maccherone (@lmaccherone)
- **Sponsor**: None (seeking sponsor)
- **PR**: TBD

## Abstract

This extension lets an MCP client keep copies of resources current. A server records each change to its resources at a moment on a timeline, a string that sorts after every moment before it, such as `"108"`. For each change, it sends `mcp.resource.collection.updated`, an event of SEP-3415's Events extension, to each subscription covering a resource the change touched. The event names the collection the subscription watches and each resource in it that changed, with its new revision, and carries none of their contents.

A client that already holds each named revision, such as the client that made the change, skips the event. Any other catches up with `resources/collection/catchup`, which brings every change to the collection after the moment the client is caught up to, each as contents or a patch. 

A new client first reads the collection with `resources/collection/read`. Writes use the Filesystems working group's strawman methods, and a resource's revision is the moment it last changed.

An event carries no state, so a lost, repeated or late one does no harm: the next catch-up brings whatever it missed. A working prototype builds all of it, as § *Reference Implementation* describes.

While the prototype and § *Motivation* mention webhooks explicitly, nothing in this proposal prevents its event from being delivered over an SSE stream, by SEP-3415's push delivery. Further, the proposal and the prototype both support a client that polls, with no stream at all (§ *Use cases*).

We bring this proposal to the Tasks and Events working group for discussion, and are looking for a sponsor.

## References

- **SEP-3415, Events,** [modelcontextprotocol#3415](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/3415), a draft, by reference at commit `b08e4f9`. This extension takes its subscriptions and delivery as they stand, and sends an event of its own, `mcp.resource.collection.updated` rather than the `mcp.resource.updated` its Open Question 2 sketches.
- **The Filesystems strawman,** [experimental-ext-filesystems#2](https://github.com/modelcontextprotocol/experimental-ext-filesystems/pull/2), the Filesystems working group's draft of resource operations, at commit `db67f78`, which the rest of this proposal calls "the strawman". It adds `resources/stat`, `resources/create`, `resources/update` and `resources/delete`. Each resource has a `revision`, a write names the one it expects as `ifMatch`, and a write's result holds a change-set entry for each resource it changed, such as `{uri, effect: "updated", previousRevision, revision}`. This extension takes its `revision` and its change-set entry, and the prototype builds its four methods.
- **The Filesystems problem statements,** [experimental-ext-filesystems](https://github.com/modelcontextprotocol/experimental-ext-filesystems/blob/main/docs/problem-statements.md), that working group's framing of its work. Problem Statement 6 asks a proposal to "Decide whether a client can subscribe to a directory", and asks "Does a client need ordering between its write response and the resulting notification?" A collection answers the first (R24), and revisions the second, since a client orders messages by revision rather than arrival (R15). This extension offers both for that group to weigh.
- **MCP 2026-07-28,** [the specification](https://modelcontextprotocol.io/specification/2026-07-28), the revision this extension builds on, for `resources/read`, pagination and `CacheableResult`.
- **SEP-2640, Skills over MCP,** [the extension](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/seps/2640-skills-extension.md), Final, which serves a skill's files as resources, such as `skill://pdf-processing/scripts/extract.py`. A client that loaded a skill can watch its files with this extension, as § *Use cases* shows.

## Motivation

**MCP clients want an *efficient* way to keep local copies of resources up to date in near *real time*.** A webhook delivers each change without the client holding a connection open.

Keeping a copy current needs only the newest value, though, not every event on the way to it, so it calls for a different contract than a stream of events, as § *SEP-3415's open question 2* argues. It differs from keeping task state in sync too, as § *Why is this different from task state sync?* says.

A client watching resources today has four problems:

1. **It must hold a stream open or poll.**
   - **Streaming.** Since [SEP-2575](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2575), `notifications/resources/updated` arrives only on a `subscriptions/listen` stream, which load-balanced, edge and serverless infrastructure handles poorly.
   - **Polling.** The polling interval is an uncomfortable trade between ***efficiency*** and ***near real time***.
2. **It can't tell whether it already holds the latest value.** The notification names only the URI, so each one costs a read, even for the client that made the change.
3. **Catching up after a disconnect is inefficient.** Since 2026-07-28 a broken stream can't resume, so a client back from a disconnect must read every resource it watches again.
4. **It can't subscribe to a collection.** `subscriptions/listen` takes resource URIs and no templates, so a client subscribes to every uri individually, and hears of a new one, if at all, only through `notifications/resources/list_changed`, which itself names no resource.

### SEP-3415's open question 2

SEP-3415's Open Question 2 sketches a minimal resource subscription over Events, with "a reserved `mcp.resource.updated` event type whose arguments are `{uri}`", re-expressing core's `notifications/resources/updated`, which names one changed resource. This extension leaves that event to Open Question 2, and sends its own, `mcp.resource.collection.updated`, naming a collection and every resource in it that one change touched (D19).

**However, a resource is state, and a watcher cares only about the latest version, not every change.** A receiver that was down while a ticket was edited, reopened and closed needs only the closed ticket. On SEP-3415's thread on 2026-10-16, Peter Alexander (@pja-ant) [wrote](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/3415#issuecomment-6215654779) that watching state and consuming messages "will need different handling and different APIs". The two "should likely share a bunch of the same machinery", he went on, but whether they should share an API "requires some thought and discussion". Here the shared machinery is the event, which carries only which resources changed and at which moment, and the different API is the catch-up, which brings the newest values. SEP-3415 itself points this way: servers "SHOULD keep event payloads minimal", agents that need stronger guarantees should "use tools to read authoritative state", and servers "MAY implement resource watching through events".

### Why is this different from task state sync?

**Task state is closer to a resource than to a stream of messages, but it is still different.** Peter put the two on the same side: "this is very much the same thing (observing task state vs stream of messages)". A task's state, though, is a small record the Tasks extension defines, so its newest value can go whole in the event itself, as SEP-3415's Open Question 4 sketches with an `mcp.task.updated` event carrying `{status, progress?, result?}`. Resources come in collections, which a client watches and catches up on as one. So this proposal starts with resources, and tasks could reuse what it builds. This work also began before the Tasks extension joined this working group, on 2026-10-16.

### Why now?

**MCP is getting webhooks.** SEP-3415's webhook delivery is the catalyst for this work, and this extension takes it as it stands. Clients accept these webhooks already, while SEP-3415 is still a draft: [ChatGPT](https://developers.openai.com/plugins/build/mcp-events) receives Events by webhook on protocol `2026-07-28`, as do the MCP packages of Vercel's AI SDK, Mastra and TanStack AI, in versions released in October 2026, and OpenClaw bundles an opt-in `mcp-events` plugin.

**MCP now serves skills as resources.** [Skills over MCP](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/seps/2640-skills-extension.md), SEP-2640, is Final, so every client that adopts it reads resources, and may use a skill only while its files match the manifest it approved. Its group chose resources in part for "a subscription model for updates", as its [decision log](https://github.com/modelcontextprotocol/ext-skills/blob/main/docs/decisions.md) records, which since 2026-07-28 reaches a client only over an open `subscriptions/listen` stream.

### What the working group's charter asks

The Tasks and Events working group's [charter](https://modelcontextprotocol.io/community/triggers-events/charter) asks for a callback mechanism "with defined ordering guarantees that hold across all transports". SEP-3415 guarantees no order across subscriptions, and says "Servers that need stronger guarantees can implement them". A timeline is such a guarantee: it orders a server's changes, which commit in the order of their moments (R5), rather than its events, which still arrive in any order (R15), and it holds whichever transport prompts a catch-up.

### Use cases

- **A client behind a gateway.** A client on a laptop or in a browser, with no URL a server can reach, names its gateway's URL, as in SEP-3415's forward-proxy deployment.
- **A collection of documents.** A client subscribes to `https://notes.example/notes/{id}`, hears of each note created, edited or deleted, and catches up after a night away.
- **A client that can't take webhooks.** A command-line client polls `resources/collection/catchup` for `https://notes.example/notes/{id}` as often as each result's `nextPollMs` asks: one request for the whole collection, and an empty result when nothing changed.
- **A skill served over MCP.** A client that loaded `skill://code-review/SKILL.md` subscribes to `skill://code-review/{+path}`, hears of each changed file, and can ask for approval again before the model next uses the skill.

§ *Specification* answers the four problems. § *Rationale* says why, against the alternatives.

## Specification

**A server records each change to its resources at a moment on a timeline.** A moment is a string that sorts after every moment before it, such as a counter zero-padded to a fixed width, so that `"042"` sorts before `"107"`, where `"42"` would not. The `tickets` timeline in the examples below begins like this:

| moment | change |
|---|---|
| `"101"` | ticket-3 opened |
| `"107"` | ticket-7 closed as a duplicate of ticket-9, both changed in one atomic step |
| `"108"` | ticket-3 closed |

The server keeps the latest moment and mints each new one later (R5). An event names each resource that changed and its new revision, so a client that already holds that revision skips it, and a catch-up brings every change after the moment the client is caught up to. A database's read replica follows its primary's oplog the same way, and wherever it stops, it holds the data as of that point, which may be behind the primary. A catch-up differs in sending each resource once, as its contents or a patch from an earlier revision, rather than every operation, but the idea is the same.

Rules are numbered R1 and up across the sections below. Each keeps its ID when others are added or removed, so the IDs follow the order the rules were raised, not the order shown, and a gap is from a rule that was dropped, or that binds only how a server is built. The key words MUST, MUST NOT, SHOULD and MAY are to be read as BCP 14 ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) defines them when, and only when, they appear in capitals.

### Definitions

| Term | Meaning | Example |
|---|---|---|
| server | An MCP server as its clients see it, at one endpoint, however many processes, replicas or database nodes serve it. | replicas sharing one database |
| client | An MCP client, together with the application it runs in, which reads, subscribes, catches up and keeps copies. MCP's architecture calls that application the host, and this proposal says client throughout. | X and Y below |
| uri-specification | A uri, a uriTemplate, or later a uriGlob (§ *uriGlob, later*). A server lists a uri in `resources/list` and a uriTemplate in `resources/templates/list`. | `tickets/{id}` |
| collection | The resources a uri-specification covers. A uri on its own is a collection of one. | every ticket `tickets/{id}` covers, or `tickets/ticket-3` alone |
| timeline | The server's ordered record of the changes to a set of resources. | `tickets` below |
| moment | The point on a timeline at which a change occurred, a string that sorts after every moment before it. | `"108"` |
| latest moment | A timeline's most recent moment. A server stores it, and mints each new moment later than it (R5). | `"107"`, for `tickets` below |
| revision | A moment at which a resource changed, which the Filesystems strawman's fields `revision` and `previousRevision` hold. A resource's revision, with nothing more said, is its latest, as `resources/stat` and `resources/read` report it (R7). | ticket-3's `"101"`, and `"108"` from step 4 |
| tombstone | What a deleted resource leaves: its uri and the moment it was deleted. | ticket-3, deleted at `"111"` |
| copy | The contents a client last received or wrote for a resource, with the resource's revision then. | ticket-3's JSON at `"101"` |
| catch-up | A `resources/collection/catchup` call and its result: every change to a uri-specification after the moment the client is caught up to, up to the result's `nextFrom`. A client is caught up, on each uri-specification, to the last `nextFrom` it applied in full for it, and holds every change to it at or before that. | steps 3 and 6 below |

### A simple happy path

Server A lists the uriTemplate `tickets/{id}` in `resources/templates/list`, whose timeline's latest moment is `"107"`: ticket-3 last changed at `"101"`, and ticket-7 and ticket-9 at `"107"`. Two clients, X and Y, hold no copies yet, and each does steps 1 to 3. Each table shows what the server and each client hold after its step, in bold where the step changed it. The examples leave out JSON-RPC envelopes and core's result fields.

#### Step 1, at `"107"`: X and Y read the collection

Each client reads the collection, which brings every ticket that exists.

```jsonc
{"method": "resources/collection/read", "params": {"uriTemplate": "tickets/{id}"}}
{"contents": [
  {"uri": "tickets/ticket-3", "mimeType": "application/json",
   "text": "{\"title\":\"Printer jams\",\"status\":\"open\"}",
   "_meta": {"io.modelcontextprotocol/resource-revision": "101"}}
  // ...and ticket-7 and ticket-9, each at "107"
 ], "nextFrom": "107"}
```

Each client now holds every ticket, and is caught up to `"107"`, the read's `nextFrom`.

| | Server A | Client X | Client Y |
|---|---|---|---|
| latest moment | `"107"` | | |
| caught up to | | **`"107"`** | **`"107"`** |
| ticket-3 | `"101"`, open | **`"101"`, open** | **`"101"`, open** |
| ticket-7 and ticket-9 | `"107"` | **`"107"`** | **`"107"`** |

- **R22.** `resources/collection/read` names a uri-specification as a catch-up does, in its params and in `Mcp-Name`, gets the same errors for it, and is authorized the same way (R13), but takes no `from`. It returns the contents of each resource in the collection, each item as `resources/read` returns it, with its revision in `_meta` (R7), and `nextFrom`, the timeline's latest moment. Each resource MUST be as it stood at `nextFrom` or later. The client catches up from `nextFrom` afterwards, and a long read comes in pages (R26). Its result is a core `CacheableResult`.
  - A uri on its own is a collection of one, and a client watching one SHOULD read it with `resources/collection/read` too, rather than `resources/read`, since only a collection read brings a `nextFrom` to catch up from. A copy's revision can be older than the history the server keeps, so a catch-up from it could get `momentExpired` (R21), while `nextFrom` is the timeline's latest moment.

#### Step 2, at `"107"`: X and Y subscribe

Each client subscribes through SEP-3415, naming the uri-specification as the event's `arguments`.

```jsonc
{"method": "events/subscribe", "params": {
  "name": "mcp.resource.collection.updated",
  "arguments": {"uriTemplate": "tickets/{id}"},
  "delivery": {"mode": "webhook", "url": "https://gateway.example/hooks/x",
               "secret": "whsec_..."}}}
```

A now holds a subscription for each client, and nothing else changes.

- **R18.** A client subscribes with `events/subscribe`, naming `mcp.resource.collection.updated` and a uri-specification as `arguments`: `{uri}`, `{uriTemplate}`, or later `{uriGlob}`, and refreshes the subscription as SEP-3415 says. A client new to the uri-specification SHOULD first read its collection (R22). Once subscribed, it MUST catch up, so no change falls between the read and the subscription.

#### Step 3, at `"107"`: X and Y catch up

Each client catches up from `"107"`, since a change made after its read and before its subscription took hold sends it no event. Here there is none, so the result is empty and nothing changes. Had a ticket changed in that gap, it would come here, as ticket-3's change does in step 6.

```jsonc
{"method": "resources/collection/catchup", "params": {"uriTemplate": "tickets/{id}",
  "from": "107", "patchMimeTypes": ["application/merge-patch+json"]}}
{"changes": [], "hasMore": false, "nextFrom": "107", "nextPollMs": 30000}
```

#### Step 4, at `"108"`: Y closes ticket-3

Y closes ticket-3 with the strawman's `resources/update`, sending the whole closed ticket, and its copy's revision as `ifMatch` (R7).

```jsonc
{"method": "resources/update", "params": {"uri": "tickets/ticket-3",
  "ifMatch": "101", "contents": {...}}}
{"changes": [{"uri": "tickets/ticket-3", "effect": "updated",
              "previousRevision": "101", "revision": "108"}]}
```

Y's copy of ticket-3 is at `"108"`, but Y stays caught up to `"107"`, since others' changes could have landed before `"108"` (R9).

| | Server A | Client X | Client Y |
|---|---|---|---|
| latest moment | **`"108"`** | | |
| caught up to | | `"107"` | `"107"` |
| ticket-3 | **`"108"`, closed** | `"101"`, open | **`"108"`, closed** |
| ticket-7 and ticket-9 | `"107"` | `"107"` | `"107"` |

- **R17.** A server MUST store a create's or an update's contents as sent, so the writer's copy is the resource at the reply's `revision`.

#### Step 5, at `"108"`: A sends the event

A POSTs the event to each subscription's webhook. It names the subscription's uri-specification and the change, and carries none of the ticket's contents, so no copy changes.

```jsonc
{"eventId": "108", "name": "mcp.resource.collection.updated",
 "timestamp": "2026-11-03T16:00:00.000Z",
 "data": {"uriTemplate": "tickets/{id}", "changes": [
   {"uri": "tickets/ticket-3", "effect": "updated",
    "previousRevision": "101", "revision": "108"}]}}
```

- **R10.** Once a change is visible to a catch-up, the server MUST send one `mcp.resource.collection.updated` to each subscription covering any resource it touched. Its `data` repeats the subscription's `arguments`, naming its uri-specification, and holds `changes`, the strawman's change-set entry for each covered resource the change touched, `revision` always present. When a change's entries would take the body past the 256 KiB SEP-3415 asks for, the server SHOULD leave `changes` out. Its `eventId` SHOULD be the moment of its change, and SEP-3415's `cursor` MUST be null. A server MAY coalesce undelivered events for one subscription into the newest, whose `changes` then hold each uri once, as a catch-up's do (R19).

#### Step 6, at `"108"`: X catches up

Y already holds ticket-3, the event's one change, at `"108"`, so it skips the catch-up. X holds it at `"101"`, so X catches up from `"107"`.

```jsonc
{"method": "resources/collection/catchup", "params": {"uriTemplate": "tickets/{id}",
  "from": "107", "patchMimeTypes": ["application/merge-patch+json"]}}
{"changes": [
  {"uri": "tickets/ticket-3", "effect": "updated",
   "previousRevision": "101", "revision": "108",
   "patchMimeType": "application/merge-patch+json",
   "patch": {"status": "closed"}}],
 "hasMore": false, "nextFrom": "108", "nextPollMs": 30000}
```

X's copy is at `"101"`, the patch's `previousRevision`, so X applies it (R19), and is caught up to `"108"`. Y stays caught up to `"107"` until its next catch-up.

| | Server A | Client X | Client Y |
|---|---|---|---|
| latest moment | `"108"` | | |
| caught up to | | **`"108"`** | `"107"` |
| ticket-3 | `"108"`, closed | **`"108"`, closed** | `"108"`, closed |
| ticket-7 and ticket-9 | `"107"` | `"107"` | `"107"` |

- **R11.** When a client receives `mcp.resource.collection.updated`, it MUST catch up on the uri-specification the event names, unless it already holds each uri in `changes`, or its tombstone, at that entry's `revision` or later, which lets the writer skip its own change. An event with no `changes` always brings a catch-up.
- **R13.** A catch-up names a uri-specification, as `uri` or `uriTemplate`, the moment the client is caught up to as `from`, and any patch formats it applies as `patchMimeTypes`. Over Streamable HTTP, its request MUST carry `Mcp-Name` holding that `uri` or `uriTemplate`, as `resources/read`'s carries its `uri`. It stores nothing and needs no subscription, and the server MUST authorize it as a `resources/read` of each covered resource. A catch-up's `changes` MUST hold every covered resource changed after `from` and at or before its `nextFrom`, as it stood at `nextFrom`, a deleted one as a tombstone, so a resource missing from them did not change in that span. With `hasMore` false, its `nextFrom` MUST be no earlier than any moment on its timeline that the server had sent when it was read. A client that polls SHOULD wait the result's `nextPollMs` before its next catch-up, with a floor, as in SEP-3415's poll mode (D12), except while `hasMore` is true (R14).
  - A uri-specification on a timeline gets the timeline's latest moment as its `nextFrom` before any resource it covers exists. The server MUST answer `InvalidParams` (`-32602`) to a catch-up with no `from`, and to one naming a uri-specification it neither lists nor covers (R24), with `data.reason` `"notFound"`.

### Timelines and moments

A timeline's scope is the server's choice, and no client sees it: a client subscribes, catches up and keeps the moment it is caught up to for each uri-specification. Each of the two examples below serves tickets and notes. The names `tickets`, `notes` and `work` are the server's own.

In the first example below, `tickets/{id}` and `notes/{id}` each have a timeline of their own, so writes to one never wait on the other (R5), and a client watching both subscribes and catches up on each. § *A simple happy path* starts from this one, at `"107"`:

```yaml
listed:                       # uri-specifications MAY point to separate timelines
  tickets/{id}: tickets
  notes/{id}: notes
timelines:
  tickets:
    latest: "107"             # the timeline's latest moment
    resources:                # each one's revision, the moment it last changed
      tickets/ticket-3: "101"
      tickets/ticket-7: "107" # closing ticket-7 as a duplicate of ticket-9
      tickets/ticket-9: "107" #   changed both in one atomic step
  notes:
    latest: "042"
    resources:
      notes/plan: "042"
```

In the second example below, closing a ticket also closes its note, in one atomic step, so tickets and notes share a timeline, and the server also lists `{type}/{id}`, which covers both, beside the templates that describe each type (R1):

```yaml
listed:
  tickets/{id}: work
  notes/{id}: work
  "{type}/{id}": work         # covers both so only one shared timeline is used
timelines:
  work:
    latest: "108"
    resources:
      tickets/ticket-3: "101"
      tickets/ticket-9: "108" # ticket-9 and its note closed in one atomic step
      notes/ticket-9: "108"
```

A catch-up on `{type}/{id}` brings ticket-9 and its note as they stood at one moment. Separate catch-ups on `tickets/{id}` and `notes/{id}` can leave a client holding ticket-9 closed and its note still open until the second one.

- **R1.** A server keeps one or more timelines, from one for everything it serves to one for each uri-specification it lists. Every resource a uri-specification covers is on that uri-specification's timeline, so uri-specifications that cover a resource in common MUST share one, and each resource is on exactly one timeline. Example: `tickets/{id}` and `{kind}/ticket-3` both cover `tickets/ticket-3`, so they share a timeline though neither contains the other. A server assigns timelines over every uri-specification it could ever list to anyone, since listings change and differ by who asks.
- **R2.** Every change to a resource, its creation and deletion included, MUST happen at a new moment on its timeline, which becomes the timeline's latest moment and the resource's revision.
- **R3.** Moments on a timeline MUST strictly increase in byte order: ULIDs, or a counter zero-padded to a fixed width, but ISO timestamps only at a precision finer than the time between any two changes.
  - Kubernetes works the same way. An object's `resourceVersion` names the version it last changed at, a client watches a collection for changes after one, and an update fails if it has moved on. [KEP-5504](https://www.kubernetes.dev/resources/keps/5504/), stable in v1.35, made them orderable within one resource type, as moments are within one timeline. Clients had compared them only for equality, as ETags compare.
- **R4.** A server MUST give one moment to a change that touches several resources in one atomic step, so resources that can change together share a timeline.
- **R5.** Changes on a timeline MUST commit in the order of their moments, so once a client has seen a moment, no change at or before it can still appear: SEP-3415's "safe-to-persist watermark", held by a timeline instead of a subscription. A server checks that each new moment is later than the timeline's latest one, and commits, in one atomic step. In Postgres at read committed, for example, that's `UPDATE timelines SET latest = $new WHERE id = $timeline AND latest < $new` in byte order (`COLLATE "C"`), minting again when no row changes. A timeline per server serializes all its writes, and one per uri-specification only that one's.
- **R7.** Each of a resource's revisions, in the strawman's sense, MUST be the moment of that change, so `ifMatch` checks the latest, and this extension's clients can also order revisions. Every change-set entry in a write's result MUST carry `revision`, a deletion's included. A `resources/read` result MUST carry each item's revision in `_meta`, under the strawman's provisional key, `io.modelcontextprotocol/resource-revision`.

### What a catch-up brings

- **R19.** Each entry in `changes` is one resource's change from `from` to the catch-up's `nextFrom` (R13): the strawman's change-set entry, with contents added. `previousRevision` is the resource's revision at `from`, absent if it did not exist then. `effect` is `deleted` if the change ends with the resource gone, else `created` if it did not exist at `from`, else `updated`. A non-deletion MUST carry `contents`, one `TextResourceContents` or `BlobResourceContents`, or a `patch` from the contents at `previousRevision`, with its `patchMimeType`, in a listed format that applies. A merge patch ([RFC 7396](https://www.rfc-editor.org/rfc/rfc7396)) applies to one `application/json` text entry, and patches its parsed `text`. JSON-based patches travel as JSON, others as strings. The server MUST send `contents` when no listed format applies, when there is no `previousRevision`, or when it no longer keeps those contents, and MAY send them whenever they're smaller than the patch.
  - A client MUST apply a patch only to a copy at its `previousRevision`. Otherwise it reads the resource (D2).

### Less common cases, most likely first

**A read that comes in pages.** From `"107"`, as in § *A simple happy path*, X reads `tickets/{id}` two tickets to a page, while Y updates ticket-3 and then ticket-9:

```text
X -> A   resources/collection/read tickets/{id}
A -> X   page 1: ticket-3 at "101", ticket-7 at "107", nextFrom "107", nextCursor
Y -> A   resources/update ticket-3, at "108"
Y -> A   resources/update ticket-9, at "109"
X -> A   resources/collection/read, with that cursor
A -> X   page 2: ticket-9 at "109", nextFrom "107"   page 1's again
X -> A   resources/collection/catchup from "107"
A -> X   ticket-3 at "108", ticket-9 at "109", nextFrom "109"
```

After page 2, X's copies match no one moment: ticket-3 is as it stood at `"107"`, and ticket-9 as at `"109"`. Catching up from `"109"`, the newest revision X holds, would miss ticket-3's change at `"108"`. So every page carries the first page's `nextFrom`, and X catches up from it, ignoring ticket-9, which it already holds (R15). `nextFrom` also gives a client that reads an empty collection a moment to catch up from.

- **R26.** A long collection read comes in pages, as core's `resources/list` does: the server returns an opaque `nextCursor`, and the client sends it back as `cursor`. Every page MUST repeat the first page's `nextFrom`. A resource that doesn't change while the client pages MUST come exactly once. The client's copies match no one moment until it has every page and has caught up from `nextFrom`, which brings whatever changed while it paged, deletions included.

**A catch-up that stops short.** X is caught up to `"100"` and holds ticket-5 at `"090"`. Then ticket-5 changes at `"102"` and again at `"104"`, and A stops the first result at `"102"` to keep it small. Other tickets are left out:

```text
X -> A   resources/collection/catchup from "100"
A -> X   ticket-5 at "102", a patch from "090", hasMore, nextFrom "102"
X -> A   resources/collection/catchup from "102"   at once, since hasMore is true
A -> X   ticket-5 at "104", a patch from "102", nextFrom "104", nextPollMs 30000
```

Each result leaves X's copies as they stood at its `nextFrom`, so the next one's patches start where the copies are, as a read replica's do (§ *Specification*). X could stop after either result and still hold the collection as it stood at one moment, which a read's pages don't give (R26). Sent as `contents` instead, ticket-5 would travel twice. A catch-up that reaches the timeline's latest moment brings each resource once, at its revision.

- **R14.** A server MAY stop a catch-up short of the timeline's latest moment, to keep its result small. It MUST NOT stop partway through one moment's changes, and when it stops short it MUST set `hasMore` true, with `nextFrom` where it stopped, as in SEP-3415's poll mode. It MAY stop short with no changes, as after scanning many changes to another collection on a shared timeline. The client applies the changes, and is caught up to `nextFrom`. While `hasMore` is true, it SHOULD catch up again at once, ignoring `nextPollMs`, as SEP-3415's poll mode does. A catch-up MAY run past the server's usual size to finish a moment. A server that can't send one moment's changes in a single result MUST answer `InvalidParams` (`-32602`) with `data.reason` `"momentTooLarge"`, and the client starts over (R23), since a collection read pages freely and the catch-up after it starts past that moment.
  - Kubernetes reads every page of a list from one snapshot, so its `continue` token expires, after five minutes by default. A catch-up that stops short needs no snapshot, since the next is an ordinary catch-up, which expires only as any `from` does (R21).

**A writer's copy runs ahead of its catch-ups.** From `"107"`, as in § *A simple happy path*, X updates ticket-3 at `"109"`, and then Y updates it at `"110"`. X's copy is at `"109"`, but X stays caught up to `"107"` (R9):

| | Server A | Client X |
|---|---|---|
| latest moment | `"110"` | |
| caught up to | | `"107"` |
| ticket-3 | `"110"`, Y's update | `"109"`, X's update |

X then catches up from `"107"`:

```jsonc
{"uri": "tickets/ticket-3", "effect": "updated",
 "previousRevision": "101", "revision": "110",    // ticket-3 as of "107"
 "patchMimeType": "application/merge-patch+json", "patch": {...}}
// X's copy is at "109", not "101", so X reads ticket-3 with resources/read.
```

- **R9.** A client that writes a resource makes what it sent its copy, at the revision the reply gives that resource (R17). It takes nothing else from the reply. The reply lists every resource the write changed, such as a note closed along with its ticket, but holds no contents for the others, so the client gets them from a catch-up. The client MUST NOT move what it is caught up to on a reply. Example: X is caught up to `"107"`. Y updates ticket-9 at `"108"`, then X updates ticket-3 at `"109"`. Had X counted itself caught up to `"109"`, it would never fetch ticket-9's change.

**Messages that cross.**

```text
From "107" as in the simple happy path:
X -> A   resources/update ticket-3, ifMatch "101"
A -> X   event: ticket-3 at "109"              arrives before the reply
         X doesn't hold "109", so it catches up from "107"
Y -> A   resources/delete ticket-3, at "111"
A -> X   catch-up: ticket-3 deleted at "111"   X keeps the tombstone
A -> X   reply: ticket-3 at "109"              earlier than "111", so ignored
```

- **R15.** A client MUST ignore a write's reply, a catch-up's entry or a tombstone whose revision is no later than its copy's, and MUST NOT move what it is caught up to backward, so messages can arrive in any order.
- **R16.** A client SHOULD run one catch-up or collection read per uri-specification at a time, counting the catch-ups that follow while `hasMore` is true as one. An event that arrives meanwhile waits until they end, and the client then applies R11 to it.

**Missed events.**

- **R12.** A client SHOULD catch up on each uri-specification it subscribed to whenever it might have missed events: after each SEP-3415 refresh, on reconnecting to its gateway or a `subscriptions/listen` stream, on starting, and on a `gap` envelope. A subscription with no expiry still gets SEP-3415's occasional refresh.
  - A client can miss events without knowing, as a backend can when its webhook endpoint is down while the process holding its copies stays up. SEP-3415 retries each event, so a short outage loses none, and a missed event costs nothing once another event brings a catch-up, which brings every change after the moment the client is caught up to. A quiet collection stays stale at most until the next refresh, whose `deliveryStatus` says whether deliveries had been failing, so such a client sets its refresh interval to the staleness it can tolerate, even when granted no expiry.

**A template the server doesn't list.** A client that loaded `skill://code-review/SKILL.md` also wants the skill's other files, such as `skill://code-review/scripts/lint.py`, from a server that lists one template, `skill://{skill}/{+path}`, for all its skills. The client subscribes to `skill://code-review/{+path}`, and the server accepts it, since the listed template matches every uri the client's could. To check, the server turns the client's template into a sample uri and matches it against the listed template, as this pseudo-code does:

```text
covers(listed, clientTemplate):
  1. Write clientTemplate as a sample uri, replacing each {name} with the
     stand-in SEG and each {+name} with the stand-in ANY.
  2. Match the sample against listed as you would any uri, except that:
     - a literal in listed never matches a stand-in,
     - a {name} in listed matches SEG but not ANY, since ANY may hold a slash,
     - a {+name} in listed matches both.
  3. If they match, listed covers clientTemplate. If not, the server
     answers notFound.
```

| Listed | Named by the client | Sample | Matches? |
|---|---|---|---|
| `skill://{skill}/{+path}` | `skill://code-review/{+path}` | `skill://code-review/ANY` | Yes: `{skill}` matches `code-review`, and `{+path}` matches `ANY` |
| `skill://{skill}/{+path}` | `skill://code-review/scripts/{+file}` | `skill://code-review/scripts/ANY` | Yes: `{skill}` matches `code-review`, and `{+path}` matches `scripts/ANY` |
| `{type}/{id}` | `tickets/{id}` | `tickets/SEG` | Yes: `{type}` matches `tickets`, and `{id}` matches `SEG` |
| `tickets/{id}` | `tickets/{+rest}` | `tickets/ANY` | No: `{id}` holds one path segment, but `ANY` may stand for several, such as `a/b`, so the server answers `notFound` |

- **R24.** A client MAY name a uriTemplate the server doesn't list, in a subscription, a catch-up or a collection read, as long as a template the server lists matches every uri the client's could. The client's template then shares the listed one's timeline (R1). A server MAY check as the pseudo-code and table above show, and MUST answer `notFound` when it can't tell.

**Starting over.** A client caught up to a moment too old (R21), or whose catch-up meets a moment too large for one result (R14), reads the collection again, as a new client does (R18).

- **R23.** A client that held copies before reading a collection MUST replace each one the read's pages name, even with an earlier revision, as after a restore (R21). It MUST also drop each one in that collection that neither the read's pages nor the catch-ups after them named, if its revision is at or before the last catch-up's `nextFrom`.

**A moment too old to answer from.**

- **R21.** A server MUST keep the moment of every change, deletions included, back to a moment of its choosing, and MAY drop old contents at any time. It MUST answer a catch-up whose `from` is earlier than that with `InvalidParams` (`-32602`) and `data.reason` `"momentExpired"`, and the client starts over (R23, D4). A server restored from a backup MUST first mint a moment past every one it might have sent, and make it the oldest it answers from, as etcd's `snapshot restore` does with `--bump-revision` and `--mark-compacted`. A server SHOULD keep that history longer than the time it grants between refreshes, and longer than a read of its largest collection takes, so that a client can catch up at each refresh (R12) and after reading any collection.

**Resources a client may not read.**

- **R25.** Catch-ups and collection reads MUST cover only the resources the principal may read, since they carry contents (R13, R22). Events SHOULD cover only those too, as SEP-3415's check at delivery time is a SHOULD, since an event carries only uris and revisions (R10). When what a principal may read changes while the resources don't, the server MAY mint a moment that changes no resource, as after a restore (R21), and answer that principal's catch-ups from before it with `momentExpired`. Its clients then start over (R23), which brings a shared resource or drops a revoked one.

### Declaring the extension

- **R20.** A server declares this extension in `capabilities.extensions` under a provisional identifier, `io.modelcontextprotocol/resource-events`, as SEP-2133 asks. A server that declares it MUST serve `resources/collection/read` and `resources/collection/catchup`, and list `mcp.resource.collection.updated` in `events/list`, with whichever of SEP-3415's delivery modes it offers. The extension reserves the prefix `resources/collection/` for its methods, beside core's and the strawman's under `resources/`. The event's name waits on Open Question 2's decision about an `mcp.*` prefix.

### uriGlob, later

A uriGlob is a third kind of uri-specification. A glob is the familiar way to write a filesystem-like collection, and reads more clearly than a template that could describe much the same one: `file:///tickets/**/*.md`, the example the Filesystems working group's Problem Statement 2 gives, against `file:///tickets/{+path}.md`. Its match is also defined by its own rules, where a template's is left open. RFC 6570 defines how a template expands, and says matching one "only works well if the template expressions are delimited by the beginning or end of the URI or by characters that cannot be part of the expansion". Problem Statement 2 asks for glob matching under a URI prefix to find resources, and the strawman defers globs, so a uriGlob would let a client watch what it found with the same pattern. This extension offers that group three rules, and adopts whatever it settles:

- `*` matches any characters within one segment, and `**` zero or more whole segments.
- A uriGlob has at most one `**`, so a matcher checks the segments before it from the URI's start and those after it from the end, and two globs' overlap (R1) stays easy to test.
- A glob has a field of its own, `uriGlob`, since `*` is legal in a URI's path.

## Rationale

Each decision keeps the number the rules cite it by, and opens by restating its rule, so it reads without scrolling back to the Specification.

### Subscribing and events

**D1. A client subscribes with SEP-3415's `events/subscribe`.** It subscribes with `events/subscribe` (R18), and catches up with `resources/collection/catchup`, which stores nothing (R13), so SEP-3415's is the one subscription lifecycle. A subscribe method of this extension's own, such as `resources/webhooks/subscribe`, would be a second lifecycle beside it, repeating what SEP-3415 already specifies for webhooks: verifying the callback, signing each delivery, refreshing and expiring subscriptions, and reporting `deliveryStatus`. Each client and gateway that already takes Events, such as those § *Why now?* names, would then need a second subscription API to watch resources.

**D5. Events carry no contents.** An event names what changed and carries none of it (R10), so a lost, repeated or late one does no harm. A change then costs a subscriber an event and a catch-up, one round trip more than a delivery carrying the change, with about the same bytes. The writer pays nothing extra, and N changes before a catch-up cost N events and one catch-up. Deliveries that carry each change's contents or a patch would need a `fromVersion` to order them, and a resubscribe that drops the deliveries still queued.

**D19. The extension sends an event of its own, one for each change on each subscription.** `mcp.resource.collection.updated` goes to each subscription covering a resource the change touched. Its `data` names the subscription's uri-specification, and lists the change's entries that the subscription covers (R10, R11). It leaves `mcp.resource.updated` to SEP-3415's Open Question 2. A server may coalesce undelivered events, as Peter agreed for replacing an event with a newer one. Peter's `mcp.resource.updated`, with data of our own, would take a name Open Question 2 may define otherwise, for core's `notifications/resources/updated`, whose `uri` names the changed resource. One event for each resource changed would send a change touching 500 tickets as 500 deliveries, 499 of them skipped after the first catch-up. An event naming only the uri-specification and the moment would let a writer skip it only by taking the whole moment as its own, which is wrong when its write also changed a resource whose contents it lacks, as closing a ticket closes its note. Leaving the uri-specification to the subscription `id` in `X-MCP-Subscription-Id` would have each client map it back to its subscription.

### Moments and revisions

**D6. Moments are ordered.** Moments sort (R3, R5), so one string serves as a resource's version, its revision, and the moment a client is caught up to, at the cost of serializing each timeline's writes. Versions compared only for equality, as ETags are, would leave a client unable to tell which of two revisions is the newer, which R11's skip and R15 both need.

**D17. "Moment" and "revision" stay two words.** § *Definitions* uses moment, latest moment, revision and caught up to. A moment is a point on a timeline, and a revision is a moment at which one resource changed, so both stay. One word for both would ask the strawman to rename its fields, and would leave "a resource's moment", "the timeline's moment" and "a client's moment" each meaning a different latest. "Current moment" gave way to "latest moment", since a timeline moves only when a change occurs.

### Catching up

**D2. A patch starts from `previousRevision`.** A catch-up's patch applies to the resource as it stood at the client's `from`, and a client whose copy is at another revision reads the resource instead (R19). Patching from each copy's own revision would need the server to learn every copy's revision, which a catch-up never sends. Sending contents only would give up much of the saving.

**D4. A `from` too old gets `momentExpired`.** The client then starts over (R21), as Kubernetes answers a watch from a compacted `resourceVersion` with `410 Gone`, and core an expired cursor with `-32602`. A flag on an ordinary result carrying the whole current state, or SEP-3415's `truncated: true`, is a signal a client could miss, and so keep copies of resources since deleted.

**D12. A catch-up's result is a plain result, and paces polling with `nextPollMs`.** A client that polls waits the result's `nextPollMs`, as in SEP-3415's poll mode (R13, R14). Core's `CacheableResult` would serve badly: its `ttlMs` is a freshness hint, which clients "SHOULD NOT treat … as a polling interval", and a cache keyed on the request could answer step 6's catch-up with step 3's empty result, since both send `from` `"107"`. A rule that a client never answers a catch-up from a cache would bind no gateway that follows core, and with no pacing field, a server would have no way to slow clients that poll. A collection read stays a `CacheableResult` (R22), since a cached read is an older snapshot whose `nextFrom` matches it.

**D13. A long catch-up is a run of short ones.** A server may stop a catch-up short, bringing each resource as it stood where it stopped (R13, R14), so a long catch-up is a run of ordinary ones, as polls are in SEP-3415, and every patch applies in order. Four other kinds of paging were weighed against it. Taking the last entry's `revision` as the moment the client is caught up to, beside a field holding the timeline's latest moment, gave the client two rules, and could not stop short with no changes. Pages bringing each resource at its revision cost a read for each resource changed on both sides of a page's end. Sending the first page's `from` again on every page, with a `cursor` beside it, needs both fields and a rule for which one a patch starts from. A fixed end sent back on every page, as `through`, cost a field and a catch-up after the last page.

**D15. A catch-up never stops partway through a moment.** Stopping there would leave the client's copies matching no moment until its next catch-up. So a server runs a result long to finish a moment. Such a result is never larger than a start-over, which sends every resource in the collection, and a server that still can't finish one answers `momentTooLarge` (R14). Reusing `momentExpired` would tell a client its `from` had expired when it hadn't. SEP-3415's `ResourceExhausted`, whose code its Open Question 5 may fold into `-32602`, would give a client a second code for the one response, starting over.

### Reading collections

**D14. Reading a collection has a method of its own.** `resources/collection/read` takes no `from`, and pages with core's opaque `cursor` and `nextCursor`, as `resources/list` does (R22, R26). Their pages differ: a client can stop after any catch-up and hold the collection as it stood at one moment (R14), but must read every page of a collection read, then catch up, to do so (R26). A client reads the collection, subscribes, then catches up from the read's `nextFrom` (R18), so every catch-up names a `from`. A catch-up with no `from` would give one method two kinds of result, and need a `cursor` beside `from`, with an error for naming both. `resources/template/read` would need a sibling for uriGlobs, and reads like fetching the template itself. SEP-2640's `resources/directory/read` lists one directory's children without their contents. Subscribing before reading also works, but leaves a new client's events waiting while it pages.

**D20. A client may name a template the server doesn't list, when a listed one covers it.** A server answers `notFound` for one it can't show is covered (R24). R24's pseudo-code checks it by writing the client's template as a sample uri, with a stand-in for each expression, and matching that as any uri. The check never wrongly accepts, and it handles a listed template with part of it filled in, as a skill or a type is. Listed templates alone would leave a client that wants one skill, on a server listing `skill://{skill}/{+path}`, watching every skill, unless the server lists a template for each. An exact check would first have to settle what RFC 6570's expressions match, empty values and levels 3 and 4 included, then compare automata that can grow exponentially.

**D25. A `resources/read` result MUST carry each item's revision.** R7 puts it in `_meta`, so a copy a client got from a core read, as a skills client gets `SKILL.md`, can be compared without reading it again. The MUST binds only servers that declare this extension, and gives the strawman's open question about `resources/read` a use case. R19's fallback could read through `resources/collection/read` alone, which needs no hook into core's read, but that would leave copies from core reads without a revision. A SHOULD, as the strawman's "when the server can provide it", would give R19 two routes.

### Writes

**D7. A client's own writes use the strawman's four methods.** Their replies carry the revision of each resource a write changed (R7, R17), which lets the writer skip the event its own write sends (R9, R11). A write made another way, such as through a server's tools, returns no revision a client can rely on, so its writer would catch up on its own change. Changes from anywhere else still land at moments and send events (R2, R10).

## Backward Compatibility

None. The extension adds two methods and one event, and binds only a server that declares it (R20). Its one rule on a core method, R7's revision for each `resources/read` item, adds a key under `_meta`, which a client that doesn't know the extension ignores (D25).

## Security Implications

This extension sends its events by SEP-3415's delivery modes as they stand, so that SEP's rules cover every delivery, as its rules on callback URLs, signing and endpoint verification cover every webhook POST. It adds four points of its own:

- **Events carry no contents** (R10), so a delivery that reaches the wrong receiver reveals uris and revisions, never what a resource holds.
- **A catch-up or a collection read is authorized as a `resources/read`** of each resource it covers (R13, R22), so catch-ups and reads MUST cover only what the principal may read, and events SHOULD (R25).
- **`nextFrom` moves with every change on a timeline** (R13), so it can tell a principal when resources it may not read change. A server that minds can avoid it, as by giving those resources a timeline of their own where R1 allows.
- **Both methods carry their `uri` or `uriTemplate` in `Mcp-Name`** (R13, R22), where a gateway or proxy sees it, as core's `resources/read` carries its `uri`.

**On Cloudflare Workers, the platform's egress already does SEP-3415's address check.** SEP-3415 asks a server to reject callback URLs whose address isn't globally routable, at delivery time. With the `global_fetch_strictly_public` flag, a Worker's fetch refuses a name that resolves to a private, loopback, link-local or other special-purpose address with an error of Cloudflare's own, 1016 or 1002, and names built to rebind to a private address reached none. Two ranges, `192.0.0.0/24` and `192.0.2.0/24`, go out and time out. On 2026-11-03, the prototype's webhook POSTs and its fetches of client metadata documents met the same outcome for each of 30 names, so a server on Workers needs no address check of its own for either. Cloudflare's documentation promises none of this, so the prototype's deployed tier measures it again on every run.

## Reference Implementation

[lumenize/resource-webhooks-on-events](https://github.com/lumenize/resource-webhooks-on-events) builds this extension on Cloudflare Workers, without an MCP SDK, and with the spec's `schema.ts` as its only schema. It has two parts, each a Durable Object:

- **A, the server.** It serves tickets and notes, with the strawman's four methods for writes and zero-padded counters for moments. Clients register by Client ID Metadata Document.
- **B, a gateway** in SEP-3415's forward-proxy deployment, which verifies each delivery and passes it to its client over a WebSocket.

A client library, `src/client/copies.ts`, keeps copies by the client's rules: R9, R11, R12, R15, R16, R18, R19, R21 and R23. A serves two layouts in its tests. One puts tickets and notes on one timeline and also lists `{type}/{id}`. The other gives each its own timeline, so closing a ticket closes its note at a moment on each, which breaks R4 on purpose, to test a write that commits once per timeline. The deployed Worker serves the first.

### Tested, and not yet tested

- **457 tests pass,** and a check fails the run unless a passing test names every rule above but R25. Each new test was mutation-checked: broken code turned it red.
- **Each MUST of SEP-3415's that the prototype uses has a test.**
- **MCP's conformance suite,** `@modelcontextprotocol/conformance` 0.2.0-alpha.12 at revision 2026-07-28, passes against A and against the client, with a baseline that lists the features neither offers, such as prompts.
- **A deployed tier runs on Cloudflare.** Two clients read the tickets, subscribe and catch up. One closes a ticket and skips its own event, and the other catches up with a merge patch.
- **Designed, not built:** several writers minting moments at once, subscriptions granted no expiry, R12's catch-up on a `subscriptions/listen` stream and the `deliveryStatus` it cites, uriGlob, access that differs by principal (R25), and a restore from a real backup, whose steps an operator reset takes in the tests (R21).
- **Only the prototype's own client and server have run the extension so far.** The strawman's methods are built for the happy path and a conflicting write.

## Open Questions

IDs follow the order the questions were raised, not the order shown, and a gap is a question answered or withdrawn.

- **Q6. Authorization.** A principal may be able to read only some of the resources a uri-specification covers, such as some of the tickets under `tickets/{id}`.
  - (a) The server answers a subscription, catch-up or collection read with an error unless the principal may read every covered resource, so a client can watch only a collection it may read whole.
  - (b) The server answers with only the resources the principal may read, so a client can watch `tickets/{id}` and see just the tickets it may read.
  - This draft takes (b), in R25, and leaves the choice to the working group.
- **Q7. Starting to watch a collection in one call.** The simple happy path shows a client making three calls in a row when it first connects: (1) it reads the collection, (2) subscribes with `events/subscribe`, and (3) catches up (§ *A simple happy path*, steps 1 to 3). One call would be simpler to use.
  - (a) Keep the three calls, so SEP-3415's `events/subscribe` stays the one subscription lifecycle (D1), and add a helper to the SDKs that makes them.
  - (b) Add `resources/collection/subscribe` to this extension, which subscribes and returns the collection's contents. It would be a second subscription lifecycle beside `events/subscribe`, and its result would page as a collection read's does.
  - This draft takes (a), and leaves the choice to the working group.
