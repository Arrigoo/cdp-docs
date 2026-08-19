# Arrigoo Integration Guide

This document describes how systems integrate with Arrigoo: the available integration
patterns, the event data model, the endpoints used to send data in and out, and the
real-time / event-driven capabilities built on top of them.

---

## 1. Integration overview and recommended patterns

Arrigoo is designed around a small number of well-defined integration surfaces. Rather
than a bespoke connector per source system, every integration falls into one of four
patterns. Most implementations use two or three of them together.

| Pattern | Direction | Trigger | Typical use |
|---|---|---|---|
| **Browser tracking** | In | Visitor activity | Behavioural events, sessions, consent, on-site personalisation |
| **Server-side endpoints** | In | Source system pushes | Orders, CRM updates, transactional events, backend systems |
| **API integrations (pull)** | In | Cron schedule | Nightly CRM/ERP/marketing-platform syncs, bulk profile loads |
| **Actions (push)** | Out | Event, segment change, schedule | Webhooks, marketing-automation handover, mail, AI enrichment |

### 1.1 Browser tracking

The Arrigoo tracking script runs in the browser, maintains the customer id (`cid`) and
session id (`sid`), and posts events to the public event endpoint. It also receives the
customer's profile back on the same request, which makes on-site personalisation possible
without a second round trip. See [section 3.1](#31-the-browser-tracking-endpoint-event).

### 1.2 Server-side custom endpoints

For anything that does not happen in a browser — an order confirmed in the webshop
backend, a support case closed, a subscription renewed — Arrigoo lets you define
**custom endpoints** in the admin UI. There are two kinds:

- **Event endpoints** (`POST /event-endpoint/{slug}`) — receive activity and write events.
- **Profile endpoints** (`POST /profile-endpoint/{slug}`) — receive attributes and write
  profile properties.

Each endpoint is a configuration, not code. You define:

- a **slug** (the public path segment),
- a **parser** that says where the records live in the payload and how remote fields map
  onto Arrigoo fields,
- an **identifier** (which field in the payload identifies the customer, and as which
  identifier type),
- an **auth type**.

This means the sending system does not have to conform to an Arrigoo payload format.
It sends its own natural JSON, and the endpoint's mapping adapts it. Adding a new source
system is a configuration task, not a development task.

**Recommended:** use custom endpoints as the default for server-to-server integration.
They are real-time, they are individually authenticated and logged, and they can be added
or changed without a deployment.

### 1.3 API integrations (scheduled pull)

Where the source system cannot push — or where a periodic full reconciliation is wanted —
Arrigoo can pull. An **API integration** is a scheduled outbound request whose response is
parsed into profile updates using the same parser/mapping model as the inbound endpoints.

Supported source types include:

- **HTTP** — any REST/JSON API, with configurable method, headers and request body.
- **OAuth2** — client-credentials style APIs, using a stored connection.
- **Microsoft Graph / Dynamics** — Dataverse entity sets (`contacts`, `accounts`, …).
- **Agillic** — recipient exports, with automatic pagination.

Integrations run on a cron schedule, support pagination via a `next_url` expression,
persist the previous request URL/timestamp so runs can be incrementally windowed
(`{{ req.time(-7d) }}`), and write an entry to the integration log on every run.
Credentials are never stored in the integration itself — they are referenced as
`{{ x.secret_name }}` and resolved from the encrypted secret store at request time.

Inventory (product/content) data uses the same mechanism through
**inventory integrations**, writing to the inventory rather than to profiles.

**Recommended:** use API integrations for bulk and reconciliation loads, and custom
endpoints for the real-time delta. The two combine well: a nightly pull guarantees
completeness, the endpoint guarantees freshness.

### 1.4 Actions (push out)

Outbound integration is handled by **actions**. An action is a configured operation —
a webhook, an email, a marketing-platform call, an AI prompt — that is bound to a
**trigger**. See [section 4](#4-real-time-and-event-driven-integrations).

### 1.5 Other surfaces

- **CSV import** — `POST /v1/csv-event/{configKey}/upload` for file-based event delivery,
  using a stored column mapping.
- **Feeds** — `GET /feed/{slug}/{cid}` returns a personalised content/product feed for a
  customer; used for on-site recommendations and email content blocks.
- **Admin API** — the full `/v1/*` REST API (customers, events, properties, segments,
  inventory, configuration) for direct integration and operational tooling.
- **MCP API** — `/v1/mcp` exposes profiles, segments, events, inventory and feeds to
  AI agents.

---

## 2. Event structure and event data model

### 2.1 One event shape for everything

Arrigoo uses **a single, flat event structure**. A page view, a purchase, a support call
logged in the CRM and an email click imported from the marketing platform are all the same
record type, distinguished only by the value of `evt`.

```json
{
  "cid":    "4b0e9c31-…",
  "sid":    17,
  "evt":    "purchase",
  "topics": ["outdoor", "footwear"],
  "strval": "SKU-10293",
  "intval": 1299,
  "src":    "webshop",
  "url":    "https://example.com/checkout/receipt",
  "ref":    "https://google.com/",
  "fv":     false,
  "ns":     true,
  "ident":  { "id_type": "email", "id_value": "customer@example.com" },
  "ctime":  "2026-08-19T10:14:22Z"
}
```

| Field | Type | Meaning |
|---|---|---|
| `cid` | string | Customer id. Assigned by Arrigoo; stable across sessions and devices once identified. |
| `sid` | int | Session id. |
| `evt` | string | Event type. The only field that varies the meaning of the record. |
| `topics` | string[] | Free-form classification — categories, tags, product groups, content topics. |
| `strval` | string | The event's string value (product id, article title, search term, campaign name…). |
| `intval` | int | The event's numeric value (amount, quantity, score, duration…). |
| `src` | string | Where the event came from (`webshop`, `endpoint-4`, `crm`, …). |
| `url` | string | The URL the event relates to. |
| `ref` | string | Referrer. |
| `fv` | bool | First visit. |
| `ns` | bool | New session. |
| `ident` | object | Identifier carried with the event (`id_type` + `id_value`), used to resolve or create the profile. |
| `ctime` | string | Event timestamp (RFC 3339). Defaults to receipt time. |

Supported identifier types are `email`, `phone`, `foreignid1`, `foreignid2`, `foreignid3`
and `agillic_id`.

### 2.2 Why the structure is deliberately simple

The unified structure is a design decision, and it pays off in several places:

- **Any source maps onto it.** Because there is one shape with a generic string value, a
  generic numeric value and a topic list, a new source system never requires a new event
  schema, a schema migration, or a new table. Mapping a remote payload onto
  `evt` / `strval` / `intval` / `topics` is a configuration step.
- **Segmentation and property rules work across all sources uniformly.** A property
  aggregation like "sum of `intval` for `evt = purchase` over the last 90 days" behaves
  identically whether those purchases came from the browser, an endpoint, a CSV or a
  nightly pull. Rules do not have to be re-implemented per source.
- **Queries stay fast and predictable.** One physical event model means one set of indexes
  and one query pattern, rather than a join graph that grows with each integration.
- **Integrations stay cheap to add and to change.** Most competing models require a
  schema definition per event type before data can flow. Here, an event spec is a
  lightweight validation rule, not a table definition.
- **No dead ends.** Because `topics` is a list and unmapped detail can be carried in
  `strval`, sources rarely need to be re-integrated later when requirements change.

The trade-off is deliberate: Arrigoo does not try to be a general-purpose warehouse of
arbitrarily nested event payloads. It keeps the behavioural layer small and uniform, and
puts richness into the **profile property** layer, where it is queryable and actionable.

### 2.3 Event specs

Each event type has an **event spec** that defines validation and handling:

| Field | Meaning |
|---|---|
| `evt` | The event type name. |
| `topics`, `strval`, `intval` | Requirement flags: `1` = required, `0` = optional, `-1` = must not be present. |
| `description` | Human-readable description. |
| `protected` | The event type is excluded from general read/search surfaces. |
| `protected_content` | The event's content is treated as sensitive. |

Events that fail validation against their spec are rejected with `400`. Events whose type
has no spec are rejected — this prevents unregistered event types from silently
accumulating.

### 2.4 From event to profile

Events are the raw material; **profile properties** are the derived, queryable state.
Property definitions declare conditions over events (event type, value ranges, URL or
topic matches, recency windows) and an accumulator (`sum`, `avg`, `count`, `max`, `min`,
group counts, tallies, history series, trends). When events arrive, the affected customer
is queued for re-aggregation, properties are recalculated, and segment membership is
re-evaluated.

Properties are typed (`str`, `strs`, `int`, `float`, `bool`, `date`, `datetime`, `map`,
`tally`, `history`, `trend`, `calculation`), and selected properties can be flagged as
identifiers, which is how an inbound property update can also identify or merge a profile.

---

## 3. How data and events are sent to and from Arrigoo

### 3.1 The browser tracking endpoint (`/event`)

Most first-party behavioural data originates in the browser. The Arrigoo tracking script
is loaded on the site, establishes and persists the customer id, manages session state,
and posts each tracked event as JSON.

```
POST /event            (alias: POST /v1/event)
Content-Type: application/json
```

Request body: an event object as described in [section 2.1](#21-one-event-shape-for-everything).

The endpoint does four things in one round trip:

1. **Validates** the event against its event spec.
2. **Enforces the account's event rate limit** (`429` if exceeded).
3. **Resolves the profile** — from `cid`, from the `ident` block if present, or from the
   Agillic click cookie (`ag-uid`) when a visitor arrives from an Agillic email.
4. **Queues the event** for storage, property aggregation and action evaluation, and
   **returns the customer's current profile** so the page can personalise immediately.

Response:

| Status | Meaning |
|---|---|
| `200` | Event accepted; body contains the profile. |
| `204` | Event accepted; no profile payload to return (e.g. first visit, or nothing changed). |
| `400` | Event failed validation against its spec. |
| `429` | Account event limit reached. |

Profile payload (`200`):

```json
{
  "cid": "4b0e9c31-…",
  "p":   [ { "lab": "customer_segment_value", "val": 4200 } ],
  "s":   ["high_value", "newsletter_subscriber"]
}
```

`p` = properties, `s` = segments. The compact keys keep the response small
enough to be safe on every event.

A separate lightweight endpoint, `POST /hb`, records engagement heartbeats (time on page,
scroll dwell) and is sent via `navigator.sendBeacon` where available.

### 3.2 The event endpoint (server-side events)

For events originating in backend systems, define an **event endpoint** in the admin UI.

```
POST /event-endpoint/{slug}
Content-Type: application/json
```

Configuration:

| Setting | Purpose |
|---|---|
| `slug` | Public path segment for this endpoint. |
| `event_type` | The `evt` written for every record received here. Must have an event spec. |
| `parser.entry_path` | Where in the payload the records live (empty = root array). |
| `parser.single_entry` | Whether the payload is one record rather than a list. |
| `parser.identifier_source` / `identifier_type` | Which payload field identifies the customer, and as which identifier type. |
| `parser.mappings` | Remote field → event field (`strval`, `intval`, `topics`, `topics_add`, `src`, `url`, `ctime`, …). |
| `auth_type` | `none`, `api` (issued bearer token), or `api_key` (fixed shared secret). |
| `status` | Enable/disable without deleting. |

Behaviour per request:

1. The endpoint is looked up by slug and checked for `status` and authorisation.
2. The body is parsed into records using `parser`.
3. For each record the identifier is extracted, and the customer is **identified or
   created**.
4. The mapped values are applied to an event of the configured `event_type`, tagged with
   `src = endpoint-{id}`.
5. The event is queued for processing.

Batches are supported — a single request may contain many records. The response reports
acceptance; per-record outcomes are written to the endpoint log, visible in the admin UI
along with the most recent raw payload received (useful when onboarding a new sender).

Example — a webshop posting its own natural order format:

```json
[
  { "orderId": "SO-4471", "customerEmail": "a@example.com", "total": 1299,
    "categories": ["outdoor", "footwear"] }
]
```

with `identifier_source = customerEmail`, `identifier_type = email`, and mappings
`orderId → strval`, `total → intval`, `categories → topics`.

### 3.3 The profile endpoint (server-side profile data)

Profile attributes — as opposed to activity — are sent to a **profile endpoint**.

```
POST /profile-endpoint/{slug}      (alias: POST /inbound/{slug})
Content-Type: application/json
```

Configuration mirrors the event endpoint, except that `parser.mappings` map remote fields
onto **profile properties** (by system title) rather than onto event fields, and there is
no `event_type`. Date and datetime properties can declare a source `format` so
non-ISO date strings are parsed correctly.

Behaviour per request:

1. Lookup by slug, `status` check, authorisation.
2. Parse the body into records.
3. Convert each record into a profile update: an identifier plus a set of properties.
4. **Upsert** the customer — create if unknown, update if known, merging identifiers.

Setting the endpoint's type to `update_only` makes it non-creating: records that do not
match an existing profile are skipped. This is the right setting for enrichment feeds that
should never introduce new profiles.

Example:

```json
{ "email": "a@example.com", "loyalty_tier": "gold", "signup_date": "15-03-2024" }
```

### 3.4 Authentication on custom endpoints

Every custom endpoint chooses its own scheme:

- **`none`** — public. Only appropriate for low-sensitivity, high-volume signals.
- **`api`** — a bearer token issued through `/auth/api`, validated per request.
- **`api_key`** — a fixed shared secret, generated in the admin UI, stored encrypted in
  the secret store and compared in constant time. Sent as `Authorization: Bearer …` or
  `X-API-KEY`.

Because the key is per endpoint, a compromised or rotated credential affects one
integration rather than the whole platform.

### 3.5 Reading data out over the API

| Endpoint | Purpose |
|---|---|
| `GET /v1/customer/{cid}` | Full profile: properties, segments, consents, identifiers. |
| `POST /v1/customer/{ident_type}` | Look up a customer by identifier value. |
| `POST /v1/customer/filter` | Query profiles by property conditions. |
| `GET /v1/customer/segment/{seg}` | All customers in a segment. |
| `POST /v1/event-api/search` | Search events by type, customer and date range. |
| `GET /v1/event-api/cid/{cid}` | A customer's event history. |
| `GET /feed/{slug}/{cid}` | Personalised content/product feed for a customer. |
| `GET /v1/inventory` | Product/content inventory. |
| `DELETE /v1/customer/{cid}` | Erase a profile (GDPR). |

### 3.6 Sending data out

Outbound delivery is always an **action**. See the next section.

---

## 4. Real-time and event-driven integrations

Arrigoo is event-driven internally. Inbound events are placed on a queue, consumed
asynchronously, stored, aggregated into properties, and evaluated against triggers. This
keeps the ingest path fast and the outbound path decoupled and retryable.

### 4.1 Triggers

A **trigger** binds an action to a condition:

| Trigger type | Fires when | Latency |
|---|---|---|
| `event` | An event of a given type is recorded | Real time (queue latency) |
| `segment` | A customer **enters** or **leaves** a named segment | On segment evaluation run |
| `delete` | A profile is deleted | Real time |
| Scheduled | A cron-style `scheduled_action` to execute actions on all users in one or more segments | On schedule |

A trigger carries `trigger_type`, `trigger_type_id` (the event type or segment system
title), optional `conditions` (for segment triggers, `{"type": "enter"}` or
`{"type": "leave"}`), and a reference to the action configuration to run.

Event triggers are held in memory in the consumer and reloaded on change, so evaluating
them adds no per-event database round trip.

### 4.2 The execution path

```
inbound event ──▶ CUSTOMER_EVENTS queue ──▶ event consumer
                                              │
                                              ├─▶ store event
                                              ├─▶ queue profile for re-aggregation
                                              └─▶ match event triggers
                                                        │
                                                        ▼
                                              ACTION_CHANNEL queue
                                                        │
                                                        ▼
                                              action processor ──▶ external system
                                                        │
                                                        └─▶ action log
```

When a trigger matches, the action is queued with a full payload: the **customer**
(properties, segments, consents, identifiers), the **event** that caused it, the action
**configuration**, and — for chained actions — a **context** object holding the previous
step's response.

### 4.3 Action types

| Type | Does |
|---|---|
| `webhook` | Arbitrary HTTP request: configurable URL, method, headers, body template, auth, and rate limit. |
| `email` | Sends a templated email. |
| `agillic_add_to_static_target_group` / `agillic_remove_from_static_target_group` | Target-group membership in Agillic. |
| `agillic_trigger_flow` | Starts an Agillic flow for the recipient. |
| `agillic_one_to_many` | One-to-many send. |
| `agillic_achieve_event` | Registers an Agillic event (async, resolved via callback). |
| `agillic_person_data` | Upserts recipient data in Agillic (async, resolved via callback). |
| `agillic_decentralized_campaign` | Creates a decentralised email campaign. |
| `ai_prompt` | Sends the prompt plus the profile and event history to an LLM; the response is handed to a mandatory follow-up action. |
| `render_properties` | Recomputes/renders profile properties. |
| `delete` | Deletes the profile, then fires the `delete` triggers. |

### 4.4 Templating

Action payloads — URL, headers and body — are templates. Placeholders are substituted at
execution time:

| Placeholder | Resolves to |
|---|---|
| `{{ p.property_name }}` | A profile property. |
| `{{ c.consent_name }}` | A consent value. |
| `{{ i.identifier_type }}` | An identifier (e.g. `{{ i.email }}`). |
| `{{ s.segment_name }}` | Segment membership. |
| `{{ x.secret_name }}` | A secret from the encrypted secret store. |
| `{{ ctx.body.field }}` | A field from the previous action's response (chained actions). |
| `{{ req.time(-7d) }}` | Computed request metadata, e.g. a relative timestamp. |
| `{{ ii.… }}` / `{{ iif.… }}` | An inventory item or a personalised feed entry. |

Modifier functions are available on placeholders (for example URL encoding and regex
extraction), so most payload shaping is done in configuration rather than in code.

### 4.5 Chaining, throttling and observability

- **Chaining.** Any request can name a `followup_action`. The response of the first call
  is passed to the next as `{{ ctx.* }}`. This composes multi-step outbound flows — for
  example: enrich via an external API → build a message with an AI prompt → deliver by
  webhook — without custom code.
- **Throttling.** Actions declare a concurrency/rate limit, honoured per action
  configuration, so a burst of events cannot overwhelm a downstream API.
- **Asynchronous callbacks.** Actions whose target completes out of band POST back to
  `/action/callback/{configKey}/{execId}`, and the final outcome is recorded against the
  original execution.
- **Retry.** Events carry a retry counter and failed processing is re-queued.
- **Logging.** Every action execution writes a log entry with the customer id, the
  triggering event, the response code and the payload. Every inbound endpoint and every
  scheduled integration keeps its own log and 30-day statistics, visible in the admin UI.

### 4.6 Choosing between real-time and scheduled outbound

- **Event trigger** — for reactions that must be immediate: abandoned basket, form
  submitted, high-intent page viewed, transaction completed.
- **Segment trigger** — for state changes rather than moments: a customer becoming
  high-value, lapsing, or qualifying for a programme. Fires on segment entry or exit, so
  it is naturally deduplicated — a customer is not re-triggered while they remain in the
  segment.
- **Scheduled action** — for batch handovers and periodic synchronisation where the
  downstream system prefers bulk over stream.

---

## 5. A typical implementation

A common end-to-end setup combines all four patterns:

1. **Tracking script** on the website — behavioural events, sessions, consent, and the
   profile returned on each event for on-site personalisation.
2. **Event endpoint** from the webshop/ERP — orders, returns, subscription changes,
   in real time and authenticated with a per-endpoint API key.
3. **Profile endpoint** from the CRM (`update_only`) — service attributes, tier, contact
   preferences.
4. **API integration** against the marketing platform — nightly pull of sends, opens and
   clicks, plus a reconciliation of recipient state.
5. **Properties and segments** derived from the combined event stream.
6. **Actions** pushing outward — a segment-entry trigger handing customers to the
   marketing platform, and an event trigger firing a real-time webhook for high-intent
   behaviour.

Steps 2–6 are configuration in the admin UI. Only step 1 requires a change to the website.
