# Actions

Actions are automated tasks triggered by events, segment changes, or schedules.

In the database, actions are stored with a type key (e.g. `webhook`) and a specification JSON field. The key tells the system how to interpret the specification. Each row contains an instance of an action type.

The `action_trigger` table defines when to trigger actions. Each row holds a reference to an action and keys indicating whether it fires on an event, a segment transition, or a profile deletion.

## Action Types

### Webhook

Sends an HTTP request to an external URL.

**Setup:**

1. Select **Webhook** as the action type.
2. Enter a description.
3. Set the **HTTP Method** (GET, POST, PUT, PATCH, DELETE) and the **URL**.
4. Optionally add a **Request Body** (JSON) and **Headers**.
5. Optionally select a **Connection** for authentication (OAuth2, API key, etc).
6. You can add headers for API keys etc. Do use the secrets function if adding API keys.

Use placeholders in the URL, body, and headers: `{{p.email}}` for customer properties, `{{x.api_key}}` for secrets.

When a **Microsoft Graph** connection is selected, the body editor is replaced by a **Field Mapping** section. Map CDP properties to Microsoft API fields (e.g. `email` -> `mail`). The request payload is generated from the mappings automatically.

### Email

Sends an email using a template.

**Setup:**

1. Select **Email Action** as the action type.
2. Enter a description.
3. Set the **Template** identifier for the email content.
4. Set the **Recipient** address. Use `{{p.email}}` to send to the customer's email.

### Render Properties

Recalculates selected computed customer properties on demand. Useful when properties must be up-to-date before a follow-up action runs or other instant availability.

Render properties must be triggered by an event trigger which will make the updated property available in the response of the next event request and subsequently with the `window.argo` functions.

**Setup:**

1. Select **Render Properties** as the action type.
2. Enter a description.
3. Check the specific **properties to recalculate**, or leave all unchecked to recalculate all computed properties.
4. Optionally select a **Follow-up Action** to execute after rendering. The follow-up action receives the customer with freshly updated values.

**Note:** Render Properties can only be used with **event** triggers.

## Placeholders

Available in webhook URLs, headers, and body templates:

| Syntax                | Description                               |
| --------------------- | ----------------------------------------- |
| `{{p.property_name}}` | Customer property value (by sys_title)    |
| `{{x.secret_key}}`    | Secret from the secrets store             |

## Triggers

Actions fire via triggers, configured on the Triggers page:

- **Event**: Fires on every incoming event of a specific type.
- **Segment**: Fires when a customer enters or leaves a segment.
- **Delete**: Fires when a customer profile is deleted.
