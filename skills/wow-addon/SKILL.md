---
name: wow-addon
description: Create, check and publish a World of Warcraft Forever addon (Interface 16001, Lua 5.1) with the WowAddons command line, including step-by-step guides that run on GoblinGPS. Use when the user asks for a WoW Forever addon, a fix to one, a guide with an arrow, translations of an addon, or to publish an addon on wowaddons.gg.
---

# World of Warcraft Forever addon

You make an addon with a player and put it on wowaddons.gg. The method is nine
steps, in order. Do not skip one because the addon looks small.

The `wowaddons` command line, how to install it, the AddOns folder and the
account link are in https://wowaddons.gg/agents.md: read it first. Every verb
takes `--json`.

## 1. Before any code

Write three lines and keep them in front of you:

- What the addon does for the player, in one sentence.
- The first value moment: what the player sees in the first minute, with no data saved yet.
- The game functions it needs, each checked to exist in Forever.

Forever is not Classic. Its client is the modern one: `## Interface: 16001`,
Lua 5.1, a retail-style API (`C_Map`, `C_QuestLog`, `C_Item`, `GetPlayerFacing`).
Before you call a function or register an event, read its page:
https://wowaddons.gg/forever-api/C_Spell.md (any namespace or name) has every
argument and return as Blizzard documents Forever, and
https://wowaddons.gg/forever-api?q=NAME says whether a name is in the game,
deprecated (with what to write instead) or absent. Blizzard's source is
https://github.com/Gethe/wow-ui-source/tree/forever. Prefer the `C_`
namespaces. Some values can be secret (positions in instances, some fights):
guard them before any arithmetic.

```lua
local function open(v) return v ~= nil and not (issecretvalue and issecretvalue(v)) end
```

Nothing protected in combat: check `InCombatLockdown()` before you change a secure frame.

## 2. Data: never invented

Never write a coordinate, an ID, a drop rate or a level from memory. Take each
from a source and write the source in a comment beside it.

- Wowhead Forever tooltips, one JSON per thing and language:
  `https://nether.wowhead.com/forever/tooltip/{item|quest|npc|object}/ID?locale=N`
  (0 enUS, 1 koKR, 2 frFR, 3 deDE, 4 zhCN, 6 esES, 7 ruRU, 8 ptBR, 10 zhTW, 11 esMX; no Italian).
  It gives the name in each language, the icon and the tooltip text.
- QuestieDB (https://github.com/Questie/QuestieDB), `data/Forever/` and
  `src/corrections/Forever/`: spawns of NPCs and objects, quest givers and
  enders, quest levels. Its spawns are keyed by zone (area) ID; `C_Map` and
  GoblinGPS take a uiMapID: convert with its `support/Forever/Zones/areaIdToUiMapId.lua`.
- The game client itself, at runtime: `C_Map.GetAreaInfo(areaID)` and
  `C_Map.GetMapInfo(uiMapID).name` for place names, `C_Item.GetItemNameByID(id)`
  for an item's name (it can be nil until the item is cached: fall back, draw again later).

A number that is not Forever's own (a drop rate from classic tables, a time
measured once) is an estimate: the addon says so to the player, in words.

## 3. Languages

The game has 11 locales: enUS, frFR, deDE, esES, esMX, ptBR, itIT, ruRU, koKR, zhCN, zhTW
(`GetLocale()`; treat enGB as enUS). One `Locales.lua`, loaded first, with a
table per language and English as the fallback for a missing key or language:

```lua
local _, ns = ...
local L = GetLocale() == "enGB" and "enUS" or GetLocale()
local S = {
  enUS = { hello = "%d quests left in %s" },
  frFR = { hello = "%d quêtes restantes à %s" },
}
S.esMX = S.esMX or S.esES
function ns.T(key, ...)
  local s = (S[L] and S[L][key]) or S.enUS[key] or key
  return select("#", ...) > 0 and s:format(...) or s
end
```

- Place names come from the client (`C_Map.GetAreaInfo`), never hard-coded per language.
- Other names (NPCs, objects, items) come from Wowhead's tooltip per locale; Italian falls back to English.
- Lua 5.1's `string.format` has no positional `%1$s`. When a language needs the
  arguments in another order, rewrite that sentence so the arguments come in the
  order the code passes them, or pass one sentence per order.

## 4. Build

One folder, `<AddOns>/<Name>/`, `Name` in letters and digits, with `<Name>.toc`:

```
## Interface: 16001
## Title: Pull Timer
## Notes: One sentence that says what it does.
## Notes-frFR: La même phrase en français.
## Version: 1.0.0
## IconTexture: Interface\Icons\INV_Misc_PocketWatch_01
## SavedVariables: PullTimerDB
## SavedVariablesPerCharacter: PullTimerChar
## Dependencies: GoblinGPS
Locales.lua
Core.lua
```

- `Title-xxXX` and `Notes-xxXX` give the other languages; `Dependencies` only when it truly needs another addon.
- Saved variables exist only after `ADDON_LOADED` or `PLAYER_LOGIN`: initialise them there (`PullTimerDB = PullTimerDB or {}`).
- Everything `local`; the files share the table `local name, ns = ...` gives each. Only the saved variables, the `SLASH_` names and a TOC's `AddonCompartmentFunc` are global.
- `Author` is the player's own creator name, or leave it out.
- No em dash and no en dash in any text the player reads: a comma, a colon or parentheses.
- A `README.md` in the folder becomes the addon's page.

## 5. Checks, every change

1. `luac -p` on every Lua file (`luac5.1 -p` on some systems): syntax.
2. `wowaddons check --json`: every TOC against the game, and every function, `C_`
   function and event the Lua uses against what Forever has. Fix each item of `problems`:
   a missing name comes with the `C_` call to write instead and its page (read it, the
   returns often differ). The `!` lines too: a deprecated call (with what to write
   instead), a needless global and an `OnUpdate` that works every frame without
   reading its elapsed time are what addon forums remove (r/wowaddons audits for them).
3. `wowaddons try <Name> --json`: the addon run once outside the game, in a real Lua
   5.1 against a stand-in of Forever (its files in order, the addons it needs first,
   `ADDON_LOADED`, `PLAYER_LOGIN`, ten seconds of timers and `OnUpdate`, each slash
   command typed bare). Each of `errors` has `file`, `line`, `message`, `when`, `stack`
   and `sure` (the game would say the same; false: the stand-in may be what differs,
   read the line first). `printed` is what the chat would show. For logic that depends
   on the game's state (no data, half way, done, each faction), add your own stand-ins
   of the functions it calls in a Lua 5.1 mock.

4. In the game: run `wowaddons errors --json` once before the player starts the
   game (it installs a small witness addon that notes Lua errors). A new addon
   folder loads only when the game starts: ask for a full restart. After that,
   changed files need only `/reload`. After each `/reload`,
   `wowaddons errors <Name> --json` gives the errors the game saved; check
   `saved_seconds_ago` matches the reload just made. Go on until it is empty and
   the player says it works.

## 6. Roast before shipping

Read the addon as a hostile reviewer, and fix what you find:

- Wrong data: each number traced to its source? An estimate said as one?
- Wrong game version: a Classic-only or removed function, a retail feature Forever lacks?
- Unclear text: would a player who never read the code understand each line, in each language?
- Overlap: does a frame sit on top of the minimap, the quest tracker, the action
  bars or another popular addon's frame? Can the player move it and hide it?

## 7. Screenshot

A real in-game capture of the addon working: the player presses Print Screen
(or `/run Screenshot()`), then `wowaddons shot <Name> --json` copies the newest
capture into `<Name>/screenshots/` (the first file, `01.png`, is the cover).
Check `seconds_ago`. Never a mockup, never an edited image: the product as it is.
Tell the player their chat and character name may show.

## 8. Publish

1. `wowaddons whoami --json`; if `not_linked`, link the app to the player's
   account first (`wowaddons link start --json`, then `wowaddons link wait`, as agents.md says).
2. Dependencies first: an addon whose `## Dependencies` names another addon the
   player made is refused until that one is published.
3. `wowaddons publish <Name> --json` gives the addon's page `url`: give it to the player.
4. A change: raise `## Version`, publish again.
5. Shared on r/wowaddons? It takes CurseForge, Wago, WoWInterface or GitHub links
   only: with the player's yes, a GitHub release of the very archive wowaddons.gg
   serves, or `wowaddons mirror` on to their CurseForge and Wago projects (agents.md,
   its step 7). The post itself is the player's own words, with an `AI Disclaimer:` line.

Players who install an addon get its dependencies installed with it when the
same creator published them; otherwise its page links to what it needs.

## 9. Guides: on GoblinGPS

A step-by-step guide (a chain of quests, a recipe to farm, a place to reach)
does not draw its own arrow or panel. It declares `## Dependencies: GoblinGPS`
and calls `GoblinGPS.register({...})`: GoblinGPS draws every guide in one
panel, one tab each, and one arrow for the guide followed, with distance, time
left and the boat or zeppelin across continents. The full API and a minimal
guide: [goblingps.md](goblingps.md).
