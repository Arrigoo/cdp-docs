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

Returns the list of available tools with their input schemas.

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

## Error codes

| Code     | Meaning |
|----------|---------|
| `-32700` | Parse error — request body is not valid JSON. |
| `-32600` | Invalid Request — `jsonrpc` field is not `"2.0"`. |
| `-32601` | Method not found. |
| `-32602` | Invalid params — missing required arguments or unknown tool name. |
| `-32603` | Internal error — database or server failure. |
| `-32002` | Resource not found — customer lookup returned no result. |

---
