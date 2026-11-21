# WoW Forever addon skill

An Agent Skill that teaches Claude Code, Codex, Cursor or any AI coding agent to make a World of
Warcraft Forever addon (Interface 16001, Lua 5.1), check it against Forever's own API, try it
outside the game, and publish it so players install it in one click on
[wowaddons.gg](https://wowaddons.gg).

```
npx skills add cyclop-studio/wow-forever-addon-skill
```

Or copy `skills/wow-addon` into your agent's skills folder (`~/.claude/skills/` for Claude Code).
Then tell your AI one sentence: "Make me an addon that ...".

- What it does: writes the addon, runs `wowaddons check` (missing or deprecated calls, needless
  globals, every-frame OnUpdate) and `wowaddons try` (a real Lua 5.1 against a stand-in of the
  client), reads your Lua errors after a /reload, publishes with screenshots.
- The Forever API it checks against: [wowaddons.gg/forever-api](https://wowaddons.gg/forever-api),
  6,623 functions and 1,816 events, each as markdown for your AI.
- Publish without the skill: [wowaddons.gg/publish](https://wowaddons.gg/publish).

Free, for creators and players. The copy served at wowaddons.gg/skills/wow-addon is the reference.
