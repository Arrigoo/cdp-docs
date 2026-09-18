# Arrigoo CDP API endpoints

## Authentication

Initially, you create an API key by going to the account settings (the cog wheel at the top right) and select the sub menu item 'API keys'.

Give the key a name, an optional description and select a role.

The interface will respond by showing the API key and secret in a notification. The key cannot be requested again, so be sure to store it.

Use the key name and secret to retrieve a refresh and access token.

```bash
POST /auth/api
Body: {"key": "THE KEY NAME", "secret": "THE SECRET"}
Headers:
    Content-type: application/json
    Accept: application/json
```

Response example:

```json
{
    "refresh_token": "f8xxxx85-exx3-4xxc-9xxa-18xxxxxxxx10",
    "access_token": "06xxxx1d-9xx2-4xxe-axx1-bdxxxxxxxx6c",
    "scope": "api_viewer",
    "expiry": "2025-03-21T10:35:24+01:00"
}
```

An access token must be used on every request to extract or update data as a bearer token. Example:

```bash
POST /v1/stat/segment
Headers:
    Content-type: application/json
    Accept: application/json
    Authorization: Bearer 06xxxx1d-9xx2-4xxe-axx1-bdxxxxxxxx6c
```

Access tokens are relatively short lived (see expiry in the response) and must be refreshed frequently using the refresh token.

```bash
POST /auth/api/refresh
Body: { "refresh_token": "f8xxxx85-exx3-4xxc-9xxa-18xxxxxxxx10"}
```

Response:

```json
{
    "access_token": "16xxxx1d-9xx2-4xxe-axx1-bdxxxxxxxx6b",
    "expiry": "2025-03-21T11:17:08+01:00"
}
```

The authorization header is omitted in the description below for the sake of simplicity.

## Profile data

### Fetch all profiles

```bash
GET /v1/customer?page=1
Optional parameters:
page: integer
```

For performance reasons, the properties are omitted from the response.

**Response:**

```json
[
    {
        "cid": "xxx",
        "s": ["segment1", "segment2"],
        "i": [
            {
                "id_type": "foreignId1",
                "id_value": "bxre35"
                },
            {
                "id_type": "email",
                "id_value": "bob@bib.dk"
            }
        ]
    },
    {
        "cid": "yxyxyx",
        ...
    },
    ...
]
```

### Get a single profile

```bash
GET /v1/customer/{profile UUID}
```

**Response:**

```json
{
    "p": [
        {
            "lab": "name",
            "val": "Bib Bobsen"
        }
    ],
    "s": ["subscriber","frequent_visitor"],
    "cid": "f72af6a3-487a-41db-9ac0-d0a6ccfde14e",
    "i": [
        {
            "id_type": "email",
            "id_value": "ibbib@bob.dk"
        },
        {
            "id_type": "foreignid1",
            "id_value": "ytrewq"
        }
    ],
    "ctime": "2025-03-21T11:32:22.503163Z",
    "mtime": "2025-03-21T11:32:22.503163Z"
}
```

### Fetch all customers in a specific segment

```bash
GET /v1/customer/segment/{segment_system_title}
```

NB: The segment is identified by the system title, not the ID.

**Response:**

```json
[
    {
        "cid": "xxx",
        "s": ["segment1", "segment2"],
        "i": [
            {
                "id_type": "foreignId1",
                "id_value": "bxre35"
                },
            {
                "id_type": "email",
                "id_value": "bob@bib.dk"
            }
        ]
    },
    {
        "cid": "yxyxyx",
        ...
    },
    ...
]
```

## Events

Events can be extracted with basic filtering. They will be sorted with the latest first.

```bash
POST /v1/event-api/search
```

Payload:
```json
{
    "cid": "b73884e8-a5e7-450b-83cc-572202d451d6",
    "evt": "pageview",
    "from": "2026-01-01T11:11",
    "to": "2026-02-25T11:11"
}
```
All parameters can be omitted.

Response:
```json
[
    {
        "id": 639515,
        "cid": "b13784e8-aee7-450b-83cc-572202d451d6",
        "sid": 1769700277,
        "evt": "pageview",
        "topics": [
            "topic1",
            "topic2"
        ],
        "ctime": "2026-01-25T21:11:56Z",
        "strval": "value",
        "intval": 68,
        "src": "web",
        "url": "https://arrigoo.io/products/item1",
        "ident": {
            "id_type": "",
            "id_value": ""
        }
    },
    {
        "id": 639406,
        "cid": "b13784e8-aee7-450b-83cc-572202d451d6",
        "sid": 1769700279,
        "evt": "pageview",
        "topics": [
            "topic1",
            "topic2"
        ],
        "ctime": "2026-01-22T13:40:00Z",
        "strval": "value",
        "intval": 73,
        "src": "web",
        "url": "https://arrigoo.io/login",
        "ident": {
            "id_type": "",
            "id_value": ""
        }
    },
   ...
    {
        "id": 639494,
        "cid": "b13784e8-aee7-450b-83cc-572202d451d6",
        "sid": 1769700277,
        "evt": "pageview",
        "topics": [
            "topic1",
            "topic2"
        ],
        "ctime": "2025-10-31T10:41:19Z",
        "strval": "value",
        "intval": 73,
        "src": "web",
        "url": "https://arrigoo.io/register",
        "ident": {
            "id_type": "",
            "id_value": ""
        }
    }
]
```

## Statistics

The statistics for the dashboard and numbers displayed in the interface are all available through the API.

They are based on aggregated datasets that are build at night.

```bash
GET /v1/stat/segment
```

**Response:**

```json
{
    "types": [
        "active",
        "inactive"
    ],
    "entries": [
        {
            "total": 2943,
            "entry": "Binge reader",
            "stats": {
                "active": 2940,
                "inactive": 3
            }
        },
        {
            "total": 4348,
            "entry": "Is subscriber",
            "stats": {
                "active": 4348,
                "inactive": 0
            }
        },
        {
            "total": 3770,
            "entry": "Not subscriber",
            "stats": {
                "active": 1297,
                "inactive": 2473
            }
        },
        {
            "total": 3601,
            "entry": "Top reader",
            "stats": {
                "active": 3009,
                "inactive": 592
            }
        }
    ]
}
```

```bash
GET /v1/stat/event
```

Daily event stats for the past month.

**Response**
```json
{
    "types": [
        "checkout",
        "login",
        "nl_signup",
        "nl_unsubscribe",
        "pageview",
        "pageview_nl",
        "purchase"
    ],
    "entries": [
        {
            "total": 137,
            "entry": "2025-02-25T00:00:00Z",
            "stats": {
                "checkout": 137,
                "login": 129,
                "nl_signup": 114,
                "nl_unsubscribe": 439,
                "pageview": 466,
                "pageview_nl": 409,
                "purchase": 438
            }
        },
        {
            "total": 121,
            "entry": "2025-02-26T00:00:00Z",
            "stats": {
                "checkout": 121,
                "login": 128,
                "nl_signup": 126,
                "nl_unsubscribe": 453,
                "pageview": 466,
                "pageview_nl": 409,
                "purchase": 352
            }
        },
        ...
   ]
}
```

```bash
GET /v1/stat/event/month
```

Monthly event stats.

**Response**
```json
{
    "types": [],
    "entries": [
        {
            "total": 32572,
            "entry": "2024-12",
            "stats": {
                "total": 32572
            }
        },
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
        },
        {
            "total": 23504,
            "entry": "2025-03",
            "stats": {
                "total": 23504
            }
        }
    ]
}
```

## Configuration: properties, segments and event types

The definitions behind the data — properties, segments and event types — can also be read
and created through the API.

Note what an API key may do here. Looking things up works with any API key. **Creating**
only works for event types; `POST /v1/property` and `POST /v1/segment` are answered with
`403 Forbidden` for every API key, whatever its role, and can currently only be done from
the admin interface. Changing and deleting definitions (`PUT` and `DELETE`) is closed to
API keys as well.

| Endpoint | What it does | Available to an API key |
|---|---|---|
| `GET /v1/property` | All property definitions | Yes |
| `GET /v1/property/{label}` | One property, by its label | Yes |
| `POST /v1/property` | Create a property | No |
| `GET /v1/segment` | All segment definitions | Yes |
| `GET /v1/segment/{id}` | One segment, by its numeric ID | Yes |
| `POST /v1/segment` | Create a segment | No |
| `GET /v1/event-spec` | All event types | Yes |
| `POST /v1/event-spec` | Create an event type | Yes |

For what the fields mean and how to choose them, see the
[segmentation guide](../segmentation-guide-tech.md).

### Properties

```bash
GET /v1/property
```

**Response:**

```json
[
    {
        "id": 12,
        "label": "Total spend, 365 days",
        "sys_title": "total_spend_365d",
        "value_type": "int",
        "description": "Sum of purchase amounts over the past year",
        "identifier_type": "",
        "identifier_merge": false,
        "protected": false,
        "protected_content": false,
        "count": 4348,
        "event_requirements": [
            {
                "property": 12,
                "event_type": ["purchase"],
                "accumulator": "sum",
                "max_age": 365
            }
        ]
    },
    ...
]
```

`count` is the number of profiles that currently hold a value for the property. It is updated every 15 minutes. The
`event_requirements` block is the recipe that turns events into the value; it is described
in full in the segmentation guide. `identifier_type` is set on the few string properties
that double as a profile identifier (`email`, `phone`, `foreignid1-3`, `agillic_id`) and is
empty on all others.

A single property is looked up by its label, not its system title:

```bash
GET /v1/property/Total%20spend,%20365%20days
```

Create a property by posting the same structure. `label`, `sys_title` and `value_type` are
the required parts:

```bash
POST /v1/property
Body:
{
    "label": "Articles read, 90 days",
    "sys_title": "articles_read_90d",
    "value_type": "int",
    "description": "Number of article pageviews in the past 90 days",
    "event_requirements": [
        {
            "event_type": ["pageview"],
            "accumulator": "count",
            "max_age": 90
        }
    ]
}
```

The response is the stored property, including the `id` it was given. Use that ID when you
reference the property in a segment condition.

Labels must be unique. A `POST` with a label an existing property already uses is rejected
with `409 Conflict` and an explanation:

```json
{
    "error": "A property with the label \"Articles read, 90 days\" already exists"
}
```

A property with no label is rejected with `400 Bad Request`. The `sys_title` is not checked
the same way, so look the property list up first if you are not sure whether the system
name is taken.

### Segments

```bash
GET /v1/segment
```

**Response:**

```json
[
    {
        "id": 7,
        "title": "High value outdoor customers",
        "sys_title": "high_value_outdoor",
        "description": "Spent over 5,000 in the last year with outdoor affinity",
        "properties": [12, 19],
        "conditions": [
            { "pt": 12, "rt": "", "field": "int",  "op": ">",         "v1": "500000", "v2": "" },
            { "pt": 19, "rt": "", "field": "strs", "op": "intersect", "v1": "[\"outdoor\"]", "v2": "" }
        ]
    },
    ...
]
```

A single segment is looked up by its numeric ID, not its system title:

```bash
GET /v1/segment/7
```

(The endpoint that lists the profiles in a segment, `GET /v1/customer/segment/{sys_title}`,
uses the system title instead. See [Profile data](#profile-data).)

Create a segment by posting the same structure without an ID:

```bash
POST /v1/segment
Body:
{
    "title": "Engaged readers",
    "sys_title": "engaged_readers",
    "description": "Read at least 10 articles in the past 90 days",
    "conditions": [
        { "pt": 31, "field": "int", "op": ">", "v1": "10" }
    ]
}
```

Each condition names a property by its ID in `pt`, the property's value type in `field`, an
operator in `op`, and one or two values in `v1` and `v2` (`v2` only for `between`). All
conditions must match for a profile to be in the segment.

The response repeats what you sent; it does not include the new ID. Read the segment list
again to get it.

### Event types

```bash
GET /v1/event-spec
```

**Response:**

```json
[
    {
        "evt": "purchase",
        "description": "A completed order",
        "topics": 1,
        "strval": 1,
        "intval": 1,
        "protected": false,
        "protected_content": false,
        "context_event": false,
        "context_spec": null
    },
    ...
]
```

An event type must exist before events of that type are accepted; events with an unknown
type are rejected.

```bash
POST /v1/event-spec
Body:
{
    "evt": "newsletter_signup",
    "description": "Signed up for a newsletter",
    "topics": 0,
    "strval": 1,
    "intval": -1
}
```

`topics`, `strval` and `intval` say what an event of this type must carry:

| Value | Meaning |
|---|---|
| `1` | Required. An event without the field is rejected. |
| `0` | Optional. The field may be present or absent. |
| `-1` | Disallowed. The field must be empty or the event is rejected. |

`protected` keeps the event type out of general read surfaces, including the MCP, and
`protected_content` keeps the event's content back where the type is still listed. Both
default to `false`.

Posting an event type that already exists overwrites the stored definition, so the same
call both creates and updates.

#### Context events

A normal event carries three pieces of data: `topics`, `strval` and `intval`. That is
enough for "viewed article X" but not for "bought these four items at these prices". A
**context event** adds a nested JSON object, so the whole payload arrives as one event.

Two things set a context event apart:

- The event type is declared with `"context_event": true`, and `context_spec` describes the
  keys the payload may carry.
- Events of that type are sent to `POST /v1/event/context` instead of `POST /v1/event`. The
  two are not interchangeable: a context event type is rejected on the basic endpoint, and
  the context endpoint rejects a type that is not a context event.

Declare the event type first:

```bash
POST /v1/event-spec
Body:
{
    "evt": "purchase",
    "description": "A completed order with its order lines",
    "topics": 0,
    "strval": 1,
    "intval": 1,
    "context_event": true,
    "context_spec": {
        "order_id":            { "required": 1, "type": "string" },
        "currency":            { "required": 1, "type": "string" },
        "products@*.sku":      { "required": 1, "type": "string" },
        "products@*.price":    { "required": 1, "type": "float" },
        "products@*.quantity": { "required": 0, "type": "int" },
        "customer.vip":        { "required": 0, "type": "bool" }
    }
}
```

Then send the event:

```bash
POST /v1/event/context
Body:
{
    "evt": "purchase",
    "cid": "b73884e8-a5e7-450b-83cc-572202d451d6",
    "strval": "order-10093",
    "intval": 74900,
    "src": "web",
    "context": {
        "order_id": "10093",
        "currency": "DKK",
        "products": [
            { "sku": "AB-100", "price": 499.00, "quantity": 1 },
            { "sku": "CD-220", "price": 250.00, "quantity": 2 }
        ],
        "customer": { "vip": true }
    }
}
```

The response is the same as for a basic event. A payload that breaks the schema is rejected
with `400 Bad Request` and a message naming the key:

```json
{
    "error": "context key \"products@1.price\": must be of type string"
}
```

##### Writing the context schema

Each entry in `context_spec` is a key pattern with a requirement and a type.

The pattern is the path to the value. Object levels are joined with `.`, and array elements
are written with `@`: `@0` for one specific position, `@*` for any position. So
`products@*.price` covers `products@0.price`, `products@1.price` and so on.

`required` works exactly like `topics`, `strval` and `intval`:

| Value | Meaning |
|---|---|
| `1` | Required. At least one value must match the pattern. |
| `0` | Optional. |
| `-1` | Disallowed. No value may match the pattern. |

`type` is one of `string`, `int`, `float` or `bool`, and every matching value must be of
that type. Whole numbers are stored as integers however they were written, so `499.00` is
an integer like `499` is. A value declared as `float` accepts whole numbers too, while one
declared as `int` rejects anything with a decimal part — so declare prices and other
amounts that can have decimals as `float`.

Keys the schema does not mention are stored as sent and not validated. Declaring the keys
you rely on means a sender that changes its payload fails immediately instead of quietly
producing empty properties.

A payload is limited to 200 values in total, 8 levels of nesting, keys of 128 characters
and strings of 1024 characters. Values must be strings, numbers or booleans; `null` is
dropped.

##### Using the context in properties

A property reads a context value by prefixing the pattern with `context.`, for example
`context.products@*.price` as the value or the group field of an event requirement. That is
what makes the payload useful: one `purchase` event can feed spend per product category, a
list of bought SKUs and an order count at the same time. The segmentation guide covers the
property side.

Events are returned with their context nested again, the way you sent it, so
`POST /v1/event-api/search` gives back the `context` object rather than the flat keys the
server stores internally.
