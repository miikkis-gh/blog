---
title: From Blind Reloads to a Real Development Pipeline
slug: from-claude-to-mechanic
publishDate: 18 Feb 2026
description: How I went from writing WoW addons with just Claude Code and no feedback loop to discovering Mechanic and building a proper development pipeline for TBC Classic.
---

I started making WoW addons with Claude Code and a folder. No linter. No tests. No version control. No API reference. Claude could read and write my files — that part worked. The problem was everything after that. Claude was blind. It couldn't see what happened after a `/reload`, couldn't verify if an API existed in TBC Classic, couldn't run a single test. I was the eyes, the ears, and the error log.

---

## The Pure-Claude Code Era

My first addon was **BoneyardTBC** — guild management for TBC Classic Anniversary. GearScore, roster sync, a leaderboard. The first revisions (r1–r4, around Feb 10) were pure Claude Code sessions.

The loop looked like this:

1. Describe a feature to Claude
2. Claude writes the `.lua` files
3. `/reload` in WoW
4. Something breaks — I read the error and describe it back
5. Claude has no idea if the API it used even exists in TBC
6. Claude guesses a fix
7. Back to step 3

Code writing was fast. Everything else was slow.

Claude's WoW API knowledge is a cocktail of retail and Classic. It would confidently drop `C_Spell.GetSpellInfo()` — retail only — when TBC uses the old global `GetSpellInfo()`. I wouldn't find out until the `/reload` blew up.

No git either. Just loose files in a folder. When your dev process is "write, reload, describe error," version control isn't top of mind.

---

## The Traces Left Behind

I went digging through my files recently. The archaeological evidence was everywhere.

**Phase plans next to the code.** BoneyardTBC has `ATTENDANCE_PHASE1.md`, `PHASE2`, `PHASE3` — Claude broke a feature into phases and saved the plans alongside the Lua. No project memory, no skills, no reference library. Just markdown as a poor man's session persistence.

**A frozen architecture doc.** `ARCHITECTURE.md`, 782 lines, still says "Version: r7" on line 32. A fossil. Claude generated it, the workflow shifted, and nobody ever touched it again.

**A 973-line UI reference guide.** `Beautiful_Addon_UI_Design_for_20505.md` — every frame-level UI technique in TBC's modern client. `BackdropTemplate` patterns, texture paths, animation groups. Claude researched this from the web when I wanted to know what was possible on Interface 20505. It landed in the dev folder and stayed there.

**No git anywhere.** Not a single `.git` directory across any of my addons. All loose folders. That's what happens when every session starts fresh and your feedback loop runs through a human.

---

## Discovering Mechanic — Built for the Wrong Client

I found [**Mechanic**](https://github.com/Falkicon/Mechanic) — a dev hub for WoW addon developers by Falkicon. Python CLI, MCP server, web dashboard, sandbox for unit tests without launching the game.

One problem: **Mechanic was built for retail** (Interface 120001). I was on TBC Classic Anniversary (Interface 20505). Completely different API surfaces. Retail has `C_Spell.GetSpellInfo()`, TBC has the global `GetSpellInfo()`. Retail has `C_Container`, TBC has direct bag functions. Entire namespaces — `C_ClassTalents`, `C_Traits`, `C_DelvesUI` — flat out don't exist. The API database Mechanic ships with was populated from retail FrameXML. Every lookup was potentially wrong.

### Finding the Right FrameXML

Mechanic has `api.download` to pull FrameXML from [Townlong Yak](https://www.townlong-yak.com/framexml). Claude ran it, populated the database — with retail APIs. Townlong Yak only hosts retail builds. No TBC section. So now I had a nice searchable database full of wrong answers.

I pointed Claude to [Gethe's `wow-ui-source`](https://github.com/Gethe/wow-ui-source) on GitHub — a community archive of FrameXML for every client variant. The trick was the branch name. No `tbc` branch, no `bcc`. TBC Classic Anniversary lives on `classic_anniversary` — FrameXML 2.5.5, 191 Blizzard addons.

Cloned it, pointed Mechanic's populator at it. **3,459 TBC-accurate APIs** replaced the retail database. I also had to yank `C_Engraving` (26 APIs) and `C_Seasons` (2 APIs) — Season of Discovery leftovers that leaked into the Anniversary client's shared codebase.

After that, `api.search` gave TBC-correct answers. No more retail ghosts sneaking into my code.

### The MCP Bridge

This was the real unlock. Mechanic exposes tools as an MCP server, so Claude Code calls them directly mid-session:

- **`addon.output`** — reads in-game errors and logs after a `/reload`
- **`addon.lint`** — runs Luacheck without leaving the conversation
- **`sandbox.test`** — executes Busted unit tests against WoW API stubs
- **`api.search`** — looks up function signatures from the TBC database

The new loop:

1. Describe a feature
2. Claude reads the files, writes the code
3. I `/reload` in WoW
4. Claude calls `addon.output` and reads the errors itself
5. Claude fixes them. Done.

I stopped being the error relay. Claude could see the Lua errors, look up correct APIs, lint the code, and run tests — all in one session.

---

## Building the Pattern Library

I copied well-known addons into my dev folder and had Claude tear them apart. Extract patterns, document techniques, figure out how the pros do it.

32 addon analyses. 224 pattern documents. 74,000 lines. Organized into 27 topic guides — combat log parsing, tooltip hooking, map pin rendering, secure frame handling, data architecture.

Instead of generating Lua from first principles, Claude now looks up how DBM or Details! solved the same problem. Turns out production code is a better teacher than a language model's imagination.

---

## What I Learned

**The bottleneck was never the code.** Claude could always write Lua. The bottleneck was the feedback loop — blind to errors, no API verification, no tests. I was the only bridge between the code and the game, and I was a slow, lossy bridge.

**Reference material compounds.** That 973-line UI guide? Still useful. The pattern library from 32 addons? Every new feature is faster because Claude starts from proven patterns instead of hallucinating Lua.

**You don't need to understand every line.** I'm not a Lua expert. I can read it, I know the patterns, and I can describe what I want. Claude handles event registration order, connected-realm normalization edge cases, GearScore math. My job is knowing what the addon should do and confirming it does.

**Tools beat talent.** Day one: one addon, no tests, constant API errors. One week later: six addons, 131 unit tests. That wasn't me getting better at Lua. That was tooling.

---

## The Numbers

| Addon | Lines of Lua | Unit Tests |
|-------|-------------|------------|
| BoneyardTBC | 7,177 | 37 |
| BoneyardTBC_DungeonOptimizer | 7,443 | 94 |
| BoneyardTBC_Officer | 10,132 | — |
| CommanderText | 1,649 | — |
| DungeonFAQ | 1,084 | — |
| DisenchantValue | 559 | — |
| **Total** | **28,044** | **131** |

28,000 lines of Lua, 131 tests, zero prior Lua knowledge, one week of evening sessions. The pipeline did the heavy lifting.
