# Building Segments from Scratch

A practical guide to going from raw events to a working, actionable segment in Arrigoo:
what each layer does, every option available at each step, and how to choose between them.

---

## 1. The three-layer model

Segmentation in Arrigoo is built in three layers. You cannot skip a layer, and understanding
why they are separate is most of the battle.

```
   EVENTS                    PROPERTIES                    SEGMENTS
   ──────                    ──────────                    ────────
   Raw, immutable            Derived, queryable            Named audiences
   facts about               state about a                 defined by
   behaviour                 profile                       property conditions

   "viewed article X"   ──▶  articles_read_90d = 14   ──▶  "engaged_reader"
   "purchased, 1299"         total_spend_365d = 8400       "high_value_customer"
   "purchased, 2100"         favourite_topic = "outdoor"   "outdoor_enthusiast"
```

| Layer | What it is | When it changes | Where it lives |
|---|---|---|---|
| **Event** | An immutable record that something happened | Written once, never updated | `event` table |
| **Property** | A computed value derived from events | Recalculated on a schedule | `cust_property_str` / `_int` / `_json` |
| **Segment** | A set of customers matching property conditions | Re-evaluated on a schedule | `cust_segment` table |

**Why properties sit in the middle.** Segments do not query events directly. They query
properties. This is deliberate:

- **Performance.** A segment like "spent over 5,000 in the last year" would otherwise scan
  and aggregate every purchase event for every customer on every evaluation. Instead the
  aggregation happens once per customer in the property layer, and the segment does a
  simple indexed comparison against a single stored number.
- **Reusability.** One `total_spend_365d` property serves twenty segments, an action
  payload, a personalisation rule, an export and a feed. Re-deriving that logic inside each
  segment would mean twenty places to change when the definition changes.
- **Inspectability.** You can look at a profile and see *why* it qualifies. The property
  value is visible, testable and explainable — a segment built on opaque event scans is not.
- **Composability.** Properties can be built on other properties (history, trend,
  calculation, top-value), which lets you express things like "spending is declining"
  that no single-pass event query could produce.

The practical consequence: **most of the work of segmentation is property design.** By the
time you get to the segment itself, you are usually just writing `property > value`.

---

## 2. Step 1 — Define the event specs

Before any event can be received, its type must be registered as an **event spec**. Events
whose type has no spec are rejected. This is intentional: it prevents typos and unplanned
event types from silently filling the database.

Create specs in the admin UI, or via the API:

```
POST /v1/event-spec
```

```json
{
  "evt": "purchase",
  "description": "A completed order",
  "topics": 1,
  "strval": 1,
  "intval": 1,
  "protected": false,
  "protected_content": false
}
```

### 2.1 The requirement flags

`topics`, `strval` and `intval` are not booleans — they are three-state requirement flags:

| Setting | Stored value | Meaning |
|---|---|---|
| **Required** | `1` (any positive) | The field must be present on the event or it is rejected with `400`. |
| **Optional** | `0` | The field may be present or absent. |
| **Disallowed** | `-1` (any negative) | The field must be missing, empty or zero, or the event is rejected. |

Use `1` for fields your properties will depend on. If `total_spend` reads `intval` from
`purchase` events, marking `intval` as required means a malformed sender fails loudly at
ingest rather than quietly producing wrong segment membership weeks later.

### 2.2 The protection flags

| Flag | Effect |
|---|---|
| `protected` | The event type is excluded from general read and search surfaces (including the MCP/AI interfaces). |
| `protected_content` | The event's content is treated as sensitive and withheld even where the type is listed. |

Use these for event types carrying personal or commercially sensitive detail that should
still drive segmentation but should not be browsable.

### 2.3 Planning your event vocabulary

A few rules of thumb that save rework:

- **Keep the number of event types small.** Distinguish types by *what happened*
  (`purchase`, `page_view`, `email_click`), not by *what it was about* — the subject belongs
  in `topics` and `strval`. Ten well-chosen types are easier to work with than eighty.
- **Decide what goes in `strval` vs `topics` early.** `strval` is the identity of the thing
  (a SKU, an article title, a search term) — one value per event. `topics` is the
  classification of the thing (categories, tags, product groups) — many values per event.
  Properties can group on either, so the split determines what you can aggregate later.
- **Put money and quantities in `intval`.** Store amounts in minor units (øre, cents) so
  they stay integers.
- **Use `src` to record the origin system.** It becomes a filter later, which matters as
  soon as the same event type arrives from two places.

---

## 3. Step 2 — Get the events flowing

Events reach Arrigoo through the browser tracking script, a server-side event endpoint, a
CSV upload or a scheduled API integration. See the
[Integration Guide](integration-guide.md) for the mechanics.

**Verify before moving on.** Property design against events you have not actually inspected
is guesswork. Check what arrived:

```
POST /v1/event-api/search      # by type, customer and date range
GET  /v1/event-api/cid/{cid}   # a specific customer's history
```

Look specifically at:

- Are `topics` populated, and with the values you expected?
- Is `intval` in the units you assumed?
- Is `src` distinguishing your sources?
- Do timestamps look right (`ctime`)?

Everything downstream is built on these fields. Fixing a mapping now costs minutes; fixing
it after twelve properties depend on it costs a full re-aggregation and a review of every
segment.

---

## 4. Step 3 — Design the properties

This is where the real work happens. A property definition answers three questions:

1. **Which events count?** — the *event filters*.
2. **How are they grouped?** — the *group field*.
3. **How are they combined into a value?** — the *accumulator*.

Create properties in the admin UI, or via `POST /v1/property`.

### 4.1 Anatomy of a property

| Field | Meaning |
|---|---|
| `label` | Human-readable name shown in the UI. |
| `sys_title` | Machine name. Used in placeholders (`{{ p.sys_title }}`), calculations and exports. Choose carefully — it is referenced everywhere. |
| `value_type` | The property's data type. Determines storage, available accumulators and segment operators. |
| `description` | What the property means and why it exists. Worth filling in: six months later this is the only record of intent. |
| `event_requirements` | The condition block — filters, grouping, accumulation. |
| `identifier` / `identifier_type` / `identifier_merge` | Marks the property as an identifier (see [4.11](#411-identifier-properties)). |
| `protected` / `protected_content` | Excludes the property from general read surfaces. |

### 4.2 Value types

The value type is the first decision, because it constrains everything after it.

| Type | UI label | Stored in | Holds | Typical use |
|---|---|---|---|---|
| `str` | String | `cust_property_str` | One string | Latest campaign, first landing page, preferred store |
| `int` | Number | `cust_property_int` | One integer (decimals are rounded on write) | Total spend, visit count, order count |
| `float` | Decimal number | `cust_property_int` | One number with decimals | Average order value, a score, a ratio |
| `bool` | Boolean | `cust_property_int` | 1 or absent | "Has purchased", "has consented" |
| `date` | Date | `cust_property_int` | Unix timestamp | First purchase date, last login |
| `datetime` | Date & Time | `cust_property_int` | Unix timestamp | Same, with time-of-day precision |
| `strs` | String List | `cust_property_json` | Ordered list of strings | Last 10 products viewed, article history |
| `map` | Map/Object | `cust_property_json` | Key → number | Spend per category, views per topic |
| `ints` | Number List | `cust_property_json` | List of numbers | Numeric series |
| `tally_slice` | Top Value | `cust_property_json` | Top-N keys of a parent map | Favourite category, top three topics |
| `history` | History | `cust_property_json` | Timestamp → snapshot | Spend measured monthly over a year |
| `trend` | Trend | `cust_property_int` | Difference between two history points | Is spend rising or falling |
| `calculation` | Calculation | `cust_property_int` | Formula over other properties | RFM score, engagement index |

Types `tally_slice`, `history`, `trend` and `calculation` are **derived** — they read other
properties rather than events. See [4.10](#410-derived-properties).

### 4.3 Event filters — deciding which events count

Every condition starts with **event types**: the list of `evt` values this property reads.
An event whose type is not listed is ignored outright. Everything below narrows further.

| Filter | JSON key | Applies to | Behaviour |
|---|---|---|---|
| Event types | `event_type` | `evt` | Required. Event must match one of the listed types exactly. |
| Max Age (Days) | `max_age` | `ctime` | Only events newer than N days count. `0` = no limit. |
| Minimum Age (Days) | `min_age` | `ctime` | Only events older than N days count. `0` = no limit. Pairs with `max_age` to express a window. |
| String Values | `str_val` | `strval` | Event's `strval` must match one of the patterns. |
| Source | `event_src` | `src` | Event's `src` must match one of the patterns. |
| Topic Contains | `topic_contains` | `topics` | At least one of the event's topics must match one of the patterns. |
| URL Contains | `url_contains` | `url` | Event's `url` must match — **substring** match, not exact. |
| Number Values in | `num_val` | `intval` | `intval` must be one of these exact numbers. |
| Number Values not in | `num_val_negative` | `intval` | `intval` must not be one of these numbers. |
| Min Number Value | `num_val_over` | `intval` | `intval` must be strictly greater than this. |
| Max Number Value | `num_val_under` | `intval` | `intval` must be strictly less than this. |

#### Pattern syntax

The four string-list filters (`str_val`, `event_src`, `topic_contains`, `url_contains`)
share a pattern syntax that makes them considerably more expressive than a plain list:

| Prefix | Meaning | Example |
|---|---|---|
| *(none)* | Literal match. Exact for `str_val`, `event_src`, `topic_contains`; substring for `url_contains`. | `outdoor` |
| `!` | **Negation.** The event is excluded if it matches. | `!internal-test` |
| `~` | **Regular expression.** | `~^SKU-1\d{4}$` |
| `!~` | Negated regular expression. | `!~^(test\|staging)` |

Positive and negative patterns can be mixed in one list, and are evaluated independently:

- If any **negative** pattern matches, the event is rejected.
- If positive patterns are present, at least one must match or the event is rejected.
- A list containing only negative patterns acts as a pure exclusion filter — everything
  passes except the matches.

An invalid regular expression never matches; it does not raise an error, so test your
patterns rather than assuming a silent filter is working.

#### Worked filter examples

*Purchases over 500 kr in the last year, excluding test orders:*

```json
{
  "event_type": ["purchase"],
  "max_age":    365,
  "num_val_over": 50000,
  "event_src":  ["!test", "!staging"]
}
```

*Article views in the outdoor section, excluding the index page:*

```json
{
  "event_type":     ["page_view"],
  "topic_contains": ["outdoor"],
  "url_contains":   ["/articles/", "!/articles/index"]
}
```

*Email clicks on any campaign whose name starts with a 2026 date code:*

```json
{
  "event_type": ["email_click"],
  "str_val":    ["~^2026-\\d{2}-"]
}
```

### 4.4 Group Field — how events are bucketed

The **group field** (`aggregate`) decides what the events are grouped by before the
accumulator runs. It is the difference between "how much did they spend" and "how much did
they spend *per category*".

| Option | JSON value | Groups by | Produces |
|---|---|---|---|
| Total (no grouping) | `total` *(default)* | One bucket for everything | A single number |
| Event type | `type` | `evt` | One bucket per event type |
| String value | `str` | `strval` | One bucket per product, article, campaign… |
| Topics | `topic` | Each topic on the event | One bucket per topic — **an event with three topics contributes to all three** |
| Session | `session` | `sid` | One bucket per session |
| Source | `source` | `src` | One bucket per source system |
| Referrer | `referrer` | `ref` | One bucket per referrer |
| URL | `url` | `url` | One bucket per page address |

Grouping is what turns an `int` property into a `map` property. If you group by anything
other than `total`, you almost always want `value_type: map`, because the result is a set
of key → number pairs rather than one number.

The `topic` option is the most useful and the most easily misunderstood: because `topics`
is a list, one event lands in *every* one of its topic buckets. A purchase tagged
`["outdoor", "footwear"]` with `intval: 1299` adds 1299 to both. Sums across topic buckets
therefore exceed the true total — that is correct behaviour for "affinity per topic", but
wrong if you wanted a total.

### 4.5 Accumulators — how values are combined

The accumulator turns each bucket into a number. Which accumulators are meaningful depends
on whether the property is a single number (`int`) or a bucketed map (`map`).

#### For `map` properties (grouped)

| Accumulator | JSON value | Result per bucket |
|---|---|---|
| None | `none` | The number of events in the bucket (same as count) |
| Sum per grouping item | `sum` | Sum of `intval` across the bucket's events |
| Count per grouping item | `count` | Number of events in the bucket |
| Average per grouping item | `avg` | Sum of `intval` ÷ number of events in the bucket |
| Max value per grouping item | `max` | Highest `intval` in the bucket |
| Min | `min` | Lowest `intval` in the bucket |

*Example — spend per category:* group by `topic`, accumulator `sum`, value type `map`.
Result: `{"outdoor": 8400, "footwear": 2100, "camping": 950}`.

*Example — visits per session count:* group by `session`, accumulator `count`, value type
`map`. Result: one entry per session with its page-view count.

#### For `int` properties (single number)

| Accumulator | JSON value | Result |
|---|---|---|
| Count per grouping item | `count` | Total number of qualifying events |
| Sum per grouping item | `sum` | Sum of `intval` across all qualifying events |
| Average per grouping item | `avg` | Total `intval` ÷ total event count |
| Single number average of value | `single_avg_sum` | Same as `avg` — total value ÷ total count |
| Single number average of count | `single_avg_count` | Total event count ÷ number of distinct buckets — i.e. *average events per group* |
| Count of group items | `group_count` | The number of distinct buckets — i.e. *how many different things* |
| Max value per grouping item / Single number max value | `max`, `single_max` | The highest `intval` across all qualifying events |
| Min | `min` | The lowest `intval` across all qualifying events |

The last two are the reason to combine an `int` value type with a non-`total` group field:

- `group_count` grouped by `str` = "how many distinct products have they bought".
- `group_count` grouped by `session` = "how many sessions did they have".
- `single_avg_count` grouped by `session` = "average page views per session".

### 4.6 Thresholds and gates

These do not change the value — they decide whether the property is written **at all**. A
property that fails a gate is removed from the profile, which in turn removes the customer
from any segment built on it. This is how you express "at least" and "at most" conditions
without a segment operator.

| Gate | JSON key | Applies to | Effect |
|---|---|---|---|
| Min Occurrences | `min_occurrences` | Event count | Property is removed unless at least N events qualified. |
| Max Occurrences | `max_occurrences` | Event count | Property is removed if more than N events qualified. |
| Accumulated Value Above | `acc_value_over` | Final value | Property is removed unless the value exceeds this. |
| Accumulated Value Below | `acc_value_under` | Final value | Property is removed unless the value is below this. |

For `map` properties, `acc_value_over` / `acc_value_under` are applied **per bucket** —
buckets outside the range are dropped, and the property is removed only if every bucket
is dropped. This is a clean way to build "categories they actually care about" rather than
"every category they ever touched": set `acc_value_over` to a meaningful floor and the
long tail disappears.

For `min_occurrences` / `max_occurrences` there is a fast pre-check: if the customer's
total event count is already below the minimum or above the maximum before filtering, the
property is skipped without evaluating conditions.

### 4.7 String processing — for `str` and `strs` properties

String properties do not accumulate; they select and collect. These options control what
is read and how it is cleaned up.

| Option | JSON key | Applies to | Effect |
|---|---|---|---|
| Event Field Source | `event_field_source` | `str`, `strs` | Which event field the value is read from: `strval` (default), `intval`, `src`, `url`, or `topics` (list properties only). |
| Event Order | `event_order` | `str`, `int` | `latest` (default) keeps the newest qualifying event; `first` keeps the earliest. |
| Regex Extract | `regex_extract` | `str`, `strs`, `map` keys | Apply a regex and store capture group 1 if present, otherwise the whole match. Values with no match are skipped. |
| Regex Omit | `regex_omit` | `str`, `strs`, `map` keys | Skip any value matching this regex. |
| Unique | `unique` | `strs` | Deduplicate the collected list before applying limits. |
| Min Count | `min_count` | `strs` | Remove the property if fewer than N items were collected. |
| Max Count | `max_count` | `strs` | Truncate the list to at most N items. |

Note the collection order for `strs`: values are collected and then reversed, so the list
reads **oldest first**, and `max_count` truncation therefore keeps the oldest items. If you
want "the last 10 things they viewed", combine a tight `max_age` with `max_count` rather
than relying on truncation order alone.

`regex_extract` is the most useful of these in practice. It lets you derive a clean
dimension from a messy field without changing the sender:

```json
{
  "event_type":         ["page_view"],
  "event_field_source": "url",
  "regex_extract":      "/articles/([a-z-]+)/",
  "unique":             true,
  "max_count":          20
}
```

This turns `https://example.com/articles/winter-hiking/12345?ref=nl` into `winter-hiking`
and collects up to twenty distinct sections.

Applied to a `map` property, `regex_extract` and `regex_omit` operate on the **grouping
key** rather than the value — which is how you strip query strings or normalise URLs before
they become buckets.

`event_order` also works on `int` properties. Setting it bypasses accumulation entirely:
the property takes the chosen single event's `intval`. This is how you express "the value of
their most recent order" as opposed to "their total spend".

### 4.8 Date properties

`date` and `datetime` properties store the timestamp of a qualifying event as a Unix
integer. The only choice is which event:

| Option | JSON value | Result |
|---|---|---|
| Date of latest event | `latest_event` *(default)* | Timestamp of the newest qualifying event |
| Date of first occurring event | `first_event` | Timestamp of the oldest qualifying event |

Set via `event_date`. Combined with event filters, this covers most recency and tenure
needs: `first_event` on `purchase` gives customer tenure; `latest_event` on `page_view`
gives last-seen; `latest_event` on a filtered event type gives "last time they did X".

Because they are stored as timestamps, date properties can be compared against **relative**
values in segments (see [6.4](#64-dates-and-relative-time)), which is what makes
"purchased in the last 30 days" a maintenance-free segment rather than one you have to keep
editing.

### 4.9 Retention — smoothing property removal

By default, when no events qualify on an aggregation pass, the property is **removed**.
With a rolling `max_age` window this causes churn: a customer who bought 89 days ago is in
the segment, and tomorrow they are out — and if they buy again next week, they re-enter,
firing entry actions a second time.

`retention` (days) softens this. When the aggregator would remove a property but the
existing value is younger than the retention window, the value is kept as-is — no write,
no delete, no segment churn.

```json
{ "event_type": ["purchase"], "max_age": 90, "accumulator": "sum", "retention": 30 }
```

Set `retention` to `0` (the default) for immediate removal. Use it wherever an action fires
on segment entry and you do not want repeat firings from boundary flapping.

### 4.10 Derived properties

Four value types read **other properties** instead of events. They declare their inputs in
`parents` (a list of property IDs). If a parent is missing from a profile, the derived
property is removed from it too.

Dependency order is resolved automatically — parents are always computed before children in
the same aggregation run, so you can build several layers deep without scheduling anything.

#### `tally_slice` — top values of a map

Reads a parent `map` property and stores its top-N keys by value.

| Field | Meaning |
|---|---|
| `parents` | The map property to read. |
| `tally_slice` | How many top keys to keep. |

*Example:* parent `spend_per_category` (map), `tally_slice: 1` → `favourite_category`.
With `tally_slice: 3` you get the top three as a list, ready for a "one or more of" segment
condition or a personalisation feed.

This is the standard way to turn a rich behavioural map into something a marketer can
segment on directly.

#### `history` — periodic snapshots

Reads a parent property and appends its current value to a timestamped map, building a time
series.

| Field | Meaning |
|---|---|
| `parents` | The property to snapshot. |
| `history_interval` | Minimum days between snapshots. A run inside the interval writes nothing. |
| `history_limit` | Maximum snapshots to keep. The oldest are discarded first. |

*Example:* parent `total_spend_365d`, `history_interval: 30`, `history_limit: 12` → a
twelve-month rolling record of spend.

The interval is measured from the newest existing snapshot, not from the row's modification
time, so cadence stays correct even when the property row is rewritten for other reasons.

#### `trend` — direction of change

Reads a parent `history` property and stores the difference between two snapshots.

| Field | Meaning |
|---|---|
| `parents` | The history property to read. |
| `trend_target` | Offset of the newer point. `0` = the most recent snapshot. |
| `trend_duration` | How many snapshots back to compare against. Minimum 1. |

The value is `snapshot[target] − snapshot[target + duration]`. A positive number means
growth, negative means decline. With `history_interval: 30`, `trend_target: 0` and
`trend_duration: 3` you get "change in spend over the last three months".

The trend is not computed if the history has fewer than two entries or does not span the
requested window — meaning new customers simply have no trend property rather than a
misleading zero.

#### `calculation` — a formula over other properties

Evaluates an arithmetic expression whose variables are other properties' `sys_title`s.

| Field | Meaning |
|---|---|
| `parents` | The properties used in the formula. |
| `calculation` | The expression. |

```json
{
  "parents":     [12, 15, 19],
  "calculation": "(total_spend_365d / 100) + (order_count_365d * 10) - days_since_last_order"
}
```

Two behaviours worth knowing:

- **`date` and `datetime` parents are converted to "days ago"** before entering the
  formula. A property holding a purchase timestamp arrives as the number of days since that
  purchase — which is exactly what you want for a recency score, and is why the example
  above subtracts it directly.
- **The result is stored as an integer** (truncated). Scale your formula accordingly:
  multiply before you divide.

Calculations are the standard way to build RFM scores, engagement indices and lead scores
that segments can then band into tiers with `between`.

### 4.11 Identifier properties

A property can be marked as an **identifier**, which links its value to the profile's
identity rather than merely describing it.

| Field | Meaning |
|---|---|
| `identifier` / `identifier_type` | Which identifier this property populates: `email`, `phone`, `foreignid1`–`3`, `agillic_id`. |
| `identifier_merge` | When true, if the value matches another existing profile, the two profiles are **merged**. |

This is how anonymous browsing history survives login: an event carrying an email
identifier resolves to the known profile, and `identifier_merge` consolidates the anonymous
profile into it. Enable `identifier_merge` deliberately — merges are not reversible.

---

## 5. Step 4 — Run and verify the aggregation

Properties are not computed on write. They are computed by a scheduled aggregation job.

### 5.1 The cadence

| Job | Default schedule | What it does |
|---|---|---|
| `property assign` | Every minute | Recomputes properties for profiles **queued for update** — those touched by a new event or profile write. |
| `property assign --force` | Nightly, 01:05 | Recomputes properties for **every** profile. Catches time-based changes (a `max_age` window rolling over) that no event triggered. |
| `segment segmentize` | Every 5 minutes | Re-evaluates every segment and updates membership. |
| `segment action` / `trigger-queue-segment-action` | Every minute | Processes queued segment entry/exit actions. |

The incremental run is what makes near-real-time segmentation possible: an event arrives,
the profile is queued, properties are recomputed within a minute, and the segment picks it
up within five. The nightly forced run is what keeps rolling windows honest — a customer
whose last purchase ages out of a 90-day window generates no event, so only the full pass
can notice.

### 5.2 Running it manually

```bash
# Recompute all properties for one customer — the fastest way to test a definition
arrigoocli property assign --cid <customer-id>

# Recompute only specific properties, for all queued profiles
arrigoocli property assign --id 12,15,19

# Full rebuild of everything (expensive — this is the nightly job)
arrigoocli property assign --force

# Re-evaluate one segment
arrigoocli segment segmentize --id 7

# List segments with the properties and segments they depend on
arrigoocli segment list
```

Both jobs take a lock so overlapping runs are skipped; `--force` bypasses the lock.

### 5.3 Verifying

After defining a property, check it on a profile you understand before building anything on
top of it:

```
GET /v1/customer/{cid}
```

Common causes when a property is missing from a profile:

| Symptom | Likely cause |
|---|---|
| Property absent on every profile | No events match the filters — check event types first, then patterns |
| Absent on some profiles | A gate is firing: `min_occurrences`, `acc_value_over`/`_under` |
| Absent, and it is a derived property | A parent property is missing on that profile |
| Value is right but stale | The aggregation has not run since the events arrived |
| A `map` property is empty | Every bucket was dropped by `acc_value_over` / `acc_value_under` |

---

## 6. Step 5 — Build the segment

A segment is a name plus a list of conditions. Each condition names a property (or another
segment), an operator, and one or two values.

```
POST /v1/segment
```

```json
{
  "title":       "High value outdoor customers",
  "sys_title":   "high_value_outdoor",
  "description": "Spent over 5,000 in the last year with outdoor affinity",
  "conditions": [
    { "pt": 12, "field": "int",  "op": ">",         "v1": "500000" },
    { "pt": 19, "field": "strs", "op": "intersect", "v1": "[\"outdoor\"]" },
    { "pt": 22, "field": "date", "op": ">",         "v1": "-90d" }
  ]
}
```

| Condition field | Meaning |
|---|---|
| `pt` | The property ID (or segment ID when `field` is `segment`). |
| `field` | The property's value type — determines which operators are available. |
| `op` | The operator. |
| `v1` / `v2` | The comparison values. `v2` is used only by `between`. |

### 6.1 Conditions are combined with AND

**All conditions must match.** Conditions are joined as a set intersection; there is no OR
between them, and no nesting or grouping.

To express OR, build the alternatives as separate segments and combine them with segment
conditions, or move the logic into the property layer — a `calculation` property or a
carefully filtered property can often express in one condition what would otherwise need a
disjunction.

This constraint is less limiting than it sounds, and it is a large part of why segment
evaluation stays fast and predictable. But it does mean segment design and property design
have to be considered together.

### 6.2 Operators by property type

**String (`str`)**

| Operator | `op` | Matches |
|---|---|---|
| Equals | `=` | Exact value |
| Not equals | `!=` | Anything other than the value |
| Contains | `contains` | Substring match |
| Does not contain | `not_contains` | Substring absent |
| One of | `one_of` | Value is in a comma-separated list |
| Is empty | `is_empty` | No value stored, or an empty string |
| Is not empty | `not_empty` | Any non-empty value |

**Number, Trend, Calculation (`int`, `trend`, `calculation`)**

| Operator | `op` | Matches |
|---|---|---|
| Equals / Not equals | `=`, `!=` | Exact comparison |
| Greater / Less than | `>`, `<`, `>=`, `<=` | Numeric comparison |
| Between | `between` | Inclusive range, `v1` to `v2` |
| One of | `one_of` | Value is in a comma-separated numeric list |
| Is empty / Is not empty | `is_empty`, `not_empty` | Presence of any value |

**String List, Top Value, Number List (`strs`, `tally_slice`, `ints`)**

| Operator | `op` | Matches | Value format |
|---|---|---|---|
| All of | `eq` | The list contains **every** given value | JSON array |
| One or more of | `intersect` | The list contains **at least one** given value | JSON array |
| None of | `not_contains` | The list contains **none** of the given values | JSON array |
| Not equal to | `neq` | The list does not contain all the given values | JSON array |
| Is empty / Is not empty | `is_empty`, `not_empty` | Presence of the property | — |

`intersect` is the workhorse here — it is how "interested in outdoor **or** camping" is
expressed despite conditions being AND-only, because the OR lives inside the single
condition.

`not_contains` also matches customers who have **no value at all** for the property. A
customer with no topic affinities does not contain any of them. This is usually what you
want; be aware of it when the property is sparsely populated.

**Map, History (`map`, `history`)**

Same operators as string lists, matching against the map's **keys**.

**Boolean (`bool`)**

| Operator | `op` | Matches |
|---|---|---|
| True | `true` | A stored value greater than zero |
| False | `false` | **No stored value for this property** |

Note the asymmetry: `false` means *absent*, not *stored as zero*. Because a boolean property
is removed when no events qualify, absence is the normal representation of false — but if a
profile somehow holds a stored `0`, it matches neither `true` nor `false`. Prefer
`not_empty` when you mean "has a value either way".

**Date, Date & Time (`date`, `datetime`)**

| Operator | `op` | Matches |
|---|---|---|
| Later than | `>` | After the given moment |
| Before | `<` | Before the given moment |
| Equals / Not equal to | `=`, `!=` | Exact timestamp comparison |
| Between | `between` | Inclusive range, `v1` to `v2` |
| Is empty / Is not empty | `is_empty`, `not_empty` | Presence of any value |

**Segment (`field: "segment"`)**

| Operator | `op` | Matches |
|---|---|---|
| In | `in` | Currently an active member of the segment |
| Not in | `not_in` | Not currently a member |
| Never in | `never_in` | Has never been a member, at any point |
| Has been in | `was_in` | Was a member and has since left |

### 6.3 Building on other segments

Segment conditions make segments composable, and they cover the cases the AND-only rule
would otherwise make awkward:

- **Layering.** `in: high_value` AND `intersect: ["outdoor"]` — refine a broad audience
  without restating its definition.
- **Exclusion.** `in: newsletter_subscribers` AND `not_in: recently_contacted`.
- **Approximating OR.** Define `outdoor_interest` and `camping_interest` separately, then a
  third segment with `in: outdoor_interest` — and a fourth with `in: camping_interest` — and
  target both in the campaign. Or better: express the OR inside one `intersect` condition.
- **Lifecycle.** `was_in: active_customer` AND `not_in: active_customer` identifies churned
  customers; `never_in: purchasers` identifies genuine prospects, distinct from lapsed ones.

`never_in` versus `not_in` is a distinction worth internalising: `not_in` includes people
who left, `never_in` does not. For win-back campaigns you want `was_in`; for acquisition you
want `never_in`.

### 6.4 Dates and relative time

Date conditions accept **relative** values, which is what keeps rolling segments from
needing maintenance:

| Format | Meaning |
|---|---|
| `-30d` | 30 days ago |
| `-6h` | 6 hours ago |
| `-90m` | 90 minutes ago |
| `7d` | 7 days from now (future-dated properties) |
| `2026-08-19` | Absolute date (`date` properties) |
| `2026-08-19 14:30` | Absolute date and time (`datetime` properties) |

So "purchased in the last 90 days" is `field: date`, `op: >`, `v1: -90d` — evaluated afresh
on every run, with no scheduled edit required. Prefer relative values over absolute ones for
anything recurring; reserve absolute dates for genuinely fixed windows like a campaign
period.

---

## 7. Step 6 — Evaluation, membership and actions

### 7.1 How membership is updated

Every five minutes, each segment's conditions are compiled to a single query and executed.
The result is compared against current membership:

- Customers in the result but not currently members are **added** (entry).
- Customers currently members but not in the result are **deactivated** (exit).
- Membership rows are deactivated, not deleted — which is what makes `was_in` possible.

### 7.2 Triggering actions on entry and exit

Segments become actionable through **segment triggers**. Bind an action to a segment with a
condition of `{"type": "enter"}` or `{"type": "leave"}`, and it fires for each customer
crossing that boundary on an evaluation run. See [Actions](actions.md) and the
[Integration Guide](integration-guide.md#4-real-time-and-event-driven-integrations).

Because triggers fire on the **transition**, not on membership, a customer who stays in a
segment is not re-triggered. This is the main reason to think about property churn: a
property that flickers in and out of existence produces repeated entry events. Use
`retention` ([4.9](#49-retention--smoothing-property-removal)) to prevent it.

For reactions that must be faster than the five-minute segment cycle, use an **event
trigger** (fires within seconds of the event) or a **property update trigger** (fires when a
specific property changes during aggregation) instead.

### 7.3 Where segments are used

| Surface | How |
|---|---|
| Actions | Entry/exit triggers, and `{{ s.segment_name }}` in payload templates |
| On-site personalisation | Returned in the tracking response as `s` on every event |
| Marketing platforms | Pushed as target-group membership via actions |
| Feeds | Personalised content and product feeds scoped by segment |
| Export | `GET /v1/customer/segment/{seg}` |
| Insights | Cohort analysis and LLM-assisted summaries across segments |

---

## 8. A complete worked example

**Goal:** a segment of high-value customers with a demonstrated outdoor affinity whose
spending is not declining, so a loyalty campaign can be targeted at them.

### Step 1 — Event spec

```json
{ "evt": "purchase", "description": "Completed order",
  "topics": 1, "strval": 1, "intval": 1 }
```

Topics carry product categories, `strval` the SKU, `intval` the order value in øre.

### Step 2 — Base properties

**`total_spend_365d`** — value type `int`

```json
{ "event_type": ["purchase"], "max_age": 365,
  "aggregate": "total", "accumulator": "sum",
  "event_src": ["!test"], "retention": 30 }
```

**`spend_per_category`** — value type `map`

```json
{ "event_type": ["purchase"], "max_age": 365,
  "aggregate": "topic", "accumulator": "sum",
  "acc_value_over": 50000 }
```

Only categories with more than 500 kr of spend survive, so the long tail is excluded.

**`last_purchase`** — value type `date`

```json
{ "event_type": ["purchase"], "event_date": "latest_event" }
```

### Step 3 — Derived properties

**`top_categories`** — value type `tally_slice`, parent `spend_per_category`,
`tally_slice: 3`. The three categories they spend most in.

**`spend_history`** — value type `history`, parent `total_spend_365d`,
`history_interval: 30`, `history_limit: 12`. A monthly record of rolling annual spend.

**`spend_trend_3m`** — value type `trend`, parent `spend_history`,
`trend_target: 0`, `trend_duration: 3`. Change in annual spend over the last three months.

### Step 4 — The segment

```json
{
  "title":     "Loyalty campaign — outdoor",
  "sys_title": "loyalty_outdoor",
  "conditions": [
    { "pt": 12, "field": "int",         "op": ">",         "v1": "500000"          },
    { "pt": 21, "field": "tally_slice", "op": "intersect", "v1": "[\"outdoor\",\"camping\"]" },
    { "pt": 15, "field": "date",        "op": ">",         "v1": "-180d"           },
    { "pt": 24, "field": "trend",       "op": ">=",        "v1": "0"               }
  ]
}
```

Read as: spent over 5,000 kr in the last year, **and** outdoor or camping is among their top
three categories, **and** they purchased within the last 180 days, **and** their spending is
not declining.

Note how the OR ("outdoor or camping") lives inside one `intersect` condition rather than
needing a disjunction between conditions, and how "not declining" — impossible to express
against raw events — becomes a simple numeric comparison because the property layer did the
work.

### Step 5 — Act on it

Bind an action to the segment with `{"type": "enter"}` to hand new members to the marketing
platform, and `{"type": "leave"}` to remove them when they no longer qualify.

---

## 9. Design guidance

**Build properties for reuse, segments for campaigns.** Properties are infrastructure and
should be general (`total_spend_365d`); segments are cheap and disposable and can be
specific (`q3_loyalty_push`). If you find yourself defining a property for exactly one
segment, ask whether a slightly more general property would serve several.

**Name `sys_title`s as if you will never rename them.** They appear in calculations, action
templates, exports and feeds. Include the window in the name when it matters —
`total_spend_365d` rather than `total_spend` — so the definition is legible at the call
site.

**Put the OR in the property, the AND in the segment.** Conditions are AND-only, so any
disjunction has to be resolved earlier: multiple event types in one property, multiple
patterns in one filter, multiple values in one `intersect`.

**Filter at ingest, not in the property, where you can.** A property with eight negative
patterns is a sign that unwanted events are being collected in the first place. It is
cheaper to filter at the endpoint mapping.

**Use `max_age` on nearly everything behavioural.** Lifetime aggregates grow monotonically
and stop discriminating. A rolling window keeps segments responsive to current behaviour —
and pair it with `retention` so the window edge does not cause churn.

**Watch out for the boundary.** Any segment built on a rolling window will have customers
crossing in and out. If an action fires on entry, decide explicitly whether re-entry should
re-fire, and use `retention` and action throttling accordingly.

**Test on a known profile first.** `arrigoocli property assign --cid <id>` followed by
`GET /v1/customer/{cid}` on a customer whose history you understand will surface a wrong
filter in seconds. A full rebuild to discover the same thing takes hours.

**Check segment size after the first evaluation.** A segment of zero usually means a gate
removed the property; a segment of everyone usually means a filter is not applied. Both are
faster to diagnose immediately than after a campaign has gone out.

### float vs int

`float` and `int` share `cust_property_int` (the column is `double precision`) and behave
identically in segments — same operators (`=`, `!=`, `>`, `<`, `one_of`, `between`,
is/not empty), same accumulators. The only difference is precision: an `int` (and `trend`
/ `calculation`) value is rounded before it is stored and returned as an integer from the
API, while `float` keeps its decimals end to end.

Protected (encrypted) float properties support exact-match operators only, like every
encrypted value; the match literal is normalised (`1.50` matches `1.5`), but values of
magnitude ≥ 1e15 or < 1e-4 are outside the exact-match guarantee.
