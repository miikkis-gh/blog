---
title: From Blind Reloads to a Real Development Pipeline
slug: from-claude-to-mechanic
publishDate: 18 Feb 2026
description: How I went from writing WoW addons with just Claude Code and no feedback loop to discovering Mechanic and building a proper development pipeline for TBC Classic.
---

# From Blind Reloads to a Real Development Pipeline

I started making WoW addons with nothing but Claude Code and a development folder. No linter, no test framework, no version control, no API reference. Claude Code could read and write the addon files directly — that part was fine. The problem was everything else: Claude had no way to see what happened after a `/reload` in WoW, no way to look up whether an API actually existed in TBC Classic, and no way to run tests. I was the bridge for all of that.

A couple of weeks later I was building new addons from scratch in single evening sessions. This is the story of what changed in between, and the archaeological evidence I found in my own codebase when I went looking for traces of that early workflow.

---

## The Pure-Claude Code Era

My first addon was **BoneyardTBC** — a guild management tool for TBC Classic Anniversary Edition. GearScore calculation, guild roster sync, a leaderboard. The earliest revisions (r1 through r4, around Feb 10) were built entirely through Claude Code sessions.

Claude Code could read my addon files and write Lua directly — no manual copy-pasting of code. But the development loop still had a massive blind spot:

1. Describe a feature to Claude Code
2. Claude writes the `.lua` files directly
3. `/reload` in WoW
4. Something breaks — I read the error in chat and describe it back to Claude
5. Claude has no idea if the API it just used actually exists in TBC Classic
6. Claude suggests a fix based on what it thinks the API surface looks like
7. Repeat from step 3

The code writing was fast. The feedback was slow. Claude couldn't see in-game errors, couldn't verify API signatures against TBC Classic's actual surface, couldn't run any tests. Every error had to flow through me. And Claude's knowledge of WoW APIs is a mix of retail and Classic — it would confidently use `C_Spell.GetSpellInfo()` (retail only) when it should have used the old global `GetSpellInfo()` (which is what TBC has). I wouldn't know until the `/reload` failed.

There was no version control — the addon lives as loose files in a development folder with no `.git` directory. No linting to catch issues before they hit the game. No test framework. Just Claude writing code and me reporting back what happened.

---

## The Traces Left Behind

When I recently went digging through my files looking for evidence of this early workflow, the traces were everywhere.

**Phase plan documents sitting inside the addon folder.** BoneyardTBC has three files: `ATTENDANCE_PHASE1.md`, `ATTENDANCE_PHASE2.md`, `ATTENDANCE_PHASE3.md` — a multi-phase plan for building an attendance tracking feature. Claude Code broke the feature into phases and saved the plans next to the code so the next session could pick up where it left off. No project memory yet, no skills, no reference library — just markdown files as a way to carry knowledge between sessions.

**A frozen architecture document.** `ARCHITECTURE.md` is 782 lines long and still says "Version: r7" on line 32. Claude generated it as a snapshot of the system at that point in time. Once the development workflow shifted, nobody went back to update it. It's a fossil from a specific moment.

**A 973-line UI reference guide.** `Beautiful_Addon_UI_Design_for_20505.md` is a comprehensive document covering every frame-level UI technique available in TBC Classic's modern client — `BackdropTemplate` patterns, texture paths, animation groups, preset backdrop tables. Claude Code researched this from the web when I wanted to understand what was possible for addon interfaces on Interface 20505. It ended up as a reference document saved in the development folder.

**No git anywhere.** None of my addons have their own repositories. They're all loose folders. When your development process is "write code, reload, describe errors," and each session starts mostly fresh, version control isn't the first thing you think about.

---

## Discovering Mechanic — And Realizing It Was Built for Retail

At some point I found [**Mechanic**](https://github.com/Falkicon/Mechanic) — a development hub for WoW addon developers built by Falkicon. It has a Python CLI, an MCP server that exposes tools to AI coding assistants, a web dashboard, and a sandbox for running unit tests without launching the game.

There was one problem: **Mechanic was built for retail WoW** (Interface 120001). I was building for TBC Classic Anniversary Edition (Interface 20505). These are very different API surfaces. Retail has `C_Spell.GetSpellInfo()`, TBC has the old global `GetSpellInfo()`. Retail has `C_Container`, TBC has direct bag functions. Entire namespaces like `C_ClassTalents`, `C_Traits`, `C_DelvesUI` don't exist in TBC. Using them would just silently fail or throw errors in-game.

The API database that ships with Mechanic — the thing Claude uses to look up function signatures — was populated from retail FrameXML. Every `api.search` result was potentially wrong for my client.

### Finding the Right FrameXML

Mechanic has an `api.download` command that pulls FrameXML source from [Townlong Yak](https://www.townlong-yak.com/framexml), a community site that archives Blizzard's UI source after each patch. But Townlong Yak only hosts retail builds. There's no TBC Classic section.

After some digging I found [Gethe's `wow-ui-source`](https://github.com/Gethe/wow-ui-source) repository on GitHub — a community effort that archives FrameXML for every WoW client variant. The key was finding the right branch. There's no `tbc` or `bcc` branch. The TBC Classic Anniversary client lives on the `classic_anniversary` branch, which contains FrameXML version 2.5.5 with 191 Blizzard addons.

I cloned that branch into `_dev_/framexml/2.5.5/` and pointed Mechanic's API populator at it. The result: **3,459 TBC-accurate APIs** replacing the retail database. I also had to manually remove `C_Engraving` (26 APIs) and `C_Seasons` (2 APIs) because those are Season of Discovery features that leaked into the Anniversary client's shared codebase.

From that point on, when Claude ran `api.search` to find how to do something, it got TBC-correct answers. No more retail APIs sneaking into my addon code and breaking on `/reload`.

### The MCP Bridge

The bigger insight was the **MCP integration**. Mechanic exposes its tools as an MCP server, which means Claude Code can call them directly during a session:

- **`addon.output`** — read in-game errors, console logs, and test results after a `/reload`
- **`addon.lint`** — run Luacheck on addon code without leaving the conversation
- **`sandbox.test`** — execute Busted unit tests against WoW API stubs
- **`api.search`** — look up WoW API signatures from the TBC-accurate database

The development loop got dramatically tighter:

1. Describe a feature to Claude Code
2. Claude reads the relevant files, writes the code
3. I `/reload` in WoW and confirm it's done
4. Claude calls `addon.output` and reads the errors directly — no need for me to relay them
5. Claude fixes the issues and we go again

The difference sounds small, but it changed everything. Claude could now see the actual Lua errors, search for correct TBC API signatures, lint the code, and run tests — all within the same session, without me acting as the go-between.

---

## Building the Pattern Library

The other thing that made a huge difference was studying how production addons actually work. I copied several well-known addons into my development folder and had Claude analyze their source code to extract patterns.

This turned into a **reference library**: 32 addon source analyses producing 224 pattern documents (74,000 lines), organized into a cookbook of 27 topic guides. Combat log parsing. Tooltip hooking. Map pin rendering. Secure frame handling. Data architecture. The works.

The pattern library is still evolving — I started with analysis of a handful of addons and have been expanding it over the last couple of days into a full cookbook with 27 topic guides. When Claude needs to implement a feature, it can look up how production addons solved the same problem instead of generating Lua from first principles.

---

## When the Pipeline Started Working

The difference became obvious around mid-February. Features that used to take multiple sessions of back-and-forth — describe the error, explain the context, try again — started landing in a single pass. Claude could look up the correct TBC API, write the code, and after my `/reload` it could read the errors itself and fix them without me explaining anything.

Building a new addon went from a strenuous back-and-forth to something that flowed naturally. The pipeline handled all the friction that used to drain the energy out of it.

The contrast with the early days was clear. What used to be a draining cycle of blind reloads and relaying errors had become a smooth loop where Claude could see and fix problems on its own.

---

## What I Learned

**The bottleneck was never the code writing.** Claude Code could always write Lua files directly. The bottleneck was the feedback loop — Claude couldn't see what happened in the game, didn't know which APIs actually existed in TBC Classic, and couldn't run tests. I was the only bridge between the code and the runtime, and that bridge was slow and lossy.

**Reference material compounds.** That 973-line UI guide I wrote early on? It's still useful. The pattern library built from studying 32 addons? It makes every new feature faster because Claude starts from a proven approach instead of guessing. The investment in building good reference material pays off across every future session.

**You don't need to understand every line.** I'm not a Lua expert. I can read it, I understand the patterns, and I can describe what I want clearly. Claude handles the implementation details — the event registration order, the edge cases in connected-realm name normalization, the math in GearScore formulas. My job is knowing what the addon should do and testing that it actually does it.

**Tools matter more than talent.** The difference between my first week (one addon, no tests, constant API errors) and two weeks later (multiple addons, 131 unit tests across the suite) wasn't skill improvement. It was tooling. The MCP bridge, the TBC API database, the pattern library, the sandbox test runner — each one removed a friction point, and the compound effect was dramatic.

---

## The Numbers

For the curious, here's where things stand:

| Addon | Lines of Lua | Unit Tests |
|-------|-------------|------------|
| BoneyardTBC | 7,177 | 37 |
| BoneyardTBC_DungeonOptimizer | 7,443 | 94 |
| BoneyardTBC_Officer | 10,132 | — |
| CommanderText | 1,649 | — |
| DungeonFAQ | 1,084 | — |
| DisenchantValue | 559 | — |
| **Total** | **28,044** | **131** |

Plus 782 lines of architecture documentation, three phase plan documents, a UI reference guide, and a pattern library that I keep adding to.

All built with Claude, starting from zero Lua knowledge, in about two weeks of evening sessions.
