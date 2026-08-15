# Argus assistant skills

The working briefs the in-app assistant runs on. **All 16 already ship
inside Argus** — they are published here to be read and adapted, not
installed.

Each file is in the shape `POST /v1/skills/save` accepts, with `skillId` and
`entryTabs` deliberately omitted, so editing one and posting it creates your
own custom skill rather than overwriting a built-in.

| Skill | Tools | What it is for |
| --- | --- | --- |
| [`app-guide.json`](./app-guide.json) | _docs and navigation only_ | Explain Argus features, find where things live, and take the user there. |
| [`projects.json`](./projects.json) | `projects` pack | Group work that spans tabs into a project, and keep notes and rules about it. |
| [`profiles.json`](./profiles.json) | `profiles` pack | Create, organize, and launch browser profiles; explain fingerprint options. |
| [`proxies.json`](./proxies.json) | `proxies` pack | Add, test, rename, and assign proxies to profiles. |
| [`cookies.json`](./cookies.json) | `cookies` pack | Explain cookie import/export and assign cookie sets to profiles. |
| [`dataset-organizer.json`](./dataset-organizer.json) | `datasets` pack | Inspect, clean, dedupe, and edit datasets collected in the Data tab. |
| [`automations.json`](./automations.json) | `automations` pack | Explain, build, edit, delete and run automations from chat. |
| [`schedule.json`](./schedule.json) | `schedule` pack | Read the calendar and set up workflows that run at a time you pick. |
| [`session-hygiene.json`](./session-hygiene.json) | _docs and navigation only_ | Fix cookie sets and proxies shared across too many profiles. |
| [`troubleshooter.json`](./troubleshooter.json) | _docs and navigation only_ | Diagnose a real failure from its actual error text and name the fix. |
| [`data-collection.json`](./data-collection.json) | _docs and navigation only_ | Plan how a site's data lands in a dataset, and build the collector that fills it. |
| [`bulk-profile-check.json`](./bulk-profile-check.json) | _docs and navigation only_ | Check many profiles against a site (login vs feed vs suspended) and tag the outliers, dataset-first. |
| [`agent-access.json`](./agent-access.json) | _docs and navigation only_ | Connect Claude Code, Cursor or your own scripts to this workspace. |
| [`workspace-setup.json`](./workspace-setup.json) | _docs and navigation only_ | Take a first profile end to end: proxy, cookies, launch, then scale. |
| [`orchestrator.json`](./orchestrator.json) | _docs and navigation only_ | Split large requests: repeatable work becomes a scheduled automation, one-off work gets delegated. |
| [`skill-author.json`](./skill-author.json) | _docs and navigation only_ | Write and edit the briefs this assistant works from. |

9 of the 16 use a tool set that is not one of the 7 packs Argus
defines (`projects`, `profiles`, `proxies`, `cookies`, `datasets`, `automations`, `schedule`). A skill gets tools
only by naming a pack — naming individual tools is not possible by design —
so those 9 carry no `toolPack` and post as documentation-and-navigation
skills. The brief is still the useful part.
