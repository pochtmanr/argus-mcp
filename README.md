# Argus — API and MCP reference

Documentation, the assistant skill briefs, and the MCP registry manifest for
[Argus](https://www.browserargus.com), an anti-detect browser for managing hundreds of isolated
browser profiles — each with its own fingerprint, proxy and cookie jar.

> **Which Argus this is.** Argus is published by Simnetiq Ltd (UK company
> number 16861177) at [www.browserargus.com](https://www.browserargus.com).
> Other companies and software products share the name Argus and are unrelated
> to this one.

This is a local API. It listens on loopback on the machine running Argus, and nothing off that machine can reach it. There is no hosted web API.

## There is nothing here to install

The Argus MCP server ships **inside the launcher's own application bundle** and
is started through it with `ELECTRON_RUN_AS_NODE`. There is no npm package, no
`npx` command and nothing new listening: it speaks JSON-RPC over stdin and
stdout to the client that launched it, and reaches the local API on your behalf.

For most clients the launcher writes the config file for you and shows you the
path first. Two of them have no file it can write, and hand you a snippet
instead. See [Integrations and MCP setup](https://www.browserargus.com/integrations).

This repository is the **documentation** for that server, not the server.

## What is here

| Path | What it is |
| --- | --- |
| [`docs/api-reference.md`](./docs/api-reference.md) | All 55 endpoints and 53 tools — paths, fields, curl examples, status codes. |
| [`docs/openapi.json`](./docs/openapi.json) | The same table as an OpenAPI 3.1 spec, for generating a client. |
| [`docs/mcp-tools.md`](./docs/mcp-tools.md) | The tool list an MCP client receives, and which endpoint each one fronts. |
| [`skills/`](./skills) | The briefs the in-app assistant runs on, as readable JSON. |
| [`agents/`](./agents) | Drop-in rules for your own `CLAUDE.md`, `AGENTS.md` or `.cursorrules`. |
| [`server.json`](./server.json) | The MCP registry manifest. |

## Quick start

1. [Download Argus](https://www.browserargus.com/downloads) and open the launcher.
2. Mint a key in the **API** tab. It is shown once.
3. Call it:

```sh
curl -s "http://127.0.0.1:39219/v1/profiles" \
  -H "Authorization: Bearer $ARGUS_KEY"
```

The first call from a new client raises an approve-or-deny card in the launcher
and blocks until you answer it.

To drive it from a coding agent instead, connect an MCP client — Claude Code,
Codex, Cursor, Gemini CLI, Windsurf, VS Code, Zed and others are supported — and
the 53 tools appear on their own.

## About `skills/`

An Argus *skill* is the working brief the in-app assistant loads for one job:
instructions, worked examples, a reference shelf, and the pack of tools it may
reach.

**These twelve already ship inside Argus.** They are published here to be read
and to be adapted — not installed, because you already have them. Each file is
in the shape `POST /v1/skills/save` accepts, so editing one and posting it
creates *your own* custom skill:

```sh
curl -X POST "http://127.0.0.1:39219/v1/skills/save" \
  -H "Authorization: Bearer $ARGUS_KEY" \
  -H "Content-Type: application/json" \
  -d @skills/proxies.json
```

Note what the files deliberately omit:

- **No `skillId`.** Posting one without an id creates a new custom skill.
  Posting a *built-in's* id would instead write a local override of that
  built-in, which is not what a reader of this repo is asking for.
- **No `entryTabs`.** A custom skill can claim a tab from a built-in, and an
  installed copy silently displacing the shipped skill's tab suggestion is a
  surprise nobody asked for.
- **`toolPack` only where one fits.** Argus defines six packs
  (`profiles`, `proxies`, `cookies`, `datasets`, `automations`,
  `schedule`) and a skill gets tools by naming one — naming individual tools is
  not possible by design. Six of the twelve briefs use a tool set that is not one
  of those packs; their files carry no `toolPack`, so posting one as-is gives a
  skill with documentation and navigation tools only. The brief is still the
  useful part.

## Links

- Website: https://www.browserargus.com
- API reference: https://www.browserargus.com/api-reference
- Integrations and MCP setup: https://www.browserargus.com/integrations
- Everything as one file: https://www.browserargus.com/llms-full.txt

## Licence

MIT — see [LICENSE](./LICENSE). The grant covers the contents of this repository: the generated documentation, the agent rules and the skill briefs. Argus itself — the launcher, the hardened browser and the MCP server — is proprietary and is not part of this repository.
