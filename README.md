# Subspace Brawl

A space battle your chat plays. You run the game on your PC and add it to OBS as a browser source; your viewers type
`!join` in Twitch (or YouTube) chat and their ship flies into the fight on your stream. Ships fly and fight on their
own: chat decides when to fire missiles, raise shields, ram, drop mines or retreat, and cheers and channel point
redeems shake it up.

Website: https://subspacebrawl.com

## Download

**[Download for Windows](https://subspacebrawl.com/download)** (the latest installer, `SubspaceBrawl-Setup.exe`)

- Windows only for now. The installer isn't code-signed yet, so Windows SmartScreen may warn before it runs
  ("More info" → "Run anyway").
- Every release here is copied from the build server, and the website only hands out an installer whose SHA-256
  matches the build it verified. Each release lists its SHA-256; to check yours in PowerShell:
  `Get-FileHash SubspaceBrawl-Setup.exe`.
- Pre-Steam releases are `0.x`; `1.0.0` will be the Steam launch.

## Quick start

1. Install and open Subspace Brawl.
2. **Connect Twitch**: in the game's Settings, press CONNECT, then click Authorize on twitch.tv/activate. That's it: no
   developer keys, no redirect URLs, no port forwarding. YouTube live chat can join the same battle (Settings →
   YOUTUBE).
3. **Add the overlay to OBS**: a Browser source with `http://localhost:2087/` at 1920×1080. For a vertical stream add
   `http://localhost:2087/?vertical` at 1080×1312 (any size works: `?w=…&h=…`). Settings → VIEWS lists these URLs.
   Every source shows the same screen, and only the game's own window plays sound.
4. Pick a mode, open the lobby and press START. **Viewers type `!join`** in chat.

No viewers yet? Fill the lobby with bots (four skill levels), or add friends' Twitch accounts as named bots.

## Chat commands

| Command | Does |
|---|---|
| `!join` / `!play` | Join the round (viewers join again each round) |
| `!join red` / `blue` / `green` / `yellow` | Team modes: pick a side while it has room |
| `!missile` | A heat-seeking missile; `!missile 3` (or `!missiles 3`) fires a volley of what you have charged |
| `!shields` | A shield ring around your ship for a few seconds |
| `!ram` | Burn hard with a half shield over the nose: head-on rams don't hurt you |
| `!retreat` | Pull back home; in team modes your base heals you inside its ring |
| `!mine` | Drop a mine that drifts, arms, then hunts the nearest enemy; `!mines 3` drops several |
| `!space color <hex \| name \| rgb \| reset>` | Your name color (saved) |

Cheers: every 10 bits is an orbital strike on a random ship (never yours). Cheer with `!bounty @name` to put bounties
on pilots instead. Channel point rewards can be linked to effects in the game's menu: Nuke, calamities, Mine Supply,
EMP, Grow, Bounty, Missile Supply, and cosmetics (trails, shield skins, death explosions, laser skins).

## Modes

- **Factions**: team deathmatch. Win by time (most kills) or by stock (Fleet Reserves: each team shares a pool of ships).
- **Free-for-all**: every ship for itself. Win by time (most kills) or by stock (everyone has their own lives).
- **Boss**: every pilot against a giant boss, sharing one pool of lives.

## Fields

- **Deep Space**: the classic sky, rocks, a quarter of them ice.
- **Over Earth**: satellites drift by and break apart; with hard bots some carry turrets.
- **Lunar Surface**: more and bigger rocks, far more meteor showers.
- **Over Mars**: red rocks and Dust Storms.
- **Over Jupiter**: radiation fields keep opening.
- **Saturn's Rings**: mostly ice, ice showers more often.
- **Near the Sun**: Solar Flares sweep fire across the fight.

## API

A REST API, a live event stream and an MCP server run on the game itself (`http://localhost:2087`), for bots, stream
tools and AI assistants: **[API docs on the website](https://subspacebrawl.com/api-docs)** (a copy is in
[docs/api.md](docs/api.md)).

## Help and bug reports

Open an issue in this repo's [Issues](../../issues): what happened, what you expected, the game version (the release
tag), and your OBS / overlay setup if it's about the overlay.

## Credits

Development and design: Shibiko. Special effects: Redgarr, Shibiko. Assets: Shibiko, Kenney (kenney.nl).
