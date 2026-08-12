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

- **You cannot create a profile from here.** Creating an identity is done in the
  app, because a script that mints fifty makes fifty that look alike.
- **A 403 is not a missing profile and is not worth retrying.** It means the key
  is scoped to folders that do not contain that profile, or it is trying to
  author an automation while folder-scoped. Automations are shared across every
  folder and have none of their own, so authoring one needs an unscoped key.
- **Datasets are not on the API.** They are managed in the app and by the
  in-app assistant. No endpoint reaches them.
- **Never hand a launched profile real credentials or tokens.** It is a separate,
  anonymous browser process with no account of its own, and that is the point.
- **Proxy listings never return credentials.** Do not plan around reading one
  back.
- **The five page-driving tools need a profile that is already open.** They
  attach over CDP to a running profile; launch it first.

## The tools

- **Profiles**: `argus_list_profiles`, `argus_get_profile`, `argus_create_profile`, `argus_update_profile`, `argus_profile_notes`, `argus_add_profile_note`, `argus_assign_proxy`, `argus_launch_profile`, `argus_profile_session`, `argus_close_profile`, `argus_delete_profile`, `argus_update_fingerprint`
- **Proxies**: `argus_list_proxies`, `argus_check_proxy`
- **Automations**: `argus_list_automations`, `argus_automation_schema`, `argus_get_automation`, `argus_create_automation`, `argus_update_automation`, `argus_delete_automation`, `argus_run_automation`, `argus_automation_runs`
- **Schedule**: `argus_list_schedule_entries`, `argus_create_schedule_entry`, `argus_update_schedule_entry`, `argus_delete_schedule_entry`, `argus_schedule_history`, `argus_keep_awake`, `argus_set_keep_awake`, `argus_schedule_notifications`, `argus_set_schedule_notifications`
- **Connectors**: `argus_list_connectors`, `argus_create_connector`, `argus_update_connector`, `argus_delete_connector`, `argus_test_connector`
- **Workspace**: `argus_list_folders`, `argus_list_statuses`
- **Notifications**: `argus_telegram_status`, `argus_set_telegram_pref`, `argus_set_telegram_bot`
- **Tables**: `argus_table_columns`, `argus_set_table_columns`
- **Skills**: `argus_skills_schema`, `argus_list_skills`, `argus_get_skill`, `argus_save_skill`, `argus_delete_skill`
- **Driving a page**: `argus_list_tabs`, `argus_navigate`, `argus_read_page`, `argus_screenshot`, `argus_eval`

Full request and response detail: https://www.browserargus.com/api-reference.md
Machine-readable spec: https://www.browserargus.com/openapi.json
