# Driving Argus from this project

Argus is an anti-detect browser. A *profile* is an isolated browser identity
with its own fingerprint, proxy and cookie jar. This project can drive those
profiles either as MCP tools or over plain HTTP.

## Before anything works

- **The Argus launcher must be open.** The API starts with it and stops with it.
  A refused connection almost always means the launcher is closed, not that the
  address is wrong.
- **Base URL is `http://127.0.0.1:39219`.** It listens on loopback, on this machine.
  Nothing off it can reach the API, and there is no hosted web API to point at
  instead.
- **Auth is `Authorization: Bearer <YOUR_API_KEY>`.** Keys are minted in the
  launcher's API tab and shown once — only a hash is stored, so a lost key is
  replaced rather than recovered.
- **The first call from a new client blocks.** The launcher raises an
  approve-or-deny card and holds the request until a human answers. Do not treat
  the pause as a hang, and do not retry — a retry queues a second card.

## Rules that save a wasted turn

- **A 403 is not a missing profile and is not worth retrying.** It means the key
  is scoped to folders that do not contain that profile, or it is trying to
  author an automation while folder-scoped. Automations are shared across every
  folder and have none of their own, so authoring one needs an unscoped key.
- **Creating a profile is available; purging one is not.** `argus_create_profile`
  mints one — only `name` is required, and everything else takes the same
  defaults the app's own New Profile dialog uses. The asymmetry is at the other
  end: `argus_delete_profile` only moves a profile to Trash, where the app can
  restore it and the on-disk browser data stays, and it never sends
  `permanent`. An irreversible purge stays a human action in the app.
- **Datasets are on the API in full.** Creating one, replacing its declared
  columns, appending, updating and deleting rows, Trash and restore — all of it
  has a tool. Read with `argus_query_rows` (a list of `{column, op, value}`
  conditions, optionally aggregated) or `argus_sample_rows` for a page in
  insertion order; both are capped and paged, so plan on filtering rather than
  on pulling a dataset whole. Rows are indexed by column *key*, which never
  changes when a column is renamed — write against keys, not names.
- **A launched session carries the profile's own identity.** Whatever the
  profile already holds comes with it, including its cookies, so a profile with
  a cookie set assigned opens *already signed in*. Do not send it anything the
  profile is not meant to have.
- **Sharing is enforced and it refuses.** `argus_assign_proxy` is *refused*
  when the proxy would then be shared by more profiles than the user's limit
  allows, or when it would put two profiles for the same platform behind one
  exit; the refusal names the profiles already on it and the limit it was judged
  against. Repeating the call with `acknowledgeSharing: true` proceeds anyway —
  shared egress is a threshold rather than a prohibition — but that flag is for
  after the user has been told and has agreed, not for getting past a refusal on
  your own. A cookie set is stricter and is not a setting at all: exactly one
  profile may hold it, because two profiles on one set is one account signed in
  twice. `argus_launch_profile` blocks on neither, but its reply carries a
  `warnings` array when the profile is over a limit — say so rather than
  opening a batch silently, and call `argus_sharing_report` first, which
  returns the whole picture in one call instead of one refusal at a time.
- **Proxy listings never return credentials.** Do not plan around reading one
  back.
- **The five page-driving tools need a profile that is already open.** They
  attach over CDP to a running profile; launch it first. Every one of them takes
  `profileId`.

## Projects, and the documents they carry

A workspace may hold **projects**: named pieces of work spanning profiles,
proxies, automations and datasets — "grow 20 accounts for Client X" — each with
a goal, a brief and a set of rules. `argus_list_projects` lists them with an
index of each one's documents; `argus_project_context` returns one project's
goal, its brief, and the *full text* of every rule it keeps. Read that before
acting on anything a project holds: the rules are binding and are written down
nowhere else.

The documents themselves are MCP **resources** rather than tools, so nothing is
sent until it is asked for. `resources/list` enumerates them and each is read
at `argus://project/<id>/<path>` — `argus://project/client-x/PROJECT.md`,
`argus://project/client-x/rules/posting-hours.md`. Fetch the few you need.

`argus_delegate` hands a goal to Argus's own orchestrator instead of driving
the project's profiles yourself. It already holds that project's brief and
notes, and it enforces the project's autonomy level, its refusal list and its
acting hours in code rather than by being asked nicely.

## The tools

- **Profiles**: `argus_list_profiles`, `argus_get_profile`, `argus_create_profile`, `argus_update_profile`, `argus_profile_notes`, `argus_add_profile_note`, `argus_assign_proxy`, `argus_sharing_report`, `argus_launch_profile`, `argus_profile_session`, `argus_close_profile`, `argus_delete_profile`, `argus_update_fingerprint`, `argus_fingerprint_check`, `argus_trash_profiles`, `argus_list_trashed_profiles`, `argus_restore_profiles`, `argus_purge_profiles`
- **Proxies**: `argus_list_proxies`, `argus_create_proxy`, `argus_update_proxy`, `argus_check_proxy`, `argus_delete_proxy`, `argus_reimport_proxies`
- **Cookies**: `argus_assign_cookie_set`, `argus_unassign_cookie_set`, `argus_list_cookie_sets`, `argus_update_cookie_set`, `argus_trash_cookie_sets`, `argus_list_trashed_cookie_sets`, `argus_restore_cookie_sets`, `argus_purge_cookie_sets`
- **Automations**: `argus_list_automations`, `argus_list_recipes`, `argus_set_up_recipe`, `argus_automation_schema`, `argus_get_automation`, `argus_create_automation`, `argus_update_automation`, `argus_delete_automation`, `argus_run_automation`, `argus_automation_runs`, `argus_search_automations`, `argus_automation_summary`
- **Scrapers**: `argus_list_scrapers`, `argus_get_scraper`, `argus_sample_scraper`, `argus_run_scraper`, `argus_scraper_runs`
- **Schedule**: `argus_list_schedule_entries`, `argus_create_schedule_entry`, `argus_update_schedule_entry`, `argus_delete_schedule_entry`, `argus_schedule_history`, `argus_keep_awake`, `argus_set_keep_awake`, `argus_schedule_notifications`, `argus_set_schedule_notifications`, `argus_schedule_day`
- **Triggers**: `argus_list_event_triggers`, `argus_create_event_trigger`, `argus_update_event_trigger`, `argus_delete_event_trigger`, `argus_trigger_history`
- **Datasets**: `argus_list_datasets`, `argus_create_dataset`, `argus_update_dataset`, `argus_update_dataset_schema`, `argus_update_dataset_options`, `argus_sample_rows`, `argus_query_rows`, `argus_add_rows`, `argus_update_rows`, `argus_delete_rows`, `argus_trash_datasets`, `argus_list_trashed_datasets`, `argus_restore_datasets`, `argus_purge_datasets`, `argus_dedupe_rows`
- **Projects**: `argus_list_projects`, `argus_project_context`, `argus_delegate`, `argus_create_project`, `argus_add_to_project`, `argus_remove_from_project`, `argus_read_project_doc`, `argus_write_project_doc`, `argus_delete_project_doc`, `argus_make_plan`, `argus_delegate_many`
- **Connectors**: `argus_list_connectors`, `argus_create_connector`, `argus_update_connector`, `argus_delete_connector`, `argus_test_connector`
- **Workspace**: `argus_list_folders`, `argus_list_statuses`, `argus_get_proxy_share_limit`, `argus_set_proxy_share_limit`
- **Notifications**: `argus_telegram_status`, `argus_set_telegram_pref`, `argus_set_telegram_bot`
- **Tables**: `argus_table_columns`, `argus_set_table_columns`
- **Skills**: `argus_skills_schema`, `argus_list_skills`, `argus_get_skill`, `argus_save_skill`, `argus_delete_skill`
- **Driving a page**: `argus_switch_tab`, `argus_new_tab`, `argus_page_recording`, `argus_save_page_recording`, `argus_page_snapshot`, `argus_click`, `argus_type`, `argus_press_key`, `argus_scroll`, `argus_wait_for`, `argus_extract`, `argus_select_option`, `argus_list_tabs`, `argus_navigate`, `argus_read_page`, `argus_screenshot`, `argus_eval`
- **Mail**: `argus_list_mailboxes`, `argus_list_messages`, `argus_read_message`, `argus_read_new_message`, `argus_open_compose`, `argus_list_compose`, `argus_set_compose_fields`, `argus_send_compose`, `argus_close_compose`
- **Knowledge base**: `argus_list_docs`, `argus_read_doc`, `argus_search_docs`, `argus_get_context`, `argus_recent_errors`
- **Connection**: `argus_connection_status`

Full request and response detail: https://www.browserargus.com/api-reference.md
Machine-readable spec: https://www.browserargus.com/openapi.json
