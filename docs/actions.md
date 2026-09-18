# Actions

An **action** is something the CDP does for a profile: call another system over
a webhook, add a recipient to a target group, ask a language model for a
suggestion, recalculate properties, or delete the profile.

Actions never run on their own. Something has to set them off, and there are two
ways to do that:

- A **trigger** fires an action the moment something happens to a profile — an
  event arrives, the profile enters a segment, a property changes.
- A **schedule** runs an action for every profile in one or more segments at a
  fixed time, using a cron expression.

So a working setup is always at least two pieces: the action, which says *what
to do and where to send it*, and a trigger or schedule, which says *when*. They
are edited on separate pages, and one action can be reused by any number of
triggers and schedules.

You will find all three under **Actions** in the menu:

| Menu item | Page | What it holds |
|---|---|---|
| **Actions** | `/admin/actions` | The actions themselves |
| **Triggers** | `/admin/triggers` | What sets an action off |
| **Schedules** | `/admin/scheduled-actions` | Actions that run on a clock |

---

## 1. A first action, start to finish

The shortest useful example: post a message to a webhook whenever a `purchase`
event arrives.

**Create the action.** Go to **Actions → Actions**, click **Add Action** and
fill in:

- **Action Type**: `Webhook`
- **Description**: `Notify order system` — this is the name you will pick from
  in the trigger list, so make it recognisable.
- **HTTP Method**: `POST`
- **URL**: the address to call.
- **Request Body**: the JSON to send, with placeholders for profile values:

  ```json
  {
    "customer": "{{p.email}}",
    "total": "{{p.order_total}}"
  }
  ```

- **Request Headers**: add `Content-Type` / `application/json`.

Click **Preview** next to the body to see it rendered against a real profile
before you save. Then **Save Action**.

**Create the trigger.** Go to **Actions → Triggers**, click **Add Action
Trigger**, choose **Event**, pick `purchase` under **Event Type**, select
`Notify order system` under **Action**, and save.

That is it. The next `purchase` event fires the webhook. Note that event
triggers are cached in memory by the background worker and reloaded about once a
minute, so a brand-new event trigger may take up to a minute to become live.

**Check it ran.** Open the action from the list and click **View**. The
**Action Log** at the bottom lists each run with the event, the profile ID, the
time and a status — **Success** when the receiving system accepted the request,
**Error** when it refused it or could not be reached. See
[section 7](#7-what-happens-when-an-action-runs).

---

## 2. Triggers — when an action fires

Each trigger binds one action to one condition. On **Add Action Trigger** you
first pick the kind:

| Trigger | Fires when | How quickly |
|---|---|---|
| **Event** | An event of the chosen type is recorded | Within seconds. New or changed event triggers take up to a minute to load. |
| **Segment** | A profile enters or leaves the chosen segment | When segments are recalculated, every 5 minutes |
| **Delete Profile** | Any profile is deleted | Immediately |
| **Profile Merge** | Two profiles are merged into one | Immediately |
| **Property Update** | One of the chosen properties changes value | When the property is recalculated |

**Event** asks for an **Event Type** and fires for every incoming event of that
type.

**Segment** asks for a **Segment** and a **Segment Action** — *Profile enters
segment* or *Profile leaves segment*. Entry and exit are naturally
de-duplicated: a profile that stays in the segment is not triggered again, so
this is the right choice for "became a high-value customer" rather than "bought
something".

**Property Update** asks which properties to watch. The first entry in the list
is **Any property update**; tick it to fire on every property change, or tick
individual properties instead. Updating any one of the selected properties is
enough.

**Delete Profile** and **Profile Merge** take no configuration. On a merge the
action runs on the surviving (master) profile, so `{{p.*}}` reads its values;
the profile that was merged away is handed over as context — `{{ctx.body.cid}}`,
`{{ctx.body.p.<label>}}`, `{{ctx.body.i.<identifier>}}` and `{{ctx.body.s}}` —
with `{{ctx.status}}` set to `merged`.

### Narrowing a trigger with profile conditions

Every trigger has an optional **Profile Conditions** card. Add conditions there
and the action only runs for profiles that match **all** of them, checked at the
instant the trigger fires. With no conditions, the action runs for every profile.

The editor is the same one used for segments: pick a **Parameter** (a property,
a segment membership, or the profile's own *Profile created* / *Profile
updated* timestamp), an **Operator**, and a **Value**. See the
[segmentation guide](../segmentation-guide.md) for what the operators mean.

This is how you keep a broad trigger narrow — fire on every `purchase` event,
but only for profiles in the `vip` segment, without building a dedicated event
type.

Two things to know:

- Conditions are combined with AND. There is no OR.
- If a condition cannot be evaluated, the action is **not** queued. Failure is
  silent from the profile's point of view, so test a new condition on a profile
  you know should match.

---

## 3. The action types

Pick the type first on the action form; the rest of the form changes to match.
Every type takes a **Description**, which is required and is how the action is
listed everywhere else.

| Type | What it does |
|---|---|
| **Webhook** | Sends any HTTP request you configure. The general-purpose option. |
| **Render Properties** | Recalculates properties for the profile. |
| **Delete Profile** | Permanently deletes the profile. |
| **AI Prompt** | Sends a prompt plus the profile and its event history to a language model. |
| **Person Data** | Upserts a recipient in Agillic MA. |
| **Add to / Remove from Static Target Group** | Agillic MA target-group membership. |
| **One-To-Many** | Writes a row to an Agillic MA one-to-many table. |
| **Trigger Flow** | Queues a recipient into an Agillic MA flow. |
| **Achieve Event** | Registers an event on an Agillic MA recipient. |

### Webhook

The workhorse. Settings:

- **Connection** — optionally use a saved connection for authentication instead
  of writing credentials into the headers. Only shown when connections exist.
- **Throttling limit** — the maximum number of copies of this action that may
  run at the same time. `0` means no limit. See
  [section 6](#6-throttling).
- **HTTP Method**, **URL** — the URL is required.
- **Request Body** — the JSON to send. It must be valid JSON; the form refuses
  to save while it is not, and shows the parser error. A **Preview** button
  renders it against a real profile.
- **Request Headers** — name/value pairs.
- **Follow-up Action** — see [section 5](#5-chaining-actions).

If you leave both the body and the field mappings empty, the CDP sends a default
body containing the whole event and the whole profile:

```json
{ "event": { … }, "customer": { … } }
```

That is often all a receiving system needs, and it is the only way to get the
full event payload into a webhook — see the note on event data in
[section 4](#4-personalising-what-is-sent).

### Render Properties

Recalculates properties for the profile on the spot, rather than waiting for the
scheduled run. Tick the properties under **Properties to recalculate**, or leave
them all unticked to recalculate everything. Only properties that are built from
events are listed.

Useful as the first step of a chain: recalculate, then send the fresh values
onward with a follow-up action, which is re-read from the database after the
recalculation.

This action type can only be attached to an **Event** trigger.

### Delete Profile

Takes no settings. It permanently deletes the profile, and then runs everything
registered on the **Delete Profile** trigger, so you can notify downstream
systems as part of the deletion. This cannot be undone.

### AI Prompt

- **Model** — only models whose provider has a registered API key are listed.
  Add keys under **Account → LLM Keys**.
- **Prompt** — free text, which may contain `{{p.…}}` placeholders.
- **Follow-up Action** — **required**. The prompt action does nothing with the
  answer itself; it hands it to the follow-up as `{{ctx.body.response}}`.

The model receives the prompt, the profile and up to the 200 most recent events.
Properties marked *Protected Content* and protected event types are excluded,
and protected properties are never resolved in the prompt.

This runs once per profile that triggers it, and each run costs tokens. Put
profile conditions on the trigger before pointing this at a busy event.

### The Agillic action types

All of them need an **Agillic Connection**, set up under Connections first, and
a property holding the Agillic recipient ID. Beyond that:

- **Person Data** maps CDP properties onto Agillic person-data fields, grouped
  by folder, with optional one-to-many data. Include the field Agillic uses as
  the recipient key (usually `EMAIL`) among the mappings, or the recipient
  cannot be matched.
- **Add to / Remove from Static Target Group** needs a target group. Only groups
  marked static in Agillic are offered.
- **One-To-Many** writes a row into an Agillic table you select, with column
  mappings.
- **Trigger Flow** needs the **Flow ID**. No payload is sent.
- **Achieve Event** needs an **Event Name**, and optionally a **context** and a
  **conversion** JSON block.

**Achieve Event** and **Person Data** complete asynchronously: the CDP sends the
request, and Agillic calls back later with the outcome, which is what appears in
the action log. A follow-up action on these two receives the immediate
acknowledgement, not the eventual result.

If the property holding the recipient ID is empty for a profile, the Agillic
action is skipped quietly rather than sending a request with a blank recipient.
It is not recorded as an error, so a mapping mistake here shows up as "nothing
happened".

---

## 4. Personalising what is sent

URLs, headers, bodies, prompts and field mappings are all templates. Two syntaxes work, and the CDP picks per field automatically:

```
{{p.email}}         legacy placeholders
{{ .p.email }}      templates — adds conditionals, functions, combining values
```

Do not mix the two in one field: the choice is made across the whole value, so
the losing syntax is sent through as literal text. The **Preview** dialog warns
when a value mixes them.

The most used prefixes:

| Placeholder | Resolves to |
|---|---|
| `{{p.<label>}}` | A profile property |
| `{{p._cid}}` | The internal profile ID |
| `{{i.<identifier>}}` | An identifier, e.g. `{{i.email}}` |
| `{{s.<segment>}}` | The segment name when the profile is a member, empty otherwise |
| `{{c.<scope>.<type>}}` | A consent status; `DENY` when nothing is recorded |
| `{{x.<secret>}}` | A secret from the secret store |
| `{{ctx.…}}` | The previous action's result, in a chain |

Properties are matched on the property's **label**, spelled and capitalised
exactly as it appears in the property list. A placeholder that resolves to
nothing becomes an empty string rather than an error, which is why a misspelt
label shows up as a blank field rather than a failure.

For the full reference — functions, conditionals, inventory lookups, field
mappings — see [templating.md](../templating.md).

Three limitations worth knowing before you design a payload:

- **Event fields are not available as placeholders.** There is no `{{e.…}}`
  prefix. If a webhook needs the event's own values, leave the request body
  empty and use the default body, which contains the whole event.
- **Secrets do not resolve in a webhook request body** with the legacy syntax.
  `{{x.token}}` works in the URL and in headers — which is where credentials
  belong — but in the body it is left unresolved. The template syntax
  `{{ .x.token }}` does work there.

---

## 5. Chaining actions

Most action types have a **Follow-up Action** field naming another action to run
once this one finishes. The first action's result is passed to the second as
`{{ctx.status}}` and `{{ctx.body.*}}`, so a chain can use what the previous step
returned:

```
AI Prompt ──▶ Webhook
   writes the message      sends {{ctx.body.response}} onward
```

What ends up in the context depends on the first action:

| First action | `{{ctx.status}}` | `{{ctx.body}}` |
|---|---|---|
| Webhook | The HTTP status code | The response, if it was a JSON object |
| AI Prompt | `ok` | `response` and `model` |
| Render Properties | — | Empty; the profile is re-read first |
| Agillic (async) | The acknowledgement status | The acknowledgement, not the final outcome |

Chains are how multi-step outbound flows are built without code: look something
up in an external API, use the answer in a prompt, deliver the result by
webhook.

Be careful pointing two actions at each other. There is no depth limit and no
loop detection, so `A → B → A` runs until you break it.

---

## 6. Throttling

The **Throttling limit** on a webhook caps how many copies of that one action
may be in flight at the same time. Set it when the receiving system is slower
than a burst of events, or rejects parallel calls.

It is a **concurrency cap, not a rate limit**: it does not spread calls over
time, it only prevents more than N running at once. `0` disables it. The cap
applies per action, and per worker process — if the installation runs more than
one action worker container, each has its own allowance.

---

## 7. What happens when an action runs

When a trigger matches, the action is placed on a queue with a snapshot of the
profile, the triggering event and the action's configuration. A pool of workers
(10 by default) picks it up.

**Timing.** Event triggers fire within seconds, apart from the up-to-a-minute
delay before a newly saved trigger is loaded. Segment entry and exit are
detected when segments are recalculated, every 5 minutes. Schedules are checked
every minute.

**Order is not guaranteed.** Actions are not ordered, and two actions for the
same profile can run at the same time on different workers. Do not build a chain
by creating two triggers on the same event and hoping they run in order — use
**Follow-up Action**, which is sequential by construction.

**Where to look afterwards.** Open an action and click **View**:

- **Action Statistics** — executions over the last 30 days.
- **Action Log** — one row per run, with the event, profile ID, time and status.
  The eye icon opens the payload that was sent.

Schedules have their own **Log** card on the schedule's detail page.

A run is marked **Success** only when the receiving system accepted the request.
If it answered with an error status — a `404` because the URL is wrong, a `500`
because it failed on its side — the run is marked **Error**, and the message
names the status code and the URL. Open the payload view to see exactly what was
sent.

### Two things the log will not tell you

These are current behaviours, and they matter when you are working out why
something did not arrive:

1. **A failed delivery is not retried.** Whether the target refused the request
   or could not be reached at all, the action is not sent again. The log entry
   is the record that it happened; nothing will retry it for you.
2. **An action with invalid settings is dropped silently.** If required fields
   are missing the run does not happen, and there is no row in the log to say so.

If an action did not run at all, work backwards: does the trigger exist and
point at the right action; do the profile conditions actually match that profile;
for an event trigger, was the trigger saved more than a minute ago; for an
Agillic action, does the profile have a value in the recipient-ID property.

---

## 8. Schedules

A schedule runs one action for **every profile currently in the selected
segments**, on a cron expression. Use it for batch handovers and periodic
synchronisation, where the receiving system would rather have a nightly run than
a stream.

Under **Actions → Schedules**, click **New Schedule**:

- **Title** — required.
- **Schedule** — a five-field cron expression, for example `0 8 * * 1` for 08:00
  every Monday. The form shows a plain-language reading of what you typed
  underneath, and *Invalid cron expression* when it cannot parse it.
- **Active** — untick to keep the schedule without running it.
- **Action to trigger** — the action to run.
- **Segments** — one or more. The action runs once per profile in them.

Two differences from triggers:

- Profile conditions live on triggers, not schedules. A schedule fans out to
  **every** member of the chosen segments, so the segment definition is the only
  filter.
- A schedule re-runs for the same profiles every time it fires, for as long as
  they remain in the segment. It is not de-duplicated the way segment entry is.

Mind the size of the segments. A schedule over a large segment queues one action
per profile, and for an AI Prompt action that is one model call per profile.

---

## 9. Who can do what

The **Actions**, **Triggers** and **Schedules** pages are visible to every role,
including viewers — actions are part of understanding how the installation
behaves, so being able to read them is deliberate.

| | Super Admin | Admin | Viewer |
|---|---|---|---|
| See the lists, open **View**, read logs and statistics | Yes | Yes | Yes |
| Create, edit and delete actions, triggers and schedules | Yes | Yes | No |

A viewer still sees the **Add Action** and **Duplicate** buttons on some pages,
but is sent back to the dashboard on clicking them, and the server rejects the
change regardless.

See [access-and-roles.md](access-and-roles.md) for the roles in full. Note that
API keys cannot reach the action configuration at all, whatever their role.
