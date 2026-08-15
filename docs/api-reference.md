# The Argus local API

This is a local API. It listens on loopback on the machine running Argus, and nothing off that machine can reach it. There is no hosted web API.

- Base URL: `http://127.0.0.1:39219`
- Auth: `Authorization: Bearer <YOUR_API_KEY>`
- Bodies: `Content-Type: application/json`
- Surface: 84 endpoints, 85 agent tools

## Keys and approval

- Keys are created in the launcher, under the API tab. The raw value is shown once, at creation — only a hash is stored, so a lost key is replaced rather than recovered.
- A key can be scoped to specific folders, or left unscoped. A scoped key sees only the profiles in its folders, and may only close a session it opened itself.
- The first time a new client calls the API, the launcher raises an approve-or-deny prompt and holds the request until you answer it.

A launched session carries whatever identity the profile already holds, including its cookies — so it may well open already signed in. Do not send it anything the profile is not meant to have. The launch reply carries a `warnings` array: a non-empty one means that profile shares its cookie set or its proxy past the limit, which for a cookie set means the account is signed in from another profile too. Launching is not blocked for it, but say so rather than opening a batch silently, and call `argus_sharing_report` before a batch to see the whole picture in one call.

## Profiles

A profile is an isolated browser identity with its own proxy, fingerprint and cookie jar. You can create one, read it, change it, put it on a proxy, and open it for automation. Deleting is the one half-move: a profile goes to Trash and can be brought back, and the irreversible purge stays an in-app action.

### GET /v1/profiles

List profiles (optional ?folder=<id>)

- MCP tool: `argus_list_profiles`

List the Argus browser profiles this key can see. Each profile is an isolated browser identity with its own proxy, fingerprint and cookies.

Fields:
- `folder` (string) — Only profiles in this folder id (argus_list_folders). Sent as the ?folder= query parameter, not a body. Omit for every profile the key can see.

```sh
curl -X GET "http://127.0.0.1:39219/v1/profiles" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/profiles/get

Read one profile

- MCP tool: `argus_get_profile`

Read one profile: its proxy, status, tags, folder and fingerprint. The reply also carries `sharing` — empty when nothing is wrong, and otherwise the cookie-set or proxy this profile shares past its limit, with the other profiles involved and the fix.

Fields:
- `profileId` (string, required) — From argus_list_profiles.

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

Change a profile's name, status, tags, colour, avatar, folder, proxy mode, start URL, login URL or launch automation. Assigning a specific proxy is a separate call (argus_assign_proxy); setting proxyMode to direct or free_proxy here clears whatever proxy the profile was on.

Fields:
- `profileId` (string, required) — From argus_list_profiles.
- `name` (string) — The profile's display name.
- `status` (string) — Workflow status label. argus_list_statuses has the ones this workspace uses.
- `tags` (tags) — At most 5; extras are dropped. Replaces the whole list.
- `color` (string) — A preset name like blue or violet, or a #rrggbb hex.
- `avatar` (string) — The mark shown beside the name: "brand:<slug>" for one of the built-in site logos (brand:instagram, brand:facebook, brand:x, brand:tiktok, ...), or "" to go back to the initials. Uploaded pictures are set in the app.
- `folderId` (string) — Folder to file it in (argus_list_folders).
- `proxyMode` (string) — assigned, direct or free_proxy. assigned requires the profile to already have a proxy (use argus_assign_proxy to set one); direct and free_proxy clear it.
- `startUrl` (string) — URL the profile opens on launch. "" to clear.
- `loginUrl` (string) — The sign-in page this profile's stored email/password belong to. A note, not an action -- nothing fills the form in; an automation reads it as {{profile.login_url}}. "" to clear. Distinct from startUrl, which is where a launch lands.
- `automationId` (string) — Automation to run on every launch (argus_list_automations). "" to detach.
- `automationVars` (object) — This profile's answers to automations' declared parameters, keyed by automation id: {"<automationId>": {"city_name": "Dortmund"}}. This is how one automation runs a different city per profile. Replaces the whole map, so read it back with argus_get_profile and merge rather than sending one automation on its own. Values for a parameter the automation no longer declares are ignored at run time. See argus_automation_schema for the parameter vocabulary.

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

Put a profile on a proxy from the library. A profile whose proxy mode is "assigned" needs a working proxy before it will launch. REFUSED when the proxy would then be shared by more profiles than the user's limit allows, or when it would put two profiles for the same platform behind one exit; the refusal names the profiles already on it and the limit it was judged against. Sharing an exit links the accounts behind it. Repeat the call with acknowledgeSharing true if the user wants it anyway -- unlike a shared cookie-set, shared egress is a normal way to run throwaway profiles, so this is a threshold and not a prohibition. argus_sharing_report shows the whole picture before you start.

Fields:
- `profileId` (string, required) — From argus_list_profiles.
- `proxyId` (string, required) — From argus_list_proxies.
- `acknowledgeSharing` (boolean) — Proceed even though the proxy would then be shared past the limit. Only after the user has been told and has agreed -- the refusal this overrides names the profiles already on that exit and the limit it was judged against.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/assign-proxy" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "proxyId": "<id>" }'
```

### GET /v1/sharing/report

Every cookie-set and proxy shared past its limit, with the limits themselves

- MCP tool: `argus_sharing_report`

List every cookie-set and proxy in this workspace that is shared past its limit, and the limits themselves. CALL THIS BEFORE assigning cookie-sets or proxies, before launching a batch of profiles, and before building a workflow that does either -- one call here replaces discovering the same state one refusal at a time. A cookie-set may be held by exactly one profile, and that is a fixed rule rather than a setting: two profiles on one set is one account signed in twice. A proxy may be shared up to the limit reported, which the user sets per machine. Each warning carries the profiles involved and the remedy to apply. An empty warnings array means nothing is over its limit.

```sh
curl -X GET "http://127.0.0.1:39219/v1/sharing/report" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/profiles/launch-automation

Open a profile for automation; returns its CDP url. Reuses a running session unless relaunch is set

- MCP tool: `argus_launch_profile`

Open a profile in the Argus browser, ready for automation. If it is already open this returns the existing session rather than restarting it; pass relaunch=true to force a fresh window (which closes the current one). A profile set to use an assigned proxy needs that proxy to be working before it will launch. The session carries whatever identity the profile already holds, including its cookies — do not send it anything the profile is not meant to have. The reply carries a `warnings` array: a non-empty one means this profile shares its cookie-set or its proxy past the limit, which for a cookie-set means this account is signed in from another profile too. Launching is NOT blocked for it — the conflict already existed — but tell the user rather than opening a batch of them silently, and use argus_sharing_report before a batch to see it in one call.

Fields:
- `profileId` (string, required) — From argus_list_profiles.
- `relaunch` (boolean) — Close any existing session and start a new one. Omitted or false reuses a session that is already open.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/launch-automation" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "relaunch": false }'
```

### POST /v1/profiles/cdp

Where a running profile's CDP endpoint is, without launching it

- MCP tool: `argus_profile_session`

Check whether a profile is currently open for automation, and where its debugging endpoint is. Call this before launching to avoid disturbing a session that is already running.

Fields:
- `profileId` (string, required) — From argus_list_profiles. Answers whether it is open and where its debugging endpoint is; never launches it.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/cdp" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>" }'
```

### POST /v1/profiles/close-automation

Close a session this key opened

- MCP tool: `argus_close_profile`

Close a profile session this key opened.

Fields:
- `profileId` (string, required) — From argus_list_profiles. Only a session this key opened can be closed here.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/close-automation" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>" }'
```

### POST /v1/profiles/delete

Move a profile to Trash (permanent: true to purge; the tool only soft-deletes)

- MCP tool: `argus_delete_profile`

Move a profile to Trash, where the app can restore it. This does not remove the profile's on-disk browser data. Permanent deletion is only available in the app, not over this tool.

Fields:
- `profileId` (string, required) — From argus_list_profiles. Moves it to Trash, where the app can restore it; the on-disk browser data stays. argus_delete_profile never sends `permanent`, so an irreversible purge stays an in-app action.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/delete" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>" }'
```

### POST /v1/profiles/update-fingerprint

Re-roll or override a profile's fingerprint

- MCP tool: `argus_update_fingerprint`

Change parts of a profile's fingerprint. The fields you send are merged into the stored fingerprint; anything you omit keeps its value. Read the current one with argus_get_profile first. Changing the device identity of a profile that is holding a logged-in session looks exactly like a stolen cookie and can get the session challenged -- prefer a new profile for a new identity.

Fields:
- `profileId` (string, required) — From argus_list_profiles.
- `fingerprint` (object, required) — The fingerprint fields to change, merged into the stored one -- anything omitted keeps its value. Any subset of these 21 keys, and nothing else (an undeclared key is dropped): os (Windows 11 | Windows 10 | macOS | Ubuntu | Android | iOS), browser_version, user_agent, language, timezone, geolocation, webrtc (Proxy only | Disabled | Real | Custom), canvas (Real | Noise | Block), webgl (Real | Noise | Block), webgpu (Real | Block), client_rects (Real | Noise | Block), audio (Real | Noise | Block), webgl_vendor, webgl_renderer, screen (Auto, or a size like 1920x1080), cpu_model, cpu_cores (number), memory_gb (number), media_devices, do_not_track (boolean), rotate_on_launch (boolean -- re-rolls the noise seeds on every launch, unsafe for a profile holding a session). Everything else is a string. Read the current fingerprint with argus_get_profile first.

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

List the proxies in this account's library.

```sh
curl -X GET "http://127.0.0.1:39219/v1/proxies" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/proxies/create

Add proxy

- MCP tool: `argus_create_proxy`

Add a proxy to the library: host, port, type http or socks5, optional username and password. The proxy lands UNCHECKED -- call argus_check_proxy, or wait for the background sweep, before assigning it to profiles, because at launch a proxy that does not respond blocks the launch rather than falling back to the real address. Returns the saved proxy and its id, which argus_update_proxy and argus_assign_proxy take.

Fields:
- `host` (string, required) — The proxy's address.
- `port` (number, required) — The port to dial.
- `name` (string) — What the Proxies tab calls it. Defaults to host:port.
- `type` (string) — http or socks5. Defaults to http.
- `username` (string) — Stored, never returned by any route.
- `password` (string) — Stored, never returned by any route.

```sh
curl -X POST "http://127.0.0.1:39219/v1/proxies/create" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "US proxy", "type": "socks5", "host": "1.2.3.4", "port": 1080 }'
```

### POST /v1/proxies/update

Update a proxy

- MCP tool: `argus_update_proxy`

Change an existing proxy's name, type, host, port, username or password. Only the fields you send are written. Changing the connection details clears the stored check result -- the status you saw before is stale until the next check -- and the password is stored, never returned. To re-test after a change, call argus_check_proxy.

Fields:
- `proxyId` (string, required) — From argus_list_proxies.
- `name` (string)
- `type` (string) — http or socks5.
- `host` (string)
- `port` (number)
- `username` (string)
- `password` (string)

```sh
curl -X POST "http://127.0.0.1:39219/v1/proxies/update" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "proxyId": "<id>", "name": "US proxy" }'
```

### POST /v1/proxies/check

Check reachability and egress IP

- MCP tool: `argus_check_proxy`

Check a proxy's reachability and egress IP.

Fields:
- `host` (string, required) — The proxy's address.
- `port` (number, required) — The port to dial.
- `username` (string) — Only if the proxy authenticates.
- `password` (string) — Only if the proxy authenticates.
- `type` (string) — http or socks5. Defaults to http.

```sh
curl -X POST "http://127.0.0.1:39219/v1/proxies/check" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "host": "1.2.3.4", "port": 1080, "type": "socks5" }'
```

### POST /v1/proxies/delete

Remove a proxy

- MCP tool: `argus_delete_proxy`

Permanently delete a proxy. There is no Trash for proxies and no undo: the row is gone and every profile that used it is left without a proxy. The user must approve this in the app, and the approval card names how many profiles are affected.

Fields:
- `proxyId` (string, required) — From argus_list_proxies.

```sh
curl -X POST "http://127.0.0.1:39219/v1/proxies/delete" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "proxyId": "<id>" }'
```

### POST /v1/proxies/reimport

Re-import proxies from a list of rows

- MCP tool: `argus_reimport_proxies`

Re-import proxies from rows like {ip, port_socks5, port_http, username?, password?, country?}. A row matching an existing proxy by type+host+port UPDATES it and keeps its id, so profiles already assigned to it stay assigned; anything else is created. Every touched proxy has its check columns cleared -- the background sweep re-checks them.

Fields:
- `proxies` (objects, required) — The rows to merge in. ip or host names the address; port_socks5 or port_http (or port) gives the port and picks the type.

```sh
curl -X POST "http://127.0.0.1:39219/v1/proxies/reimport" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "proxies": [{ "ip": "1.2.3.4", "port_socks5": 1080, "country": "US" }] }'
```

## Cookies

Cookie sets are jars kept in the workspace and attached to profiles, so an account signed in once can be carried into a profile without signing in again. A set is held by exactly one profile at a time — two profiles on one set is one account signed in twice. The two bulk-import routes, which read files off your own disk, are HTTP-only: no agent tool fronts them.

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

### POST /v1/profiles/assign-cookies

Set which profiles launch with a cookie-set

- MCP tool: `argus_assign_cookie_set`

Set which profiles launch with a saved cookie-set. profileIds is the COMPLETE list of profiles holding the set afterwards -- profiles that hold it today and are missing from your list are detached, so send the whole list rather than just the additions. A profile carries at most one cookie-set, so assigning replaces whatever it had. REFUSED by default when the result would put the set on more than one profile: that is one account signed in twice, on two fingerprints and usually two exits, which is the pattern platforms ban for. The refusal names the profiles involved; repeat the call with acknowledgeSharing true only if the user has been told and wants it anyway. Cookies are seeded at each profile's NEXT launch, not immediately.

Fields:
- `cookieSetId` (string, required) — The saved set, from argus_list_cookie_sets or argus_sharing_report.
- `profileIds` (strings, required) — The complete list of profiles holding this set afterwards. An empty array detaches it from every profile and parks the set.
- `acknowledgeSharing` (boolean) — Proceed even though the result puts one session on more than one profile. Only send this when the user has been told and has agreed.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/assign-cookies" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "cookieSetId": "<id>", "profileIds": ["<id>"] }'
```

### POST /v1/profiles/unassign-cookies

Detach profiles from whatever cookie-set they hold

- MCP tool: `argus_unassign_cookie_set`

Detach profiles from whatever cookie-set they currently hold. This is the fix for a shared session: leave the set on the one profile that should keep it and unassign the rest. The set itself is not deleted and stays in the library. A detached profile keeps whatever its own browser directory already holds on disk, so nothing is logged out right now -- but its next launch seeds no cookies, which for a profile that relied on the set means it opens signed out.

Fields:
- `profileIds` (strings, required) — Profiles to detach. A profile that holds no set is left alone rather than failing the call.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/unassign-cookies" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileIds": ["<id>"] }'
```

### GET /v1/cookies

The workspace's cookie sets, metadata only

- MCP tool: `argus_list_cookie_sets`

The workspace's cookie sets: name, domain, cookie count, status, folder, tags and how many profiles use each. Cookie VALUES are never returned over this API -- they are session credentials and stay in the launcher; only names and counts leave.

```sh
curl -X GET "http://127.0.0.1:39219/v1/cookies" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/cookies/update

Change a cookie set's name, status, colour, folder or tags

- MCP tool: `argus_update_cookie_set`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Change a cookie set's name, status label, colour, folder or tags. Only the fields you send are written. Cookie VALUES never change through the API -- they are session credentials and stay where they are.

Fields:
- `cookieSetId` (string, required) — From argus_list_cookie_sets.
- `name` (string)
- `status` (string) — A label from argus_list_statuses; "" clears it.
- `color` (string) — A preset name or #rrggbb; "" clears it.
- `folderId` (string) — The cookie folder to file it in (argus_list_folders). Empty moves it to All cookie-sets.
- `tags` (tags) — Replaces the whole list. At most 5.

```sh
curl -X POST "http://127.0.0.1:39219/v1/cookies/update" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "cookieSetId": "<id>", "status": "Stale" }'
```

### POST /v1/cookies/trash

Move cookie sets to Trash

- MCP tool: `argus_trash_cookie_sets`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Move cookie sets to Trash (30-day recovery). Profiles using a trashed set are detached -- their next launch seeds no cookies. The user must approve this in the app, and the card names the sets.

Fields:
- `cookieSetIds` (strings, required) — From argus_list_cookie_sets.

```sh
curl -X POST "http://127.0.0.1:39219/v1/cookies/trash" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "cookieSetIds": ["<id>"] }'
```

### GET /v1/cookies/trashed

The cookie sets currently in Trash

- MCP tool: `argus_list_trashed_cookie_sets`

The cookie sets currently in Trash (deleted within the last 30 days, pending purge), with when each was deleted. The fix for a set that seems to have vanished.

```sh
curl -X GET "http://127.0.0.1:39219/v1/cookies/trashed" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/cookies/restore

Bring cookie sets back out of Trash

- MCP tool: `argus_restore_cookie_sets`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Bring cookie sets back out of Trash. Profiles are NOT re-attached automatically -- a restored set sits in the library until it is assigned again with argus_assign_cookie_set.

Fields:
- `cookieSetIds` (strings, required) — From argus_list_trashed_cookie_sets.

```sh
curl -X POST "http://127.0.0.1:39219/v1/cookies/restore" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "cookieSetIds": ["<id>"] }'
```

### POST /v1/cookies/purge

Permanently delete cookie sets from Trash

- MCP tool: `argus_purge_cookie_sets`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Permanently delete cookie sets from Trash, payloads included. There is no undo. The user must approve this in the app, and the card names the sets.

Fields:
- `cookieSetIds` (strings, required) — From argus_list_trashed_cookie_sets.

```sh
curl -X POST "http://127.0.0.1:39219/v1/cookies/purge" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "cookieSetIds": ["<id>"] }'
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

Run an automation against a profile, launching the profile if it is not already open. Returns a run id immediately; the run continues in the background. The reply carries a `warnings` array: non-empty means this profile shares its cookie-set or proxy past the limit, which the run does not block but the user should hear about. One call is one profile: to run the same automation with different values on several profiles, call it once per profile with that profile's vars, which also gives you a run id per profile to poll.

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

## Datasets

### GET /v1/datasets

The workspace's datasets, with their declared columns

- MCP tool: `argus_list_datasets`

The workspace's datasets: id, name, row count, last update, and each declared column's key, name and type. The keys are what argus_query_rows and argus_add_rows index rows by -- they never change, even when a column is renamed, so write against them rather than against the names. A select or tags column also returns its options as {key, label, color}; the option KEY is what a cell stores, so write that -- a label is also accepted and resolved to its key on write.

```sh
curl -X GET "http://127.0.0.1:39219/v1/datasets" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/datasets/create

Create a dataset, optionally with starting columns

- MCP tool: `argus_create_dataset`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Create a dataset with 0 rows, optionally declaring starting columns: [{name, type, options?}]. Types: text, longText, number, checkbox, date, datetime, url, email, phone, select, tags, createdAt, rowNumber, parameter, profile, automation, proxy. Keys are derived from the names and never change. Fill it with argus_add_rows; declare more columns later with argus_update_dataset_schema. profile, automation and proxy columns hold the item's id from argus_list_profiles / argus_list_automations / argus_list_proxies; a name in a row value is resolved to its id on write. A select or tags column declares its choices with options -- without them the column has no choices and no row can be set to one.

Fields:
- `name` (string, required)
- `folderId` (string) — The dataset folder to file it in (argus_list_folders, datasets group). Omit for All datasets.
- `columns` (objects) — Starting columns, each {name, type, options?}. At most 60; type defaults to text. options is for select/tags only: ["New", "Won"] or [{label, color?}]. A key is derived from each label and is what rows store. Colours: slate, stone, red, orange, amber, yellow, lime, green, teal, cyan, sky, blue, indigo, violet, fuchsia, pink. At most 50 options.
- `icon` (string) — brand:<slug> for a built-in site logo. Omit for the plain glyph.
- `color` (string) — A preset name like blue or violet, or a #rrggbb hex.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/create" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Leads", "columns": [{ "name": "Status", "type": "select", "options": ["New", "Won"] }] }'
```

### POST /v1/datasets/update

Change a dataset's name, icon, colour or folder

- MCP tool: `argus_update_dataset`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Change a dataset's name, icon (brand:<slug>), colour, folder or library tags. Rows and columns are untouched.

Fields:
- `datasetId` (string, required) — From argus_list_datasets.
- `name` (string)
- `icon` (string) — brand:<slug> or "" to clear.
- `color` (string) — A preset name or #rrggbb; "" clears it.
- `folderId` (string) — Empty moves it to All datasets.
- `tags` (tags) — The dataset's library tags. Replaces the whole list.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/update" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "datasetId": "<id>", "name": "Leads 2026" }'
```

### POST /v1/datasets/schema

Replace a dataset's declared column list

- MCP tool: `argus_update_dataset_schema`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Replace a dataset's DECLARED column list: add, rename, retype or remove columns. Send every column you want to keep: existing ones by their key from argus_list_datasets (same key + new name renames it, same key + new type retypes it), new ones by name alone (a key is derived and never changes). Types include text, number, date, select, tags, parameter, and the workspace-item references profile, automation and proxy -- those three hold the item's id from argus_list_profiles / argus_list_automations / argus_list_proxies, and a name in a row value is resolved to its id on write. Columns you leave out are removed from the schema; their stored values are kept, hidden. Row values are stored against keys, so a rename loses nothing. A select or tags column's choices are its options: OMITTING options on a column KEEPS its existing list, so a rename never wipes it -- only omitting a whole COLUMN removes anything, and options: [] is how you clear them. To edit choices without resending every column, use argus_update_dataset_options. The user must approve this in the app, and the card shows the exact diff.

Fields:
- `datasetId` (string, required)
- `columns` (objects, required) — The complete new column list, each {key?, name, type, options?}. Keys are for existing columns; new ones go by name alone. At most 60 columns. OMITTING options keeps that column's current choices; [] clears them. Colours: slate, stone, red, orange, amber, yellow, lime, green, teal, cyan, sky, blue, indigo, violet, fuchsia, pink.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/schema" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "datasetId": "<id>", "columns": [{ "key": "email", "name": "Email", "type": "email" }] }'
```

### POST /v1/datasets/options

Edit one choice column's options

- MCP tool: `argus_update_dataset_options`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Add, rename, recolour, reorder or remove the options of ONE select or tags column. The option KEY is what rows store and never changes, so renaming only changes the visible label and rewrites no row -- never use argus_update_rows to fix an option's wording. A cell holding a removed option keeps its value and shows it as a former option. At most 50 options per column. Colours are names: slate, stone, red, orange, amber, yellow, lime, green, teal, cyan, sky, blue, indigo, violet, fuchsia, pink. Only remove and order need the user's approval in the app; adding, renaming and recolouring apply straight away because they touch no row.

Fields:
- `datasetId` (string, required) — From argus_list_datasets.
- `column` (string, required) — The select or tags column, by key or by name.
- `add` (objects) — New options: ["Bounced"] or [{label, color?}]. Without a colour, the next one in the palette is used.
- `rename` (objects) — Each {key, label}. Changes the visible label only; no row is rewritten.
- `recolor` (objects) — Each {key, color}.
- `remove` (strings) — Option keys to undeclare. Rows keep their value and show it as a former option.
- `order` (strings) — The complete remaining key list, in the order you want them drawn.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/options" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "datasetId": "<id>", "column": "Status", "add": ["Bounced"], "rename": [{ "key": "New", "label": "Fresh" }] }'
```

### POST /v1/datasets/sample

A page of a dataset's rows

- MCP tool: `argus_sample_rows`

A page of a dataset's rows in insertion order: from is 0-based, limit defaults to 10 and caps at 25. Returns the total row count, the starting position, and each row's values by column key. Use argus_query_rows for filtering or aggregation.

Fields:
- `datasetId` (string, required)
- `from` (number) — 0-based offset. Defaults to 0.
- `limit` (number) — Rows to return. Default 10, at most 25.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/sample" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "datasetId": "<id>", "from": 0, "limit": 10 }'
```

### POST /v1/datasets/query

Query a dataset's rows by where conditions, optionally aggregated

- MCP tool: `argus_query_rows`

Query a dataset's rows. where is a list of {column, op, value} with op one of contains, equals, not_equals, gt, gte, lt, lte, is_empty, not_empty. aggregate is {op: count|sum|avg|min|max, column?, group_by?} -- sum and the rest need a numeric column, and group_by splits the result into groups. limit defaults to 20 and caps at 50. Scans at most 20,000 rows and reports how many were scanned when the dataset is larger, so a partial answer is never presented as the whole truth.

Fields:
- `datasetId` (string, required)
- `where` (objects) — List of {column, op, value} conditions, ANDed together.
- `aggregate` (object) — {op, column?, group_by?} instead of rows.
- `limit` (number) — Rows to return. Default 20, at most 50.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/query" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "datasetId": "<id>", "where": [{ "column": "status", "op": "equals", "value": "new" }] }'
```

### POST /v1/datasets/rows

Append rows to a dataset

- MCP tool: `argus_add_rows`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Append rows to a dataset. Each row is a flat object keyed by the dataset's column keys (argus_list_datasets); undeclared keys are stored but render nowhere until the schema is extended. At most 1000 rows per call. Returns the persisted row count, re-counted after the write rather than trusted. More than 20 rows asks the user to approve.

Fields:
- `datasetId` (string, required)
- `rows` (objects, required) — Flat row objects, keyed by column key. At most 1000.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/rows" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "datasetId": "<id>", "rows": [{ "email": "a@b.c" }] }'
```

### POST /v1/datasets/rows/update

Rewrite dataset rows in place by id

- MCP tool: `argus_update_rows`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Rewrite rows in place by id with whole replacement objects: [{id, row}]. A row keeps its position in the dataset -- this is a real UPDATE, not a delete-and-reinsert. At most 1000 per call. The user must approve this in the app.

Fields:
- `datasetId` (string, required)
- `updates` (objects, required) — List of {id, row}: the row id from argus_sample_rows/argus_query_rows and its complete replacement object. At most 1000.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/rows/update" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "datasetId": "<id>", "updates": [{ "id": "<rowId>", "row": { "email": "b@c.d" } }] }'
```

### POST /v1/datasets/rows/delete

Delete dataset rows by id or by a where filter

- MCP tool: `argus_delete_rows`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Delete rows by id, or by the same where filter argus_query_rows takes -- a filter is resolved to real rows first, so the approval card states an exact count. At most 1000 per call. There is no undo.

Fields:
- `datasetId` (string, required)
- `ids` (strings) — Explicit row ids. Alternative to where.
- `where` (objects) — The same conditions argus_query_rows takes; resolved to matching rows first.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/rows/delete" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "datasetId": "<id>", "where": [{ "column": "status", "op": "equals", "value": "old" }] }'
```

### POST /v1/datasets/trash

Move datasets to Trash

- MCP tool: `argus_trash_datasets`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Move datasets to Trash (30-day recovery; their rows go with them). The user must approve this in the app, and the card names the datasets.

Fields:
- `datasetIds` (strings, required) — From argus_list_datasets.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/trash" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "datasetIds": ["<id>"] }'
```

### GET /v1/datasets/trashed

The datasets currently in Trash

- MCP tool: `argus_list_trashed_datasets`

The datasets currently in Trash (deleted within the last 30 days, pending purge), with when each was deleted. The fix for a dataset that seems to have vanished.

```sh
curl -X GET "http://127.0.0.1:39219/v1/datasets/trashed" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/datasets/restore

Bring datasets back out of Trash

- MCP tool: `argus_restore_datasets`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Bring datasets back out of Trash. Refused at the 200-dataset workspace cap.

Fields:
- `datasetIds` (strings, required) — From argus_list_trashed_datasets.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/restore" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "datasetIds": ["<id>"] }'
```

### POST /v1/datasets/purge

Permanently delete datasets from Trash

- MCP tool: `argus_purge_datasets`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Permanently delete datasets from Trash, rows and all. There is no undo. The user must approve this in the app, and the card names the datasets.

Fields:
- `datasetIds` (strings, required) — From argus_list_trashed_datasets.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/purge" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "datasetIds": ["<id>"] }'
```

## Projects

### GET /v1/projects

The workspace's projects, what each holds, and the index of its brain

- MCP tool: `argus_list_projects`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

The workspace's projects: each one's id, name, one-line goal, how many profiles, proxies, cookie sets, automations and datasets it holds, and the index of its local brain -- every document by path and title, without the bodies. A project is a named piece of work that spans every tab ('grow 20 accounts for Client X'), and its brain is where the rules, notes and runbooks for that work are kept. Read a project's direction with argus_project_context; read any single document as the MCP resource argus://project/<id>/<path>, which is also what resources/list enumerates.

```sh
curl -X GET "http://127.0.0.1:39219/v1/projects" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### GET /v1/projects/{id}/context

One project's goal, brief and rules, plus the titles of its other documents

- MCP tool: `argus_project_context`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

One project's direction and its constraints: the goal, the brief's front matter (its autonomy level, the tools it refuses outright, the hours work may happen in), the brief's prose, the FULL TEXT of every rule the project keeps, and the TITLES of its notes, runbooks and plans. This is how an outside orchestrator learns what a project is for and what it may not do -- read it before acting on anything the project holds, because the rules are binding and are written down nowhere else. Titles rather than bodies for everything but the rules is the point: fetch the few you actually want as resources (argus://project/<id>/<path>) instead of paying for all of them.

Fields:
- `projectId` (string, required) — From argus_list_projects. This is the {id} path segment rather than a body field -- over HTTP it goes in the URL; argus_project_context takes it as an argument and builds the path.

```sh
curl -X GET "http://127.0.0.1:39219/v1/projects/{id}/context" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### GET /v1/projects/{id}/docs/{path}

One document from a project's brain, by path

- MCP tool: none — HTTP only
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

One document from a project's brain by path -- 'PROJECT.md', 'rules/posting-hours.md', 'notes/login-moved.md', 'plans/2026-08-13.json'. This backs the MCP resource argus://project/<id>/<path> and has no tool of its own on purpose: a tool would spend context in every request on documents the model has not asked for, whereas a resource is fetched only once the client has decided it wants that body. Paths come from argus_list_projects or from resources/list; only the shape the project store keeps is served -- the brief at the root, or one of rules/, notes/, runbooks/ and plans/ one level down -- and anything else is a 404 rather than a read.

```sh
curl -X GET "http://127.0.0.1:39219/v1/projects/{id}/docs/{path}" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/projects/{id}/delegate

Hand a goal to Argus's own orchestrator, inside a project's context

- MCP tool: `argus_delegate`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Hand a goal to Argus's own orchestrator, which already holds this project's brief, rules and notes, and get back its summary. Prefer this over driving the project's profiles yourself: the orchestrator enforces the project's autonomy level, its `never` list and its acting hours in code rather than by being asked nicely, and it writes what it learns back into the project's notes where the next session will read it. The summary describes what was done or set in motion -- work that outlives the call keeps running in the launcher, so re-read argus_project_context or the project's notes for how it ended.

Fields:
- `projectId` (string, required) — From argus_list_projects. This is the {id} path segment rather than a body field -- over HTTP it goes in the URL; argus_delegate takes it as an argument and builds the path.
- `goal` (string, required) — One instruction, in plain language, as you would give it to a colleague who has already read the project's brief. Do not restate the project's rules -- the orchestrator has them.

```sh
curl -X POST "http://127.0.0.1:39219/v1/projects/{id}/delegate" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "goal": "Warm up the five new profiles for an hour each" }'
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

### GET /v1/proxy-share-limit

This machine's proxy-sharing threshold

- MCP tool: `argus_get_proxy_share_limit`

This machine's proxy-sharing threshold: how many profiles per proxy pass without a warning chip, whether the warning is on, and whether two profiles on the same platform escalate ahead of the count. It is a per-MACHINE preference (localStorage), not a workspace setting -- two people can honestly disagree about how much sharing is too much, and argus_sharing_report judges against it.

```sh
curl -X GET "http://127.0.0.1:39219/v1/proxy-share-limit" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/proxy-share-limit

Change this machine's proxy-sharing threshold

- MCP tool: `argus_set_proxy_share_limit`

Change this machine's proxy-sharing threshold. `enabled` turns the warning chip on or off; `limit` is profiles per proxy that pass without escalating (1-50, clamped); `flagSamePlatform` makes two profiles on the same platform escalate ahead of the count. Only the fields you send are written. Local to this machine -- it changes what THIS launcher's tables warn about, not what a teammate's do.

Fields:
- `enabled` (boolean)
- `limit` (number) — Profiles per proxy that pass without escalating. 1-50; clamped.
- `flagSamePlatform` (boolean)

```sh
curl -X POST "http://127.0.0.1:39219/v1/proxy-share-limit" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "limit": 5 }'
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

Fields:
- `profileId` (string, required) — The profile whose session to read. It must already be open -- argus_launch_profile opens one.

### argus_navigate (MCP tool)

Point a page at a URL and wait for it to settle

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

Fields:
- `profileId` (string, required) — The running profile whose active page to point somewhere.
- `url` (string, required) — Where to send the active page.

### argus_read_page (MCP tool)

Read a page's visible text, whole or by CSS selector

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

Fields:
- `profileId` (string, required) — The running profile whose active page to read.
- `selector` (string) — CSS selector; defaults to the whole body.
- `maxChars` (number) — How much text to return. Defaults to 20000.

### argus_screenshot (MCP tool)

Screenshot a page — JPEG by default, PNG on request

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

Fields:
- `profileId` (string, required) — The running profile whose active page to capture.
- `fullPage` (boolean) — Capture the whole scrollable page rather than the viewport.
- `png` (boolean) — Return a lossless PNG instead of the default JPEG.

### argus_eval (MCP tool)

Evaluate a JavaScript expression in a page and return its value

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

Fields:
- `profileId` (string, required) — The running profile whose active page to evaluate in.
- `expression` (string, required) — A JavaScript expression. Promises are awaited and the resolved value returned.

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
