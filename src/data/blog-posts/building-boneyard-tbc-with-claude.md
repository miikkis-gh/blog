---
title: Building a WoW TBC Classic Addon with Claude Code and Mechanic
slug: building-boneyard-tbc-with-claude
publishDate: 17 Feb 2026
description: How I used Claude Code and the Mechanic development hub to build BoneyardTBC — a guild management addon for WoW TBC Classic Anniversary Edition.
---

# Building a WoW TBC Classic Addon with Claude Code and Mechanic

I recently built **BoneyardTBC**, a guild management addon for World of Warcraft TBC Classic Anniversary Edition, using [Claude Code](https://claude.ai/code) as my AI pair programmer and a custom development hub called **Mechanic** to streamline the feedback loop. This is the story of how that went.

---

## What Is BoneyardTBC?

BoneyardTBC is a WoW addon that gives guilds better visibility into their own roster. It tracks:

- **GearScore** for every guild member (both addon users and non-users via silent background inspection)
- **Bulletin Board** — a guild-wide in-game announcement system with timers
- **DungeonOptimizer** — a route planner that tracks rep grind progress across TBC heroics and raids
- **Officer Analytics** — weekly activity charts and GearScore progression graphs

The addon talks to itself over WoW's addon message channel (`GUILD`), so data propagates across all guild members who have it installed. Non-addon users get their GearScore populated silently whenever someone inspects them.

---

## The Toolchain

### Mechanic

Mechanic is a local development hub I built alongside the addon. It consists of:

- A **Python CLI** (`mechanic`) for linting, formatting, testing, and releasing addons
- An **MCP server** that exposes all of those tools to Claude Code directly
- A **dashboard** (web UI) that shows in-game errors, console output, and test results in real time
- A **sandbox** for running Busted unit tests against WoW API stubs without launching the game

The key feature is the **SavedVariables bridge** — Mechanic watches for WoW's SavedVariables files to update after a `/reload`, then parses them and surfaces errors and `print()` output in the dashboard. This closes the feedback loop between code changes and in-game results.

### Claude Code

Claude Code acted as my pair programmer throughout. Because Mechanic exposes tools via MCP, Claude could:

- Search the WoW API database (`api.search`) to find the right function signatures
- Lint code (`addon.lint`) with Luacheck without leaving the conversation
- Run unit tests (`sandbox.test`) and see failures inline
- Read in-game error output (`addon.output`) after I confirmed a `/reload`

The development loop looked like this:

1. Describe a feature to Claude
2. Claude reads the relevant files, proposes an implementation
3. I approve, Claude edits the files
4. I `/reload` in WoW and tell Claude it's done
5. Claude calls `addon.output` to read errors and console logs
6. Repeat

---

## Interesting Technical Challenges

### TBC Classic API Differences

TBC Classic Anniversary Edition runs on a different API surface than retail WoW. Many C_* namespaced APIs don't exist or behave differently. Early on I had to:

- Clone the `Gethe/wow-ui-source` repository on the `classic_anniversary` branch
- Parse the FrameXML source into a local API database (3,459 TBC APIs)
- Generate Luacheck stubs and sandbox stubs from that database

This meant Claude was always searching against TBC-accurate API documentation, not retail.

### Silent Inspection Tracking

One of the more interesting features was populating roster GearScore for guild members who don't have the addon. The solution:

- Hook the `INSPECT_READY` event whenever the WoW inspect frame opens
- Verify the inspected target is a guild member using `GetGuildRosterInfo()` (not just guild name — connected realms share a channel)
- Calculate GearScore from the equipped item data silently in the background
- Broadcast the result to all other addon users via the `GUILD` addon message channel

The tricky part was realm name normalization. On connected realms, player names include a suffix like `Pensatankki-Spineshatter`. Every lookup needed to strip the realm before matching against roster keys.

### Unit Testing Without WoW

WoW addons normally require the game client to run. Mechanic's sandbox solves this with generated API stubs and a Busted test runner. By the end of the project, BoneyardTBC had:

- **37 unit tests** for the core addon (GearScore, Sync)
- **94 unit tests** for DungeonOptimizer (Optimizer, Sync, Tracker, Utils)

Running the full suite takes a few seconds and catches regressions before they ever hit the game.

### Keeping Claude in Context

The project grew to multiple modules: `Core.lua`, `GearScore.lua`, `Sync.lua`, `Inspection.lua`, `BulletinBoard.lua`, `AddonDetection.lua`, plus a full UI layer and a companion addon (DungeonOptimizer).

To keep Claude oriented across sessions I maintained two things:

- **`ARCHITECTURE.md`** — a 782-line living document describing every module, its responsibilities, data flow, and message protocols
- **`MEMORY.md`** — a persistent memory file in the Claude Code project that tracked version history, known patterns, and architectural decisions

Claude reads these at the start of sessions and stays coherent across days of work.

---

## What Worked Well

**The MCP feedback loop** was the biggest win. Instead of switching between a terminal, a text editor, and the game, everything flowed through the conversation. Claude would lint, test, and check output without me having to copy-paste anything.

**Incremental versioning** helped too. Each release (r7 through r10) introduced one or two features, with tests added before implementation. This kept the codebase clean and caught regressions early.

**Separate SavedVariables** for the optional DungeonOptimizer companion addon was the right call. It meant the core addon worked standalone and the companion could be installed or removed without touching core data.

---

## What Was Harder Than Expected

**In-game UI debugging** is still painful. Lua errors in WoW surface as cryptic one-liners, and the Mechanic dashboard can only show what gets logged. Taint errors (where protected UI code gets accidentally called from addon code) are especially subtle — they often manifest as silent failures rather than visible errors.

**TBC API coverage gaps** occasionally bit us. Some APIs listed in the FrameXML source weren't actually callable in the Anniversary client, or had different signatures. The only reliable check is trying it in-game.

---

## The Result

BoneyardTBC is running in production for my guild on the Spineshatter-EU realm. The GearScore leaderboard, bulletin board, and officer analytics tabs are all actively used. The DungeonOptimizer is helping people plan their heroic rep grinds for T5 content.

The whole thing was built in roughly two weeks of evening sessions — much faster than I would have managed alone, and with a higher test coverage than any addon I've written before.

If you're interested in the toolchain, Mechanic is the piece that made this possible. The combination of an in-game error bridge, a local API database, and MCP tool exposure to Claude turned addon development from a slow trial-and-error process into something that actually resembles normal software development.
