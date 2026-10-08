# Agillic Marketing Automation activity data

Agillic records what happens to every message it sends: an email is sent, opened,
clicked or bounces; an SMS is delivered or fails; a recipient is added to a
Facebook audience; someone clicks a link or visits a page. Agillic can export
this activity as CSV files. The CDP picks the files up and turns them into
statistics you can ask about, for example *"Which hour of the day do we get the
most clicks?"* or *"How many SMS failed last week, per flow?"*.

You can also have each activity row written to the recipient's profile as a CDP
event. Segments and properties can then use the activity, for example *"opened
an email in the Welcome flow in the last 30 days"*.

This guide covers:

1. [What you need](#1-what-you-need)
2. [Setting it up](#2-setting-it-up)
3. [How the import works](#3-how-the-import-works)
4. [The activity types](#4-the-activity-types)
5. [Looking at the statistics](#5-looking-at-the-statistics)
6. [Recording activity as CDP events](#6-recording-activity-as-cdp-events)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. What you need

- **An Agillic connection in the CDP.** You find it under **Connections**.
- **A super admin.** The activity import is set up under **Account → MCP**,
  which only super admins can open.
- **A place where Agillic drops its activity exports.** This is an SFTP server or
  an S3 bucket (any S3-compatible storage). Agillic must be set up to deliver its
  activity exports there. That is done in Agillic, so ask Agillic support if you
  are not sure it is in place.
- **The export files keep Agillic's own names.** The CDP recognises a file by
  the end of its name, for example
  `KA_Export_06MAY2026_…_EMAIL.CSV` or `…_SMS.CSV`. Files that have been renamed
  are not picked up.

---

## 2. Setting it up

### Step 1: Create a connection to the storage

1. Go to **Connections**, click **New Connection** and choose **SFTP** or **S3**.
2. Fill in the access details:
   - **SFTP:** Host, Port, Username and Password.
   - **S3:** Bucket, Region, Access Key, Secret Key and, for non-AWS storage,
     Endpoint.
3. In **Remote Folder**, enter the folder Agillic writes the exports to, for
   example `agillic/exports`. Leave it empty to use the top folder. The import
   reads this folder only, not its subfolders.
4. Test the connection and save it.

### Step 2: Select the connections in the MCP settings

1. Go to **Account → MCP**.
2. Under **Agillic connection**, pick the Agillic connection the activity belongs
   to. The Agillic MCP tools use the same connection.
3. Under **Activity data sync**, pick the SFTP or S3 connection from step 1.
4. Optionally, tick **Record activity as CDP events** (see
   [section 6](#6-recording-activity-as-cdp-events)). The box appears once a
   storage is selected.
5. Click **Save**.

From now on, new export files are imported automatically. To stop the import,
set **Activity data sync** back to **None**. Data that has already been imported
is kept.

> Before these settings moved to **Account → MCP**, they were set on the Agillic
> connection itself, with **Default connection** ticked. Until the MCP settings
> are saved for the first time, the CDP keeps using what was set there.

---

## 3. How the import works

The CDP regularly checks the storage folder for new export files. This runs on a
schedule set up for your CDP instance. For each export file it finds, it:

1. Reads every row and stores it.
2. Moves the file to a `processed` subfolder, sorted by year and week, for
   example `processed/2026-41/KA_Export_…_EMAIL.CSV`.

The folder therefore only holds files that are still waiting to be imported.
Files in `processed` are kept for 5 days and then deleted automatically, so
copy them elsewhere if you need to keep them longer.

A few things are worth knowing:

- **Each file is imported as a whole or not at all.** If a file cannot be read,
  for example because a timestamp is in an unexpected format, none of its rows
  are kept. The file stays in the folder, and the import tries it again on the
  next run. The other files are still imported.
- **No recipient details are kept.** The statistics only count activity. The
  email address, Agillic ID and mobile number in the files are not stored with
  the activity. They are only used to write events to the right profile when
  you [record activity as CDP events](#6-recording-activity-as-cdp-events).
- **Nothing is counted twice.** If Agillic delivers the same file again, or two
  exports cover overlapping periods, rows that have already been imported are
  skipped.
- **Times are Danish local time.** The export timestamps are read as
  Europe/Copenhagen time. All statistics use this local time, so "9 o'clock"
  means 9 o'clock on the recipient's side of Agillic, not UTC.
- **Timestamp formats are detected automatically.** Agillic writes
  `06.05.2026 08:44:43` by default. Other common date formats are also
  recognised, as long as every row in a file uses the same one.
- **Zip files are unpacked.** Agillic may deliver the exports zipped. A `.zip`
  file in the folder is unpacked, and every export in it is imported the same
  way as a loose file. Folders inside the zip don't matter. The zip is moved to
  `processed` once all its exports are imported. If one of them fails, the zip
  stays and is tried again on the next run. The exports in it that were already
  imported are not counted twice. A file named `.zip` that is not a valid zip
  is moved to `processed` without being imported.
- **Other files are left alone.** Only `.zip` files and files whose names end in
  one of the export types below are touched.

---

## 4. The activity types

Each Agillic export type is recognised by the end of its file name:

| File name ends in | What it holds | Status means |
|---|---|---|
| `_EMAIL.CSV` | Email activity in flows | Communication status: `SENT`, `OPENED`, `CLICKED`, `BOUNCED`, `UNSUBSCRIBED`, … |
| `_TRANSACTIONAL_EMAIL.CSV` | Transactional email activity | As for email |
| `_SMS.CSV` | SMS sent in flows | Gateway status reported by the SMS gateway |
| `_INBOUND_SMS.CSV` | SMS received from recipients | – |
| `_PUSH_NOTIFICATION.CSV` | Push notifications | Communication status |
| `_PRINT.CSV` | Print (letters) | Communication status |
| `_FACEBOOK_CA.CSV` | Facebook custom audience updates | The action, such as added or removed |
| `_GOOGLE_CM.CSV` | Google customer match list updates | The action, such as added or removed |
| `_LINK.CSV` | Link clicks, across channels | – |
| `_EVENT_ALL.CSV` | Agillic events | – |
| `_PROMOTION.CSV` | Promotion and proposition evaluations | – |
| `_PAGES.CSV` | Agillic page visits | – |

Every row is counted with its flow and step, where the export has them, and with
the channel for link clicks, events and promotions.

---

## 5. Looking at the statistics

The statistics are counts of activity rows per hour. You read them through the
CDP's MCP connection from an AI assistant such as Claude. To connect one, see
[MCP API](mcp.md). There are two tools:

| Tool | Covers |
|---|---|
| `get_email_activity_stats` | Email and transactional email |
| `get_activity_stats` | All the other activity types |

You usually don't call the tools yourself. Ask the assistant in plain language,
and it picks the tool and the groupings. Some examples:

- *"How are the steps in the Price drop flow distributed over the weekdays?"*
- *"What is the ratio between sent and clicked for Abandoned cart, per weekday?"*
- *"Which hour of the day do we get the most clicks, and does it vary between
  weekdays?"*
- *"How many SMS failed per flow in September?"*
- *"Which channel gets the most link clicks on weekends?"*

The tools can count by these keys, and filter on any of them:

| Key | Meaning |
|---|---|
| `flow`, `step` | Flow and step in Agillic |
| `status` | The status, see the table in [section 4](#4-the-activity-types) |
| `transactional` | Email only: transactional or not |
| `export_type` | Other activity only: `SMS`, `LINK`, `PAGES`, … |
| `channel` | Other activity only: the channel of a link click, event or promotion |
| `day` | The date |
| `weekday` | 1 = Monday … 7 = Sunday |
| `hour` | Hour of the day, 0–23 |
| `hour_ts` | A specific hour, for example `2026-09-23 13:00` |

Keep these points in mind when you read the numbers:

- **Each status is counted when it happened.** A click is counted at the time of
  the click, not at the time the email was sent. "Sent vs. clicked on Monday"
  therefore compares Monday's sends with Monday's clicks, which may belong to
  emails sent earlier.
- **Statistics lag behind the import.** They are recalculated by the CDP's
  scheduled data jobs, at least once a day. Rows that have just been imported
  may not show up until the next run.
- **No statistics yet.** Until the first calculation has run, the tools answer
  that there are no statistics on the instance yet.

---

## 6. Recording activity as CDP events

With **Record activity as CDP events** ticked under **Account → MCP**, every
imported activity row is also written as an event on the recipient's profile.
This is off by default. The statistics in section 5 work without it.

### What you get

Each activity type has its own event type:

| Activity | Event type |
|---|---|
| Email | `ag_email` |
| Transactional email | `ag_transactional_email` |
| SMS | `ag_sms` |
| Inbound SMS | `ag_inbound_sms` |
| Push notification | `ag_push_notification` |
| Print | `ag_print` |
| Facebook custom audience | `ag_facebook_ca` |
| Google customer match | `ag_google_cm` |
| Link click | `ag_link` |
| Agillic event | `ag_event` |
| Promotion | `ag_promotion` |
| Page visit | `ag_page` |

The event's time is the time of the activity. Its source is `agillic`. Each
column of the export is carried in the event's context, named after the column
in lower case with underscores. For example, an `ag_email` event looks like this:

| Context key | Example |
|---|---|
| `email` | `jane@example.com` |
| `flow` | `Welcome` |
| `step` | `Step 2` |
| `template` | `welcome_2` |
| `communication_status` | `OPENED` |
| `communication_timestamp` | `2026-09-23T13:06:23+02:00` |
| `subject_line` | `Welcome to the club` |
| `flow_execution_id` | `69abe6da-…` |
| `agillic_id` | `683AECEBC59D865E` |
| `language` | `da` |

Empty columns are left out. The BotClick column is not recorded. The event types
and their context keys are listed under **Event Specs**. You can use them like
any other context event, for example in property conditions on context keys.
Segments can then build on those properties.

### Which profile gets the event

The CDP finds the recipient's profile by the **Agillic ID** first, and then by
**email**:

- **No match:** if neither matches an existing profile, a new profile is
  created.
- **Identifiers are added:** a profile found by email that has no Agillic ID yet
  gets the Agillic ID from the export, and the other way round.
- **Properties are written:** the Agillic ID and email are written to the profile
  properties **Agillic ID** (`agillic_id`) and **Email** (`email`).
- **Conflicting identifiers:** if the email in a row already belongs to a
  different profile than its Agillic ID, the email is left where it is. Nothing
  is moved between profiles.
- **No identifiers:** rows with neither an Agillic ID nor an email are skipped.

For exports other than email, the recipient's email is taken from the first
column of the file. This works when that column is the email, which it is in
most Agillic accounts. If your account uses a different recipient key, profiles
are matched by Agillic ID only.

### What saving the settings sets up

When you save the MCP settings with the box ticked, the CDP:

- **Creates the event types** above, or resets them to their standard
  definition if they already exist.
- **Creates the profile properties** **Agillic ID** (`agillic_id`) and **Email**
  (`email`), if no properties with those system titles exist. They are created
  as protected properties with protected content, linked to the Agillic ID and
  email identifiers.

Existing properties are not changed. If a property with the label "Email" or
"Agillic ID" already exists under another system title, the property is not
created and the import does not write that value.

### Before you switch it on

- **Volume.** Link clicks, Agillic events and page visits can add up to a lot of
  events. Switch it on for an account where you know the volume, and keep an eye
  on the number of events.
- **Privacy.** Inbound SMS events contain the text the recipient sent, and email
  events contain email addresses. Mark the event types as **protected** under
  **Event Specs** if they should not be visible to MCP clients.
- **No actions are triggered.** Imported events are written straight to the
  profiles. They do not set off triggers. Properties built on them are updated
  the next time properties are recalculated.
- **Only new activity.** Only rows imported after you tick the box become
  events. Activity that was imported before is not turned into events
  afterwards.
- **Switching it off.** Untick the box to stop recording new events. Events that
  have already been recorded stay on the profiles.

---

## 7. Troubleshooting

| What you see | What to check |
|---|---|
| Files stay in the folder and are never moved to `processed` | Make sure **Account → MCP** has an **Agillic connection** selected and that **Activity data sync** points to the right storage connection. Check that the files are directly in the **Remote Folder**, not in a subfolder, and that their names end in one of the types in [section 4](#4-the-activity-types). |
| One file stays in the folder while others are moved | That file could not be imported, most often because of a timestamp the CDP cannot read or a missing timestamp column. Ask your CDP administrator to check the import log for the file. |
| The assistant says there are no statistics yet | The statistics have not been calculated since the first import. Wait for the next scheduled run. |
| Numbers seem to be shifted by one or two hours | The statistics use Danish local time (Europe/Copenhagen), not UTC. |
| No `ag_…` events on profiles | Check that **Record activity as CDP events** is ticked and saved. Only files imported after that are recorded. Rows without an Agillic ID or email are skipped. |
| Activity lands on new profiles instead of existing ones | The existing profiles have neither the Agillic ID nor the email from the export as an identifier. Check how those profiles are identified. |

Technical details for developers and operators, such as table layouts, the
statistics model and the import command, are in
[agillic-activity.md](../agillic-activity.md).
