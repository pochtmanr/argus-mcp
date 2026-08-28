# The Argus local API

This is a local API. It listens on loopback on the machine running Argus, and nothing off that machine can reach it. There is no hosted web API.

- Base URL: `http://127.0.0.1:39219`
- Auth: `Authorization: Bearer <YOUR_API_KEY>`
- Bodies: `Content-Type: application/json`
- Surface: 126 endpoints, 136 agent tools

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

### POST /v1/profiles/fingerprint-check

Check a running profile's fingerprint

- MCP tool: `argus_fingerprint_check`

Check what a page inside a running profile actually sees about the browser, and compare it against the identity the launcher declared for that profile. Reports per-check results -- WebRTC candidates, timezone, language, user agent, platform, screen, hardware, WebGL strings, noise hashes -- with a score and a verdict. What this can and cannot tell you: it measures what a page can see and compares it against what was declared, so a pass means the page-visible identity is COHERENT with the declared identity. It cannot verify the browser's spoofing internals, and it is not proof that a site will not detect the profile. The profile must already be open AND have been launched with a debugging port; a profile opened by hand usually has neither, and the reply says which of the two is missing rather than failing. `network: true` makes ONE outbound request from inside the profile (through its proxy) to an IP echo, and configures a STUN server so a reflexive WebRTC candidate can exist to be caught -- the default makes no outbound request at all.

Fields:
- `profileId` (string, required) — From argus_list_profiles. The profile must be open with a debugging port -- launch it through argus_launch_profile if it is not.
- `network` (boolean) — Opt in to the two probes that touch the network: a STUN server for reflexive ICE candidates, and one request to an IP echo through the profile's own proxy. Defaults to false, which makes no outbound request at all. Send true only when the exit IP or a WebRTC srflx leak is the question being asked -- it spends metered proxy bandwidth and puts one more request in that profile's history.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/fingerprint-check" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "network": false }'
```

### POST /v1/profiles/trash

Move profiles to the trash

- MCP tool: `argus_trash_profiles`

Move profiles to the trash, which is undoable with argus_restore_profiles. Prefer this to argus_delete_profile: a trashed profile keeps its cookies and its proxy assignment, and a deleted one is gone with them.

Fields:
- `profileIds` (strings, required) — From argus_list_profiles.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/trash" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileIds": ["<id>"] }'
```

### GET /v1/profiles/trashed

Profiles currently in the trash

- MCP tool: `argus_list_trashed_profiles`

The profiles currently in the trash, with the id argus_restore_profiles and argus_purge_profiles take. Trashed profiles do not appear in argus_list_profiles.

```sh
curl -X GET "http://127.0.0.1:39219/v1/profiles/trashed" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/profiles/restore

Restore profiles from the trash

- MCP tool: `argus_restore_profiles`

Bring trashed profiles back, with their cookies and proxy assignment intact. Ids come from argus_list_trashed_profiles.

Fields:
- `profileIds` (strings, required) — From argus_list_trashed_profiles.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/restore" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileIds": ["<id>"] }'
```

### POST /v1/profiles/purge

Delete trashed profiles permanently

- MCP tool: `argus_purge_profiles`

Permanently delete profiles that are already in the trash, with their cookie jars. There is no undo and no second copy -- this raises a card for the user to confirm before anything is destroyed.

Fields:
- `profileIds` (strings, required) — From argus_list_trashed_profiles.

```sh
curl -X POST "http://127.0.0.1:39219/v1/profiles/purge" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileIds": ["<id>"] }'
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

### GET /v1/recipes

List the prebuilt recipes, and which ones this workspace has set up

- MCP tool: `argus_list_recipes`

The prebuilt workflows that ship with Argus - 25 across eight categories on two tabs. `surface` splits them and it is the field to read first: 'scraper' means it is pointed at a target you name - a search, a URL, a profile - and the rows are the deliverable, and those are the eight on the Scraper tab (Google Maps and Instagram collectors, plus the any-site harvesters they are built out of). 'automations' means it operates your own accounts: session checks, register audits, and the dashboard readers that export a screen you are signed in to. Each entry gives its slug, the site it targets if it targets one named service, what it reads and writes, its parameters, whether it needs a signed-in profile, whether its selectors have been verified against a captured page, and whether this workspace has set it up. Read this BEFORE building a collector with argus_create_automation: if one already does the job, argus_set_up_recipe is one call instead of twelve steps.

```sh
curl -X GET "http://127.0.0.1:39219/v1/recipes" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/recipes/set-up

Create one prebuilt recipe, plus any tables it needs

- MCP tool: `argus_set_up_recipe`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Create one prebuilt workflow in this workspace, along with any tables it reads or writes that are missing. Returns the new automation id - run it with argus_run_automation. Calling it twice is safe: a recipe that is already here is returned unchanged, and its steps are never overwritten, so a workspace that edited one keeps its edits. It costs one automation slot, and a recipe sitting in Trash is refused with a 409 rather than duplicated.

Fields:
- `slug` (string, required)

```sh
curl -X POST "http://127.0.0.1:39219/v1/recipes/set-up" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "slug": "<recipe slug>" }'
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

Create an automation from a list of steps. Call argus_automation_schema first for the step vocabulary. Steps are validated before anything is stored, and the error names the exact path that failed. One step type makes this call wait on a human: a nodeScript step anywhere in the tree, including inside an if branch or a loop body, runs code on the user's own machine with the launcher's access to their files and network, so storing one asks them to approve it first and the call blocks until they answer. Nothing else here is gated -- an ordinary automation is written without a prompt.

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

Change an existing automation. Only the fields you send are written; omitting steps leaves the step tree alone. Sending a step tree that contains a nodeScript step -- code that runs on the user's own machine, at any depth, in a branch or a loop body -- asks them to approve the write first, and the call blocks until they answer. Omitting steps entirely never asks, because the tree that is already stored was approved when it landed.

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

### POST /v1/automations/search

Find automations by name, step or connector

- MCP tool: `argus_search_automations`

Find automations whose name, description, tags or step contents match a query. Use this rather than reading every document with argus_get_automation when you are looking for the one that does a particular thing.

Fields:
- `query` (string, required)

```sh
curl -X POST "http://127.0.0.1:39219/v1/automations/search" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "query": "instagram login" }'
```

### POST /v1/automations/summary

What one automation does, without its full document

- MCP tool: `argus_automation_summary`

A readable account of what an automation does: its steps in order, the connectors and datasets it touches, its parameters and how it is scheduled. Far cheaper than argus_get_automation, which returns the whole JSON document and is what you want only when you are about to rewrite it.

Fields:
- `automationId` (string, required)

```sh
curl -X POST "http://127.0.0.1:39219/v1/automations/summary" \
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

Create a scheduled workflow or AI task

- MCP tool: `argus_create_schedule_entry`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Put something on the Argus calendar. An entry is ONE of two things, and you choose by which field you send. `recurrence` is required either way: {kind, at, date?, days?, from?, until?, tz?} -- kind is 'once' (with date 'YYYY-MM-DD'), 'daily', or 'weekly' (with days, 0=Sunday..6=Saturday); at is 'HH:MM' 24-hour; tz is an optional IANA zone like 'Europe/Berlin', and without it the time is read on whatever machine runs it. `from` and `until` bound a daily or weekly schedule to a date range, both 'YYYY-MM-DD' and both inclusive -- WITHOUT `until` a repeat runs forever, so set it whenever the request names a period ('next week', 'until the 30th'); a 'once' takes neither, since it already names its day. Then send EITHER `steps` for an AUTOMATIONS workflow -- an ordered list of {automationId, profileIds, stopOnFail}, where each step runs its automation on each profile in turn, the next starts when the last finishes, and an empty profileIds means a browserless run; get ids from argus_list_automations and argus_list_profiles -- OR `prompt` for an AI TASK, which runs ONE agent turn on that instruction when it fires, using the tools the skills in `skills` carry, and reports back a sentence. Choose a workflow when the work is the same every time: it is deterministic and costs no AI tokens however often it repeats. Choose a task when the work needs a judgement or the answer IS the output ('check which proxies expire this week and tell me'); it calls a model on every run, forever. Sending both is refused. Nobody is watching when a task fires, so its prompt must stand on its own -- it cannot ask a question, and it cannot get an approval answered, so anything it writes must already be allowed by the workspace's autonomy settings. AI tasks need migration 20260844_schedule_ai_tasks; without it this answers 501. Entries fire only while the Argus launcher is open; a time it was closed for is skipped, never caught up.

Fields:
- `name` (string, required)
- `recurrence` (object, required) — {kind: 'once'|'daily'|'weekly', at: 'HH:MM', date?: 'YYYY-MM-DD', days?: int[], from?: 'YYYY-MM-DD', until?: 'YYYY-MM-DD', tz?: IANA zone}. from/until bound a daily or weekly run to a date range, inclusive; without until it repeats forever. Neither is allowed on 'once'.
- `steps` (objects) — Makes this an automations workflow. Ordered [{automationId, profileIds: string[], stopOnFail: boolean}]. At least one. Required unless `prompt` is sent instead.
- `prompt` (string) — Makes this an AI task rather than a workflow. The instruction to run unattended, at most 4000 characters. Send this OR `steps`, never both.
- `skills` (strings) — For an AI task: skill ids whose tools it may use, at most 8. Omit or send [] for the assistant's default tools. An id this build does not ship is dropped at fire time and named in the run's report, not refused here.
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

### POST /v1/schedule/day

One day of the calendar

- MCP tool: `argus_schedule_day`

Everything scheduled on one day, in clock order, with what each entry will run and whether it is enabled. Defaults to today. This is the calendar as the Schedule tab draws it; argus_schedule_history is the record of what already ran.

Fields:
- `day` (string) — YYYY-MM-DD. Defaults to today.

```sh
curl -X POST "http://127.0.0.1:39219/v1/schedule/day" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "day": "2026-08-25" }'
```

## Triggers

A trigger fires an automation when something happens rather than at a time you set — an inbound webhook, a selector appearing or disappearing on a page a profile already has open, a domain's cookies changing, another run finishing, or a file landing in a watched folder. The clock-driven half is Schedule, above. Creating or re-pointing one needs a key with no folder scope and always asks the user to approve it, because a webhook trigger mints a URL that can start runs in the workspace. Like schedules, triggers fire only while the launcher is open.

### GET /v1/event-triggers

Every event trigger in the workspace

- MCP tool: `argus_list_event_triggers`

The workspace's event triggers: what each one watches, what it runs, whether it is enabled, and when it last fired. A trigger fires an automation on an EVENT rather than on the clock -- an inbound webhook, a CSS selector appearing or disappearing on a running profile's page, a profile's cookies changing for a domain, another run finishing, or a file landing in a watched folder. Distinct from argus_list_schedule_entries, which is the wall-clock calendar. Read this before updating one: an update replaces the whole config or the whole target rather than merging. Webhook URLs are NOT returned here -- the secret in one is a credential that can start runs, and the app shows it once when the trigger is created.

```sh
curl -X GET "http://127.0.0.1:39219/v1/event-triggers" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/event-triggers/create

Create an event trigger

- MCP tool: `argus_create_event_trigger`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Run an automation when something HAPPENS rather than at a set time. `kind` picks what to watch and decides the shape of `config`: 'webhook' takes {} and answers an inbound POST; 'page' takes {profileIds, selector, on} where on is 'appears' or 'disappears', and watches only profiles that are already open; 'cookie' takes {profileIds, domain} with a bare host like 'example.com'; 'cascade' takes {sourceAutomationId, on} where on lists any of ok, partial, failed, cancelled; 'file' takes {folder, glob} with a filename pattern such as '*.csv'. `target` is what runs: {kind:'automation', automationId, profileIds} -- an empty profileIds runs without a browser -- or {kind:'entry', entryId} to run a whole scheduled workflow. `cooldownMinutes` is the floor between two fires, at least 1 and 5 by default; a flapping selector or a retried webhook would otherwise be a run each time. A cascade that would loop back into itself is refused. Triggers fire only while the launcher is open, exactly like schedules. Always asks the user to approve: a webhook trigger mints a URL that can start runs in this workspace.

Fields:
- `name` (string, required) — What to call this trigger.
- `kind` (string, required) — webhook, page, cookie, cascade or file. A trigger's kind cannot be changed later.
- `config` (object, required) — The watch settings for this kind; the description above gives each shape. Webhook takes an empty object -- its secret is minted here and never sent over this API.
- `target` (object, required) — What runs: {kind:'automation', automationId, profileIds} or {kind:'entry', entryId}. An empty profileIds is a browserless run.
- `cooldownMinutes` (number) — Minimum minutes between two fires of this trigger, 1 to 1440. 5 when omitted.
- `enabled` (boolean) — Whether it starts switched on. True when omitted.

```sh
curl -X POST "http://127.0.0.1:39219/v1/event-triggers/create" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "New leads dropped", "kind": "file", "config": { "folder": "/Users/me/Drop", "glob": "*.csv" }, "target": { "kind": "automation", "automationId": "<id>", "profileIds": [] } }'
```

### POST /v1/event-triggers/update

Change an event trigger's watch settings, target or enabled state

- MCP tool: `argus_update_event_trigger`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Change an event trigger. Every field is optional and anything omitted is left alone -- but `config` and `target` each replace their whole value rather than merging, so read the trigger with argus_list_event_triggers first and send the complete new version. Switching `enabled` off is the way to pause a trigger without losing it: a webhook then refuses its own URL, and a file watcher is dropped. This never changes a webhook's secret -- rotating one is done in the app, because it breaks every URL already handed out. Always asks the user to approve: re-pointing a live webhook aims a URL somebody already holds at a different automation.

Fields:
- `triggerId` (string, required) — From argus_list_event_triggers.
- `name` (string)
- `enabled` (boolean)
- `config` (object) — Replaces the whole watch config. Must fit the trigger's existing kind, which cannot be changed.
- `target` (object) — Replaces the whole target.
- `cooldownMinutes` (number) — Minimum minutes between two fires, 1 to 1440.

```sh
curl -X POST "http://127.0.0.1:39219/v1/event-triggers/update" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "triggerId": "<id>", "enabled": false }'
```

### POST /v1/event-triggers/delete

Delete an event trigger

- MCP tool: `argus_delete_event_trigger`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Delete an event trigger. There is no Trash for triggers and no undo: the trigger and its whole fire history go, and a webhook's URL stops working immediately. The automation it named is untouched -- a trigger only points at one. To stop a trigger without losing it, send enabled: false to argus_update_event_trigger instead.

Fields:
- `triggerId` (string, required) — From argus_list_event_triggers.

```sh
curl -X POST "http://127.0.0.1:39219/v1/event-triggers/delete" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "triggerId": "<id>" }'
```

### POST /v1/event-triggers/history

What has fired recently

- MCP tool: `argus_trigger_history`

Recent firings of the workspace's event triggers: when each fired, what it ran and how that turned out. Without a trigger id it reports every trigger. This is where to look when someone says an automation "did not run".

Fields:
- `trigger` (string) — A trigger id from argus_list_event_triggers. All triggers when omitted.
- `limit` (number) — Firings to return.

```sh
curl -X POST "http://127.0.0.1:39219/v1/event-triggers/history" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "trigger": "<id>", "limit": 20 }'
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

### POST /v1/datasets/rows/dedupe

Remove duplicate rows

- MCP tool: `argus_dedupe_rows`

Delete rows that repeat an earlier row's values in the named columns, keeping the first of each. Raises a card saying how many would go before any are removed. Column keys come from argus_list_datasets.

Fields:
- `datasetId` (string, required)
- `byColumns` (strings, required) — Column keys that together identify a duplicate.

```sh
curl -X POST "http://127.0.0.1:39219/v1/datasets/rows/dedupe" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "datasetId": "<id>", "byColumns": ["email"] }'
```

## Projects

A project is a named piece of work and the profiles, proxies, cookie sets, automations and datasets that serve it, along with a brief and a set of rules kept on that machine. Every route here needs a key with no folder scope: a project deliberately spans folders, so honouring one would mean answering with half a project. Reading a single document is the one route with no agent tool — it backs an MCP resource instead, so a body is fetched only once a client has decided it wants that body.

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

### POST /v1/projects/create

Start a project

- MCP tool: `argus_create_project`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Start a project: a named piece of work that spans profiles, proxies, automations and datasets, with a goal and a brain of its own. Returns the id the other project tools take.

Fields:
- `name` (string, required)
- `goal` (string) — One line. Becomes the project's brief.

```sh
curl -X POST "http://127.0.0.1:39219/v1/projects/create" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Client X growth", "goal": "Grow 20 accounts" }'
```

### POST /v1/projects/add

Put records into a project

- MCP tool: `argus_add_to_project`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Put profiles, proxies, cookie sets, automations or datasets into a project. Each item is {kind, id} -- kind is one of profile, proxy, cookieSet, automation, dataset. Membership is what makes the project rail show them together.

Fields:
- `projectId` (string, required)
- `items` (objects, required) — [{kind, id}] -- kind: profile | proxy | cookieSet | automation | dataset.

```sh
curl -X POST "http://127.0.0.1:39219/v1/projects/add" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "projectId": "<id>", "items": [{ "kind": "profile", "id": "<id>" }] }'
```

### POST /v1/projects/remove

Take records out of a project

- MCP tool: `argus_remove_from_project`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Take records out of a project. Removes only the membership -- the profile, proxy or automation itself is untouched. Same {kind, id} shape as argus_add_to_project.

Fields:
- `projectId` (string, required)
- `items` (objects, required) — [{kind, id}] -- kind: profile | proxy | cookieSet | automation | dataset.

```sh
curl -X POST "http://127.0.0.1:39219/v1/projects/remove" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "projectId": "<id>", "items": [{ "kind": "profile", "id": "<id>" }] }'
```

### POST /v1/projects/doc/read

Read one project document

- MCP tool: `argus_read_project_doc`

Read one document from a project's brain by path. Paths come from argus_list_projects, which indexes every document without its body. The same documents are also served as MCP resources (argus://project/<id>/<path>); this tool is for when you already know the path and want it in one call.

Fields:
- `projectId` (string, required)
- `path` (string, required) — From the project index in argus_list_projects.

```sh
curl -X POST "http://127.0.0.1:39219/v1/projects/doc/read" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "projectId": "<id>", "path": "rules/posting-hours.md" }'
```

### POST /v1/projects/doc/write

Write a project document

- MCP tool: `argus_write_project_doc`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Write a note, rule or runbook into a project's brain. Replaces the document at that path -- read it first with argus_read_project_doc unless you mean to start it over. PROJECT.md is the brief every agent reads before acting, so edit it deliberately.

Fields:
- `projectId` (string, required)
- `path` (string, required) — Relative, one level deep, .md or .json.
- `body` (string, required) — The whole document. This replaces what is there.

```sh
curl -X POST "http://127.0.0.1:39219/v1/projects/doc/write" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "projectId": "<id>", "path": "notes/2026-08-25.md", "body": "..." }'
```

### POST /v1/projects/doc/delete

Delete a project document

- MCP tool: `argus_delete_project_doc`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Delete one document from a project's brain. Raises a card first: a project note is often the only record of a decision, and there is no trash for it.

Fields:
- `projectId` (string, required)
- `path` (string, required)

```sh
curl -X POST "http://127.0.0.1:39219/v1/projects/doc/delete" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "projectId": "<id>", "path": "notes/old.md" }'
```

### POST /v1/orchestration/plan

Lay out a multi-step plan

- MCP tool: `argus_make_plan`

Record a plan as an ordered list of tasks, which the app shows as a live checklist and which argus_delegate_many can then work through. Use it when a request needs several rounds of work and the user should be able to see where it has got to.

Fields:
- `tasks` (objects, required) — [{title, detail?}] in the order they should happen.

```sh
curl -X POST "http://127.0.0.1:39219/v1/orchestration/plan" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "tasks": [{ "title": "Check every proxy" }] }'
```

### POST /v1/orchestration/delegate-many

Hand several goals to subagents at once

- MCP tool: `argus_delegate_many`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Hand several independent goals to Argus's own orchestrator to work in parallel, each with the project's context and rules already loaded. Use argus_delegate for one goal; use this when the jobs do not depend on each other, because they then run at the same time rather than one after another.

Fields:
- `jobs` (objects, required) — [{goal, pack?, projectId?}] -- independent goals only.

```sh
curl -X POST "http://127.0.0.1:39219/v1/orchestration/delegate-many" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "jobs": [{ "goal": "Check the US proxies" }, { "goal": "Check the EU proxies" }] }'
```

## Connectors

### GET /v1/connectors

The workspace's connectors, and the field list of every kind

- MCP tool: `argus_list_connectors`

The AI, message, data and captcha connectors this workspace has, and the catalogue of kinds one can be created from. Call this before writing a notify, aiPrompt, aiCheck, aiAgent, saveRows, loadRows or solveCaptcha step, or setting notifyConnectorId -- those fields take an id from here and there is no other way to learn one. A step must name a connector of the matching category: 'ai' for the AI steps and an aiAgent's chat model, 'message' for notify and an agent's message or approval tools, 'data' for saveRows, loadRows and an agent's data or vector tools, 'captcha' for solveCaptcha. Every connector carries its stored `config`, credentials included -- an API key reads back everything it can write, so treat a key as equivalent to the credentials behind it.

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
- `kind` (string, required) — A messaging kind (telegram, slack, discord, whatsapp, smtp), a data kind (supabase, postgres, redis, firestore, sheets, notion, airtable, file), a captcha kind (2captcha, capmonster) or an AI kind. See the `kinds` block of argus_list_connectors. The category follows from the kind and cannot be set.
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

Prove a connector works: a message connector sends a real test message, an AI connector asks for a one-word completion, a data connector performs the smallest read the service offers and never writes, and a captcha connector reads its account balance and solves nothing. Reports the service's own error text when it fails, which is usually the whole diagnosis. Do this after creating one rather than waiting for a run to fail. Note that a data connector passing only proves it can connect and read -- not that a save will be permitted.

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

The vocabulary an assistant skill is written in. Call this before argus_save_skill rather than guessing: it returns the tool PACK names a skill may claim (with the tools in each) and how many it may hold at once, the tab ids that may be claimed as entry tabs and who holds each one, every knowledge-base article id available for the reference shelf, and the field limits. Every tab is held by a built-in and a custom skill outranks one, so `claimedBy` is context rather than a blocklist -- only a tab held by another CUSTOM skill is refused. The same role argus_automation_schema plays for steps.

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

Read one skill in the shape argus_save_skill accepts: instructions, examples, entry tabs, reference shelf and tool packs. Read before editing -- a save replaces the whole document, so an unread field is a field you will erase.

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

Create or edit an assistant skill. Omit skillId to create (the id is slugged from the title and returned); title and instructions are required then. Pass a built-in's id to edit it -- that writes a local override, and only title, blurb, instructions, examples and docs apply; sending toolPacks or entryTabs for a built-in is refused, because those are code-defined. EVERY FIELD YOU SEND REPLACES THAT FIELD and fields you omit are left as they are, so call argus_get_skill first and send only what changes; an empty examples array deletes them. toolPacks must be pack names argus_skills_schema lists, up to 3 of them, and the skill gets the union of their tools; packs are the ONLY way a skill gets tools, and naming tools directly is not possible by design. Unknown doc ids and already-claimed tabs are dropped and reported back in `dropped` rather than failing the call. THIS CALL BLOCKS: it raises an approve-or-deny card in the app naming your key, and returns 403 if the user denies or lets it lapse.

Fields:
- `skillId` (string) — Omit to create a custom skill. A built-in's id edits that built-in.
- `title` (string) — Required when CREATING; omit it when editing and the current title is kept. Up to 60 characters.
- `blurb` (string) — One line, up to 200 characters. The ONLY part always in the assistant's context -- it is what the model reads to decide the skill applies, so write it as a routing decision, not a summary.
- `instructions` (string) — The working brief. Required when CREATING; omit it when editing and the current brief is kept. Never hand-write a reference shelf or an example list into it -- both are appended from the fields below.
- `examples` (strings) — Two to four verbatim requests in the user's voice, one line each, up to 120 characters.
- `toolPacks` (strings) — Up to 3 pack names from argus_skills_schema. The skill gets the union of their tools. Absent means documentation and navigation only. Custom skills only.
- `toolPack` (string) — The single-pack field toolPacks replaced. Still accepted, and read as a one-element list.
- `entryTabs` (strings) — Tab ids this skill is suggested on, taking the tab from whichever built-in covers it. Only a tab another CUSTOM skill already holds is dropped. Custom skills only.
- `docs` (strings) — Knowledge-base article ids for the reference shelf, e.g. concepts/proxies. Ids the KB does not have are dropped.
- `hidden` (boolean) — Built-ins only: true removes the skill from the assistant entirely. Undo it with argus_delete_skill on the same id.

```sh
curl -X POST "http://127.0.0.1:39219/v1/skills/save" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "title": "Warm-up runs", "blurb": "Plan and check profile warm-up passes.", "instructions": "You are now...", "examples": ["Warm up my five new profiles"], "toolPacks": ["profiles", "browser"], "docs": ["concepts/profiles-fingerprints"] }'
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

Fourteen tools with no endpoint behind them. They attach to a profile that is already open and speak to the page directly, which is how an agent reads a page, clicks through it and takes a screenshot.

### POST /v1/page/switch-tab

Bring one of a profile's tabs to the front

- MCP tool: `argus_switch_tab`

Make one of the profile's open tabs the active one, so every later page verb acts on it. Tab ids come from argus_list_tabs.

Fields:
- `profileId` (string, required)
- `tabId` (string, required) — From argus_list_tabs.

```sh
curl -X POST "http://127.0.0.1:39219/v1/page/switch-tab" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "tabId": "<id>" }'
```

### POST /v1/page/new-tab

Open a new tab in a launched profile

- MCP tool: `argus_new_tab`

Open a new tab in a launched profile and make it the active one. Without a url it opens blank. Use this rather than argus_navigate when the page you are on is worth keeping.

Fields:
- `profileId` (string, required)
- `url` (string) — Optional. Opens a blank tab when omitted.

```sh
curl -X POST "http://127.0.0.1:39219/v1/page/new-tab" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "url": "https://example.com" }'
```

### POST /v1/page/recording

What has been done to this page so far

- MCP tool: `argus_page_recording`

The steps taken on this profile's page since it was launched, in the shape an automation stores them. Read this before argus_save_page_recording to see what would be saved, and which steps you may want to drop.

Fields:
- `profileId` (string, required)

```sh
curl -X POST "http://127.0.0.1:39219/v1/page/recording" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>" }'
```

### POST /v1/page/save-recording

Save what was done to a page as an automation

- MCP tool: `argus_save_page_recording`
- Key scope: needs a key with no folder scope. Automations are shared across every folder and have none of their own, so a folder-scoped key may run them but may not author them.

Turn what you have just done to a page into a saved automation that can be run again and scheduled. Call argus_page_recording first and pass dropSteps for any the automation should not repeat -- a wrong turn, a step that only made sense once.

Fields:
- `profileId` (string, required)
- `name` (string, required) — What the saved automation is called.
- `description` (string)
- `dropSteps` (numbers) — Indexes from argus_page_recording to leave out. Numbers, 0-based.

```sh
curl -X POST "http://127.0.0.1:39219/v1/page/save-recording" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "profileId": "<id>", "name": "Daily check", "dropSteps": [3] }'
```

### argus_page_snapshot (MCP tool)

The interactive elements on a page, each with a selector to act on it

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

Fields:
- `profileId` (string, required) — The running profile whose active page to describe.
- `root` (string) — A CSS selector to scope the snapshot to one container. Use this on a page with hundreds of controls.
- `maxElements` (number) — How many elements to return. Defaults to 60.

### argus_click (MCP tool)

Click an element, or a point, in a running profile's page

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

Fields:
- `profileId` (string, required) — The running profile to click in.
- `selector` (string) — A CSS selector. From argus_page_snapshot.
- `nth` (number) — Which match, when the selector is not unique. Zero-based.
- `x` (number) — Viewport x, for content with no element to name.
- `y` (number) — Viewport y, for content with no element to name.

### argus_type (MCP tool)

Type into a field in a running profile's page

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

Fields:
- `profileId` (string, required) — The running profile to type in.
- `selector` (string, required) — The field to type into. From argus_page_snapshot.
- `text` (string, required) — The text to type.
- `clear` (boolean) — Empty the field first. Defaults to true.
- `delayMs` (number) — Per-key delay. Set this to send real key events instead of one paste-shaped insert.
- `pressEnter` (boolean) — Press Enter afterwards.

### argus_press_key (MCP tool)

Press a single key in a running profile's page

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

Fields:
- `profileId` (string, required) — The running profile to press a key in.
- `key` (string, required) — Enter, Tab, Escape, Backspace, Delete, ArrowUp, ArrowDown, ArrowLeft, ArrowRight, Home, End, PageUp, PageDown, or a single character.
- `selector` (string) — Focus this element first.

### argus_scroll (MCP tool)

Scroll a running profile's page

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

Fields:
- `profileId` (string, required) — The running profile to scroll.
- `to` (string) — top or bottom.
- `by` (number) — Scroll by this many pixels instead.
- `selector` (string) — Scroll this element into view instead.

### argus_wait_for (MCP tool)

Wait until a page reaches a condition

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

Fields:
- `profileId` (string, required) — The running profile to wait on.
- `selector` (string) — Wait until this appears.
- `selectorGone` (string) — Wait until this disappears.
- `text` (string) — Wait until the page contains this text.
- `url` (string) — Wait until the URL contains this.
- `timeoutMs` (number) — How long to wait. Defaults to 15000, capped at 120000.

### argus_extract (MCP tool)

Read values out of a running profile's page

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

Fields:
- `profileId` (string, required) — The running profile to read from.
- `selector` (string, required) — What to read. From argus_page_snapshot.
- `what` (string) — text, html, value or attr. Defaults to text.
- `attr` (string) — Which attribute, when what is attr.
- `all` (boolean) — Return every match rather than the first.

### argus_select_option (MCP tool)

Choose an option in a dropdown

- No endpoint. It attaches to a profile that is already open and speaks to the page over CDP, so launch the profile first.

Fields:
- `profileId` (string, required) — The running profile whose dropdown to set.
- `selector` (string, required) — The select element. From argus_page_snapshot.
- `value` (string, required) — The option's value, or its visible label.

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

## Mail

### GET /v1/mail/mailboxes

The mailboxes this workspace can read

- MCP tool: `argus_list_mailboxes`

The email accounts this workspace has connected, with the id every other mail tool takes. Each is read through the exit its linked profile uses, so a mailbox and the profile it belongs to stay correlated -- you never supply a proxy.

```sh
curl -X GET "http://127.0.0.1:39219/v1/mail/mailboxes" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/mail/messages

List a mailbox folder

- MCP tool: `argus_list_messages`

Headers from one folder of a mailbox, newest first: uid, from, subject and date. Bodies are not included -- argus_read_message fetches one. Folder defaults to the inbox.

Fields:
- `mailboxId` (string, required) — From argus_list_mailboxes.
- `folder` (string) — Defaults to the inbox.
- `limit` (number) — Messages to return.

```sh
curl -X POST "http://127.0.0.1:39219/v1/mail/messages" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "mailboxId": "<id>", "folder": "INBOX", "limit": 20 }'
```

### POST /v1/mail/message

Read one message

- MCP tool: `argus_read_message`

One message in full, by uid, including its body and any links in it. The uid and folder come from argus_list_messages, and a uid is only meaningful inside its own folder.

Fields:
- `mailboxId` (string, required)
- `folder` (string, required) — The folder the uid came from.
- `uid` (number, required) — From argus_list_messages.

```sh
curl -X POST "http://127.0.0.1:39219/v1/mail/message" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "mailboxId": "<id>", "folder": "INBOX", "uid": 4211 }'
```

### POST /v1/mail/wait

Wait for a message that has not arrived yet

- MCP tool: `argus_read_new_message`

Wait for a message matching a sender or subject pattern and return it when it lands. This is how a confirmation or verification code is collected: send the form first, then call this. It holds the connection until the message arrives or timeoutSeconds passes, and is the one route allowed to wait past the usual budget -- at most about 100 seconds, so poll again rather than asking for more.

Fields:
- `mailboxId` (string, required)
- `fromPattern` (string) — Substring or regex matched against the sender.
- `subjectPattern` (string) — Substring or regex matched against the subject.
- `sinceSeconds` (number) — Only consider messages this recent. Guards against an older match.
- `timeoutSeconds` (number) — How long to wait before giving up.
- `folders` (string) — Comma-separated folders to watch. Defaults to the inbox.

```sh
curl -X POST "http://127.0.0.1:39219/v1/mail/wait" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "mailboxId": "<id>", "fromPattern": "noreply@", "timeoutSeconds": 120 }'
```

### POST /v1/mail/compose/open

Start a draft

- MCP tool: `argus_open_compose`

Start a draft on one mailbox and return its draftId. Nothing is sent: fill it with argus_set_compose_fields and send it with argus_send_compose, which is what raises the card. Opening a draft is deliberately a separate step from sending one, so a half-written message cannot go out.

Fields:
- `mailboxId` (string, required)
- `to` (string)
- `cc` (string)
- `subject` (string)
- `body` (string)

```sh
curl -X POST "http://127.0.0.1:39219/v1/mail/compose/open" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "mailboxId": "<id>", "to": "someone@example.com", "subject": "Hello" }'
```

### GET /v1/mail/compose

Drafts that are open right now

- MCP tool: `argus_list_compose`

The drafts currently open, with their ids and what each one has in it so far. Use it to find a draftId you have lost track of before sending or closing it.

```sh
curl -X GET "http://127.0.0.1:39219/v1/mail/compose" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### POST /v1/mail/compose/set

Fill in a draft

- MCP tool: `argus_set_compose_fields`

Set fields on an open draft. Only the fields you pass change; the rest keep what they had. Without a draftId it acts on the only open draft, and says so if there is more than one.

Fields:
- `draftId` (string) — From argus_open_compose or argus_list_compose.
- `to` (string)
- `cc` (string)
- `bcc` (string)
- `subject` (string)
- `body` (string)

```sh
curl -X POST "http://127.0.0.1:39219/v1/mail/compose/set" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "draftId": "<id>", "body": "..." }'
```

### POST /v1/mail/compose/send

Send a draft

- MCP tool: `argus_send_compose`

Send an open draft. This one always raises a card showing the recipient, subject and body: mail is the one thing here that leaves the machine and reaches another person, and it cannot be recalled.

Fields:
- `draftId` (string) — The only open draft when omitted.

```sh
curl -X POST "http://127.0.0.1:39219/v1/mail/compose/send" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "draftId": "<id>" }'
```

### POST /v1/mail/compose/close

Discard a draft

- MCP tool: `argus_close_compose`

Throw away an open draft without sending it.

Fields:
- `draftId` (string) — The only open draft when omitted.

```sh
curl -X POST "http://127.0.0.1:39219/v1/mail/compose/close" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "draftId": "<id>" }'
```

## Knowledge base

### POST /v1/kb/docs

The knowledge-base index

- MCP tool: `argus_list_docs`

Index the knowledge base: every article's id and one-line summary, without the bodies. Four wings -- pages (one per screen), concepts (how each part behaves), troubleshooting (error, cause, fix) and practice (outside domain knowledge the work needs, dated, and NOT about Argus). Read an article with argus_read_doc. Consult this before telling a user the app cannot do something.

Fields:
- `wing` (string) — Narrow to one wing: pages, concepts, troubleshooting or practice. All four when omitted.

```sh
curl -X POST "http://127.0.0.1:39219/v1/kb/docs" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "wing": "troubleshooting" }'
```

### POST /v1/kb/doc

Read one knowledge-base article

- MCP tool: `argus_read_doc`

One knowledge-base article in full. Ids come from argus_list_docs or argus_search_docs. These are the same articles the in-app assistant reads, and they are the authority on how a feature actually behaves.

Fields:
- `docId` (string, required) — From argus_list_docs or argus_search_docs.

```sh
curl -X POST "http://127.0.0.1:39219/v1/kb/doc" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "docId": "<id>" }'
```

### POST /v1/kb/search

Search the knowledge base

- MCP tool: `argus_search_docs`

Search the knowledge base and get back the articles that match, best first, with the passage that matched. Start here when you do not know which article answers the question -- it is cheaper than reading the index and then guessing.

Fields:
- `query` (string, required)

```sh
curl -X POST "http://127.0.0.1:39219/v1/kb/search" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{ "query": "why did my proxy check fail" }'
```

### GET /v1/context

What this workspace actually holds

- MCP tool: `argus_get_context`

An honest account of the workspace as it is right now: how many profiles, proxies, automations, datasets and mailboxes there are, which are in trouble, and what is running. Call this before answering a question about what the user has -- it is the difference between a real answer and a guess.

```sh
curl -X GET "http://127.0.0.1:39219/v1/context" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

### GET /v1/errors/recent

What has gone wrong lately

- MCP tool: `argus_recent_errors`

The errors this workspace has hit recently -- failed runs, proxy checks, mail sends -- with what was being done at the time. Read it before debugging anything, and before claiming something works.

```sh
curl -X GET "http://127.0.0.1:39219/v1/errors/recent" \
  -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json"
```

## Connection

### argus_connection_status (MCP tool)

Confirm this MCP connection works, and say as whom

- No endpoint. It asks the MCP server about this connection itself -- whether it is authenticated and what it can see -- so it needs nothing open and no arguments.

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
