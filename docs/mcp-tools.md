# Argus MCP tools

136 tools, across 126 endpoints. This is the list an MCP client receives from `tools/list`; the full request and response detail for each one is in [api-reference.md](./api-reference.md).

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
| `argus_sharing_report` | `GET /v1/sharing/report` | Every cookie-set and proxy shared past its limit, with the limits themselves |
| `argus_launch_profile` | `POST /v1/profiles/launch-automation` | Open a profile for automation; returns its CDP url. Reuses a running session unless relaunch is set |
| `argus_profile_session` | `POST /v1/profiles/cdp` | Where a running profile's CDP endpoint is, without launching it |
| `argus_close_profile` | `POST /v1/profiles/close-automation` | Close a session this key opened |
| `argus_delete_profile` | `POST /v1/profiles/delete` | Move a profile to Trash (permanent: true to purge; the tool only soft-deletes) |
| `argus_update_fingerprint` | `POST /v1/profiles/update-fingerprint` | Re-roll or override a profile's fingerprint |
| `argus_fingerprint_check` | `POST /v1/profiles/fingerprint-check` | Check a running profile's fingerprint |
| `argus_trash_profiles` | `POST /v1/profiles/trash` | Move profiles to the trash |
| `argus_list_trashed_profiles` | `GET /v1/profiles/trashed` | Profiles currently in the trash |
| `argus_restore_profiles` | `POST /v1/profiles/restore` | Restore profiles from the trash |
| `argus_purge_profiles` | `POST /v1/profiles/purge` | Delete trashed profiles permanently |

## Proxies

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_proxies` | `GET /v1/proxies` | List proxies |
| `argus_create_proxy` | `POST /v1/proxies/create` | Add proxy |
| `argus_update_proxy` | `POST /v1/proxies/update` | Update a proxy |
| `argus_check_proxy` | `POST /v1/proxies/check` | Check reachability and egress IP |
| `argus_delete_proxy` | `POST /v1/proxies/delete` | Remove a proxy |
| `argus_reimport_proxies` | `POST /v1/proxies/reimport` | Re-import proxies from a list of rows |

## Cookies

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_assign_cookie_set` | `POST /v1/profiles/assign-cookies` | Set which profiles launch with a cookie-set |
| `argus_unassign_cookie_set` | `POST /v1/profiles/unassign-cookies` | Detach profiles from whatever cookie-set they hold |
| `argus_list_cookie_sets` | `GET /v1/cookies` | The workspace's cookie sets, metadata only |
| `argus_update_cookie_set` | `POST /v1/cookies/update` | Change a cookie set's name, status, colour, folder or tags |
| `argus_trash_cookie_sets` | `POST /v1/cookies/trash` | Move cookie sets to Trash |
| `argus_list_trashed_cookie_sets` | `GET /v1/cookies/trashed` | The cookie sets currently in Trash |
| `argus_restore_cookie_sets` | `POST /v1/cookies/restore` | Bring cookie sets back out of Trash |
| `argus_purge_cookie_sets` | `POST /v1/cookies/purge` | Permanently delete cookie sets from Trash |

## Automations

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_automations` | `GET /v1/automations` | List the org's automations, with step counts |
| `argus_list_recipes` | `GET /v1/recipes` | List the prebuilt recipes, and which ones this workspace has set up |
| `argus_set_up_recipe` | `POST /v1/recipes/set-up` | Create one prebuilt recipe, plus any tables it needs |
| `argus_automation_schema` | `GET /v1/automations/schema` | The step catalogue: every step type, its fields and how they validate |
| `argus_get_automation` | `POST /v1/automations/get` | Read one automation, including its full step tree |
| `argus_create_automation` | `POST /v1/automations/create` | Create an automation. Steps are validated before anything is stored |
| `argus_update_automation` | `POST /v1/automations/update` | Change an automation's name, description, steps or wiring |
| `argus_delete_automation` | `POST /v1/automations/delete` | Move an automation to Trash. It stops running on launch and on its schedule |
| `argus_run_automation` | `POST /v1/automations/run` | Run an automation against a profile, launching it if needed |
| `argus_automation_runs` | `POST /v1/automations/runs` | Recent runs of one automation, with status and errors |
| `argus_search_automations` | `POST /v1/automations/search` | Find automations by name, step or connector |
| `argus_automation_summary` | `POST /v1/automations/summary` | What one automation does, without its full document |

## Schedule

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_schedule_entries` | `GET /v1/schedule/entries` | Every scheduled workflow in the workspace |
| `argus_create_schedule_entry` | `POST /v1/schedule/entries/create` | Create a scheduled workflow or AI task |
| `argus_update_schedule_entry` | `POST /v1/schedule/entries/update` | Change a scheduled workflow's time, steps, colour or enabled state |
| `argus_delete_schedule_entry` | `POST /v1/schedule/entries/delete` | Delete a scheduled workflow |
| `argus_schedule_history` | `POST /v1/schedule/occurrences` | What the calendar actually ran over a range of days |
| `argus_keep_awake` | `GET /v1/schedule/keep-awake` | Whether this computer is being held awake for the schedule |
| `argus_set_keep_awake` | `POST /v1/schedule/keep-awake` | Hold this computer awake so scheduled runs are not missed |
| `argus_schedule_notifications` | `GET /v1/schedule/notifications` | Whether the signed-in user hears about their scheduled days |
| `argus_set_schedule_notifications` | `POST /v1/schedule/notifications` | Turn the schedule's Telegram messages on or off |
| `argus_schedule_day` | `POST /v1/schedule/day` | One day of the calendar |

## Triggers

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_event_triggers` | `GET /v1/event-triggers` | Every event trigger in the workspace |
| `argus_create_event_trigger` | `POST /v1/event-triggers/create` | Create an event trigger |
| `argus_update_event_trigger` | `POST /v1/event-triggers/update` | Change an event trigger's watch settings, target or enabled state |
| `argus_delete_event_trigger` | `POST /v1/event-triggers/delete` | Delete an event trigger |
| `argus_trigger_history` | `POST /v1/event-triggers/history` | What has fired recently |

## Datasets

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_datasets` | `GET /v1/datasets` | The workspace's datasets, with their declared columns |
| `argus_create_dataset` | `POST /v1/datasets/create` | Create a dataset, optionally with starting columns |
| `argus_update_dataset` | `POST /v1/datasets/update` | Change a dataset's name, icon, colour or folder |
| `argus_update_dataset_schema` | `POST /v1/datasets/schema` | Replace a dataset's declared column list |
| `argus_update_dataset_options` | `POST /v1/datasets/options` | Edit one choice column's options |
| `argus_sample_rows` | `POST /v1/datasets/sample` | A page of a dataset's rows |
| `argus_query_rows` | `POST /v1/datasets/query` | Query a dataset's rows by where conditions, optionally aggregated |
| `argus_add_rows` | `POST /v1/datasets/rows` | Append rows to a dataset |
| `argus_update_rows` | `POST /v1/datasets/rows/update` | Rewrite dataset rows in place by id |
| `argus_delete_rows` | `POST /v1/datasets/rows/delete` | Delete dataset rows by id or by a where filter |
| `argus_trash_datasets` | `POST /v1/datasets/trash` | Move datasets to Trash |
| `argus_list_trashed_datasets` | `GET /v1/datasets/trashed` | The datasets currently in Trash |
| `argus_restore_datasets` | `POST /v1/datasets/restore` | Bring datasets back out of Trash |
| `argus_purge_datasets` | `POST /v1/datasets/purge` | Permanently delete datasets from Trash |
| `argus_dedupe_rows` | `POST /v1/datasets/rows/dedupe` | Remove duplicate rows |

## Projects

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_projects` | `GET /v1/projects` | The workspace's projects, what each holds, and the index of its brain |
| `argus_project_context` | `GET /v1/projects/{id}/context` | One project's goal, brief and rules, plus the titles of its other documents |
| `argus_delegate` | `POST /v1/projects/{id}/delegate` | Hand a goal to Argus's own orchestrator, inside a project's context |
| `argus_create_project` | `POST /v1/projects/create` | Start a project |
| `argus_add_to_project` | `POST /v1/projects/add` | Put records into a project |
| `argus_remove_from_project` | `POST /v1/projects/remove` | Take records out of a project |
| `argus_read_project_doc` | `POST /v1/projects/doc/read` | Read one project document |
| `argus_write_project_doc` | `POST /v1/projects/doc/write` | Write a project document |
| `argus_delete_project_doc` | `POST /v1/projects/doc/delete` | Delete a project document |
| `argus_make_plan` | `POST /v1/orchestration/plan` | Lay out a multi-step plan |
| `argus_delegate_many` | `POST /v1/orchestration/delegate-many` | Hand several goals to subagents at once |

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
| `argus_get_proxy_share_limit` | `GET /v1/proxy-share-limit` | This machine's proxy-sharing threshold |
| `argus_set_proxy_share_limit` | `POST /v1/proxy-share-limit` | Change this machine's proxy-sharing threshold |

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
| `argus_switch_tab` | `POST /v1/page/switch-tab` | Bring one of a profile's tabs to the front |
| `argus_new_tab` | `POST /v1/page/new-tab` | Open a new tab in a launched profile |
| `argus_page_recording` | `POST /v1/page/recording` | What has been done to this page so far |
| `argus_save_page_recording` | `POST /v1/page/save-recording` | Save what was done to a page as an automation |
| `argus_page_snapshot` | _CDP, no endpoint_ | The interactive elements on a page, each with a selector to act on it |
| `argus_click` | _CDP, no endpoint_ | Click an element, or a point, in a running profile's page |
| `argus_type` | _CDP, no endpoint_ | Type into a field in a running profile's page |
| `argus_press_key` | _CDP, no endpoint_ | Press a single key in a running profile's page |
| `argus_scroll` | _CDP, no endpoint_ | Scroll a running profile's page |
| `argus_wait_for` | _CDP, no endpoint_ | Wait until a page reaches a condition |
| `argus_extract` | _CDP, no endpoint_ | Read values out of a running profile's page |
| `argus_select_option` | _CDP, no endpoint_ | Choose an option in a dropdown |
| `argus_list_tabs` | _CDP, no endpoint_ | List the open pages in a running profile |
| `argus_navigate` | _CDP, no endpoint_ | Point a page at a URL and wait for it to settle |
| `argus_read_page` | _CDP, no endpoint_ | Read a page's visible text, whole or by CSS selector |
| `argus_screenshot` | _CDP, no endpoint_ | Screenshot a page — JPEG by default, PNG on request |
| `argus_eval` | _CDP, no endpoint_ | Evaluate a JavaScript expression in a page and return its value |

## Mail

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_mailboxes` | `GET /v1/mail/mailboxes` | The mailboxes this workspace can read |
| `argus_list_messages` | `POST /v1/mail/messages` | List a mailbox folder |
| `argus_read_message` | `POST /v1/mail/message` | Read one message |
| `argus_read_new_message` | `POST /v1/mail/wait` | Wait for a message that has not arrived yet |
| `argus_open_compose` | `POST /v1/mail/compose/open` | Start a draft |
| `argus_list_compose` | `GET /v1/mail/compose` | Drafts that are open right now |
| `argus_set_compose_fields` | `POST /v1/mail/compose/set` | Fill in a draft |
| `argus_send_compose` | `POST /v1/mail/compose/send` | Send a draft |
| `argus_close_compose` | `POST /v1/mail/compose/close` | Discard a draft |

## Knowledge base

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_list_docs` | `POST /v1/kb/docs` | The knowledge-base index |
| `argus_read_doc` | `POST /v1/kb/doc` | Read one knowledge-base article |
| `argus_search_docs` | `POST /v1/kb/search` | Search the knowledge base |
| `argus_get_context` | `GET /v1/context` | What this workspace actually holds |
| `argus_recent_errors` | `GET /v1/errors/recent` | What has gone wrong lately |

## Connection

| Tool | Endpoint | What it does |
| --- | --- | --- |
| `argus_connection_status` | _CDP, no endpoint_ | Confirm this MCP connection works, and say as whom |

## Resources

The server advertises `resources` alongside `tools`, and serves one resource family: the documents that make up a project's brain — its brief, its rules, its notes and its runbooks.

| | |
| --- | --- |
| URI | `argus://project/<id>/<path>` |
| Listed by | `resources/list` |
| Read by | `resources/read` |

`argus_list_projects` and `argus_project_context` give you a project's document *index* — the titles and paths, which is what belongs in a prompt. A body is fetched one at a time, as a resource, once you have decided you want that body. That is why `GET /v1/projects/{id}/docs/{path}` has no tool of its own: it is the same document, and giving it one would invite an agent to pull a hundred notes into context to find the one it needed.

## Endpoints with no tool

4 routes are reachable over HTTP only, on purpose. An agent cannot call them however plainly they are documented — with the one exception noted above: the project-document route is reached as a resource rather than a tool.

- `POST /v1/cookies/bulk-match` — Match exported cookie files in a folder to profiles by name
- `POST /v1/cookies/push-local` — Attach a cookie file on disk to one profile
- `POST /v1/monitoring/report` — Report a run's outcome from an external script
- `GET /v1/projects/{id}/docs/{path}` — One document from a project's brain, by path
