# Building Segments from Scratch

A step-by-step guide to going from raw events to a working, actionable segment using the
Arrigoo admin interface: what each screen does, every option you can choose, and how to
decide between them.

This guide assumes no technical background. Everything described here is done by clicking
through the admin interface — no code, no API calls.

> A companion document, [Segmentation Guide (technical)](segmentation-guide-tech.md),
> covers the same ground in terms of the underlying data model and API for developers.

---

## 1. How segmentation works in Arrigoo

Segmentation is built in three layers. You cannot skip a layer, and understanding why they
are separate is most of the battle.

```
   EVENTS                    PROPERTIES                    SEGMENTS
   ──────                    ──────────                    ────────
   Things that               Facts about a                 Named audiences
   happened                  profile, worked               defined by
                             out from events               property conditions

   "viewed article X"   ──▶  Articles read = 14       ──▶  "Engaged reader"
   "purchased, 1299"         Total spend = 8400            "High value customer"
   "purchased, 2100"         Favourite topic = outdoor     "Outdoor enthusiast"
```

| Layer | Where you manage it | What it is |
|---|---|---|
| **Events** | Configuration → Event types | The raw record that something happened. Never changes once written. |
| **Properties** | Configuration → Properties | A value worked out from those events. Recalculated automatically. |
| **Segments** | Segments | A named group of profiles matching a set of property conditions. |

### Why properties sit in the middle

Segments do not look at events directly. They look at properties. This is deliberate, and
it has three consequences worth knowing:

- **Speed.** A segment like "spent over 5,000 in the last year" would otherwise have to add
  up every purchase for every profile, every time it runs. Instead, the adding-up happens
  once per profile in the property layer, and the segment simply compares one stored number.
- **Reuse.** One "Total spend, last year" property can serve twenty segments, a campaign
  message, a personalisation rule and an export. If that logic lived inside each segment,
  you would have twenty places to update when the definition changes.
- **Transparency.** You can open a profile and see exactly *why* it qualifies — the property
  value is right there on screen. A segment built on hidden calculations cannot be checked
  this way.

The practical consequence: **most of the work of segmentation is designing properties.** By
the time you reach the segment itself, you are usually just saying "this number is bigger
than that one".

---

## 2. Step 1 — Set up your event types

**Where:** Configuration → Event types

Before Arrigoo will accept an event, its type must be registered here. Events of an
unregistered type are rejected. This is deliberate — it stops typos and unplanned event
types from quietly filling up your data.

Click **New event type** and fill in the form:

| Field | What to enter |
|---|---|
| **Event Name** | The machine name, e.g. `purchase`, `page_view`, `email_click`. Lower case, no spaces. |
| **Description** | When and why this event is sent. Worth writing properly — it is the only record of intent later. |
| **Topics** | Whether the event's topic list is Allowed or Required. |
| **String Value** | Whether the event's text value is Allowed or Required. |
| **Integer Value** | Whether the event's number value is Allowed or Required. |
| **Protected** | Hides this event type from general browsing and from the AI/Insights features. |
| **Protected Content** | Treats the event's content as sensitive and withholds it even where the type is visible. |

### Understanding the three event fields

Every event in Arrigoo carries the same small set of fields. This is what makes the system
flexible — a purchase, a page view and an email click are all the same shape, and the same
property tools work on all of them.

| Field | Holds | Examples |
|---|---|---|
| **Topics** | A list of classifications | `["outdoor", "footwear"]`, `["news", "politics"]` |
| **String Value** | The identity of the thing, one value | A product code, an article title, a search term, a campaign name |
| **Integer Value** | A number | An order value, a quantity, a score, seconds spent |
| Source | Which system sent it | `webshop`, `crm`, `newsletter` |
| URL | The page it relates to | The article or product page |

### Choosing Allowed vs Required

- **Allowed** — the field may be present or absent. Use when it genuinely varies.
- **Required** — the event is rejected if the field is missing.

Set a field to **Required** whenever a property will depend on it. If your "Total spend"
property reads the Integer Value of purchase events, marking Integer Value as Required
means a broken sender fails loudly and immediately, rather than quietly producing wrong
segment membership that nobody notices for weeks.

> **Caution:** the dropdown also offers **Disallowed**. It does not currently work as its
> name suggests — an event type saved with Disallowed behaves as though the field were
> Required. Use Allowed or Required only, until this is corrected.

### Planning your event vocabulary

A few rules of thumb that save a lot of rework:

- **Keep the number of event types small.** Distinguish types by *what happened*
  (`purchase`, `page_view`, `email_click`), not by *what it was about* — the subject belongs
  in Topics and String Value. Ten well-chosen types are far easier to work with than eighty.
- **Decide the split between String Value and Topics early.** String Value is the identity
  of the thing (one per event). Topics is the classification of the thing (many per event).
  Properties can group on either, so this choice determines what you can analyse later.
- **Put money and quantities in Integer Value.** Store amounts in the smallest unit (øre,
  cents) so they stay whole numbers.
- **Make sure Source is set** by whoever sends the events. It becomes a useful filter the
  moment the same event type starts arriving from two places.

---

## 3. Step 2 — Check the events are arriving

**Where:** Data → Events

Before designing properties, look at what actually arrived. Designing against events you
have not inspected is guesswork.

Use the event log to filter by type, profile and date range, and check:

- Are **Topics** populated, and with the values you expected?
- Is **Integer Value** in the units you assumed — kroner or øre?
- Does **Source** distinguish your different systems?
- Do the timestamps look right?

You can also open any individual profile (Data → Profiles) and use its **Events** tab to see
that person's full history in order.

Everything downstream is built on these fields. Fixing a mapping now takes minutes. Fixing
it after twelve properties depend on it means a full recalculation and a review of every
segment.

---

## 4. Step 3 — Design your properties

**Where:** Configuration → Properties → **New property**

This is where the real work happens. A property definition answers three questions:

1. **Which events count?** — the Event Conditions
2. **How are they grouped?** — the Group Field
3. **How are they turned into a value?** — the Accumulator

### 4.1 The property form

The top of the form covers identity:

| Field | What to enter |
|---|---|
| **Property Name** | The human name shown throughout the interface, e.g. "Total spend, last year". |
| **System name** | The machine name, e.g. `total_spend_365d`. Used in campaign messages, calculations and exports. Choose carefully — it is referenced in many places and renaming it is disruptive. |
| **Description** | What this property means and why it exists. |
| **Value Type** | The kind of value it holds. See below. |
| **Retention (days)** | How long to keep the value when it would otherwise be removed. See [4.10](#410-retention--stopping-people-flickering-in-and-out). |
| **Protected** / **Protected Content** | Hides the property from general browsing and from the AI/Insights features. |

Below that, the condition builder has three sections: **Event Types to Match**,
**Event Conditions**, and **Property Value Processing**. Fields are added to the last two
by clicking **Add Event Condition** / **Add Processing Condition** and picking from the
list — you only ever see the options you have chosen to use, so the form stays readable.

### 4.2 Value Type — the first decision

The Value Type constrains everything after it: which options appear in the condition
builder, and which comparisons are available when you get to the segment.

| Value Type | Holds | Use it for |
|---|---|---|
| **String** | One piece of text | Latest campaign, first landing page, preferred store |
| **Number** | One whole number | Total spend, visit count, number of orders |
| **Boolean** | Yes / no | "Has purchased", "has given consent" |
| **Date** | A date | First purchase date, last login |
| **Date & Time** | A date with a time | The same, where time of day matters |
| **String List** | An ordered list of text values | Last 10 products viewed, article reading history |
| **Map/Object** | A set of labelled numbers | Spend per category, views per topic |
| **Number List** | A list of numbers | A numeric series |
| **Top Value** | The highest-scoring entries from a Map | Favourite category, top three topics |
| **History** | Snapshots taken over time | Spend measured monthly across a year |
| **Trend** | The change between two snapshots | Is spending rising or falling |
| **Calculation** | A formula using other properties | Loyalty score, engagement index |

The last four — **Top Value**, **History**, **Trend** and **Calculation** — are *derived*.
They read other properties rather than events, and their forms look different. See
[4.11](#411-derived-properties--building-on-other-properties).

### 4.3 Event Types to Match

The first thing to set. Tick the event types this property should read. Events of any other
type are ignored completely.

You can tick more than one. This is useful when several event types mean the same thing for
your purposes — for example an "engagement" property that counts `page_view`,
`video_play` and `download` together.

### 4.4 Event Conditions — narrowing down which events count

Click **Add Event Condition** to add any of these filters. Each one narrows the set of
events further.

| Condition | What it does |
|---|---|
| **Max Age (Days)** | Only count events from the last N days. Leave unset for no limit. |
| **String Values** | Only count events whose String Value matches one of these. |
| **Topic Contains** | Only count events with one of these topics. |
| **Source** | Only count events from one of these source systems. |
| **URL Contains** | Only count events whose URL contains one of these. |
| **Number Values in** | Only count events whose Integer Value is one of these exact numbers. |
| **Number Values not in** | Exclude events whose Integer Value is one of these numbers. |
| **Min Number Value** | Only count events whose Integer Value is above this. |
| **Max Number Value** | Only count events whose Integer Value is below this. |

#### The pattern shortcuts

The four text filters — String Values, Topic Contains, Source and URL Contains — accept more
than plain values. Two prefixes make them far more powerful:

| Type this | Meaning | Example |
|---|---|---|
| `outdoor` | Match this value | Matches the topic "outdoor" |
| `!test` | **Exclude** anything matching this | Excludes the source "test" |
| `~^SKU-1` | Match a **pattern** (regular expression) | Matches any product code starting `SKU-1` |
| `!~^(test\|staging)` | Exclude anything matching a pattern | Excludes test and staging sources |

You can mix included and excluded values in the same box:

- If any **excluded** value matches, the event is skipped.
- If you have listed any **included** values, at least one must match.
- A box containing only exclusions passes everything except those matches — a convenient
  way to say "all sources except our test system".

Note that **URL Contains** matches anywhere in the URL, whereas String Values, Topic
Contains and Source must match the whole value exactly (unless you use a `~` pattern).

If you use a pattern that is not valid, it simply never matches — you will not see an error
message. Test your patterns by checking a known profile rather than assuming they work.

#### Worked examples

*Purchases over 500 kr in the last year, ignoring test orders:*

- Event Types to Match: `purchase`
- Max Age (Days): `365`
- Min Number Value: `50000` (øre)
- Source: `!test`, `!staging`

*Article views in the outdoor section, but not the section index page:*

- Event Types to Match: `page_view`
- Topic Contains: `outdoor`
- URL Contains: `/articles/`, `!/articles/index`

*Clicks on any campaign whose name starts with a 2026 date code:*

- Event Types to Match: `email_click`
- String Values: `~^2026-\d{2}-`

### 4.5 Group Field — how events are bucketed

Found under **Property Value Processing**, the **Group Field** decides what the events are
grouped by before they are counted or added up. It is the difference between "how much did
they spend" and "how much did they spend *per category*".

| Group Field | Groups events by | Result |
|---|---|---|
| **Total (no grouping)** | Nothing — everything in one bucket | A single number |
| **Event type** | The event type | One bucket per event type |
| **String value** | The String Value | One bucket per product, article or campaign |
| **Topics** | Each topic on the event | One bucket per topic |
| **Session** | The visit | One bucket per session |
| **Source** | The sending system | One bucket per source |
| **Referrer** | Where the visitor came from | One bucket per referrer |

If you choose anything other than **Total**, you will usually want the Value Type to be
**Map/Object**, because the result is a set of labelled numbers rather than one number.

**Topics deserves special attention.** Because an event can carry several topics, one event
lands in *every* one of its topic buckets. A purchase of 1,299 kr tagged both "outdoor" and
"footwear" adds 1,299 to both. The buckets therefore add up to more than the true total —
which is exactly right for measuring *affinity per topic*, and exactly wrong if you wanted
a total. Use Total grouping for totals.

> **Note:** the Group Field list also contains **URL**. This option is not currently
> functional and will produce an empty property. To analyse by URL, use a String List
> property with Event Field Source set to URL instead.

### 4.6 Accumulator — how the values are combined

Also under **Property Value Processing**, the **Accumulator** turns each bucket into a
number. Which choices make sense depends on whether you are building a single **Number** or
a bucketed **Map/Object**.

#### For Map/Object properties

| Accumulator | Result for each bucket |
|---|---|
| **None** | How many events fell into the bucket |
| **Sum per grouping item** | The Integer Values added together |
| **Count per grouping item** | How many events fell into the bucket |
| **Average per grouping item** | The average Integer Value in that bucket |

*Example — spend per category:* Group Field **Topics**, Accumulator **Sum per grouping
item**, Value Type **Map/Object**. Result: outdoor 8,400 · footwear 2,100 · camping 950.

#### For Number properties

| Accumulator | Result |
|---|---|
| **Count per grouping item** | The total number of matching events |
| **Sum per grouping item** | All Integer Values added together |
| **Average per grouping item** | Total value divided by number of events |
| **Single number average of value** | The same — total value divided by number of events |
| **Single number average of count** | Average number of events per group |
| **Count of group items** | How many *different* buckets there were |

The last two are the reason to combine a **Number** value type with a Group Field other
than Total:

- **Count of group items** grouped by **String value** = "how many different products have
  they bought"
- **Count of group items** grouped by **Session** = "how many visits have they made"
- **Single number average of count** grouped by **Session** = "average page views per visit"

> **Caution:** the list also offers **Max value per grouping item**, **Min**, and **Single
> number max value**. These do not currently work for single **Number** properties — the
> result will be `0`. Do not build segments on them. To find someone's largest single order,
> build a **Map/Object** grouped by String value and read the top entry with a **Top Value**
> property.

### 4.7 Thresholds — requiring a minimum before the property exists

These options do not change the value. They decide whether the property is saved to the
profile **at all**. If a threshold is not met, the property is removed — which in turn
removes that person from any segment built on it.

| Option | Effect |
|---|---|
| **Min Occurrences** | The property only exists if at least N events matched. |
| **Max Occurrences** | The property is removed if more than N events matched. |
| **Accumulated Value Above** | The property only exists if the final value is above this. |
| **Accumulated Value Below** | The property only exists if the final value is below this. |

This is how you say "at least three purchases" without needing a segment condition for it.

For **Map/Object** properties, the two Accumulated Value options are applied to *each
bucket* — buckets outside the range are dropped, and the property only disappears entirely
if every bucket is dropped. This is the neat way to build "categories they genuinely care
about" instead of "every category they have ever touched": set a sensible floor and the long
tail of one-off purchases disappears.

### 4.8 Working with text and lists

These options appear for **String** and **String List** properties, which select and collect
values rather than adding them up.

| Option | Available for | What it does |
|---|---|---|
| **Event Field Source** | String, String List | Which part of the event to read: String value, Number value, Source, URL, or Topics (lists only). |
| **Event Order** | String, Number | **Latest occurrence** (default) or **First occurrence** — which matching event to take the value from. |
| **Regex Extract** | String, String List, Map | Pull a piece out of the value using a pattern. Values with no match are skipped. |
| **Regex Omit** | String, String List, Map | Skip any value matching this pattern. |
| **Unique** | String List | Remove duplicates from the collected list. |
| **Min Count** | String List | Remove the property if fewer than N items were collected. |
| **Max Count** | String List | Keep at most N items in the list. |

**Regex Extract** is the most useful of these in practice. It lets you pull a clean value out
of a messy one without changing anything at the sending end. For example, reading the URL
`https://example.com/articles/winter-hiking/12345?ref=nl` with the pattern
`/articles/([a-z-]+)/` gives you `winter-hiking` — turning raw page addresses into a tidy
list of sections the person reads.

Two behaviours to be aware of:

- **Collected lists are ordered oldest first**, so **Max Count** keeps the oldest entries.
  If you want "the last 10 things they viewed", combine a short **Max Age (Days)** with
  **Max Count** rather than relying on the ordering alone.
- Applied to a **Map/Object** property, Regex Extract and Regex Omit work on the *bucket
  labels* rather than the values — which is how you tidy up URLs before they become buckets.

**Event Order** also works on **Number** properties, where it changes the behaviour
completely: instead of adding events up, the property takes the value of that one event.
This is how you express "the value of their most recent order" as distinct from "their total
spend".

### 4.9 Dates

**Date** and **Date & Time** properties store when something happened. The only choice is
which event to take:

| Event date | Result |
|---|---|
| **Date of latest event** (default) | When they most recently did this |
| **Date of first occurring event** | When they first did this |

Combined with event conditions, this covers most recency needs. First occurrence on
`purchase` gives you customer tenure. Latest occurrence on `page_view` gives you last-seen.
Latest occurrence on a filtered event type gives you "the last time they did X".

Date properties can be compared against **relative** times in segments — "in the last 90
days" — which is what lets you build a rolling segment that never needs editing. See
[6.4](#64-working-with-dates).

### 4.10 Retention — stopping people flickering in and out

By default, if no events match on a recalculation, the property is removed from the profile.
With a rolling window this causes churn: someone who bought 89 days ago is in your segment,
and tomorrow they are out. If they buy again next week they come back in — and any campaign
that fires on entry fires a second time.

**Retention (days)** smooths this over. If the property would be removed, but the existing
value is younger than the retention period, it is left alone instead.

*Example:* a "Total spend, last 90 days" property with Retention set to `30` keeps its value
for a further month after the last qualifying purchase ages out, rather than vanishing the
moment the window rolls past.

Leave it at `0` for immediate removal. Set it wherever a campaign fires on segment entry and
you do not want repeat firings from people hovering on the boundary.

### 4.11 Derived properties — building on other properties

Four value types read *other properties* instead of events. Choose one of these Value Types
and the form changes to ask for a source property rather than event conditions.

Arrigoo works out the correct order automatically, so you can build several layers deep
without arranging anything yourself.

#### Top Value — the leaders from a Map

Reads a Map/Object property and keeps its highest-scoring entries.

| Field | What to enter |
|---|---|
| **Property to Slice** | The Map/Object property to read |
| **Slice Value** | How many top entries to keep |

*Example:* reading "Spend per category" with Slice Value `1` gives you "Favourite category".
With `3` you get their top three, ready to use in a segment condition or a personalised
content feed.

This is the standard way to turn a rich behavioural map into something you can segment on
directly.

#### History — snapshots over time

Reads a property and records its value periodically, building up a picture over time.

| Field | What to enter |
|---|---|
| **Source property** | The property to snapshot |
| **Interval (days)** | How often to take a snapshot |
| **Max records** | How many snapshots to keep. The oldest are dropped. |

*Example:* snapshotting "Total spend, last year" every `30` days with a limit of `12` gives
you a twelve-month rolling record of spending.

#### Trend — the direction of change

Reads a History property and works out whether the number is going up or down.

| Field | What to enter |
|---|---|
| **History property** | The History property to read |
| **Target** | Which snapshot to measure from. `0` is the most recent. |
| **Duration** | How many snapshots back to compare against |

The result is positive for growth, negative for decline. With monthly snapshots, a Target of
`0` and a Duration of `3` gives you "change in spending over the last three months".

If there is not enough history yet, the property simply does not exist on that profile —
rather than showing a misleading zero. New customers therefore fall out of trend-based
segments naturally.

#### Calculation — a formula across properties

Works out a number from other properties using a formula. Enter it in the **Equation** field
using each property's system name as a variable:

```
(total_spend_365d / 100) + (order_count_365d * 10) - days_since_last_order
```

Two things to know:

- **Date properties become "days ago"** when used in a formula. A property holding someone's
  last purchase date arrives in the equation as the number of days since that purchase —
  which is exactly what you want for a recency score, and why the example above subtracts it.
- **The result is rounded down to a whole number.** Multiply before you divide so you do not
  lose precision.

Calculations are the standard way to build loyalty scores, engagement indices and lead
scores, which segments can then band into tiers.

### 4.12 Identifier properties

A property can be marked as an **identifier**, which means its value identifies the person
rather than just describing them.

| Field | Effect |
|---|---|
| **Identifier Type** | Which identifier this fills: Email, Phone, or one of your custom ID fields. |
| **Identifier Merge** | If the value matches another existing profile, the two profiles are combined into one. |

This is how anonymous browsing history survives someone logging in or clicking through from
an email. Turn on **Identifier Merge** deliberately — merging profiles cannot be undone.

---

## 5. Step 4 — Wait for the calculation, then check it

Properties are not worked out the instant an event arrives. They are recalculated by a
background job on a schedule.

### 5.1 What runs when

| What | How often | What it covers |
|---|---|---|
| Property recalculation | Every minute | Profiles that have changed — anyone with a new event or updated data |
| Full property rebuild | Nightly, around 01:00 | Every profile, to catch time-based changes |
| Segment evaluation | Every 5 minutes | All segments, updating who is in and who is out |
| Segment actions | Every minute | Campaigns and integrations triggered by people entering or leaving |

So in normal use: an event arrives, the property updates within about a minute, and segment
membership catches up within five.

The nightly rebuild matters more than it sounds. If someone's last purchase drifts outside a
90-day window, *nothing happens* to trigger a recalculation — no event arrives, because the
change is simply the passage of time. Only the full nightly pass notices. This is why
rolling-window segments update overnight rather than to the minute.

### 5.2 Checking your property works

**Where:** Data → Profiles → open a profile → **Profile Properties**

Pick someone whose history you actually understand and check the value is what you expect.
This takes seconds and catches most mistakes.

If the property is missing or wrong:

| What you see | Most likely cause |
|---|---|
| Missing on every profile | No events match. Check Event Types to Match first, then your text filters. |
| Missing on some profiles | A threshold is removing it — Min Occurrences, or an Accumulated Value limit. |
| Missing, and it is a derived property | The source property is missing on that profile. |
| The value is `0` on a Number property | The Accumulator is Max, Min or Single number max — see the caution in [4.6](#46-accumulator--how-the-values-are-combined). |
| The value is right but out of date | The recalculation has not run since the events arrived. Wait a minute and refresh. |
| A Map/Object property is empty | Every bucket was excluded by the Accumulated Value thresholds. |

---

## 6. Step 5 — Build the segment

**Where:** Segments → **New segment**

| Field | What to enter |
|---|---|
| **Title** | The name people will see, e.g. "High value outdoor customers". |
| **System name** | The machine name, used in campaigns and integrations. |
| **Description** | What this audience represents and what it is for. |

Then click **Add condition** for each rule. Each condition asks for a property, a comparison
and a value.

### 6.1 All conditions must be true

**Conditions are combined with AND.** Everyone in the segment matches every condition. There
is no OR between conditions, and no bracketing or grouping.

This sounds more limiting than it is in practice, and there are three ways round it:

- **Put the OR inside one condition.** List properties offer "One or more of", which covers
  "interested in outdoor *or* camping" in a single condition.
- **Put the OR in the property.** A property can match several event types, and its text
  filters accept several values at once.
- **Build separate segments and combine them.** Segments can be used as conditions in other
  segments (see [6.3](#63-building-segments-from-other-segments)).

The trade-off is deliberate: it is what keeps segment evaluation fast and predictable across
every profile in the system. But it does mean property design and segment design have to be
thought about together.

### 6.2 The comparisons available

Which comparisons you see depends on the property's Value Type.

**String properties**

| Comparison | Matches |
|---|---|
| Equals / Not equals | Exactly this value, or anything else |
| Contains / Does not contain | The text appears anywhere in the value |
| One of | The value is in a comma-separated list you provide |
| Is empty / Is not empty | Whether there is any value at all |

**Number, Trend and Calculation properties**

| Comparison | Matches |
|---|---|
| Equals / Not equals | Exactly this number, or anything else |
| Greater than / Less than | Straightforward numeric comparison |
| Between | Anything within a range, inclusive |
| One of | The value is in a comma-separated list of numbers |
| Is empty / Is not empty | Whether there is any value at all |

**String List, Top Value and Number List properties**

| Comparison | Matches |
|---|---|
| **All of** | The list contains *every* value you give |
| **One or more of** | The list contains *at least one* of the values you give |
| **None of** | The list contains *none* of the values you give |
| Not equal to | The list does not contain all the given values |
| Is empty / Is not empty | Whether the property exists at all |

**One or more of** is the workhorse here — it is how you express an OR despite conditions
being AND-only, because the OR lives inside the single condition.

Be aware that **None of** also matches people who have *no value at all* for that property.
Someone with no recorded interests does not contain any of them. This is usually what you
want, but worth remembering when the property is only filled in for a minority of profiles.

**Map/Object and History properties**

The same comparisons as lists, matched against the bucket labels.

**Boolean properties**

| Comparison | Matches |
|---|---|
| **True** | The property has a value |
| **False** | The property has *no* value |

Note the asymmetry: False means "not recorded", not "recorded as no". Because a boolean
property is removed when nothing matches, absence is the normal way "no" is represented.

**Date and Date & Time properties**

| Comparison | Matches |
|---|---|
| Later than / Before | After or before the given moment |
| Equals / Not equal to | Exactly this moment |
| Between | Within a range |
| Is empty / Is not empty | Whether the property exists at all |

**Segment conditions**

| Comparison | Matches |
|---|---|
| **In** | Currently a member of that segment |
| **Not in** | Not currently a member |
| **Never in** | Has never been a member, at any point |
| **Has been in** | Was a member and has since left |

### 6.3 Building segments from other segments

Segment conditions make segments composable, and they cover most of what the AND-only rule
would otherwise make awkward:

- **Refining.** *In* "High value" AND *One or more of* outdoor — narrow a broad audience
  without restating its definition.
- **Excluding.** *In* "Newsletter subscribers" AND *Not in* "Contacted this week".
- **Win-back.** *Has been in* "Active customers" AND *Not in* "Active customers" finds people
  who have lapsed.
- **Acquisition.** *Never in* "Purchasers" finds genuine prospects — as distinct from people
  who used to buy and stopped.

The difference between **Not in** and **Never in** is worth internalising: *Not in* includes
people who left, *Never in* does not. For win-back campaigns you want *Has been in*; for
acquisition you want *Never in*.

### 6.4 Working with dates

Date conditions accept **relative** values, which is what keeps rolling segments from needing
constant maintenance:

| Type this | Meaning |
|---|---|
| `-30d` | 30 days ago |
| `-6h` | 6 hours ago |
| `-90m` | 90 minutes ago |
| `2026-08-19` | A specific date |
| `2026-08-19 14:30` | A specific date and time |

So "purchased in the last 90 days" is a Date property, **Later than**, `-90d`. This is
re-evaluated every time the segment runs, so it stays correct forever without you touching
it.

Use relative values for anything recurring. Save specific dates for genuinely fixed windows,
such as a particular campaign period.

### 6.5 Test before you save

Click **Test segment size** at the bottom of the form. It tells you how many profiles
currently match, without saving anything.

This is the single most valuable habit in segment building:

- **Zero matches** almost always means a threshold removed a property, or a filter is too
  narrow. Check the property on a known profile.
- **Everyone matches** almost always means a condition is not doing what you think — often a
  Contains that is too loose, or an Is not empty where you meant a value comparison.
- **A plausible number** means you are probably right, but check a couple of individual
  profiles anyway before sending anything.

---

## 7. Step 6 — Using the segment

### 7.1 What you see on the segment page

**Where:** Segments → open a segment

| Section | Shows |
|---|---|
| **Active Members** | How many profiles are currently in the segment |
| **Former Members** | How many have been in it and left |
| **Total Profiles** | The size of your whole database, for context |
| **Conditions** | The rules, in readable form |
| **Member Profiles** | The actual people, clickable through to their profiles |
| **Total number of profiles the last 30 days** | How the segment has grown or shrunk |
| **Number of profiles entering and leaving the last 30 days** | The churn — useful for spotting boundary flapping |

That last chart is worth watching. High entering *and* leaving figures on a stable-sized
segment mean people are hovering on a boundary, and any campaign firing on entry will be
firing repeatedly at the same people. That is what **Retention** on the underlying property
is for.

### 7.2 Triggering campaigns and integrations

**Where:** Actions → Triggers → **New trigger**

Choose the **Segment** trigger type, pick your segment, and choose whether it fires when
someone **enters** or **leaves**. Then select the action to run — a webhook, an email, a
handover to your marketing platform.

Because triggers fire on the *transition*, someone who stays in the segment is not triggered
again. This is the main reason to care about property churn: a property that keeps appearing
and disappearing produces repeated entry events for the same person.

If you need something to happen faster than the five-minute segment cycle, use an **Event**
trigger (fires within seconds of the event) or a **Property update** trigger (fires when a
specific property changes) instead of a segment trigger.

You can also run an action against everyone in a segment on demand: open the segment, go to
the **Actions** tab, choose an action and click **Trigger Now**.

### 7.3 Where else segments appear

| Where | How it is used |
|---|---|
| Website personalisation | Segment membership is available to your website on every page view |
| Marketing platforms | Pushed as target group membership via actions |
| Content and product feeds | Personalised recommendations scoped by segment |
| Insights | Cohort analysis and AI-assisted summaries comparing segments |
| Export | Download the member list |

---

## 8. A complete worked example

**Goal:** find high-value customers with a genuine outdoor interest whose spending is not
declining, so a loyalty campaign can be targeted at them.

### Step 1 — Event type

Configuration → Event types → New:

- **Event Name:** `purchase`
- **Description:** Completed order
- **Topics:** Required (product categories)
- **String Value:** Required (product code)
- **Integer Value:** Required (order value in øre)

### Step 2 — Base properties

**"Total spend, last year"** — Value Type **Number**

- Event Types to Match: `purchase`
- Max Age (Days): `365`
- Source: `!test`
- Group Field: **Total (no grouping)**
- Accumulator: **Sum per grouping item**
- Retention (days): `30`

**"Spend per category"** — Value Type **Map/Object**

- Event Types to Match: `purchase`
- Max Age (Days): `365`
- Group Field: **Topics**
- Accumulator: **Sum per grouping item**
- Accumulated Value Above: `50000`

Only categories with more than 500 kr of spend survive, so occasional purchases do not
register as interests.

**"Last purchase"** — Value Type **Date**

- Event Types to Match: `purchase`
- Event date: **Date of latest event**

### Step 3 — Derived properties

**"Top categories"** — Value Type **Top Value**
Property to Slice: *Spend per category* · Slice Value: `3`

**"Spend history"** — Value Type **History**
Source property: *Total spend, last year* · Interval: `30` days · Max records: `12`

**"Spend trend, 3 months"** — Value Type **Trend**
History property: *Spend history* · Target: `0` · Duration: `3`

### Step 4 — The segment

Segments → New segment. Title: "Loyalty campaign — outdoor". Four conditions:

| Property | Comparison | Value |
|---|---|---|
| Total spend, last year | Greater than | `500000` |
| Top categories | One or more of | `outdoor`, `camping` |
| Last purchase | Later than | `-180d` |
| Spend trend, 3 months | Greater than or equal | `0` |

Read as: spent over 5,000 kr in the last year, **and** outdoor or camping is among their top
three categories, **and** they bought something in the last six months, **and** their
spending is not falling.

Click **Test segment size** before saving.

Note how the "outdoor *or* camping" OR sits inside a single condition, and how "spending is
not falling" — impossible to express against raw events — becomes a simple numeric
comparison because the property layer did the work first.

### Step 5 — Act on it

Actions → Triggers → New trigger. Segment trigger, this segment, on **enter**, running your
marketing platform handover action. Add a second trigger on **leave** to remove them when
they no longer qualify.

---

## 9. Practical advice

**Build properties for reuse, segments for campaigns.** Properties are infrastructure and
should be general — "Total spend, last year". Segments are cheap and disposable and can be
as specific as you like — "Q3 loyalty push". If you find yourself creating a property for
exactly one segment, ask whether a slightly more general version would serve several.

**Name system names as though you can never change them.** They appear in campaign messages,
calculations, exports and feeds. Include the time window in the name when it matters —
`total_spend_365d` rather than `total_spend` — so anyone reading it knows what it means.

**Put the OR in the property, the AND in the segment.** Since conditions are AND-only, any
"either/or" has to be resolved earlier: several event types in one property, several values
in one filter, or "One or more of" in a single condition.

**Use Max Age on nearly everything behavioural.** Lifetime totals only ever grow, and
eventually stop telling you anything useful. A rolling window keeps segments responsive to
what people are doing *now* — and pair it with Retention so the edge of the window does not
cause churn.

**Think about the boundary before you attach a campaign.** Any segment built on a rolling
window will have people crossing in and out. Decide deliberately whether re-entry should
re-send, and use Retention accordingly.

**Test on a profile you know.** Recalculating one known profile and looking at the result
will surface a wrong filter in seconds. Guessing from segment sizes takes far longer.

**Check the segment size before you use it.** Both zero and everybody are much faster to
diagnose the moment you create the segment than after a campaign has gone out.
