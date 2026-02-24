---
title: Building a WoW TBC Classic Addon with Claude Code and Mechanic
slug: building-boneyard-tbc-with-claude
publishDate: 17 Feb 2026
description: How I used Claude Code and the Mechanic development hub to build BoneyardTBC — a guild management addon for WoW TBC Classic Anniversary Edition.
---

[**BoneyardTBC**](https://www.curseforge.com/wow/addons/boneyardtbc) is a guild management addon for WoW TBC Classic Anniversary Edition. GearScore tracking, a bulletin board, a dungeon optimizer, officer analytics. I built it with [Claude Code](https://claude.ai/code) and [**Mechanic**](https://github.com/Falkicon/Mechanic) as the development backbone.

---

## What It Does

BoneyardTBC gives guilds visibility into their own roster:

- **GearScore** for every guild member — addon users and non-users alike (silent background inspection)
- **Bulletin Board** — guild-wide in-game announcements with timers
- **DungeonOptimizer** — route planner that tracks rep grind progress across TBC heroics and raids
- **Officer Analytics** — weekly activity charts and GearScore progression graphs

The addon syncs over WoW's `GUILD` addon message channel. Everyone with it installed shares data automatically. Don't have the addon? Your GearScore still gets populated whenever someone inspects you.

---

## The Toolchain

### Mechanic

[**Mechanic**](https://github.com/Falkicon/Mechanic) is a dev hub for WoW addon development. Python CLI, MCP server, web dashboard, and a sandbox for running unit tests without launching the game.

The killer feature is the **SavedVariables bridge**. Mechanic watches WoW's SavedVariables files after a `/reload`, parses them, and surfaces errors and `print()` output in the dashboard. That closes the loop between writing code and seeing what it does in-game.

### Claude Code

Claude Code did the actual coding. With Mechanic exposed via MCP, Claude could:

- Look up TBC API signatures (`api.search`)
- Lint code (`addon.lint`) with Luacheck mid-conversation
- Run unit tests (`sandbox.test`) and see failures inline
- Read in-game errors (`addon.output`) after a `/reload`

The loop:

1. Describe a feature
2. Claude reads the files, writes the code
3. I `/reload` in WoW
4. Claude reads the errors itself and fixes them
5. Repeat

No copy-pasting errors. No switching between terminal and game. Claude sees everything through Mechanic.

---

## The Hard Parts

### TBC Classic API Surface

TBC Classic Anniversary runs a completely different API surface from retail. Many `C_*` namespaces don't exist. I had to:

- Clone `Gethe/wow-ui-source` on the `classic_anniversary` branch
- Parse the FrameXML into a local API database (3,459 TBC APIs)
- Generate Luacheck stubs and sandbox stubs from that

After that, Claude searched against TBC-accurate docs instead of retail.

### Silent Inspection

Populating GearScore for guild members who don't have the addon was the interesting problem. The approach:

- Hook `INSPECT_READY` when the inspect frame opens
- Verify the target is a guild member via `GetGuildRosterInfo()` — not just guild name, because connected realms share a channel
- Calculate GearScore from equipped items silently
- Broadcast to all addon users over the `GUILD` channel

The annoying part was realm names. Connected realms give you names like `Pensatankki-Spineshatter`. Every lookup had to strip the realm suffix before matching roster keys.

### Unit Testing Without WoW

WoW addons normally need the game client to run. Mechanic's sandbox fakes it with generated API stubs and a Busted test runner.

Final count: **37 tests** for the core addon, **94 tests** for DungeonOptimizer. The full suite runs in seconds and catches regressions before they touch the game.

### Keeping Claude Oriented

The project grew to multiple modules — `Core.lua`, `GearScore.lua`, `Sync.lua`, `Inspection.lua`, `BulletinBoard.lua`, `AddonDetection.lua`, plus a UI layer and the DungeonOptimizer companion addon.

Two things kept Claude from getting lost across sessions:

- **`ARCHITECTURE.md`** — 782 lines covering every module, data flow, and message protocol
- **`MEMORY.md`** — Claude Code's persistent memory tracking version history and architectural decisions

---

## What Worked

**The MCP loop.** Everything flowed through the conversation. Claude linted, tested, and read output without me relaying anything.

**Incremental releases.** Each version (r7 through r10) added one or two features with tests first. Small steps, clean codebase.

**Separate SavedVariables** for DungeonOptimizer. The core addon works standalone. Install or remove the companion without touching core data.

---

## What Didn't

**In-game UI debugging is still awful.** Lua errors in WoW are cryptic one-liners. Taint errors are worse — protected UI code gets called from addon code and fails silently. No error, no log. Just nothing happening.

**TBC API gaps.** Some APIs in the FrameXML source aren't actually callable in the Anniversary client, or have different signatures. The only way to know is to try it in-game and watch it break.

---

## The Result

[**BoneyardTBC**](https://www.curseforge.com/wow/addons/boneyardtbc) runs in production for my guild on Spineshatter-EU. GearScore leaderboard, bulletin board, officer analytics — all actively used.

Built in one week of evening sessions. Zero prior Lua knowledge. Higher test coverage than any addon I've written before. [**Mechanic**](https://github.com/Falkicon/Mechanic) made it possible — an error bridge, a correct API database, and MCP tools turning WoW addon dev into something that resembles actual software engineering.
