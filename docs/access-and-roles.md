# Access and roles

There are two kinds of accounts in the CDP:

- **Users** are people who log in to the admin interface. You manage them under
  **Account → Users**.
- **API keys** are credentials for other systems (integrations, scripts, AI
  assistants) that call the API. You manage them under **Account → API keys**.

Each user and each API key has exactly one role. The role decides what the
account can see and change.

## How logging in works

### Signing in to the admin interface

1. Enter your email and password.
2. If two-factor authentication is enabled on the installation, you are asked
   for a code:
   - If you have set up an authenticator app (under **Account → Security**),
     enter the code from the app. You can choose to get a code by email instead.
   - Otherwise a code is emailed to you. It is valid for 10 minutes.
3. Tick **Remember me on this device** to skip the code step on this browser for
   the next 30 days. Logging out removes the remembered device.

### Staying logged in

When you log in you get two tokens:

- An **access token** that is sent with every request. It is short-lived (about
  30 minutes).
- A **refresh token** that is used only to get new access tokens. It is valid
  for 5 days from the moment you logged in.

The admin interface renews the access token in the background every 10 minutes,
so you are not interrupted while you work. If a request is rejected because the
access token has run out, for example after the computer has been asleep, the
interface renews it and repeats the request once.

You are sent back to the login page when:

- you click **Log out**,
- the refresh token has expired (5 days after you logged in), or
- your IP address changes, for example when you switch network or VPN. Tokens
  issued to the admin interface only work from the IP address they were issued
  to.

### Forgotten password

Use **Forgot password** on the login page. You receive an email with a reset
token that is valid for 10 minutes. The reset must be done from the same network
(IP address) as the request.

### Authenticating with an API key

When an API key is created, the key and its secret are shown **once**. Store the
secret safely; it cannot be shown again.

A system exchanges the key and secret for tokens:

```http
POST /auth/api
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=<key>&client_secret=<secret>
```

The response contains an `access_token`, a `refresh_token`, the key's role (in
`scope`) and an `expiry` time. Send the access token as
`Authorization: Bearer <access_token>`. Before it expires, get a new one with:

```http
POST /auth/api/refresh
Content-Type: application/x-www-form-urlencoded

refresh_token=<refresh_token>
```

The refresh token is valid for 5 days. After that, authenticate with the key and
secret again. API tokens are not bound to an IP address to support usage from cloud environments where the IP may change.

AI assistants and other MCP clients that support OAuth can connect without the
system handling the secret itself: the client opens an authorization page where
you enter the key and secret once, and the client receives its tokens from
there.

## Roles for users

| Role | Admin interface | Changes |
|---|---|---|
| **Super Admin** | Everything | Everything |
| **Admin** | Everything except account settings and limits | Everything except account settings and limits |
| **Viewer** | Everything except the Integrations and Configuration menus, and account settings and limits | Read only |

### Super Admin

Full access to all pages and all data, including users, API keys and secrets.

Super Admin is the only role that can see and change the account settings: the
**Settings** and **Limits** tabs under **Account**, which hold the account name and
contact details, the daily and monthly event limits, and how long profiles and
events are kept. Give it only to the people responsible
for the installation.

### Admin

Full access to all pages and all data, except account settings:

- view, create, edit and delete profiles, segments, actions, triggers,
  schedules, event types, properties, inventory and integrations,
- manage connections, secrets and LLM keys,
- create, edit and delete users and API keys.

The **Settings** and **Limits** tabs are not shown to an Admin, and the server
rejects any attempt to read or change the account settings.

### Viewer

For people who need to look at data and results without being able to change
anything.

A viewer **can**:

- browse profiles, events, segments, actions, triggers, schedules and inventory,
- read earlier Insights analyses (starting a new analysis is not allowed),
- see the lists of users and API keys (but not API secrets).

A viewer **cannot**:

- create, edit or delete anything. Buttons for these operations are hidden, and
  most edit forms send the viewer back to the dashboard. The server rejects any
  change a viewer attempts, even from a page where a button is still visible,
- see the **Integrations** and **Configuration** menus,
- see or change account settings and limits.

## Roles for API keys

API keys only reach the parts of the API that are meant for other systems.
Everything else, such as users, API keys, secrets, account settings, actions and
integration setup, is closed to every API key, whatever its role. Account
settings are closed to API keys even with the Super Admin role.

The table shows what each role can do per API path. A path also covers
everything below it; for example `/v1/customer` covers `/v1/customer/<cid>`.

| API path | Super Admin | Admin | Viewer | MCP |
|---|---|---|---|---|
| `/v1/event` (events, event types, event endpoints) | Read, create | Read, create | Read | – |
| `/v1/event-api` | Read | Read | Read | – |
| `/v1/property` | Read | Read | Read | – |
| `/v1/segment`, `/v1/segment-api` | Read | Read | Read | – |
| `/v1/stat` | Read | Read | Read | – |
| `/v1/customer` (profiles) | Read, create, update, delete | Read, create, update, delete | Read | – |
| `/v1/config` | Read, create, delete | Read, create, delete | Read | – |
| `/v1/insights` | Read, run analyses | Read, run analyses | Read | – |
| `/v1/mcp` | Full | Full | Full | Full |
| Anything else | – | – | – | – |

"Read" is `GET`, "create" and "run analyses" are `POST`, "update" is `PUT` and "delete"
is `DELETE`. A request outside the table is answered with `403 Forbidden`.

### Super Admin (API)

The same as Admin. Reserved for platform-level integrations.

### Admin (API)

For integrations that send data into the CDP: record events, create, update and
delete profiles, and read and write configuration rows.

### Viewer (API)

For systems that only read data, such as reporting tools. Read-only access to
the paths in the table, and full use of the MCP.

### MCP

For AI assistants such as Claude. The key can only reach the MCP endpoint
(`/v1/mcp`) and is rejected everywhere else. Through the MCP the assistant can
read and write customer data.

Before you hand out an MCP key, mark sensitive **properties** as *Protected
Content* and mark the **event types** you do not want exposed as protected.
