# Arrigoo CDP statistics API

The numbers on the dashboard and the other charts in the interface all come from the
statistics endpoints under `/v1/stat`. This page describes each of them.

Authentication works as for the rest of the API: get an access token with an API key and
send it as a bearer token. See [the API guide](api.md#authentication). Every statistics
endpoint is a `GET` and can be called with any API key, whatever its role.

## Freshness

Most statistics are read from aggregated datasets that are rebuilt during the night. Some
endpoints add the current day on top, read live from the database:

| Endpoint | History | Today |
|---|---|---|
| `event`, `action`, `import`, `user` | Nightly aggregate | Live |
| `segment-day/{sid}`, `segment-total-daily/{sid}` | Nightly aggregate | Live |
| Everything else | Nightly aggregate | Not included |

The property distributions (`property-int`, `property-float`, `property-str`) are the
exception: they are always computed live.

## Response format

Nearly all endpoints answer with the same structure:

```json
{
    "types": ["pageview", "login"],
    "entries": [
        {
            "total": 595,
            "entry": "2025-02-25T00:00:00Z",
            "stats": {
                "pageview": 466,
                "login": 129
            }
        }
    ]
}
```

- `types` lists the keys that can appear in `stats`, so a chart knows its series up front.
- `entries` holds one row per day, month, segment or value, depending on the endpoint.
- `entry` is the label of the row. Days are given as `YYYY-MM-DDT00:00:00Z`.
- `total` is the number for the row, normally the sum of `stats`.
- `stats` is the breakdown by type. It is `null` on endpoints that have no breakdown.

Endpoints with a time window return days in ascending order, and days without data are
included with zero counts.

The two profile total endpoints (`customer-total` and `customer-total-history`) use a
different format, described with them.

## Overview

| Endpoint | What it returns |
|---|---|
| [`GET /v1/stat/event`](#events-per-day) | Tracking events per day, by event type |
| [`GET /v1/stat/event/month`](#events-per-month) | Tracking events per month |
| [`GET /v1/stat/action`](#action-runs-per-day) | Action runs per day, by action |
| [`GET /v1/stat/import`](#import-runs-per-day) | Import runs per day, by integration |
| [`GET /v1/stat/log-daily`](#log-entries-per-day) | Log entries per day for one action or integration, by level |
| [`GET /v1/stat/user`](#new-profiles-per-day) | New profiles per day |
| [`GET /v1/stat/customer-total`](#profile-total) | Number of profiles and identified profiles |
| [`GET /v1/stat/customer-total-history`](#profile-total-history) | The same, per day |
| [`GET /v1/stat/profile-count-identified`](#identified-profiles-per-day) | New profiles per day, identified and unidentified |
| [`GET /v1/stat/profile-merged`](#merged-profiles-per-day) | Profiles merged per day |
| [`GET /v1/stat/segment`](#segment-members) | Members of every segment |
| [`GET /v1/stat/segment/{sid}`](#segment-members) | Members of one segment |
| [`GET /v1/stat/segment-total`](#segment-totals) | Members of every segment, with totals |
| [`GET /v1/stat/segment-total/{sid}`](#segment-totals) | Members of one segment, with totals |
| [`GET /v1/stat/segment-total-daily/{sid}`](#segment-size-per-day) | Size of one segment per day |
| [`GET /v1/stat/segment-day/{sid}`](#segment-changes-per-day) | Profiles entering and leaving one segment per day |
| [`GET /v1/stat/property-int/{pid}`](#number-property-distribution) | Value distribution of a number property |
| [`GET /v1/stat/property-float/{pid}`](#number-property-distribution) | Value distribution of a decimal property |
| [`GET /v1/stat/property-str/{pid}`](#text-property-distribution) | Most common values of a text property |

`{sid}` is the numeric ID of a segment and `{pid}` the numeric ID of a property. Both are
returned by `GET /v1/segment` and `GET /v1/property`.

## Events

### Events per day

```bash
GET /v1/stat/event
GET /v1/stat/event?date=2025-02-01
```

Tracking events per day, broken down by event type. Only events in the event log count;
action runs and imports have endpoints of their own.

| Query parameter | Description |
|---|---|
| `date` | First day, as `YYYY-MM-DD`. Defaults to 30 days ago. The nightly aggregate keeps 90 days, so earlier days come back as zero. |

The last entry is today, counted live. A day where nothing at all happened has a single
`total` key in `stats` with the value 0, so `types` may contain `total` as well as the
event types.

**Response:**

```json
{
    "types": [
        "checkout",
        "login",
        "pageview",
        "purchase"
    ],
    "entries": [
        {
            "total": 1170,
            "entry": "2025-02-25T00:00:00Z",
            "stats": {
                "checkout": 137,
                "login": 129,
                "pageview": 466,
                "purchase": 438
            }
        },
        {
            "total": 1067,
            "entry": "2025-02-26T00:00:00Z",
            "stats": {
                "checkout": 121,
                "login": 128,
                "pageview": 466,
                "purchase": 352
            }
        },
        ...
    ]
}
```

### Events per month

```bash
GET /v1/stat/event/month
```

Tracking events per month, as shown on the limits page of the account settings. It covers
the 90 days kept by the nightly aggregate, so the first month is usually partial, and the
current day is not included.

`stats` has a single `total` key and `types` is empty.

**Response:**

```json
{
    "types": [],
    "entries": [
        {
            "total": 152286,
            "entry": "2025-01",
            "stats": {
                "total": 152286
            }
        },
        {
            "total": 87882,
            "entry": "2025-02",
            "stats": {
                "total": 87882
            }
        }
    ]
}
```

## Actions and imports

### Action runs per day

```bash
GET /v1/stat/action
GET /v1/stat/action?date=2025-02-01
```

Action runs per day, broken down by action. An action run is counted from the log entries
it writes, so the keys in `stats` are the keys of the actions and scheduled actions.

| Query parameter | Description |
|---|---|
| `date` | First day, as `YYYY-MM-DD`. Defaults to 30 days ago. |

The format is the same as for [events per day](#events-per-day), including the `total` key
on empty days.

**Response:**

```json
{
    "types": ["welcome_mail", "nightly_export"],
    "entries": [
        {
            "total": 214,
            "entry": "2025-02-25T00:00:00Z",
            "stats": {
                "welcome_mail": 213,
                "nightly_export": 1
            }
        },
        ...
    ]
}
```

### Import runs per day

```bash
GET /v1/stat/import
GET /v1/stat/import?date=2025-02-01
```

Import runs per day, broken down by integration: API integrations, inventory integrations,
profile endpoints, CSV event imports and inventory feeds. The keys in `stats` are the names
of the integrations where they have one, otherwise their keys.

| Query parameter | Description |
|---|---|
| `date` | First day, as `YYYY-MM-DD`. Defaults to 30 days ago. |

The format is the same as for [events per day](#events-per-day).

### Log entries per day

```bash
GET /v1/stat/log-daily?key=int_12&source=integration
GET /v1/stat/log-daily?key=welcome_mail&source=action&days=7
```

Log entries per day for one action or integration, broken down by log level (`DEBUG`,
`INFO`, `WARN`, `ERROR`, `FATAL`). This is the chart on the detail page of an action or
integration.

| Query parameter | Description |
|---|---|
| `key` | Required. The log key of the action or integration, see the table below. |
| `source` | Required. Where the log entries come from, see the table below. |
| `days` | Number of days back. Defaults to 30. The aggregate keeps 90 days. |

| What | `source` | `key` |
|---|---|---|
| Action | `action` | The key of the action |
| API integration | `integration` | `int_` followed by the integration ID, e.g. `int_12` |
| Inventory integration | `integration_inventory` | The ID of the integration |
| Profile endpoint | `integration_inbound` | The ID of the endpoint |
| CSV event import | `integration_csv` | The key of the import |

Leaving out `key` or `source` gives `400 Bad Request`. Only days with log entries are
returned, and the current day is not included.

**Response:**

```json
{
    "types": ["INFO", "ERROR"],
    "entries": [
        {
            "total": 25,
            "entry": "2025-02-25T00:00:00Z",
            "stats": {
                "ERROR": 1,
                "INFO": 24
            }
        },
        ...
    ]
}
```

## Profiles

All profile statistics leave out profiles that have been merged into another profile.

### New profiles per day

```bash
GET /v1/stat/user
GET /v1/stat/user?date=2025-02-01
```

The number of profiles created per day. The last entry is today, counted live.

| Query parameter | Description |
|---|---|
| `date` | First day, as `YYYY-MM-DD`. Defaults to 30 days ago. |

There is no breakdown: `types` is empty and `stats` is `null`.

**Response:**

```json
{
    "types": [],
    "entries": [
        {
            "total": 312,
            "entry": "2025-02-25T00:00:00Z",
            "stats": null
        },
        ...
    ]
}
```

### Profile total

```bash
GET /v1/stat/customer-total
```

The number of profiles, and how many of them are identified (have at least one
identifier such as an email address), as of the latest nightly run. The response is an
object with the day of the count as its only key and `[profiles, identified profiles]` as
the value.

**Response:**

```json
{
    "2025-02-26T00:00:00Z": [48211, 30977]
}
```

### Profile total history

```bash
GET /v1/stat/customer-total-history
```

Meant to return the [profile total](#profile-total) for every day, in the same format with
one key per day. The `start` and `end` query parameters are accepted but not applied.

Note: the nightly dataset currently only keeps the latest day, and this endpoint returns an
empty object `{}`. Use `customer-total` for the current numbers and
[`user`](#new-profiles-per-day) for the growth over time.

### Identified profiles per day

```bash
GET /v1/stat/profile-count-identified
```

Profiles created per day over the last 90 days, split by whether they are identified
today. Up to the latest nightly run.

**Response:**

```json
{
    "types": ["identified", "unidentified"],
    "entries": [
        {
            "total": 312,
            "entry": "2025-02-25T00:00:00Z",
            "stats": {
                "identified": 41,
                "unidentified": 271
            }
        },
        ...
    ]
}
```

### Merged profiles per day

```bash
GET /v1/stat/profile-merged
```

Profiles merged into another profile per day over the last 30 days, counting profiles that
were also created within those 30 days. Up to the latest nightly run.

**Response:**

```json
{
    "types": ["merged"],
    "entries": [
        {
            "total": 17,
            "entry": "2025-02-25T00:00:00Z",
            "stats": {
                "merged": 17
            }
        },
        ...
    ]
}
```

## Segments

A segment member is either `active` (currently in the segment) or `inactive` (was in the
segment and has left it).

### Segment members

```bash
GET /v1/stat/segment
GET /v1/stat/segment/{sid}
```

Active and inactive members per segment, as of the latest nightly run, ordered by segment
title. With a segment ID, `entries` holds only that segment. `entry` is the segment title.

Read the member counts from `stats`. `total` on this endpoint holds the number of inactive
members; use [`segment-total`](#segment-totals) when you need the sum.

**Response:**

```json
{
    "types": [
        "active",
        "inactive"
    ],
    "entries": [
        {
            "total": 3,
            "entry": "Binge reader",
            "stats": {
                "active": 2940,
                "inactive": 3
            }
        },
        {
            "total": 2473,
            "entry": "Not subscriber",
            "stats": {
                "active": 1297,
                "inactive": 2473
            }
        }
    ]
}
```

### Segment totals

```bash
GET /v1/stat/segment-total
GET /v1/stat/segment-total/{sid}
```

As [segment members](#segment-members), but `total` is the sum of active and inactive
members.

Note: the variant with a segment ID currently returns an empty `entries` list. Use
`GET /v1/stat/segment/{sid}` for a single segment.

**Response:**

```json
{
    "types": ["active", "inactive"],
    "entries": [
        {
            "total": 2943,
            "entry": "Binge reader",
            "stats": {
                "active": 2940,
                "inactive": 3
            }
        },
        ...
    ]
}
```

### Segment size per day

```bash
GET /v1/stat/segment-total-daily/{sid}
```

The number of active and inactive members of one segment at the end of each day, for the
last 30 days and today. The figure for today is an estimate: yesterday's count plus the
profiles that entered today, minus those that left.

**Response:**

```json
{
    "types": ["active", "inactive"],
    "entries": [
        {
            "total": 2941,
            "entry": "2025-02-25T00:00:00Z",
            "stats": {
                "active": 2938,
                "inactive": 3
            }
        },
        ...
    ]
}
```

### Segment changes per day

```bash
GET /v1/stat/segment-day/{sid}
GET /v1/stat/segment-day/{sid}?date=2025-02-01
```

How many profiles entered (`active`) and left (`inactive`) one segment per day. The last
entry is today, counted live.

| Query parameter | Description |
|---|---|
| `date` | First day, as `YYYY-MM-DD`. Defaults to 30 days ago. The nightly aggregate keeps 60 days. |

**Response:**

```json
{
    "types": ["active", "inactive"],
    "entries": [
        {
            "total": 14,
            "entry": "2025-02-25T00:00:00Z",
            "stats": {
                "active": 12,
                "inactive": 2
            }
        },
        ...
    ]
}
```

## Properties

The property distributions are computed live and can be restricted to the members of a
segment.

| Query parameter | Description |
|---|---|
| `sid` | Optional. Only count profiles in this segment (numeric segment ID). Other values are ignored. |

A property ID that is not a number gives an empty result.

### Number property distribution

```bash
GET /v1/stat/property-int/{pid}
GET /v1/stat/property-float/{pid}?sid=4
```

The values of a number or decimal property, counted in up to 20 equally wide buckets. The
lowest and highest 0.1% of the values are left out, so a few extreme values don't flatten
the chart. Both endpoints work for both kinds of property.

`entry` is the range of the bucket: whole numbers (`"10-15"`) for a number property, two
decimals (`"0.50-1.25"`) for a decimal property. `total` is the number of profiles in the
bucket. `types` is empty and `stats` is `null`.

**Response:**

```json
{
    "types": [],
    "entries": [
        {
            "total": 1204,
            "entry": "0-5",
            "stats": null
        },
        {
            "total": 860,
            "entry": "5-10",
            "stats": null
        },
        ...
    ]
}
```

### Text property distribution

```bash
GET /v1/stat/property-str/{pid}
GET /v1/stat/property-str/{pid}?sid=4
```

The 50 most common values of a text or text list property, most common first. For a list
property each element is counted, so a profile counts once for every distinct value in its
list. `total` is the number of profiles holding the value. Values of encrypted properties
are decrypted in the response.

Other property types give an empty result.

**Response:**

```json
{
    "types": [],
    "entries": [
        {
            "total": 2210,
            "entry": "Copenhagen",
            "stats": null
        },
        {
            "total": 1408,
            "entry": "Aarhus",
            "stats": null
        },
        ...
    ]
}
```
