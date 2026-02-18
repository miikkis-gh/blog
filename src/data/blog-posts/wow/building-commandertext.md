---
title: Building CommanderText — One-Click Raid Callouts for TBC Classic
slug: building-commandertext
publishDate: 18 Feb 2026
description: How I built a quick-access message overlay for WoW TBC Classic raid leaders — from message pools to a hand-built settings panel.
---

# Building CommanderText — One-Click Raid Callouts for TBC Classic

I've been spamming TBC heroics as a warrior tank, and at the tempo these groups move, you don't have time to type. You're chain-pulling, tab-sundering, watching healer mana — and somewhere in between you need to tell the group to CC moon, hold DPS, or go all out. Stopping to type means slowing the group down.

So I built [**CommanderText**](https://www.curseforge.com/wow/addons/commandertext) — a panel of buttons that sends the right message to the right channel in one click.

![CommanderText overlay showing message buttons and DBM pull timers](/assets/blog/wow/commandertext-overlay.webp)

---

## The Idea

The concept is simple: a small overlay with message buttons that send text to party or raid chat. Left-click sends normally, right-click sends as a raid warning. The addon figures out which channel to use — raid if you're in a raid, party if you're in a party, say if you're solo.

On top of that, I wanted DBM pull timer integration. Instead of typing `/pull 10` I wanted a row of buttons: Pull (10s), Pull (15s). One click.

That's the whole addon. No data sync, no saved history, no complex state. Just buttons that send messages.

---

## Message Pools

A single static message per button would get old fast — your group would see the exact same string 10 times in a dungeon run. So from the start, each button has a label (what you see on the button) and a **pool of 5 variations** (what actually gets sent):

```lua
{
    label = "Hold DPS",
    pool = {
        "Hold DPS, building threat",
        "Wait for threat please",
        "Let me build threat first",
        "Hold DPS, need aggro",
        "Wait for Sunders",
    },
    order = 7
}
```

Every click picks a random variation from the pool. In r4 I added no-repeat logic — it tracks the last sent index on the message object so you never get the same phrase twice in a row. Small detail, but it makes the callouts feel like you're actually typing them. The only thing that gives it away is that you're moving and can't actually be typing at the same time.


---

## The Settings Panel

The settings panel ended up being the biggest piece of the addon. It covers:

- **Overlay**: width, background opacity, content opacity, lock position, frame layer
- **Minimap**: show/hide toggle, reset position
- **Pull timers**: checkboxes for 5s/10s/15s/20s/30s durations
- **Messages**: add new messages, reorder with up/down buttons, hide toggle, delete with confirmation

Everything scrolls, so it doesn't matter how many messages you add. Each message row has inline controls to move it up, move it down, hide it from the overlay without deleting, or remove it entirely.

---

## Smart Channel Detection

One thing I wanted to get right was making the buttons "just work" regardless of group state. Left-click auto-detects the right channel — raid chat if you're in a raid, party chat if you're in a party, say if you're solo. Right-click always attempts a raid warning, but gracefully falls back — if you're not in a raid it sends to party, if you're not leader or assistant it sends to regular raid chat.

The Ready Check button takes this further: it only appears when you're group leader or assistant. Promote someone else to leader? The button disappears instantly.

---

## Four Releases in One Day

CommanderText went from nothing to r4 in a single day:

- **1.0.0**: Core overlay with message buttons and pull timers
- **r2**: Full settings panel with sliders, message management, dynamic pull timer configuration
- **r3**: Minimap button improvements (middle-click hide, right-click settings), settings footer with CurseForge link
- **r4**: Two new default messages with funny variations, no-repeat randomization

Each release was a clean increment. The settings panel in r2 was the big one. The later releases were small quality-of-life improvements that came from actually using the addon while grinding dungeons.

---

## The Result

[**CommanderText**](https://www.curseforge.com/wow/addons/commandertext) is a small addon that does one thing well. It sits in the corner of my screen during dungeon runs and saves me from typing the same callouts over and over. The random message pools make it feel natural, and the settings panel gives enough control without overcomplicating things.

Sometimes the right addon is the one that replaces a keyboard with a button.
