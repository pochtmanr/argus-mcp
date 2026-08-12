# The Argus local API

This is a local API. It listens on loopback on the machine running Argus, and nothing off that machine can reach it. There is no hosted web API.

- Base URL: `http://127.0.0.1:39219`
- Auth: `Authorization: Bearer <YOUR_API_KEY>`
- Bodies: `Content-Type: application/json`
- Surface: 55 endpoints, 53 agent tools

## Keys and approval

- Keys are created in the launcher, under the API tab. The raw value is shown once, at creation — only a hash is stored, so a lost key is replaced rather than recovered.
- A key can be scoped to specific folders, or left unscoped. A scoped key sees only the profiles in its folders, and may only close a session it opened itself.
- The first time a new client calls the API, the launcher raises an approve-or-deny prompt and holds the request until you answer it.

A launched profile is a separate, anonymous browser process. Never hand it credentials or tokens — it has no account of its own, and that is the point.

## Profiles

A profile is an isolated browser identity with its own proxy, fingerprint and cookie jar. You can read them, change them, and open one for automation — but you cannot mint one from here. Creating an identity is done in the app, because a script that makes fifty of them makes fifty that look alike.

### GET /v1/profiles

List profiles (optional ?folder=<id>)

- MCP tool: `argus_list_profiles`

```sh
curl -X GET "http://127.0.0.1:39219/v1/profiles" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/profiles/get

Read one profile

- MCP tool: `argus_get_profile`

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/get" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>" }'
```

### POST /v1/profiles/create

Create a profile

- MCP tool: `argus_create_profile`

Create a browser profile. Only name is required; everything else gets the same defaults the app's own New Profile dialog uses -- status Ready, a random colour, a stable Windows 11 fingerprint, no proxy. proxyMode 'assigned' needs proxyId to name a proxy from the library (argus_list_proxies); omit both to create a direct-connection profile. Returns the created profile. Its id is permanent: it doubles as the on-disk browser-data directory name.

Fields:
- `name` (string, required) — The profile's display name.
- `folderId` (string) — Folder to file it in. Omit for the root. Folder ids come from argus_list_folders.
- `status` (string) — Workflow status label. Defaults to Ready. argus_list_statuses has the ones this workspace uses.
- `tags` (tags) — Free-text labels, at most 5. Extras are dropped.
- `color` (string) — A preset name like blue or violet, or a #rrggbb hex. Defaults to a random preset.
- `avatar` (string) — "brand:<slug>" for a built-in site logo (brand:instagram, brand:facebook, ...). Omit for initials.
- `proxyMode` (string) — assigned, direct or free_proxy. Defaults to assigned when proxyId is given, direct otherwise.
- `proxyId` (string) — A proxy from the library (argus_list_proxies). Implies proxyMode assigned.
- `startUrl` (string) — URL the profile opens on launch.
- `fingerprintOs` (string) — Windows 11, Windows 10, macOS, Ubuntu, Android or iOS. Defaults to Windows 11.
- `randomizeFingerprint` (boolean) — Pick a random realistic device identity for that OS instead of the stable default.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/create" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "warmup-01", "proxyId": "<id>", "tags": ["warmup"] }'
```

### POST /v1/profiles/update

Update name, status, tags, colour, avatar, folder, proxy mode, start URL or launch automation

- MCP tool: `argus_update_profile`

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/update" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "status": "Ready", "proxyMode": "direct", "startUrl": "https://example.com" }'
```

### POST /v1/profiles/notes

Read a profile's notes, newest first

- MCP tool: `argus_profile_notes`

The notes on a profile, newest first -- what it is for, who it belongs to, what has been tried on it. Each note carries its author and when it was written. Read this before acting on an unfamiliar profile: it is where the humans running this workspace write down the things the profile's name and tags cannot say.

Fields:
- `profileId` (string, required) — The profile whose notes to read.
- `limit` (number) — How many of the newest notes to return. Defaults to 50.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/notes" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "limit": 50 }'
```

### POST /v1/profiles/notes/add

Append a note to a profile

- MCP tool: `argus_add_profile_note`

Append a note to a profile's thread -- what you did to it, what you found, why it is set up the way it is. The note is filed under this API key's own name and marked as written by an agent, so it is distinguishable from what a person wrote. Notes are append-only over the API: there is no tool to edit or delete one, including your own.

Fields:
- `profileId` (string, required) — The profile to annotate.
- `body` (string, required) — The note itself, up to 2000 characters.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/notes/add" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "body": "Warmed up for 20 minutes; no checkpoint." }'
```

### POST /v1/profiles/assign-proxy

Put a profile on a proxy from the library

- MCP tool: `argus_assign_proxy`

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/assign-proxy" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "proxyId": "<id>" }'
```

### POST /v1/profiles/launch-automation

Open a profile for automation; returns its CDP url. Reuses a running session unless relaunch is set

- MCP tool: `argus_launch_profile`

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/launch-automation" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "relaunch": false }'
```

### POST /v1/profiles/cdp

Where a running profile's CDP endpoint is, without launching it

- MCP tool: `argus_profile_session`

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/cdp" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>" }'
```

### POST /v1/profiles/close-automation

Close a session this key opened

- MCP tool: `argus_close_profile`

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/close-automation" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>" }'
```

### POST /v1/profiles/delete

Move a profile to Trash (permanent: true to purge; the tool only soft-deletes)

- MCP tool: `argus_delete_profile`

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/delete" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "permanent": false }'
```

### POST /v1/profiles/update-fingerprint

Re-roll or override a profile's fingerprint

- MCP tool: `argus_update_fingerprint`

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/update-fingerprint" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "fingerprint": { "canvas": "Noise" } }'
```

## Proxies

The proxy library, and the check that tells you whether an entry still works. Listing proxies never returns their credentials: the response is read by a model as often as by your code.

### GET /v1/proxies

List proxies

- MCP tool: `argus_list_proxies`

```sh
curl -X GET "http://127.0.0.1:39219/v1/proxies" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/proxies/create

Add proxy

- MCP tool: none — HTTP only

```sh
curl -X POST "http://127.0.0.1:39219/v1/proxies/create" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "US proxy", "type": "socks5", "host": "1.2.3.4", "port": 1080 }'
```

### POST /v1/proxies/update

Update a proxy

- MCP tool: none — HTTP only

```sh
curl -X POST "http://127.0.0.1:39219/v1/proxies/update" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "proxyId": "<id>", "name": "US proxy" }'
```

### POST /v1/proxies/check

Check reachability and egress IP

- MCP tool: `argus_check_proxy`

```sh
curl -X POST "http://127.0.0.1:39219/v1/proxies/check" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "host": "1.2.3.4", "port": 1080, "type": "socks5" }'
```

### POST /v1/proxies/delete

Remove a proxy

- MCP tool: none — HTTP only

```sh
curl -X POST "http://127.0.0.1:39219/v1/proxies/delete" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "proxyId": "<id>" }'
```

### POST /v1/proxies/reimport

Re-import proxies from a file on disk

- MCP tool: none — HTTP only

```sh
curl -X POST "http://127.0.0.1:39219/v1/proxies/reimport" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "filePath": "/path/to/proxies.txt" }'
```

## Cookies

Bulk cookie import, for moving a jar you already have into a profile. Both routes are HTTP-only — no agent tool fronts them.

### POST /v1/cookies/bulk-match

Match exported cookie files in a folder to profiles by name

- MCP tool: none — HTTP only

```sh
curl -X POST "http://127.0.0.1:39219/v1/cookies/bulk-match" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "folderPath": "/path/to/cookies", "profileIds": ["<id>"] }'
```

### POST /v1/cookies/push-local

Attach a cookie file on disk to one profile

- MCP tool: none — HTTP only

```sh
curl -X POST "http://127.0.0.1:39219/v1/cookies/push-local" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "filePath": "/path/to/cookies.json" }'
```

## Monitoring

Report the outcome of a run back to the launcher, with an optional screenshot, so a sweep across many profiles has somewhere to land.

### POST /v1/monitoring/report

Report a run's outcome from an external script

- MCP tool: none — HTTP only

```sh
curl -X POST "http://127.0.0.1:39219/v1/monitoring/report" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "status": "ok" }'
```

## Automations

Author and run automations. Creating, changing and deleting one needs a key with no folder scope, because automations are shared across every folder and have none of their own. A folder-scoped key may still list, read and run them.

### GET /v1/automations

List the org's automations, with step counts

- MCP tool: `argus_list_automations`

List the automations in this workspace. Returns id, name, description, step count and where each is wired in.

```sh
curl -X GET "http://127.0.0.1:39219/v1/automations" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### GET /v1/automations/schema

The step catalogue: every step type, its fields and how they validate

- MCP tool: `argus_automation_schema`

The step vocabulary an automation is written in. Call this before argus_create_automation rather than guessing field names.

```sh
curl -X GET "http://127.0.0.1:39219/v1/automations/schema" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/automations/get

Read one automation, including its full step tree

- MCP tool: `argus_get_automation`

Read one automation's full step tree, in the same shape argus_create_automation accepts.

Fields:
- `automationId` (string, required)

```sh
curl -X POST "http://127.0.0.1:39219/v1/automations/get" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "automationId": "<id>" }'
```

### POST /v1/automations/create

Create an automation. Steps are validated before anything is stored

- MCP tool: `argus_create_automation`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Create an automation from a list of steps. Call argus_automation_schema first for the step vocabulary. Steps are validated before anything is stored, and the error names the exact path that failed.

Fields:
- `name` (string, required)
- `description` (string)
- `steps` (steps, required) — The step tree. See argus_automation_schema for the field list of each type.
- `tags` (tags) — Free-text labels, at most 5. Extras are dropped.
- `pinned` (boolean) — Show as a tile on every profile's start page.
- `timeoutMs` (number) — Whole-run ceiling in milliseconds. Capped at 600000.
- `closeOnFinish` (boolean) — Close the browser when the run ends. Defaults true for runs started over the API.
- `icon` (string) — Card icon, as brand:<slug> from the shared brand catalog (brand:facebook, brand:instagram, ...). Empty clears it.
- `color` (string) — Card colour: slate, blue, green, violet, red, amber, or a #rrggbb hex. Tints the card's plate, or its frame when an icon is set.
- `folderId` (string) — File it in an automation folder. Use the id from the automations list in argus_list_folders. Empty leaves it in All automations. Filing only: it changes nothing about how or when the automation runs.
- `notifyOn` (string) — Notify when a run finishes: always or failure. Empty clears it. Delivery is the Argus bell and a desktop notification, plus notifyConnectorId when set.
- `notifyConnectorId` (string) — A message connector id to additionally send the finish notification through. Requires notifyOn.
- `variables` (object) — Seed variables every run starts with, readable as {{vars.<name>}}. Prefer `parameters` for anything a person or a profile should be able to set -- a declared parameter of the same name shadows an entry here.
- `parameters` (objects) — What this automation asks for before it runs: an ordered list of {name, kind, label?, required?, default?, options?, hint?}. `name` matches ^[A-Za-z_][A-Za-z0-9_]*$ and is read from any interpolated step field as {{vars.<name>}}. `kind` is one of text, textarea, number, boolean, select, secret, list; `options` is required on select, and a boolean may not be required. Each profile can hold its own value (argus_update_profile automationVars), and a run can override them (argus_run_automation vars).
- `schedule` (object) — Run on a schedule while the launcher is open: {enabled, kind: interval|daily|weekly, everyMinutes?, at: "HH:MM"?, days: [0-6]?, profileIds: ["<id>"]}. Missed slots are skipped. Null clears it.

```sh
curl -X POST "http://127.0.0.1:39219/v1/automations/create" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Sign in", "steps": [{ "id": "s1", "type": "goto", "url": "https://example.com" }] }'
```

### POST /v1/automations/update

Change an automation's name, description, steps or wiring

- MCP tool: `argus_update_automation`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Change an existing automation. Only the fields you send are written; omitting steps leaves the step tree alone.

Fields:
- `automationId` (string, required)
- `name` (string)
- `description` (string)
- `steps` (steps) — Replaces the whole step tree. Omit to leave it unchanged.
- `tags` (tags) — Replaces the whole list. At most 5.
- `pinned` (boolean)
- `timeoutMs` (number)
- `closeOnFinish` (boolean)
- `icon` (string) — Card icon, as brand:<slug> from the shared brand catalog. Empty clears it.
- `color` (string) — Card colour: slate, blue, green, violet, red, amber, or a #rrggbb hex. Tints the card's plate, or its frame when an icon is set.
- `folderId` (string) — Move it to an automation folder. Use the id from the automations list in argus_list_folders. Empty moves it back to All automations. Filing only: it changes nothing about how or when the automation runs.
- `notifyOn` (string) — Notify when a run finishes: always or failure. Empty clears it.
- `notifyConnectorId` (string) — A message connector id to additionally send the finish notification through. Requires notifyOn.
- `variables` (object) — Replaces the seed variables every run starts with. A declared parameter of the same name shadows an entry here.
- `parameters` (objects) — Replaces the whole list of what this automation asks for before it runs: an ordered list of {name, kind, label?, required?, default?, options?, hint?}. `name` matches ^[A-Za-z_][A-Za-z0-9_]*$ and is read from any interpolated step field as {{vars.<name>}}. `kind` is one of text, textarea, number, boolean, select, secret, list; `options` is required on select, and a boolean may not be required. Each profile can hold its own value (argus_update_profile automationVars), and a run can override them (argus_run_automation vars).
- `schedule` (object) — Replaces the schedule: {enabled, kind: interval|daily|weekly, everyMinutes?, at: "HH:MM"?, days: [0-6]?, profileIds: ["<id>"]}. Null clears it.

```sh
curl -X POST "http://127.0.0.1:39219/v1/automations/update" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "automationId": "<id>", "name": "Sign in" }'
```

### POST /v1/automations/delete

Move an automation to Trash. It stops running on launch and on its schedule

- MCP tool: `argus_delete_automation`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Move an automation to Trash, where it stays for 30 days and can be restored from the app. It immediately stops running on launch and on any schedule, and disappears from argus_list_automations. Profiles keep it attached so that a restore puts them back as they were. Past runs stay readable. There is no API to restore or to delete permanently: both are done in the app.

Fields:
- `automationId` (string, required)

```sh
curl -X POST "http://127.0.0.1:39219/v1/automations/delete" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "automationId": "<id>" }'
```

### POST /v1/automations/run

Run an automation against a profile, launching it if needed

- MCP tool: `argus_run_automation`

Run an automation against a profile, launching the profile if it is not already open. Returns a run id immediately; the run continues in the background. One call is one profile: to run the same automation with different values on several profiles, call it once per profile with that profile's vars, which also gives you a run id per profile to poll.

Fields:
- `automationId` (string, required)
- `profileId` (string, required)
- `vars` (object) — Values for this run, readable as {{vars.<name>}}. A name the automation declares as a parameter is coerced to that parameter's kind and beats both the automation's default and the profile's own stored value; any other name is passed through as a plain seed variable, exactly as this field has always worked.

```sh
curl -X POST "http://127.0.0.1:39219/v1/automations/run" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "automationId": "<id>", "profileId": "<id>" }'
```

### POST /v1/automations/runs

Recent runs of one automation, with status and errors

- MCP tool: `argus_automation_runs`

Recent runs of an automation, newest first: status (running, ok, partial, failed, cancelled), timing, the failed step and the error. This is how to learn how a run started with argus_run_automation ended.

Fields:
- `automationId` (string, required)
- `limit` (number) — How many runs, newest first. Default 20, at most 50.

```sh
curl -X POST "http://127.0.0.1:39219/v1/automations/runs" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "automationId": "<id>" }'
```

## Schedule

A schedule entry is an ordered list of automations with a time on it — a different thing from the schedule an automation carries itself, which runs that one automation alone. Entries fire only while the launcher is open: a time it was closed for is recorded as missed and skipped, never caught up. Occurrences is the record of what ran, not the plan for what will.

### GET /v1/schedule/entries

Every scheduled workflow in the workspace

- MCP tool: `argus_list_schedule_entries`

The workspace's scheduled workflows: each one's id, name, colour, whether it is enabled, when it repeats, and its ordered steps. A schedule entry is several automations run one after another at a wall-clock time -- distinct from an automation's OWN schedule, which is the `schedule` field on argus_get_automation. Read this before updating one: an update replaces the whole recurrence or the whole step list, so you need what is there now.

```sh
curl -X GET "http://127.0.0.1:39219/v1/schedule/entries" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/schedule/entries/create

Create a scheduled workflow

- MCP tool: `argus_create_schedule_entry`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Schedule automations to run one after another at a set time. `recurrence` is {kind, at, date?, days?, from?, until?, tz?}: kind is 'once' (with date 'YYYY-MM-DD'), 'daily', or 'weekly' (with days, 0=Sunday..6=Saturday); at is 'HH:MM' 24-hour; tz is an optional IANA zone like 'Europe/Berlin', and without it the time is read on whatever machine runs it. `from` and `until` bound a daily or weekly schedule to a date range, both 'YYYY-MM-DD' and both inclusive -- WITHOUT `until` a repeat runs forever, so set it whenever the request names a period ('next week', 'until the 30th'); a 'once' takes neither, since it already names its day. `steps` is an ordered list of {automationId, profileIds, stopOnFail}: each step runs its automation on each profile in turn, the next step starts when the last finishes, and an empty profileIds means a browserless run. Get ids from argus_list_automations and argus_list_profiles. Entries fire only while the Argus launcher is open; a time it was closed for is skipped, never caught up.

Fields:
- `name` (string, required)
- `recurrence` (object, required) — {kind: 'once'|'daily'|'weekly', at: 'HH:MM', date?: 'YYYY-MM-DD', days?: int[], from?: 'YYYY-MM-DD', until?: 'YYYY-MM-DD', tz?: IANA zone}. from/until bound a daily or weekly run to a date range, inclusive; without until it repeats forever. Neither is allowed on 'once'.
- `steps` (objects, required) — Ordered [{automationId, profileIds: string[], stopOnFail: boolean}]. At least one.
- `enabled` (boolean) — Defaults to true. A disabled entry keeps its steps, stops firing, and is never counted as missed.
- `color` (string) — The calendar dot's colour: slate, blue, green, violet, red, amber, or a #rrggbb hex. Random when omitted.

```sh
curl -X POST "http://127.0.0.1:39219/v1/schedule/entries/create" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Weekday scrape", "recurrence": { "kind": "weekly", "at": "09:00", "days": [1, 2, 3, 4, 5], "from": "2026-08-17", "until": "2026-08-21" }, "steps": [{ "automationId": "<id>", "profileIds": ["<id>"], "stopOnFail": true }] }'
```

### POST /v1/schedule/entries/update

Change a scheduled workflow's time, steps, colour or enabled state

- MCP tool: `argus_update_schedule_entry`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Change a scheduled workflow. Every field is optional and anything omitted is left alone -- but `recurrence` and `steps` each replace their whole value rather than merging, so read the entry with argus_list_schedule_entries first and send the complete new version. Switching `enabled` off is the way to pause a schedule without losing it; adding `until` to its recurrence is the way to make it stop on a given day and need nobody to come back for it.

Fields:
- `entryId` (string, required)
- `name` (string)
- `recurrence` (object) — Replaces the whole recurrence. Same shape as on create.
- `steps` (objects) — Replaces the whole step list. Omit to leave the steps unchanged.
- `enabled` (boolean)
- `color` (string)

```sh
curl -X POST "http://127.0.0.1:39219/v1/schedule/entries/update" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "entryId": "<id>", "enabled": false }'
```

### POST /v1/schedule/entries/delete

Delete a scheduled workflow

- MCP tool: `argus_delete_schedule_entry`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Delete a scheduled workflow. There is no Trash for schedule entries and no undo: the entry and its whole run history go. The automations it named are untouched -- an entry only points at them. To stop a schedule without losing it, send enabled: false to argus_update_schedule_entry instead.

Fields:
- `entryId` (string, required)

```sh
curl -X POST "http://127.0.0.1:39219/v1/schedule/entries/delete" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "entryId": "<id>" }'
```

### POST /v1/schedule/occurrences

What the calendar actually ran over a range of days

- MCP tool: `argus_schedule_history`

What the schedule actually did between two dates: one row per fired-or-missed slot, with its status (running, ok, partial, failed, missed), the per-step outcome, and the run ids each step produced. 'missed' means the launcher was not open at that time -- those are skipped, never caught up. This is the record of what happened; argus_list_schedule_entries is the plan for what will.

Fields:
- `from` (string, required) — First day of the range, 'YYYY-MM-DD', inclusive.
- `to` (string, required) — Last day of the range, 'YYYY-MM-DD', inclusive. At most 92 days after `from`.

```sh
curl -X POST "http://127.0.0.1:39219/v1/schedule/occurrences" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "from": "2026-08-01", "to": "2026-08-31" }'
```

### GET /v1/schedule/keep-awake

Whether this computer is being held awake for the schedule

- MCP tool: `argus_keep_awake`

Whether Keep awake is on for the machine running this launcher. Schedules and automation timers only tick while the launcher is running, so a computer that idles to sleep misses them -- 'missed' is what the calendar then records, and missed times are skipped rather than caught up. This reports what the power blocker is actually doing, not what was last asked of it.

```sh
curl -X GET "http://127.0.0.1:39219/v1/schedule/keep-awake" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/schedule/keep-awake

Hold this computer awake so scheduled runs are not missed

- MCP tool: `argus_set_keep_awake`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Turn Keep awake on or off for the machine running this launcher, and persist it across restarts. It prevents IDLE sleep only: a closed lid and a manual sleep both still sleep the machine, and no setting here can override them -- say that rather than promising an overnight run on a shut laptop. The reply is what the power blocker actually reports, so a refused start comes back as enabled:false rather than as an error.

Fields:
- `enabled` (boolean, required)

```sh
curl -X POST "http://127.0.0.1:39219/v1/schedule/keep-awake" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "enabled": true }'
```

### GET /v1/schedule/notifications

Whether the signed-in user hears about their scheduled days

- MCP tool: `argus_schedule_notifications`

Whether the signed-in user gets Telegram messages about their scheduled days: `notifyFailures` when a scheduled run fails, partly fails or is missed, and `dailySummary` once the day's last run has settled. Both default to off. These are PER USER, not per workspace -- like argus_set_telegram_pref and unlike an automation's notifyOn -- and they are inert unless `telegramLinked` is true, whatever the switches say. `overrides` lists the single days that disagree with the defaults, over the range you ask for; a null in one means that day inherits. Distinct from the per-automation messages argus_telegram_status reports: these are about the Schedule tab's calendar as a whole, and there is no per-schedule setting.

Fields:
- `from` (string) — First day of the override range, 'YYYY-MM-DD'. Omit both to skip the overrides.
- `to` (string) — Last day of the override range, 'YYYY-MM-DD', inclusive. At most 92 days after `from`.

```sh
curl -X GET "http://127.0.0.1:39219/v1/schedule/notifications" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/schedule/notifications

Turn the schedule's Telegram messages on or off

- MCP tool: `argus_set_schedule_notifications`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Turn the signed-in user's schedule Telegram messages on or off. With no `day` this sets the default for every day. With a `day` it overrides that one day only, and there the row is written whole: send both switches, because one you leave out goes back to inheriting the default rather than keeping its value, and sending null for both removes the override. Setting a switch on does NOT create a Telegram link -- check `telegramLinked` on argus_schedule_notifications first, since without one nothing is ever sent. Linking is done in the app; no route can do it.

Fields:
- `notifyFailures` (boolean) — Message this user when a scheduled run fails, partly fails, or is missed.
- `dailySummary` (boolean) — Message this user once the day's last scheduled run has reached a terminal state.
- `day` (string) — 'YYYY-MM-DD' to override one day instead of setting the default. Null in a switch means that day inherits the default.

```sh
curl -X POST "http://127.0.0.1:39219/v1/schedule/notifications" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "notifyFailures": true, "dailySummary": false }'
```

## Connectors

### GET /v1/connectors

The workspace's connectors, and the field list of every kind

- MCP tool: `argus_list_connectors`

The AI, message and data connectors this workspace has, and the catalogue of kinds one can be created from. Call this before writing a notify, aiPrompt, aiCheck, aiAgent, saveRows or loadRows step, or setting notifyConnectorId -- those fields take an id from here and there is no other way to learn one. A step must name a connector of the matching category: 'ai' for the AI steps and an aiAgent's chat model, 'message' for notify and an agent's message or approval tools, 'data' for saveRows, loadRows and an agent's data or vector tools. Every connector carries its stored `config`, credentials included -- an API key reads back everything it can write, so treat a key as equivalent to the credentials behind it.

```sh
curl -X GET "http://127.0.0.1:39219/v1/connectors" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/connectors/create

Add a connector. Owner-only, like the Connectors view

- MCP tool: `argus_create_connector`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Add a connector -- a Telegram bot, a Slack or Discord webhook, WhatsApp, SMTP, an AI provider, or somewhere to save results: Supabase, Postgres, Redis, Firebase, Google Sheets, Notion, Airtable or a local file. Call argus_list_connectors first for the kind's field list. `config` carries the credentials; anything you send here is in this conversation's transcript, and argus_list_connectors returns stored values to every key. Returns the connector including its saved config.

Fields:
- `name` (string, required) — What the connector is called in the app.
- `kind` (string, required) — A messaging kind (telegram, slack, discord, whatsapp, smtp), a data kind (supabase, postgres, redis, firestore, sheets, notion, airtable, file) or an AI kind. See the `kinds` block of argus_list_connectors. The category follows from the kind and cannot be set.
- `config` (object) — The kind's fields, as a flat object of strings. Missing required fields are a 400 naming each one.
- `isDefault` (boolean) — Make this the default for its category, used by steps that name no connector. The first connector in a category is the default whether or not this is sent.

```sh
curl -X POST "http://127.0.0.1:39219/v1/connectors/create" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Ops Telegram", "kind": "telegram", "config": { "botToken": "...", "chatId": "-1004281234567" } }'
```

### POST /v1/connectors/update

Rename a connector, change its config or make it the default

- MCP tool: `argus_update_connector`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Change a connector. `config` merges key by key, so a chat id can be changed without re-sending the bot token, and omitting config entirely leaves every stored credential alone. The response includes the full stored config. The kind cannot be changed -- delete and recreate instead.

Fields:
- `connectorId` (string, required)
- `name` (string)
- `config` (object) — Merged into the stored config. Send only the keys that change; send a key as "" to clear it.
- `isDefault` (boolean) — true promotes this connector and demotes the previous default in its category. false is refused -- promote another one instead, so a category is never left without a default.

```sh
curl -X POST "http://127.0.0.1:39219/v1/connectors/update" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "connectorId": "<id>", "config": { "chatId": "-1004281234567" } }'
```

### POST /v1/connectors/delete

Delete a connector

- MCP tool: `argus_delete_connector`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Delete a connector. Steps that still name it start failing with a sentence saying it is gone -- they do not fall back to another connector. Check argus_list_automations first if you are not sure what uses it.

Fields:
- `connectorId` (string, required)

```sh
curl -X POST "http://127.0.0.1:39219/v1/connectors/delete" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "connectorId": "<id>" }'
```

### POST /v1/connectors/test

Send a real test message, or a one-word AI completion

- MCP tool: `argus_test_connector`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Prove a connector works: a message connector sends a real test message, an AI connector asks for a one-word completion, a data connector performs the smallest read the service offers and never writes. Reports the service's own error text when it fails, which is usually the whole diagnosis. Do this after creating one rather than waiting for a run to fail. Note that a data connector passing only proves it can connect and read -- not that a save will be permitted.

Fields:
- `connectorId` (string, required)

```sh
curl -X POST "http://127.0.0.1:39219/v1/connectors/test" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "connectorId": "<id>" }'
```

## Workspace

### GET /v1/folders

The folders profiles, proxies, cookie sets and automations are filed in

- MCP tool: `argus_list_folders`

The workspace's folders, grouped by what they file: profiles, proxies, cookies, automations. Each carries the id its folderId argument takes; the four groups are separate namespaces, so an automation's folderId must come from the automations group. A folder-scoped key sees only the profile folders it is scoped to. Read-only: folders are created and renamed in the app.

```sh
curl -X GET "http://127.0.0.1:39219/v1/folders" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### GET /v1/statuses

Every status label a profile, proxy or cookie set can carry

- MCP tool: `argus_list_statuses`

The status labels this workspace uses, built-in and custom. argus_update_profile.status takes a free string, so without this an agent can write a status the app never offers and the row lands looking broken. Read-only: custom statuses are created in the app.

```sh
curl -X GET "http://127.0.0.1:39219/v1/statuses" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

## Notifications

### GET /v1/telegram

Whether the notification bot is set up, and who is subscribed

- MCP tool: `argus_telegram_status`

The state of personal Telegram notifications: whether the workspace has a notification bot, whether the signed-in user has linked their Telegram, and which automations they get messages about. This is separate from telegram connectors -- it is per person, and it is what answers 'why did that run not message me'. Neither the bot token nor the chat id is returned.

```sh
curl -X GET "http://127.0.0.1:39219/v1/telegram" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/telegram/pref

Subscribe the signed-in user to one automation's outcomes

- MCP tool: `argus_set_telegram_pref`

Turn personal Telegram messages on or off for one automation, for the signed-in user. This is that person's own setting and is independent of the automation's notifyOn, which is the workspace-wide one. Requires a linked Telegram -- check argus_telegram_status first.

Fields:
- `automationId` (string, required)
- `notifyOn` (string) — always or failure. Empty unsubscribes. 'failure' includes a run that finished with a failed step.

```sh
curl -X POST "http://127.0.0.1:39219/v1/telegram/pref" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "automationId": "<id>", "notifyOn": "failure" }'
```

### POST /v1/telegram/bot

Set the workspace's notification bot. Owner-only

- MCP tool: `argus_set_telegram_bot`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Point the workspace at a Telegram bot from BotFather, so members can link their own chats to it. Write-only: the token is never returned by any route. Members still have to press Start in Telegram themselves, which only the app can walk them through -- this call does not link anybody.

Fields:
- `botName` (string, required) — The bot's @username, with or without the @.
- `botToken` (string, required) — The token BotFather issued. Stored for the whole workspace and never read back out over this API.

```sh
curl -X POST "http://127.0.0.1:39219/v1/telegram/bot" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "botName": "myworkspacebot", "botToken": "123456789:AA..." }'
```

## Tables

Read and set which columns the launcher's own tables show. The one part of the API that changes what you see rather than what runs.

### GET /v1/tables/columns

What each table shows, and every column it could show

- MCP tool: `argus_table_columns`

The columns the Profiles, Proxies and Cookies tables are showing, and every column they could show. Call this before argus_set_table_columns rather than guessing ids: the reply gives each column's id, label, one-line description, whether it is on, and whether it can be turned off.

```sh
curl -X GET "http://127.0.0.1:39219/v1/tables/columns" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/tables/columns

Show or hide columns on a table

- MCP tool: `argus_set_table_columns`

Show or hide columns on the Profiles, Proxies or Cookies table. Send `columns` for the exact set, or `show`/`hide` to change a few without restating the rest. Columns always render in the app's own order, so the order you send is ignored. The layout belongs to the signed-in user and follows them to every machine.

Fields:
- `table` (string, required) — profiles, proxies or cookies.
- `columns` (strings) — The exact set of visible columns, by id. Locked columns are added back.
- `show` (strings) — Columns to turn on, leaving every other column as it is.
- `hide` (strings) — Columns to turn off. Refusing a locked column is a 400.
- `reset` (boolean) — Put the table back to its default columns. Applied before show and hide.

```sh
curl -X POST "http://127.0.0.1:39219/v1/tables/columns" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "table": "profiles", "show": ["email", "fpTimezone"], "hide": ["tags"] }'
```

## Skills

A skill is the working brief the in-app assistant loads for one job — instructions, worked examples, a reference shelf, and the pack of tools it may reach. Saving one creates a custom skill, or writes a local override of a built-in. Deleting is not the mirror image: a custom skill goes, while a built-in is reset to the brief it shipped with.

### GET /v1/skills/schema

The vocabulary a skill is written in: tool packs, entry tabs, shelf ids

- MCP tool: `argus_skills_schema`

The vocabulary an assistant skill is written in. Call this before argus_save_skill rather than guessing: it returns the tool PACK names a skill may claim (with the tools in each), the tab ids that may be claimed as entry tabs and who holds each one, every knowledge-base article id available for the reference shelf, and the field limits. Every tab is held by a built-in and a custom skill outranks one, so `claimedBy` is context rather than a blocklist -- only a tab held by another CUSTOM skill is refused. The same role argus_automation_schema plays for steps.

```sh
curl -X GET "http://127.0.0.1:39219/v1/skills/schema" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### GET /v1/skills

List the assistant's skills, built-in and custom

- MCP tool: `argus_list_skills`

List the working briefs the in-app assistant can load. Each entry gives the id, title, one-line blurb, whether it is built-in, whether a built-in has been edited locally, its entry tabs and how many tools it reaches. Read one in full with argus_get_skill.

```sh
curl -X GET "http://127.0.0.1:39219/v1/skills" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/skills/get

Read one skill, including its full brief

- MCP tool: `argus_get_skill`

Read one skill in the shape argus_save_skill accepts: instructions, examples, entry tabs, reference shelf and tool pack. Read before editing -- a save replaces the whole document, so an unread field is a field you will erase.

Fields:
- `skillId` (string, required) — From argus_list_skills.

```sh
curl -X POST "http://127.0.0.1:39219/v1/skills/get" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "skillId": "proxies" }'
```

### POST /v1/skills/save

Create a custom skill, or edit a built-in one

- MCP tool: `argus_save_skill`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Create or edit an assistant skill. Omit skillId to create (the id is slugged from the title and returned); title and instructions are required then. Pass a built-in's id to edit it -- that writes a local override, and only title, blurb, instructions, examples and docs apply; sending toolPack or entryTabs for a built-in is refused, because those are code-defined. EVERY FIELD YOU SEND REPLACES THAT FIELD and fields you omit are left as they are, so call argus_get_skill first and send only what changes; an empty examples array deletes them. toolPack must be one of the packs argus_skills_schema lists; it is the ONLY way a skill gets tools, and naming tools directly is not possible by design. Unknown doc ids and already-claimed tabs are dropped and reported back in `dropped` rather than failing the call. THIS CALL BLOCKS: it raises an approve-or-deny card in the app naming your key, and returns 403 if the user denies or lets it lapse.

Fields:
- `skillId` (string) — Omit to create a custom skill. A built-in's id edits that built-in.
- `title` (string) — Required when CREATING; omit it when editing and the current title is kept. Up to 60 characters.
- `blurb` (string) — One line, up to 200 characters. The ONLY part always in the assistant's context -- it is what the model reads to decide the skill applies, so write it as a routing decision, not a summary.
- `instructions` (string) — The working brief. Required when CREATING; omit it when editing and the current brief is kept. Never hand-write a reference shelf or an example list into it -- both are appended from the fields below.
- `examples` (strings) — Two to four verbatim requests in the user's voice, one line each, up to 120 characters.
- `toolPack` (string) — A pack name from argus_skills_schema. Absent means documentation and navigation only. Custom skills only.
- `entryTabs` (strings) — Tab ids this skill is suggested on, taking the tab from whichever built-in covers it. Only a tab another CUSTOM skill already holds is dropped. Custom skills only.
- `docs` (strings) — Knowledge-base article ids for the reference shelf, e.g. concepts/proxies. Ids the KB does not have are dropped.
- `hidden` (boolean) — Built-ins only: true removes the skill from the assistant entirely. Undo it with argus_delete_skill on the same id.

```sh
curl -X POST "http://127.0.0.1:39219/v1/skills/save" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "title": "Warm-up runs", "blurb": "Plan and check profile warm-up passes.", "instructions": "You are now...", "examples": ["Warm up my five new profiles"], "toolPack": "profiles", "docs": ["concepts/profiles-fingerprints"] }'
```

### POST /v1/skills/delete

Delete a custom skill, or reset a built-in to its shipped brief

- MCP tool: `argus_delete_skill`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

On a CUSTOM skill id this removes the skill. On a BUILT-IN id it does not remove anything -- it clears the local override and restores the shipped brief, which is also how a hidden built-in comes back. That asymmetry is deliberate: built-ins ship with the app and cannot be deleted, only edited or hidden. The user must approve this in the app.

Fields:
- `skillId` (string, required) — A custom id removes it; a built-in id resets it to the shipped brief.

```sh
curl -X POST "http://127.0.0.1:39219/v1/skills/delete" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "skillId": "custom-warm-up-runs" }'
```

## Driving a page

Five tools with no endpoint behind them. They attach to a profile that is already open and speak to the page directly, which is how an agent reads a page, clicks through it and takes a screenshot.

### argus_list_tabs (MCP tool)

List the open pages in a running profile

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

### argus_navigate (MCP tool)

Point a page at a URL and wait for it to settle

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

### argus_read_page (MCP tool)

Read a page's visible text, whole or by CSS selector

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

### argus_screenshot (MCP tool)

Screenshot a page — JPEG by default, PNG on request

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

### argus_eval (MCP tool)

Evaluate a JavaScript expression in a page and return its value

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

## Status codes

Every response carries a `status` boolean; a failure adds a `msg` saying what went wrong.

- `401` — No bearer token, or one that does not match a stored key.
- `403` — The key is scoped to folders that do not contain that profile, or it is trying to author an automation while folder-scoped. Not a missing profile, and not worth retrying.
- `400` — The body failed validation. The message names the exact path that failed.
- `404` — No such route.
- `501` — A documented route that is not implemented yet.
- `503` — Argus Launcher is not open. Nothing can answer until it is.
- `504` — The launcher window did not answer in time.

---

Generated from https://www.browserargus.com/api-reference
