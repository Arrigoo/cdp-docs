# MCP API

The MCP API exposes CDP data for AI agents using the [JSON-RPC 2.0](https://www.jsonrpc.org/) protocol, compatible with the [Model Context Protocol](https://modelcontextprotocol.io/) specification.

**Base URL:** `POST /v1/mcp`

Access control is enforced per event type and property via the **protected** and **protected_content** flags configured in the admin. Protected event types are excluded entirely from results; protected-content types and properties have their values masked.

---

## Endpoints

| Method | Path       | Description |
|--------|------------|-------------|
| `GET`  | `/v1/mcp`  | Human-readable discovery document listing available tools and their schemas. |
| `POST` | `/v1/mcp`  | JSON-RPC 2.0 endpoint. All tool calls go here. |

---

## Authentication

Every request to `/v1/mcp` needs a bearer token in the `Authorization: Bearer <token>` header. Tokens are issued for an API key with the **MCP** role. Create one in the admin under **Account → API Keys**, and copy the key and secret when they are shown. The secret is only displayed once. An MCP key can only call `/v1/mcp`; it has no access to the rest of the API.

Before handing out an MCP key, make sure sensitive properties have **Protected Content** enabled and that event types you don't want exposed are marked as protected.

### Limiting the tools

**Account → MCP** holds the MCP settings. Only super admins can open it. Tick **Hide CDP tools** and click **Save** to offer MCP clients only the Agillic tools and the activity statistics (`get_activity_stats`, `get_email_activity_stats`). The profile, segment, event and inventory tools are then left out of `tools/list` and `GET /v1/mcp`, and calling one of them returns `Unknown tool`. Clients see the change the next time they list the tools. Untick the box and save again to offer every tool.

There are two ways to connect an MCP client.

### Browser authentication

Use this for MCP clients that connect to a remote server over HTTP and log in through the browser, such as custom connectors in Claude. It uses the OAuth 2.1 authorization code flow with PKCE.

1. In the MCP client, add a custom connector with the URL `https://<your-cdp-host>/v1/mcp`.
2. The first request has no token, so `/v1/mcp` answers `401` with a `WWW-Authenticate` header pointing to `/.well-known/oauth-protected-resource`. From there the client finds the login and token endpoints in `/.well-known/oauth-authorization-server`.
3. The client opens the CDP's **Authorize access** page (`/authorize`) in your browser. Enter the MCP key as **Client ID** and its secret as **Client secret**, then click **Authorize**.
4. The browser is sent back to the client, which exchanges the one-time code for an access token and a refresh token. The client refreshes the token on its own.

The login code is valid for 10 minutes and can be used only once. The page shows the address it will redirect to; check that it belongs to the client you are connecting.

Browser-based clients running on `https://claude.ai` and `https://claude.com` are allowed to call `/v1/mcp` cross-origin.

| Endpoint | Purpose |
|----------|---------|
| `GET /.well-known/oauth-protected-resource`   | Protected resource metadata (RFC 9728). |
| `GET /.well-known/oauth-authorization-server` | Authorization server metadata (RFC 8414). |
| `GET /authorize`, `POST /authorize`           | Login page. Requires `response_type=code`, `client_id`, `redirect_uri`, `code_challenge` and `code_challenge_method=S256`. |
| `POST /auth/api`                              | Token endpoint. Accepts `grant_type=authorization_code` (with `code` and `code_verifier`) or `client_credentials`. |
| `POST /auth/api/refresh`                      | Exchanges a `refresh_token` for a new access token. |

### MCP bridge (stdio clients)

Use this for MCP clients that only start local servers over stdio, such as Claude Desktop without custom connectors. The JS bridge file that can be fetched through the admin interface is a small Node.js script. It reads JSON-RPC messages from stdin, forwards them to `/v1/mcp`, and writes the responses to stdout. It fetches the token itself with the MCP key and secret.

**Fetch the bridge.** In the admin, go to **Account → API Keys** and click **MCP bridge file here** under **MCP bridge file**. Save it as `mcp-bridge.js` somewhere permanent, for example `~/mcp/mcp-bridge.js`. The bridge needs Node.js and has no other dependencies.

**Configure it** with environment variables:

| Variable          | Description |
|-------------------|-------------|
| `CDP_URL`         | The MCP endpoint, e.g. `https://<your-cdp-host>/v1/mcp`. Defaults to `http://localhost:3030/v1/mcp`. |
| `CDP_AUTH_URL`    | The token endpoint, e.g. `https://<your-cdp-host>/auth/api`. Defaults to `http://localhost:3030/auth/api`. |
| `CDP_AUTH_KEY`    | The MCP key. |
| `CDP_AUTH_SECRET` | The MCP key's secret. |

**Register it with the client.** For Claude Desktop, add the bridge to `claude_desktop_config.json` and restart the app:

```json
{
  "mcpServers": {
    "cdp": {
      "command": "node",
      "args": ["/Users/me/mcp/mcp-bridge.js"],
      "env": {
        "CDP_URL":         "https://cdp.example.com/v1/mcp",
        "CDP_AUTH_URL":    "https://cdp.example.com/auth/api",
        "CDP_AUTH_KEY":    "<mcp key>",
        "CDP_AUTH_SECRET": "<mcp secret>"
      }
    }
  }
}
```

To check the bridge by hand, run it and paste a request:

```bash
CDP_URL=https://cdp.example.com/v1/mcp CDP_AUTH_URL=https://cdp.example.com/auth/api \
CDP_AUTH_KEY=<mcp key> CDP_AUTH_SECRET=<mcp secret> node mcp-bridge.js
{"jsonrpc":"2.0","id":1,"method":"initialize"}
```

The bridge writes its log to stderr, so it never interferes with the protocol on stdout. The access token is cached for the life of the process. When it expires, the request that got the `401` fails and the next request fetches a new token.

---

## Request / response format

All requests to `POST /v1/mcp` must follow the JSON-RPC 2.0 envelope:

```json
{
  "jsonrpc": "2.0",
  "id":      1,
  "method":  "<method>",
  "params":  { }
}
```

Successful response:

```json
{
  "jsonrpc": "2.0",
  "id":      1,
  "result":  { }
}
```

Error response:

```json
{
  "jsonrpc": "2.0",
  "id":      1,
  "error":   { "code": -32601, "message": "Method not found: foo" }
}
```

---

## Methods

### `initialize`

Returns server info and capabilities. Call this first to confirm the connection.

**Request:**
```json
{ "jsonrpc": "2.0", "id": 1, "method": "initialize" }
```

**Result:**
```json
{
  "protocolVersion": "2024-11-05",
  "capabilities": { "tools": {} },
  "serverInfo": { "name": "Arrigoo CDP MCP API", "version": "1.0" }
}
```

---

### `tools/list`

Returns the list of available tools with their input schemas. The Agillic tools are only listed when an Agillic connection exists, and **Hide CDP tools** limits the list (see [Limiting the tools](#limiting-the-tools)).

**Request:**
```json
{ "jsonrpc": "2.0", "id": 2, "method": "tools/list" }
```

**Result:**
```json
{
  "tools": [
    {
      "name": "search_events",
      "description": "...",
      "inputSchema": { "type": "object", "properties": { ... } }
    },
    ...
  ]
}
```

---

### `tools/call`

Dispatches a named tool. Set `params.name` to one of the tools below and pass arguments in `params.arguments`.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id":      3,
  "method":  "tools/call",
  "params":  {
    "name":      "<tool_name>",
    "arguments": { ... }
  }
}
```

---

## Tools

### `search_events`

Search customer events with optional filtering. Events marked as `protected` are excluded entirely. Events marked as `protected_content` are returned but `topics`, `strval`, `intval`, `src`, and `url` are masked as empty/zero.

**Arguments:**

| Field       | Type       | Required | Description |
|-------------|------------|----------|-------------|
| `evts`      | `[]string` | No       | Event type names to include. Omit to include all non-protected types. |
| `cid`       | `string`   | No       | Filter to a single customer ID. |
| `page`      | `integer`  | No       | 0-indexed page number. Defaults to `0`. |
| `page_size` | `integer`  | No       | Events per page. Defaults to `50`. |

**Example request:**
```json
{
  "jsonrpc": "2.0",
  "id":      1,
  "method":  "tools/call",
  "params":  {
    "name": "search_events",
    "arguments": {
      "evts":      ["page_view", "purchase"],
      "cid":       "abc123",
      "page":      0,
      "page_size": 50
    }
  }
}
```

**Result:**
```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "events": [
      {
        "id":     1,
        "cid":    "abc123",
        "sid":    7,
        "ctime":  "2025-01-15T10:30:00Z",
        "evt":    "pageview",
        "topics": [],
        "strval": "",
        "intval": 0,
        "src":    "web",
        "url":    "https://example.com/shop"
      },
      {
        "id": 643405,
        "cid": "08aa997f-5fcd-48d3-b87d-a0080336057a",
        "sid": 0,
        "evt": "unboxing",
        "topics": [],
        "ctime": "2026-02-20T14:53:38.433335Z",
        "strval": "",
        "intval": 0,
        "ident": {}
      }
    ],
    "page": 0,
    "page_size": 50,
    "total": 6
  }
}
```

---

### `get_segment_profiles`

Retrieve customer profiles that are active members of one or more segments. Properties with `protected_content = true` are returned with `val: null`.

**Arguments:**

| Field       | Type       | Required | Description |
|-------------|------------|----------|-------------|
| `segments`  | `[]string` | **Yes**  | Segment `sys_title` values. Customers active in any of the listed segments are included. |
| `page`      | `integer`  | No       | 0-indexed page number. Defaults to `0`. |
| `page_size` | `integer`  | No       | Profiles per page. Defaults to `200`. |

**Example request:**
```json
{
  "jsonrpc": "2.0",
  "id":      2,
  "method":  "tools/call",
  "params":  {
    "name": "get_segment_profiles",
    "arguments": {
      "segments":  ["high_value", "newsletter"],
      "page":      0,
      "page_size": 100
    }
  }
}
```

**Result:** array of profile objects:
```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result":[
    {
      "cid":   "abc123",
      "ctime": "2024-06-01T08:00:00Z",
      "mtime": "2025-01-10T14:22:00Z",
      "s":     ["high_value", "newsletter"],
      "p": [
        { "lab": "first_name", "val": "Anna" },
        { "lab": "revenue",    "val": null }
      ]
    }
  ]
}
```

| Field   | Type     | Description |
|---------|----------|-------------|
| `cid`   | `string` | Customer ID. |
| `ctime` | `string` | ISO 8601 creation timestamp. |
| `mtime` | `string` | ISO 8601 last-updated timestamp. |
| `s`     | `array`  | Active segment `sys_title` values. |
| `p`     | `array`  | Properties — `lab` is the `sys_title`, `val` is the value (`null` if protected_content). |

---

### `get_profile`

Fetch the full profile for a single customer by identifier. Returns properties, segments, consents, and all known identifiers.

**Arguments:**

| Field        | Type     | Required | Description |
|--------------|----------|----------|-------------|
| `ident_type` | `string` | **Yes**  | Identifier type: `cid` (internal ID), `email`, `foreignid1`, `foreignid2`, `foreignid3`, or other configured type. |
| `id_value`   | `string` | **Yes**  | The identifier value to look up. |

**Example request:**
```json
{
  "jsonrpc": "2.0",
  "id":      3,
  "method":  "tools/call",
  "params":  {
    "name": "get_profile",
    "arguments": {
      "ident_type": "email",
      "id_value":   "anna@example.com"
    }
  }
}
```

**Result:**
```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "cid":   "abc123",
    "ctime": "2024-06-01T08:00:00Z",
    "mtime": "2025-01-10T14:22:00Z",
    "s":     ["high_value"],
    "c": [
      { "scope": "marketing", "consent_type": "email", "status": "granted" }
    ],
    "i": [
      { "id_type": "email", "id_value": "anna@example.com" }
    ],
    "p": [
      { "lab": "first_name", "val": "Anna" },
      { "lab": "revenue",    "val": null }
    ]
  }
}
```

| Field | Description |
|-------|-------------|
| `s`   | Active segment `sys_title` values. |
| `c`   | Consent records (`scope`, `consent_type`, `status`). |
| `i`   | All known identifiers (`id_type`, `id_value`). |
| `p`   | Properties — `null` val if `protected_content = true`. Properties with `protected = true` are excluded entirely. |

---

### `list_segments`

List all segment definitions.

**Arguments:** none.

**Result:**
```json
{
  "segments": [
    {
      "id":          4,
      "sys_title":   "high_value",
      "title":       "High value",
      "description": "Customers with revenue above 10.000",
      "properties":  [12],
      "conditions":  [ ... ]
    }
  ],
  "count": 1
}
```

---

### `list_event_specs`

List all event specifications, including the `protected` / `protected_content` flags and whether `strval`, `intval` and `topics` are expected.

**Arguments:** none.

**Result:**
```json
{
  "event_specs": [
    {
      "evt":               "pageview",
      "description":       "Page view",
      "strval":            1,
      "intval":            0,
      "topics":            1,
      "protected":         false,
      "protected_content": false,
      "context_event":     false,
      "context_spec":      {}
    }
  ],
  "count": 1
}
```

---

### `list_properties`

List all property definitions with `sys_title`, label, value type, identifier mapping, and event requirement conditions.

**Arguments:** none.

**Result:** `{ "properties": [ ... ], "count": <n> }`.

---

### `get_email_activity_stats` and `get_activity_stats`

Sum Agillic activity statistics: sends, opens, clicks, SMS deliveries, link clicks, page visits and so on, counted per hour. `get_email_activity_stats` covers email and transactional email. `get_activity_stats` covers the other activity types: SMS, inbound SMS, push, print, Facebook and Google audiences, link clicks, events, promotions and page visits. The data comes from the Agillic activity exports; see [Agillic activity data](agillic-activity.md) for the setup and what the numbers mean.

**Arguments:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `group_by` | `[]string` | No | Keys to count by. Both tools: `connection_key`, `flow`, `step`, `status`, `hour_ts`, `day`, `weekday`, `hour`. Email only: `transactional`. Other activity only: `export_type`, `channel`. Omit for a single total. |
| `filters` | `object` | No | Key → accepted values, e.g. `{"flow": ["Welcome"], "weekday": ["6", "7"]}`. |
| `from`, `to` | `string` | No | Inclusive day range, `yyyy-mm-dd`. |
| `order_by` | `string` | No | `keys` (default) or `event_count` (largest first). |
| `limit` | `integer` | No | Maximum rows, default 1000, at most 10000. |

**Result:** `{ "group_by": [...], "rows": [ { <keys>, "event_count": <n> } ], "total_event_count": <n>, "groups": <n>, "truncated": <bool> }`. Times are Danish local time. The full reference is in [mcp-api.md](../mcp-api.md).

---

### `list_inventory`

List inventory items, ordered by most recently modified. Returns at most 500 items. All filters are optional and combined with AND.

**Arguments:**

| Field        | Type     | Required | Description |
|--------------|----------|----------|-------------|
| `category`   | `string` | No       | Exact category match. |
| `src`        | `string` | No       | Exact source match. |
| `topic`      | `string` | No       | Must appear in the item's `topics` array. |
| `title`      | `string` | No       | Case-insensitive substring match on `title`. |
| `sys_title`  | `string` | No       | Case-insensitive substring match on `sys_title`. |
| `ctime_from` | `string` | No       | Created at or after this timestamp. |
| `ctime_to`   | `string` | No       | Created at or before this timestamp. |
| `mtime_from` | `string` | No       | Modified at or after this timestamp. |
| `mtime_to`   | `string` | No       | Modified at or before this timestamp. |

**Result:**
```json
{
  "items": [
    {
      "id":         42,
      "foreign_id": "sku-1001",
      "title":      "Espresso beans 1kg",
      "sys_title":  "espresso_beans_1kg",
      "content":    "",
      "url":        "https://example.com/p/1001",
      "img_url":    "https://example.com/img/1001.jpg",
      "src":        "shop",
      "topics":     ["coffee", "beans"],
      "category":   "product",
      "source":     "web",
      "ctime":      "2025-01-02T09:00:00Z",
      "mtime":      "2025-03-10T12:00:00Z"
    }
  ],
  "count": 1
}
```

---

### `get_inventory_item`

Fetch a single inventory item by its numeric ID. Returns one item object in the same shape as `list_inventory` items. Returns error `-32002` if no item has that ID.

**Arguments:**

| Field | Type      | Required | Description |
|-------|-----------|----------|-------------|
| `id`  | `integer` | **Yes**  | Inventory item ID. |

---

### `list_categories`

List the distinct inventory categories currently in use.

**Arguments:** none.

**Result:** `{ "categories": ["article", "product"] }`

---

### `list_feeds`

List all active inventory feed configurations.

**Arguments:** none.

**Result:**
```json
{
  "feeds": [
    {
      "config_key":  "recommended_articles",
      "description": "Recommended articles",
      "config":      { "categories": ["article"], "conditions": [ ... ], "sort": "..." }
    }
  ]
}
```

`config_key` is the feed slug used by `fetch_feed`.

---

### `fetch_feed`

Resolve a personalised inventory feed for one customer. Returns the ordered items, each with its matching `score`. Returns error `-32002` if the feed does not exist or could not be resolved.

**Arguments:**

| Field  | Type     | Required | Description |
|--------|----------|----------|-------------|
| `slug` | `string` | **Yes**  | Feed slug (`config_key` from `list_feeds`). |
| `cid`  | `string` | **Yes**  | Customer ID to personalise the feed for. |

**Result:** `{ "items": [ ... ], "count": <n> }`, items in the `list_inventory` shape plus `score`.

---

## Agillic Marketing automation (MA) tools

The tools below call the Agillic MA API using the **Agillic connection** selected under **Account → MCP**. They are only listed by `tools/list` and `GET /v1/mcp`, and can only be called, when an Agillic connection is selected there. Until the MCP settings are saved for the first time, the Agillic connection marked **Default connection**, where this used to be set, is used. Failures from Agillic (including non-2xx responses) are returned with code `-32010` and Agillic's status and error body verbatim.

### `create_agillic_campaign`

Create an Agillic MA decentralised-messaging email campaign (`POST /messages/v2/campaign/email/:stage`). The MA processes the campaign asynchronously and returns a `taskId` straight away. Wait at least 5 seconds, then call `get_agillic_task_status` with that `taskId` to see whether the campaign was created.

**Arguments:**

| Field     | Type     | Required | Description |
|-----------|----------|----------|-------------|
| `payload` | `object` | **Yes**  | The campaign document, in the exact structure below. MA rejects unknown or renamed fields. |

**Payload structure:**
```jsonc
{
  "configuration": {                              // optional
    "callback": {                                 // optional
      "on":  "every",                             // required if callback is present: "every" | "success" | "failure"
      "url": "https://..."                        // required if callback is present
    }
  },
  "details": {                                    // required
    "name":                   "newsletter-001",   // required — campaign name (used for Flow and Email)
    "flowTemplate":           "NewsletterFlowTemplate",
    "senderName":             "Cafe Connect",
    "senderEmail":            "news@example.com",
    "replyToEmail":           "news@example.com",
    "subject":                "Newsletter 001",   // required
    "schedule":               "2025-12-01 12:00:00",
    "templateName":           "newsletter.html",  // required — email HTML template name
    "targetGroupName":        "Roastery Newsletter", // required
    "limitToTargetGroupName": "Valid Newsletter", // global target group
    "utmCampaign":            "newsletter-roastery",
    "blockGroups": [                              // required
      {
        "blockGroupId": "message_group",          // required — agid in the HTML
        "messages": [                             // required
          {
            "name":            "Roastery News 001",       // required
            "maxVariants":     1,
            "collapsed":       false,
            "messageTemplate": "RoasteryMessageTemplate", // required
            "blockId":         "message-side_article",    // required — agid in the HTML
            "variants": [                                 // required
              {
                "name":            "Fallback",            // required
                "targetGroupName": "Empty Target Group",
                "fields": {                               // required — keys must match the message template's fields
                  "hero-image": "", "message-header": "", "message-content": "", "cta-link": ""
                }
              }
            ]
          }
        ]
      }
    ]
  }
}
```

All `fields` values are strings: plain text, rich text/HTML, a link, or an absolute image URL.

**Result:** `{ "status": "created", "response": { "taskId": "<uuid>" } }`

---

### `get_agillic_task_status`

Get the status of an asynchronous Agillic task (`GET /messages/v2/task/{taskId}`), such as the one started by `create_agillic_campaign`.

**Arguments:**

| Field     | Type     | Required | Description |
|-----------|----------|----------|-------------|
| `task_id` | `string` | **Yes**  | The `taskId` returned by the asynchronous call. |

**Result:** Agillic's response, `{ "taskId": "...", "status": "...", "details": { "result": ... } }`. `status` is for example `queued`, `running`, `completed` or `failed`. On a completed campaign creation the result holds the `campaignId` used by `test_agillic_campaign`.

---

### `test_agillic_campaign`

Send a test of an existing Agillic campaign (`POST /messages/v1/campaign/email/:test`).

**Arguments:**

| Field          | Type     | Required | Description |
|----------------|----------|----------|-------------|
| `campaign_id`  | `string` | **Yes**  | `campaignId` of a created campaign, from `get_agillic_task_status`. |
| `recipient_id` | `string` | **Yes**  | Agillic recipient to render the email as — an email address or another recipient identifier. |
| `send_message` | `string` | No       | `"true"` or `"false"`. Defaults to `"true"`, which delivers the test email. `"false"` only renders it. |

**Result:** Agillic's response.

---

### `get_agillic_target_groups`

List all Agillic target groups (`GET /discovery/targetgroups`).

**Arguments:** none.

**Result:**
```json
{
  "target_groups": [
    { "name": "Roastery Newsletter", "description": "", "static": false, "global": false }
  ],
  "count": 1
}
```

| Field    | Description |
|----------|-------------|
| `static` | `true` = manually maintained recipient list, `false` = rule-based. |
| `global` | `true` = available to all flows and campaigns. |

---

### `list_agillic_resources`

List images and other resources in the Agillic asset library (`GET /assets/resources/list`).

**Arguments:**

| Field    | Type     | Required | Description |
|----------|----------|----------|-------------|
| `folder` | `string` | No       | Folder to list, e.g. `Campaigns/Images`. Omit to list the root. |

**Result:** `{ "resources": [ { "name": "...", "isFolder": false, "referenced": true } ] }`. `isFolder` marks a sub-folder (pass its path as `folder` to list it); `referenced` means a template or campaign uses the resource.

---

### `upload_agillic_resource`

Upload one image or other resource to the Agillic asset library (`PUT /assets/resources`).

**Arguments:**

| Field         | Type     | Required | Description |
|---------------|----------|----------|-------------|
| `filename`    | `string` | **Yes**  | File name to store it under, including extension, e.g. `logo.png`. |
| `file_base64` | `string` | **Yes**  | The file contents, base64-encoded. |
| `folder`      | `string` | No       | Destination folder, e.g. `Campaigns/Images`. |

**Result:** `{ "response": <Agillic's response> }`

---

### `list_agillic_templates`

List email templates in the Agillic asset library (`GET /assets/templates/list`).

**Arguments:**

| Field    | Type     | Required | Description |
|----------|----------|----------|-------------|
| `folder` | `string` | No       | Folder to list, e.g. `email/automations`. Omit to list the root. |

**Result:** `{ "templates": [ { "name": "...", "isFolder": false, "referenced": false } ] }`, with the same fields as `list_agillic_resources`.

---

### `create_agillic_template`

Author a new Agillic template and upload it to the asset library (`POST /assets/templates`) as an HTML file. The tool description returned by `tools/list` carries the full Agillic markup rules the agent must follow — content blocks (`agblockgroup` / `agrepeatingblock`), `agid`, `ageditable`, `templateparam` / `blockparam`, toggles, language versions, personalisation syntax, and cross-client rendering rules for Gmail and Outlook.

Agillic cannot overwrite an existing filename, so use a new name (for example with a `_v2` suffix) for each version. A new upload may take a few seconds to appear in `list_agillic_templates`.

**Arguments:**

| Field      | Type     | Required | Description |
|------------|----------|----------|-------------|
| `filename` | `string` | **Yes**  | File name including `.html`, e.g. `welcome.html`. |
| `template` | `string` | **Yes**  | The template HTML, following the Agillic markup rules. |
| `folder`   | `string` | No       | Destination folder, e.g. `email/automations`. |

**Result:** `{ "response": <Agillic's response> }`

---

### `upload_agillic_template`

Upload existing template HTML to the asset library as-is (`POST /assets/templates`). Use this when you already have the finished HTML; use `create_agillic_template` to author a new template. The HTML should be email-friendly: inline CSS, table-based layout, and no `<script>`, external stylesheets or fonts, or flexbox/grid.

**Arguments:**

| Field      | Type     | Required | Description |
|------------|----------|----------|-------------|
| `filename` | `string` | **Yes**  | File name including `.html`, e.g. `welcome.html`. |
| `html`     | `string` | **Yes**  | The template HTML, uploaded unchanged. |
| `folder`   | `string` | No       | Destination folder, e.g. `email/automations`. |

**Result:** `{ "response": <Agillic's response> }`

---

## Error codes

| Code     | Meaning |
|----------|---------|
| `-32700` | Parse error — request body is not valid JSON. |
| `-32600` | Invalid Request — `jsonrpc` field is not `"2.0"`. |
| `-32601` | Method not found. |
| `-32602` | Invalid params — missing required arguments or unknown tool name. |
| `-32603` | Internal error — database or server failure. |
| `-32002` | Resource not found — customer, inventory item or feed lookup returned no result. |
| `-32010` | Agillic error — the Agillic connection selected under **Account → MCP** could not be resolved, or the Agillic API call failed. |

---