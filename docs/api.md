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
