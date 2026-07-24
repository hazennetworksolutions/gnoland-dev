<div align="center">

# 🤖 Connect gno.land to Your AI Assistant (gnomcp)

**A step-by-step guide to using `gnomcp` — the official gno.land MCP server —**
**with Cursor, Claude, and any other MCP-compatible AI client.**
*Read realms, evaluate expressions, audit code, and (on testnet) write — all grounded in live chain data, backed by our public Topaz full node and tx-indexer.*

[![Gnoland](https://img.shields.io/badge/Gnoland-Topaz-A259FF?style=flat-square)](https://gno.land)
[![MCP](https://img.shields.io/badge/Protocol-MCP-000000?style=flat-square)](https://modelcontextprotocol.io)
[![gnomcp](https://img.shields.io/badge/gnomcp-gnoverse%2Fgno--mcp-2496ED?style=flat-square&logo=github)](https://github.com/gnoverse/gno-mcp)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Network:** Gnoland Topaz Testnet (Chain ID: `topaz-1`)
> **Tool:** [gnomcp](https://github.com/gnoverse/gno-mcp) — MCP server + agent skill for gno.land
> **Last Updated:** July 2026

---

## Table of Contents

- [Why this matters](#why-this-matters)
- [What you get](#what-you-get)
- [Endpoints](#endpoints)
- [Step 1 — Install gnomcp](#step-1--install-gnomcp)
- [Step 2 — Add the Hazen Topaz profile](#step-2--add-the-hazen-topaz-profile)
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
OpenCode, …). We run a full Topaz node and a public **tx-indexer** — this
guide points `gnomcp` at both, unlocking its indexer-backed tools
(`gno_history`, `gno_activity`) that the default public profile doesn't have.

> ⚠️ `gnomcp` is explicitly **work-in-progress, unaudited, pre-release**
> software (per its own README). The tool API can change and the session
> write path is being reworked. Treat it accordingly — read the install
> script before running it, keep write scopes tight, and expect updates.

---

## What you get

| Category | Tools (examples) | Needs |
|---|---|---|
| Chain reads | `gno_render`, `gno_read`, `gno_eval`, `gno_packages`, `gno_account`, `gno_status` | nothing — works out of the box |
| **Indexer reads** | `gno_history`, `gno_activity` | a profile with a `tx-indexer-url` — **our profile below provides this** |
| Writes (dev/testnet only) | `gno_call`, `gno_run`, `gno_addpkg`, key/faucet tools | a funded agent key |
| Sessions (WIP) | `gno_session_propose`, `gno_session_revoke` | your own `gnokey` — gnomcp never sees your mnemonic |

> `gno_list` (realm catalog listing) is registered whenever an indexer is
> configured, but as of gnomcp v0.9.0 it always returns "not supported by
> this indexer" — it's a stub upstream, not something our indexer is
> missing. `gno_history` and `gno_activity` are fully functional against
> our indexer today (verified live).

Plus a **`gno` skill** — a knowledge pack that teaches your agent gno.land's
idioms, security taxonomy (`OriginSend` traps, `IsUser()` vs `IsUserCall()`),
and `Render()` conventions — and workflow skills like `gno-build`,
`gno-audit`, `gno-debug`, and `gno-onboard`.

---

## Endpoints

| Type | Endpoint |
|---|---|
| **Our Topaz RPC** | `https://gnoland-topaz-rpc.hazennetworksolutions.com` |
| **Our Topaz tx-indexer (GraphQL)** | `https://gnoland-topaz-indexer.hazennetworksolutions.com/graphql/query` |
| Official public RPC | `https://rpc.topaz.testnets.gno.land:443` |
| Official gnoweb | `https://topaz.testnets.gno.land` |
| Official faucet | `https://topaz.testnets.gno.land/faucet` |
| Our explorer | `https://explorer.hazennetworksolutions.com/gnoland-testnet/` |
| Our node setup guide | `https://guides.hazennetworksolutions.com/gnoland/` |

Both of our endpoints are public and require no authentication. The RPC
serves standard Tendermint2 JSON-RPC; the indexer server
([gnolang/tx-indexer](https://github.com/gnolang/tx-indexer)) actually
exposes **two** APIs on the same host — plain JSON-RPC at `/` and GraphQL at
`/graphql/query` — so the `/graphql/query` suffix above is required, not
optional. Leaving it off silently points `gnomcp` at the JSON-RPC endpoint
instead and its indexer tools fail.

---

## Step 1 — Install gnomcp

One command installs the **binary** (into `~/.local/bin`, checksum-verified)
and wires up the clients it can find automatically (Claude Code, Gemini CLI,
Codex):

```bash
curl -fsSL https://raw.githubusercontent.com/gnoverse/gno-mcp/main/scripts/install.sh | sh
```

> This pipes a script from the internet into your shell. Read
> [scripts/install.sh](https://github.com/gnoverse/gno-mcp/blob/main/scripts/install.sh)
> first — that's good practice for any `curl | sh`.

**Cursor** doesn't get auto-registered by the installer. After the binary is
in place, add it manually — Cursor Settings → MCP → **Add new MCP server**,
or edit `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "gnomcp": { "command": "gnomcp", "args": [] }
  }
}
```

**Claude Desktop** — same idea, in its own MCP config file:

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

Restart your editor/client after installing so it picks up the new server.

---

## Step 2 — Add the Hazen Topaz profile

Save our node + indexer as a named profile so `gnomcp` remembers it between
runs:

```bash
gnomcp profile add hazen-topaz \
  --rpc https://gnoland-topaz-rpc.hazennetworksolutions.com:443 \
  --chain-id topaz-1 \
  --indexer-url https://gnoland-topaz-indexer.hazennetworksolutions.com/graphql/query
```

This writes to `~/.config/gnomcp/profiles.toml`. You can also edit that file
directly — same result:

```toml
[hazen-topaz]
rpc-url        = "https://gnoland-topaz-rpc.hazennetworksolutions.com:443"
chain-id       = "topaz-1"
tx-indexer-url = "https://gnoland-topaz-indexer.hazennetworksolutions.com/graphql/query"
```

Verify it loaded — ask your agent, or run:

```bash
gnomcp profile list
```

You should see `hazen-topaz   topaz-1   https://gnoland-topaz-rpc.hazennetworksolutions.com:443   [read-only]`
(it stays read-only until you add `--master g1...` or a session — see
[Step 4](#step-4--writes--sessions-optional)).

---

## Step 3 — Try it

Just talk to your agent in plain language and tell it to use the
`hazen-topaz` profile. A few starting points:

> *"Using the hazen-topaz profile, which realms exist under gno.land/r/? List a few and tell me what they do."*

> *"Render gno.land/r/gnops/valopers on hazen-topaz and summarize the registered validators."*

> *"Give me a formal audit of gno.land/r/&lt;path&gt; on hazen-topaz before I interact with it."*
> *(uses the `gno-audit` skill — read-only, works on any chain, quoted findings with severity)*

> *"Show the recent deploy and transaction history of gno.land/r/&lt;path&gt; on hazen-topaz."*
> *(this is `gno_history` — it only works because our profile carries an indexer URL)*

New to gno.land entirely? Just ask your agent to teach you — the
`gno-onboard` skill gauges your background and gives you a guided tour.

---

## Step 4 — Writes & sessions (optional)

Everything above is read-only and safe on any chain, including mainnet.
Writing (deploying, calling functions) only works on dev/testnet chains —
never on mainnet or betanet, by design.

**Agent identity (simplest):** generate and fund a key once, then let the
agent sign with it directly:

> *"Generate an agent key for hazen-topaz, then fund it from the faucet."*

**Sessions (WIP — use with caution):** have the agent act *as you*, scoped
and time-boxed, without ever touching your mnemonic:

> *"Propose a session on hazen-topaz scoped to gno.land/r/&lt;path&gt; with a low spend limit and a short expiry."*

This prints a `gnokey maketx session create` command — you review and run it
yourself, on your own machine, with your own `gnokey`. Keep `allow_paths`
narrow and `spend_limit`/`expires_in` tight, and revoke when done:

> *"Revoke that session on hazen-topaz."*

---

## Security notes

- **Your keys never leave `gnokey`.** gnomcp never sees a mnemonic or seed phrase.
- **No signing on real-funds chains.** Writes are gated to dev/testnet; mainnet and betanet are read-only.
- **Chain output is treated as untrusted.** Every chain-derived byte (realm markdown, account data, etc.) is wrapped so it can't silently hijack your agent's instructions.
- **Prompt injection isn't fully solved anywhere in the industry.** Malicious content embedded in a realm's `Render()` output is a real, open risk — that's exactly why an audit pass (`gno-audit`) before interacting with an unfamiliar realm is worth doing.
- **Every write is logged** by gnomcp in an append-only local audit trail.

---

## Links

- [gnoverse/gno-mcp](https://github.com/gnoverse/gno-mcp) — the gnomcp source, full docs, and security model
- [gnolang/gno #5661](https://github.com/gnolang/gno/issues/5661) — the "AI agents x Gno" roadmap this fits into
- [Our Topaz explorer](https://explorer.hazennetworksolutions.com/gnoland-testnet/) — blocks, realms, validators, live consensus
- [Our Topaz node setup guide](https://guides.hazennetworksolutions.com/gnoland/) — run your own full node / validator

---

Built and operated by **Hazen Network Solutions**
🌐 [hazennetworksolutions.com](https://www.hazennetworksolutions.com) · [GitHub](https://github.com/hazennetworksolutions) · [@haznftofficial](https://twitter.com/haznftofficial)
