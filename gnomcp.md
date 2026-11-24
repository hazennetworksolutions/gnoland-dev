<div align="center">

# 🤖 Connect gno.land to Your AI Assistant (gnomcp)

**A step-by-step guide to using `gnomcp` — the official gno.land MCP server —**
**with Cursor, Claude, and any other MCP-compatible AI client.**
*Read realms, evaluate expressions, audit code, and (on testnet) write — all grounded in live chain data, backed by our public Onyx full node and tx-indexer.*

[![Gnoland](https://img.shields.io/badge/Gnoland-Onyx-A259FF?style=flat-square)](https://gno.land)
[![MCP](https://img.shields.io/badge/Protocol-MCP-000000?style=flat-square)](https://modelcontextprotocol.io)
[![gnomcp](https://img.shields.io/badge/gnomcp-gnoverse%2Fgno--mcp-2496ED?style=flat-square&logo=github)](https://github.com/gnoverse/gno-mcp)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Network:** Gnoland Onyx Testnet (Chain ID: `onyx-1`)
> **Tool:** [gnomcp](https://github.com/gnoverse/gno-mcp) — MCP server + agent skills for gno.land (checked against v0.13.1)
> **Last Updated:** November 2026

> 🔁 **Coming from the Topaz version of this guide?** Topaz (`topaz-1`) has
> been retired and our Topaz endpoints are offline. Remove the old profile
> with `gnomcp profile remove hazen-topaz` and follow
> [Step 2](#step-2--add-the-hazen-onyx-profile) to add `hazen-onyx`.

---

## Table of Contents

- [Why this matters](#why-this-matters)
- [What you get](#what-you-get)
- [Endpoints](#endpoints)
- [Step 1 — Install gnomcp](#step-1--install-gnomcp)
- [Step 2 — Add the Hazen Onyx profile](#step-2--add-the-hazen-onyx-profile)
- [Step 3 — Try it](#step-3--try-it)
- [Step 4 — Writes & sessions (optional)](#step-4--writes--sessions-optional)
- [Security notes](#security-notes)
- [Links](#links)

---

## Why this matters

gno.land is one of very few chains where the source code of every deployed
package lives **on-chain**, not just compiled bytecode. That means an AI
agent can actually *read* a realm's Gno source, reason about it, and flag
risk before you ever sign a transaction — something that isn't possible on
bytecode-only chains.

`gnomcp` is the official bridge between that on-chain source and your AI
client of choice (Cursor, Claude Code, Claude Desktop, Gemini CLI, Codex,
OpenCode, …). It ships with a built-in `testnet` profile that already points
at the official Onyx RPC and tx-indexer, so it works out of the box.

We run our own Onyx full node and a public **tx-indexer**. This guide adds
them as a second, independent profile — `hazen-onyx` — so you have a fallback
when the official endpoints are slow, rate-limited or down, and a second
source to cross-check indexer-backed answers (`gno_history`, `gno_activity`)
against.

> ⚠️ `gnomcp` is explicitly **work-in-progress, unaudited, pre-release**
> software (per its own README). The tool API can change and the session
> write path is being reworked. Treat it accordingly — read the install
> script before running it, keep write scopes tight, and expect updates.

---

## What you get

| Category | Tools (examples) | Needs |
|---|---|---|
| Chain reads | `gno_render`, `gno_read`, `gno_eval`, `gno_packages`, `gno_account`, `gno_status` | nothing — works out of the box |
| **Indexer reads** | `gno_history`, `gno_activity` | a profile with a `tx-indexer-url` — the built-in `testnet` profile and **our profile below** both have one |
| Writes (dev/testnet only) | `gno_call`, `gno_run`, `gno_addpkg`, key/faucet tools | a funded agent key, or an active session |
| Sessions (WIP) | `gno_session_propose`, `gno_session_revoke` | your own `gnokey` — gnomcp never sees your mnemonic |

> `gno_activity` accepts `since`/`until` time bounds, but the current
> tx-indexer schema has no block time on transactions, so passing them
> returns an error; without bounds it returns the realm's full
> MsgCall/MsgRun log. `gno_history` works fully against our indexer
> (verified live with the exact query gnomcp sends).

Plus the **`gno` skill** — a knowledge pack that teaches your agent gno.land's
idioms, security taxonomy (`OriginSend` traps, `IsUser()` vs `IsUserCall()`),
and `Render()` conventions — and workflow skills like `gno-build`,
`gno-audit`, `gno-debug`, and `gno-onboard`.

---

## Endpoints

| Type | Endpoint |
|---|---|
| **Our Onyx RPC** | `https://gnoland-onyx-rpc.hazennetworksolutions.com` |
| **Our Onyx tx-indexer (GraphQL)** | `https://gnoland-onyx-indexer.hazennetworksolutions.com/graphql/query` |
| Official public RPC | `https://rpc.onyx.testnets.gno.land:443` |
| Official tx-indexer (GraphQL) | `https://indexer.onyx.testnets.gno.land/graphql/query` |
| Official gnoweb | `https://onyx.testnets.gno.land` |
| Faucet | `https://faucet.gno.land` |
| Our explorer | `https://explorer.hazennetworksolutions.com/gnoland-onyx/` |
| Our node setup guide | `https://guides.hazennetworksolutions.com/gnoland/` |

Both of our endpoints are public and require no authentication. The RPC
serves standard Tendermint2 JSON-RPC; the indexer server
([gnolang/tx-indexer](https://github.com/gnolang/tx-indexer)) actually
exposes **two** APIs on the same host — plain JSON-RPC at `/` and GraphQL at
`/graphql/query` — so the `/graphql/query` suffix above is required, not
optional. gnomcp appends nothing: leaving the suffix off points it at the
JSON-RPC API and its indexer tools fail.

---

## Step 1 — Install gnomcp

gnomcp has two parts and every client needs both: the **binary** (the MCP
server) and the **plugin** (the skills and agents). One command installs the
binary into `~/.local/bin` (checksum-verified) and sets up the clients it
finds — Claude Code and Gemini CLI fully automatically, Codex with the MCP
server registered plus printed plugin steps, and OpenCode with printed steps:

```bash
curl -fsSL https://raw.githubusercontent.com/gnoverse/gno-mcp/main/scripts/install.sh | sh
```

> This pipes a script from the internet into your shell. Read
> [scripts/install.sh](https://github.com/gnoverse/gno-mcp/blob/main/scripts/install.sh)
> first — that's good practice for any `curl | sh`.

**Upgrading is the same command** — re-running it moves the binary to the
newest release and refreshes the plugin for Claude Code and Gemini CLI.

**Cursor** installs gnomcp through its own plugin flow (it reads the repo's
`.cursor-plugin/plugin.json`).

**Claude Desktop and other MCP clients** — any host that runs local stdio
servers works. With the binary installed, add it to the client's MCP config:

```json
{
  "mcpServers": {
    "gnomcp": { "command": "gnomcp", "args": [] }
  }
}
```

**Docker**, if you'd rather not install a binary:

```json
{
  "mcpServers": {
    "gnomcp": { "command": "docker", "args": ["run", "-i", "--rm", "ghcr.io/gnoverse/gnomcp:latest"] }
  }
}
```

> The container's state (audit log, sessions, agent keys) is wiped on exit
> unless you mount a volume at `/home/nonroot/.local/share/gnomcp`, and it
> can't reach a local gnodev node on `127.0.0.1`. For day-to-day use, prefer
> the binary.

Restart your editor/client after installing — plugins and MCP servers load
at startup.

---

## Step 2 — Add the Hazen Onyx profile

Save our node + indexer as a named profile so `gnomcp` remembers it between
runs:

```bash
gnomcp profile add hazen-onyx \
  --rpc https://gnoland-onyx-rpc.hazennetworksolutions.com:443 \
  --chain-id onyx-1 \
  --indexer-url https://gnoland-onyx-indexer.hazennetworksolutions.com/graphql/query
```

This writes to `~/.config/gnomcp/profiles.toml`. You can also edit that file
directly — and if you want the faucet tool to work on this profile too, do it
that way, because `profile add` has no flags for the gnoweb and faucet
fields (the last two values below are the ones gnomcp's own built-in
`testnet` profile uses):

```toml
[hazen-onyx]
rpc-url            = "https://gnoland-onyx-rpc.hazennetworksolutions.com:443"
chain-id           = "onyx-1"
tx-indexer-url     = "https://gnoland-onyx-indexer.hazennetworksolutions.com/graphql/query"
gnoweb-url         = "https://onyx.testnets.gno.land"
faucet-service-url = "https://faucet-agent.onyx.testnets.gno.land"
```

> `gnomcp profile add` refuses the built-in names (`testnet`, `mainnet`,
> `local`, `default`), which is why ours is called `hazen-onyx`.

Verify it loaded — ask your agent, or run:

```bash
gnomcp profile list
```

You should see a line like
`hazen-onyx   onyx-1     https://gnoland-onyx-rpc.hazennetworksolutions.com:443  [read-only]`.
`[read-only]` here only means no master address is set yet: agent-key writes
still work on `onyx-1`, and session writes need `--master g1...` (see
[Step 4](#step-4--writes--sessions-optional)).

---

## Step 3 — Try it

Just talk to your agent in plain language and tell it to use the
`hazen-onyx` profile. A few starting points:

> *"Using the hazen-onyx profile, which realms exist under gno.land/r/? List a few and tell me what they do."*

> *"Render gno.land/r/gnops/valopers on hazen-onyx and summarize the registered validators."*

> *"Give me a formal audit of gno.land/r/&lt;path&gt; on hazen-onyx before I interact with it."*
> *(uses the `gno-audit` skill — read-only, works on any chain, quoted findings with severity)*

> *"Show the recent deploy and transaction history of gno.land/r/&lt;path&gt; on hazen-onyx."*
> *(this is `gno_history` — served by our tx-indexer)*

New to gno.land entirely? Just ask your agent to teach you — the
`gno-onboard` skill gauges your background and gives you a guided tour.

---

## Step 4 — Writes & sessions (optional)

Everything above is read-only and safe on any chain, including mainnet.
Writing (deploying, calling functions) only works on dev/testnet chains —
never on mainnet, by design.

**Agent identity (simplest):** generate and fund a key once, then let the
agent sign with it directly:

> *"Generate an agent key for hazen-onyx, then fund it from the faucet."*

Two Onyx rules to know before you write:

- **Deploys park first.** Onyx runs an *inert* code-submission policy: a new
  package stays parked until the chain's package approver enables it.
  `gno_addpkg` reports `package_status` — `live`, or `PARKED` with the
  chain's reason — and a parked package answers reads with `package_parked`.
- **`gno_run` is restricted.** Onyx accepts `MsgRun` only from addresses in
  its `run_submitters` list; gnomcp checks this first and refuses with
  `run_not_allowed` before anything is signed. `gno_call` is unaffected.

**Sessions (WIP — use with caution):** have the agent act *as you*, scoped
and time-boxed, without ever touching your mnemonic. Add your address to the
profile first (`--master g1...` on `profile add`, or `master-address` in the
TOML), then:

> *"Propose a session on hazen-onyx scoped to gno.land/r/&lt;path&gt; with a low spend limit and a short expiry."*

This prints a `gnokey maketx session create` command — you review and run it
yourself, on your own machine, with your own `gnokey`. Keep `allow_paths`
narrow and `spend_limit`/`expires_in` tight, and revoke when done:

> *"Revoke that session on hazen-onyx."*

---

## Security notes

- **Your keys never leave `gnokey`.** gnomcp never sees a mnemonic or seed phrase.
- **No signing on real-funds chains.** Writes are gated to dev/testnet; mainnet (and any unknown chain-id) is read-only.
- **Chain output is treated as untrusted.** Every chain-derived byte (realm markdown, account data, etc.) is wrapped so it can't silently hijack your agent's instructions.
- **Prompt injection isn't fully solved anywhere in the industry.** Malicious content embedded in a realm's `Render()` output is a real, open risk — that's exactly why an audit pass (`gno-audit`) before interacting with an unfamiliar realm is worth doing.
- **Every write is logged** by gnomcp in an append-only local audit trail.

---

## Links

- [gnoverse/gno-mcp](https://github.com/gnoverse/gno-mcp) — the gnomcp source, full docs, and security model
- [gnomcp install & configuration docs](https://github.com/gnoverse/gno-mcp/blob/main/docs/gnomcp.md) — every client, profiles, write authorization
- [gnolang/gno #5661](https://github.com/gnolang/gno/issues/5661) — the "AI agents x Gno" roadmap this fits into
- [Our Onyx explorer](https://explorer.hazennetworksolutions.com/gnoland-onyx/) — blocks, realms, validators, live consensus
- [Our Onyx node setup guide](https://guides.hazennetworksolutions.com/gnoland/) — run your own full node / validator

---

Built and operated by **Hazen Network Solutions**
🌐 [hazennetworksolutions.com](https://www.hazennetworksolutions.com) · [GitHub](https://github.com/hazennetworksolutions) · [@haznftofficial](https://twitter.com/haznftofficial)
