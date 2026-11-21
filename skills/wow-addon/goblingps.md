# GoblinGPS: the engine of step-by-step guides

A guide addon only says what its steps are, from the player's own progress
(quest log, bags, spells, skills). GoblinGPS draws them: one panel with an icon
tab per guide, and one arrow for the guide the player follows (the one clicked
last, or the one that just moved on a step), with distance, time left and,
across continents, the nearest zeppelin or boat. Alone it shows nothing.

Find it with `wowaddons search GoblinGPS --json` and install it with
`wowaddons install <its id> --json`; then its source is in
`<AddOns>/GoblinGPS/GoblinGPS.lua`, the API in the comment at its top.

## The API

```lua
GoblinGPS.register({
  id = "cozy",                          -- stable: GoblinGPS saves by it
  icon = fileID or path,                -- the tab's icon
  title = "Cozy Sleeping Bag",          -- already in the player's language
  slash = { "/cozy", "/sac" },          -- optional: shows or hides this guide
  events = { "QUEST_TURNED_IN", ... },  -- progress may have changed: draw again
  on_event = function(event, ...) end,  -- optional, called before drawing (PLAYER_LOGIN included)
  state = function() return {           -- return nil: this guide does not apply to this character
    progress = "2/7",
    intro = "From level 14 ...",        -- optional, above the steps
    rows = { { text = "1. ...", state = "done" | "now" | "later", extra = " |cff..24|r" }, ... },
    lines = { { text = "...", kind = "detail" | "note" | "tip" }, ... }, -- under the current step
    place = { map = uiMapID, x = 0-100, y = 0-100, label = "...", near = 10, here = "..." } or nil,
    key = "2:37.41",                    -- changes when the step does
    finished = false, finished_text = "...",
  } end,
})
```

- `place.map` is a uiMapID, `x` and `y` in percent (0 to 100), as the world map shows them.
- `key` must change when the step changes: GoblinGPS then moves the arrow and puts the waypoint on the map.
- `finished = true` with `finished_text`: GoblinGPS prints it once and stops pointing.
- `/gps` brings back every hidden guide.

`GoblinGPS.words(S, NAMES)` returns a guide's `T(key, ...)` and `name(key)`.
`S` holds its sentences per language (English as fallback), `NAMES` its proper
names per language (English as fallback). In a sentence, `{n:key}` is a name,
`{a:ID}` an area and `{z:ID}` a map, both named by the game itself in the
player's language. Format arguments go in first, so an argument can itself be
`{a:%d}` or `{n:%s}`.

`GoblinGPS.player()` gives the player's continent and world position (nil when
secret or unknown); `GoblinGPS.world(map, x, y)` the same for a map point (x and
y from 0 to 1). Use them to choose the nearest of several places.

## A minimal guide

Where to buy Mild Spices in the Barrens. Item 2678 and its names from Wowhead
Forever's tooltips (locale 0 and 2); the two vendors' places from QuestieDB
Forever spawns.

`MildSpices/MildSpices.toc`:

```
## Interface: 16001
## Title: Mild Spices
## Title-frFR: Epices douces
## Notes: Where to buy Mild Spices in the Barrens, with GoblinGPS's arrow.
## Notes-frFR: Où acheter des Epices douces aux Tarides, avec la flèche de GoblinGPS.
## Version: 1.0.0
## Dependencies: GoblinGPS
Locales.lua
MildSpices.lua
```

`MildSpices/Locales.lua`:

```lua
local _, ns = ...
local S = {
  enUS = { buy = "Buy {n:spices}", where = "From {n:%s}, in {a:%d}.", done = "you have them." },
  frFR = { buy = "Acheter des {n:spices}", where = "Chez {n:%s}, à {a:%d}.", done = "tu les as." },
}
local NAMES = {
  spices = { enUS = "Mild Spices", frFR = "Epices douces" },
  ranik = { enUS = "Ranik" },
  tarban = { enUS = "Tarban Hearthgrain" },
}
ns.T, ns.name = GoblinGPS.words(S, NAMES)
```

`MildSpices/MildSpices.lua`:

```lua
local _, ns = ...
local T = ns.T
local SPICES = 2678 -- Mild Spices (Wowhead Forever)
-- The vendor of each side (QuestieDB Forever spawns); 1413 is the Barrens' uiMapID, 392 Ratchet, 380 the Crossroads.
local VENDOR = UnitFactionGroup("player") == "Alliance"
  and { map = 1413, x = 61.93, y = 38.70, npc = "ranik", area = 392 }
  or { map = 1413, x = 55.16, y = 32.08, npc = "tarban", area = 380 }

GoblinGPS.register({
  id = "mildspices",
  icon = C_Item.GetItemIconByID(SPICES) or 134400,
  title = ns.name("spices"),
  slash = { "/spices" },
  events = { "BAG_UPDATE_DELAYED" },
  state = function()
    if (C_Item.GetItemCount(SPICES) or 0) > 0 then
      return { finished = true, finished_text = T("done"), key = "done" }
    end
    return {
      progress = "0/1",
      rows = { { text = "1. " .. T("buy"), state = "now" } },
      lines = { { text = T("where", VENDOR.npc, VENDOR.area), kind = "detail" } },
      place = { map = VENDOR.map, x = VENDOR.x, y = VENDOR.y, label = ns.name(VENDOR.npc) },
      key = "buy",
    }
  end,
})
```

Publish the guide with `wowaddons publish MildSpices --json` as any
addon. A guide that already draws its own arrow should move to GoblinGPS: two
arrows on screen point the player two ways.
