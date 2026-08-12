# Argus MCP tools

53 tools, across 55 endpoints. This is the list an MCP client receives from `tools/list`; the full request and response detail for each one is in [api-reference.md](./api-reference.md).

This is a local API. It listens on loopback on the machine running Argus, and nothing off that machine can reach it. There is no hosted web API.

## Profiles

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_profiles` | `GET /v1/profiles` | List profiles (optional ?folder=<id>) |
| `argus_get_profile` | `POST /v1/profiles/get` | Read one profile |
| `argus_create_profile` | `POST /v1/profiles/create` | Create a profile |
| `argus_update_profile` | `POST /v1/profiles/update` | Update name, status, tags, colour, avatar, folder, proxy mode, start URL or launch automation |
| `argus_profile_notes` | `POST /v1/profiles/notes` | Read a profile's notes, newest first |
| `argus_add_profile_note` | `POST /v1/profiles/notes/add` | Append a note to a profile |
| `argus_assign_proxy` | `POST /v1/profiles/assign-proxy` | Put a profile on a proxy from the library |
| `argus_launch_profile` | `POST /v1/profiles/launch-automation` | Open a profile for automation; returns its CDP url. Reuses a running session unless relaunch is set |
| `argus_profile_session` | `POST /v1/profiles/cdp` | Where a running profile's CDP endpoint is, without launching it |
| `argus_close_profile` | `POST /v1/profiles/close-automation` | Close a session this key opened |
| `argus_delete_profile` | `POST /v1/profiles/delete` | Move a profile to Trash (permanent: true to purge; the tool only soft-deletes) |
| `argus_update_fingerprint` | `POST /v1/profiles/update-fingerprint` | Re-roll or override a profile's fingerprint |

## Proxies

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_proxies` | `GET /v1/proxies` | List proxies |
| `argus_check_proxy` | `POST /v1/proxies/check` | Check reachability and egress IP |

## Automations

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_automations` | `GET /v1/automations` | List the org's automations, with step counts |
| `argus_automation_schema` | `GET /v1/automations/schema` | The step catalogue: every step type, its fields and how they validate |
| `argus_get_automation` | `POST /v1/automations/get` | Read one automation, including its full step tree |
| `argus_create_automation` | `POST /v1/automations/create` | Create an automation. Steps are validated before anything is stored |
| `argus_update_automation` | `POST /v1/automations/update` | Change an automation's name, description, steps or wiring |
| `argus_delete_automation` | `POST /v1/automations/delete` | Move an automation to Trash. It stops running on launch and on its schedule |
| `argus_run_automation` | `POST /v1/automations/run` | Run an automation against a profile, launching it if needed |
| `argus_automation_runs` | `POST /v1/automations/runs` | Recent runs of one automation, with status and errors |

## Schedule

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_schedule_entries` | `GET /v1/schedule/entries` | Every scheduled workflow in the workspace |
| `argus_create_schedule_entry` | `POST /v1/schedule/entries/create` | Create a scheduled workflow |
| `argus_update_schedule_entry` | `POST /v1/schedule/entries/update` | Change a scheduled workflow's time, steps, colour or enabled state |
| `argus_delete_schedule_entry` | `POST /v1/schedule/entries/delete` | Delete a scheduled workflow |
| `argus_schedule_history` | `POST /v1/schedule/occurrences` | What the calendar actually ran over a range of days |
| `argus_keep_awake` | `GET /v1/schedule/keep-awake` | Whether this computer is being held awake for the schedule |
| `argus_set_keep_awake` | `POST /v1/schedule/keep-awake` | Hold this computer awake so scheduled runs are not missed |
| `argus_schedule_notifications` | `GET /v1/schedule/notifications` | Whether the signed-in user hears about their scheduled days |
| `argus_set_schedule_notifications` | `POST /v1/schedule/notifications` | Turn the schedule's Telegram messages on or off |

## Connectors

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_connectors` | `GET /v1/connectors` | The workspace's connectors, and the field list of every kind |
| `argus_create_connector` | `POST /v1/connectors/create` | Add a connector. Owner-only, like the Connectors view |
| `argus_update_connector` | `POST /v1/connectors/update` | Rename a connector, change its config or make it the default |
| `argus_delete_connector` | `POST /v1/connectors/delete` | Delete a connector |
| `argus_test_connector` | `POST /v1/connectors/test` | Send a real test message, or a one-word AI completion |

## Workspace

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_folders` | `GET /v1/folders` | The folders profiles, proxies, cookie sets and automations are filed in |
| `argus_list_statuses` | `GET /v1/statuses` | Every status label a profile, proxy or cookie set can carry |

## Notifications

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_telegram_status` | `GET /v1/telegram` | Whether the notification bot is set up, and who is subscribed |
| `argus_set_telegram_pref` | `POST /v1/telegram/pref` | Subscribe the signed-in user to one automation's outcomes |
| `argus_set_telegram_bot` | `POST /v1/telegram/bot` | Set the workspace's notification bot. Owner-only |

## Tables

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_table_columns` | `GET /v1/tables/columns` | What each table shows, and every column it could show |
| `argus_set_table_columns` | `POST /v1/tables/columns` | Show or hide columns on a table |

## Skills

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_skills_schema` | `GET /v1/skills/schema` | The vocabulary a skill is written in: tool packs, entry tabs, shelf ids |
| `argus_list_skills` | `GET /v1/skills` | List the assistant's skills, built-in and custom |
| `argus_get_skill` | `POST /v1/skills/get` | Read one skill, including its full brief |
| `argus_save_skill` | `POST /v1/skills/save` | Create a custom skill, or edit a built-in one |
| `argus_delete_skill` | `POST /v1/skills/delete` | Delete a custom skill, or reset a built-in to its shipped brief |

## Driving a page

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_tabs` | _CDP, no endpoint_ | List the open pages in a running profile |
| `argus_navigate` | _CDP, no endpoint_ | Point a page at a URL and wait for it to settle |
| `argus_read_page` | _CDP, no endpoint_ | Read a page's visible text, whole or by CSS selector |
| `argus_screenshot` | _CDP, no endpoint_ | Screenshot a page — JPEG by default, PNG on request |
| `argus_eval` | _CDP, no endpoint_ | Evaluate a JavaScript expression in a page and return its value |

## Endpoints with no tool

7 routes are reachable over HTTP only, on purpose. An agent cannot call them however plainly they are documented.

- `POST /v1/proxies/create` — Add proxy
- `POST /v1/proxies/update` — Update a proxy
- `POST /v1/proxies/delete` — Remove a proxy
- `POST /v1/proxies/reimport` — Re-import proxies from a file on disk
- `POST /v1/cookies/bulk-match` — Match exported cookie files in a folder to profiles by name
- `POST /v1/cookies/push-local` — Attach a cookie file on disk to one profile
- `POST /v1/monitoring/report` — Report a run's outcome from an external script
