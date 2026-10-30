# Subspace Brawl API

Everything the streamer menu can do, a bot or plugin can do too: through a **REST API**, a live **event stream**
(SSE), or an **MCP server** that an LLM can use as tools. All three call the same game actions.

Base URL: the game server, `http://localhost:2087` on the streamer's PC (port from `PORT`).

The browser version (play.subspacebrawl.com) has no API: no REST, event stream or MCP server (a browser tab can't
serve one). Use the desktop app for bots and plugins.

## Versions

The API is versioned on its own, separately from the game (game `0.9.x` until the Steam launch = `1.0.0`):
- **API version** (now **1.4.0**): `GET /api/v1/version` → `{ "game": "0.9.0", "api": "1.4.0", "devTools": false }`
  (`devTools`: a dev build, see "Dev builds only" below); the MCP server reports
  it too. **Major** = the `/api/vN` path: a breaking change gets `/api/v2` (v1 keeps working for a while).
  **Minor** = something new (a route, a field, an MCP tool, an event). **Patch** = a fix or a docs correction.
- Everything below documents the current version; each change gets a line in the changelog.

### Changelog
- **1.5.0**: view settings: `POST /camera/mirror` (the host's camera + mouse for mirrored views), `round.overlay.mirror`
  and `round.overlay.coViews` (a co-streamer's music / volume / mirror).
- **1.4.0**: streamers' cameras: `POST /camera/co-pilot`, `round.overlay.cams`; `/camera/follow` takes `boss:<pilot>`.
- **1.3.0**: streamers fly bosses: settings `bossPilots` (`ai` / `streamers`) and `coPilots`, `POST /bosses/pilot`.
- **1.2.0**: content packs: `state.packs`, `pack` / `packName` / `owned` on `GET /bosses` designs, `GET /fields` + MCP
  `list_fields`; a boss / field from a pack the channel doesn't own is refused (400: "Phantom is in Boss Pack 2");
  `boss.warning` `pack` and boss status `teaser`; `devTools` in `/version`. Dev builds only (404 / no tool in a
  release): `POST /effects`, `/trails/test`, `/cosmetics/test`, MCP `apply_effect`, `test_cosmetics`, `list_effects`.
- **1.1.0**: `GET /version`; ability `ram` (old name `boost` still accepted) and `mine`; fields (`POST /field`, MCP
  `set_field`); scripts on game events; `PATCH /bots/:login`; boss damage board; seasons; laser skins.
- **1.0.0**: first documented API: game, menus, pilots and effects, overlay, redeems, scripts, events, MCP.

## Auth

1. Open the streamer menu (`/admin`), connect Twitch, then **API & MCP → Make new API key**.
2. Copy the key (`sbk_…`). It's shown once; only its hash is stored. Making a new key replaces the old one.
3. Send it on every request: `Authorization: Bearer sbk_…`

Wrong or missing key → `401`. Bad input → `400 { "error": "…" }`. Anything else failing → `500 { "error": "…" }`.

Keep the key secret: it can change every setting and run the show.

## REST API (`/api/v1`)

Request and response bodies are JSON.

**Content packs.** Bosses and fields come in packs: the free base game (Leviathan, Crimson Manta, Iron Bastion,
Lumen Spire; Deep Space, Over Earth, Lunar Surface, Ceres, Over Pluto), **Boss Pack 1** (`bossPack1`: Hive Mother, Hydra, Widow,
Magnetar), **Boss Pack 2** (`bossPack2`: Singularity, Phantom, Seraph, Maelstrom), **Inner Sol System** (`arenaPack`: Near
the Sun, Over Mercury, Over Venus, Over Mars), **Gas Giants** (`arenaPack2`: Over Jupiter, Saturn's Rings, Over
Uranus, Over Neptune), **Moons of Jupiter** (`moonsPack`: Io, Europa, Ganymede, Callisto). Picking a boss or field from a pack the channel doesn't own
(`PATCH /settings`, `POST /field`, `POST /bosses`, MCP) is a 400 naming the pack; `random` only picks owned ones
(a random encounter or random BOSS-mode boss is now and then a *teaser* from an unowned pack: it fights normally and
says its pack).

**Dev builds only** (marked ⚙): routes that hand out effects or cosmetics without a redeem. A release build doesn't
contain them: they answer 404 (MCP: the tool isn't listed). `GET /version` says `devTools`.

### Game

| Method | Path | Body | Does |
|---|---|---|---|
| GET | `/state` | | Round, every pilot, settings, content packs (below) |
| GET | `/leaderboards` | | `{ day, week, month }`: top pilots `{ username, points }` |
| GET | `/settings` | | Every setting |
| PATCH | `/settings` | any settings keys | Changes only the keys given; returns all settings |
| POST | `/round/start` | | Skip the lobby / restart the round now |
| POST | `/lobby/countdown` | | Lobby START: the countdown begins (a fresh lobby waits for it) |
| GET | `/twitch/users?logins=a,b` | | Twitch accounts by name or link: `{users: [{id, login, name, avatar}], missing}` |
| POST | `/bots/join` | `{logins}` | Roster bots fly in this lobby / round without auto-join (gone when it ends) |
| PATCH | `/bots/:login` | `{name?, color?, image?}` (`""` color / image = auto / none) | Edit a roster bot (the login stays; auto-join and its lobby spot are kept; a flying bot changes at once) → `{bot}`; unknown login or bad color / image = 400 |
| PUT | `/bots/:login/image` | `{data}` (data: URL, ≤ 2 MB) | A named bot's picture → `{path}` for its `image` |
| POST | `/results/board` | `{board}` (`kills` / `out` / `damage`) | The end screen's board: kills, knockout order (stock battles' default) or damage to the bosses (BOSS mode's default) |
| POST | `/killlist/board` | `{board}` (`lives` / `kills`) | Free-for-all stock: the in-game kill list shows lives left (default) or kills |
| POST | `/trails/test` | | ⚙ Every flying pilot gets a random special trail for 5 minutes (testing) |
| POST | `/cosmetics/test` | `{ "kind": "laser" }` (optional) | ⚙ Every flying pilot gets a random trail, shield skin, death explosion and laser skin for 5 minutes (testing); `kind` (`trail` / `shield` / `explosion` / `laser`) = just that one |
| POST | `/camera/follow` | `{id}` (null = stop) | Every view's camera follows this pilot (`round.overlay.follow`); `boss:<pilot>` = a streamer's boss (`host` or a co-streamer's name) |
| POST | `/camera/co-pilot` | `{name, cam}` | A co-streamer's camera (their pilot tab and their `?for=<name>` OBS link): `auto`, `host` (the host's camera) or `boss:<pilot>`; null = their own boss (`round.overlay.cams`) |
| POST | `/camera/mirror` | `{x, y, w, h, px, py}` | The host window's camera (middle `x, y`, world `w × h` shown) and mouse (`px, py` 0–1 of its screen, null = off it), sent ~10× a second; views with mirror on (`?mirror=on`) copy it (`round.overlay.mirror`, null after 3 s unheard) |
| POST | `/music/beat` | `{bpm, beatAt}` (`{}` = no music) | The host window's music beat (`beatAt` = wall-clock ms of a beat); every view's notes shield pulses to it (`round.overlay.music`, null after 75 s unrefreshed) |
| POST | `/results/close` | | After a round: close the results board (it stays up until this) and show the lobby |
| POST | `/calamities` | `{ "type": "random" }` | `random`, `meteorShower`, `iceStorm` (Ice Shower), `thunderStorm`, `blackHole`, `radiationLeak`, `blizzard` (a Dust Storm over Mars, an Acid Storm over Venus, Supersonic Winds over Neptune), `solarFlare` (random picks it only Near the Sun) |
| POST | `/field` | `{ "field": "earth" }` | `random` or a field (`deepSpace`, `earth`, `moon`, `mercury`, `venus`, `mars`, `ceres`, `jupiter`, `saturn`, `uranus`, `neptune`, `pluto`, `sun`, `io`, `europa`, `ganymede`, `callisto`): saved as the `field` setting, and mid-round the field changes right away (the rocks are swapped for the new field's) → `{ field, setting }`; a field from a pack the channel doesn't own = 400 |
| GET | `/fields` | | `{ field, setting, fields: [{ id, name, pack, packName, owned }] }` |
| POST | `/nukes` | `{ "by": "name" }` | A nuke near the fight, never on a team base: 3 s warning, then it vaporizes everything within 450 (`by`, optional: the pilot who gets its kills) |
| GET | `/bosses` | | The bosses in the fight (`name`, `role`, `phase`: `warning` / `entering` / `fighting` / `leaving` / `down`, `hp`: `Good` / `Fair` / `Low` only, `shield`: `up` / `down`, `rage` ×1–6, `livesLeft` (Fleet Reserves encounters), `leavesIn` s (Factions encounters), `teaser`: a pack's name when it's a random pick from a pack the channel doesn't own) and every boss design (`name`, `role`, `entrance`, `pack`, `packName`, `owned`) |
| POST | `/bosses` | `{ "boss": "Hydra" }` | Call a boss into the round now (a design name or `random` = an owned one): WARNING, its entrance, then it fights until shot down; at most 3 at once; a boss from a pack the channel doesn't own = 400 |
| POST | `/bosses/pilot` | `{ "move": [1, 0], "aim": { "x": 900, "y": 400 }, "fire": true, "skill": 0 }` | The streamer's boss controls (BOSS mode, setting `bossPilots: "streamers"`): move x / y −1…1, aim = a world spot (turrets fire there while `fire`; abilities go after the pilot nearest it), `skill` = an ability slot to fire (on its cooldown). Send it while flying; 10 s without = the AI flies it |
| POST | `/strikes` | `{ "count": 3, "by": "name" }` | Orbital strikes on random ships (like a cheer) |
| POST | `/bounties` | `{ "target": "name", "stacks": 1, "by": "name" }` | Bounty on a pilot (stacks add up) |

### Menus

The game starts on its title screen. Menu pages show on every overlay (desktop and vertical) at once.

| Method | Path | Body | Does |
|---|---|---|---|
| POST | `/menu` | | Back to the title screen (a round in progress stops; pilots stay joined) |
| POST | `/lobby` | `{ "mode": "ffa" }` | From the title screen: open that mode's lobby. In a lobby or round: swap the next mode, pilots stay |
| POST | `/menu/page` | `{ "page": "boards" }` | Show a screen: `main` (title), `modes`, `boards`, `credits`; `null` closes it |

`round.phase` is `menu`, `lobby` or `playing`; `round.overlay.menu` is the open page (or null).

### Pilots and effects

| Method | Path | Body | Does |
|---|---|---|---|
| POST | `/pilots` | `{ "username": "twitchlogin" }` | Put a viewer in the game (as `!join`) |
| POST | `/pilots/:name/abilities` | `{ "ability": "missile" }` | `missile`, `shields`, `ram` (old name `boost` still works), `retreat`, `mine` (`mines` works too; drops one if charged) |
| GET | `/effects` | | Every effect: `[{ id, label, needsPilot }]` |
| POST | `/effects` | `{ "effect": "grow", "pilot": "name", "input": "", "by": "name" }` | ⚙ Apply an effect, as if redeemed |

Effect ids: `calamity:random`, `calamity:meteorShower`, `calamity:iceStorm` (Ice Shower; the id kept its old name),
`calamity:thunderStorm`, `calamity:blackHole`, `calamity:radiationLeak`, `calamity:blizzard`, `nuke`, `mines` (mine supply: +3 mine charges, over the max), `emp`, `grow`, `bounty` (input = target name), `trail` (input = style: rainbow, fire, neon, sparkler, galaxy, stars, hearts, confetti, snow, bubbles, notes,
smoke, pixels; Halloween: pumpkins, ghosts, bats, toxic, scythe,
spider; Christmas: bells, candycanes, gingerbread,
ornaments; usb; `halloween` / `christmas` = a random one of those), `shield` (shield skin,
input = the same themes), `explosion` (death explosion, input = the same themes; the pilot's missiles and mines blow up
in it too), `laser` (laser skin for the pilot's shots, input = rainbow, fire, plasma, hearts, carrots, snowballs,
candycanes, pumpkins, bubbles, notes, stars, pixels), `missiles`. Unknown input (an emoji or emote too) = a
random one; each lasts `trailMinutes`. `missiles` = +3 missile charges, over the max.
Effects with `needsPilot` need a `pilot` who is flying right now.

### Overlay

| Method | Path | Body | Does |
|---|---|---|---|
| POST | `/overlay/announce` | `{ "text": "GG!", "color": "#35e0ff", "seconds": 6 }` | Banner on the overlay (text ≤ 120, seconds 1–120) |
| POST | `/overlay/screen` | `{ "screen": "leaderboard", "seconds": 15 }` | Show a screen over the fight; `"game"` goes back |

Screens: `game`, `leaderboard` (today / week / month boards).

### Channel point redeems

| Method | Path | Body | Does |
|---|---|---|---|
| GET | `/redeems/rewards` | | The channel's rewards `[{ id, title, cost, needsInput, enabled }]` |
| GET | `/redeems/links` | | `{ rewardId: effectId }` |
| PUT | `/redeems/links/:rewardId` | `{ "effect": "emp" }` | Link a reward to an effect |
| DELETE | `/redeems/links/:rewardId` | | Unlink it |

Unlinked rewards still work by keywords in their title ("Spawn Calamity", "Drop Mines", "EMP", "Shield Skin", "Laser Skin"…).
The streamer menu has the same thing as a searchable list.

### Scripts (owner only)

The streamer's own scripts (the game's SCRIPTS page): JavaScript, TypeScript, Python or Lua, run when a channel point
reward is redeemed, a chat command is typed or a game event happens (round end, kill, boss defeated…). Only the API key or the streamer's own session reaches these; the
extension's token gets `403`. Secret values go in and never come back out of any route.

| Method | Path | Body | Does |
|---|---|---|---|
| GET | `/scripts` | | `{ scripts, runtimes: { js, ts, python, lua }, secrets: [names], events: [{ name, description, fields }] }` |
| POST | `/scripts` | `{ name, language, code, trigger, enabled }` | Add a script → the saved script (with its `id`) |
| PUT | `/scripts/:id` | same | Replace it |
| DELETE | `/scripts/:id` | | Delete it |
| POST | `/scripts/test` | `{ id }` or `{ language, code }`, `trigger?`, `event?` | Run it now with a sample event for its trigger (a game event: that event's sample; `event` overrides fields) → `{ ok, ms, output: [{ kind, text }] }` |
| GET | `/scripts/runs?after=ms` | | The latest runs (redeems, commands, game events, tests), the same shape plus `name`, `at`, `trigger` (`reward`, `command`, the event name or `test`) |
| GET | `/scripts/secrets` | | `{ secrets: [names] }` (never the values) |
| PUT | `/scripts/secrets` | `{ "OBS_PASSWORD": "…" }` | Set secrets (empty values are skipped) |
| DELETE | `/scripts/secrets/:name` | | Delete one |

`language`: `js`, `ts`, `python`, `lua`. `trigger`: `{ "type": "command", "command": "hydrate" }` (for `!hydrate`),
`{ "type": "reward", "rewardId": "…", "rewardTitle": "…" }`, `{ "type": "event", "event": "round.end" }` (any name in
the events table below except `relay` / `extension`; `{ "type": "roundEnd" }` / `{ "type": "roundStart" }` are short for
`round.end` / `round.start`) or `{ "type": "none" }` (test runs only).

A script's `event`: reward / command runs get `{ type, trigger, user, displayName, userId, platform, reward, command,
input, bits }`. Game-event runs get that event's fields (as in the events table) plus `type` (the event name),
`trigger` (`event`, or `test`) and `at`, except these two, reshaped for scripts:

- `round.end`: `{ mode, modeName, result, winnerTeam ("Red"… or null), winners: [row], players: [row] (results order,
  winners first), bitLeaders: [{ name, bits }], startedAt, endedAt, durationSeconds }`; row = `{ place, name, login
  (null: bot), bot, team (name or null), won, kills, deaths, assists, friendlyKills, damage, lastShots }`.
- `round.start`: `{ mode, modeName, pilots (count), teams: [names], roster: [{ name, login, bot, team }], field,
  fieldName }`.

Busy events (`kill` can fire many times a second) don't pile up: a script that already has a run waiting skips the
newer event; at most 3 scripts run at once and 30 wait (more are dropped with a console warning).
Output `kind`: `log`, `error` or `say`. Each run is its own sandboxed process (10 s, 256 MB); see the SCRIPTS
section of the game's docs.

### Twitch connect (owner only, like scripts)

| Method | Path | Body | Does |
|---|---|---|---|
| POST | `/twitch/connect` | | `{ url, code, interval }`: type `code` at `url` (twitch.tv/activate) and click Authorize |
| POST | `/twitch/connect/poll` | | `{ status: pending \| done \| denied \| expired, error? }`; poll every `interval` s until done |

### State shape

```json
{
  "round": {
    "mode": "dogfight", "phase": "playing", "remaining": 412000,
    "scores": [12, 9], "reserves": [0, 0], "fleetSize": 30,
    "result": null, "lobby": null, "calamities": [], "bounties": [], "season": "halloween", "field": "earth",
    "stats": null, "overlay": { "announcement": null, "screen": "game" },
    "bosses": [{ "name": "Leviathan", "phase": "fighting", "hp": "Good", "shield": "up", "rage": 1 }]
  },
  "pilots": [
    {
      "id": "chat:bob", "name": "bob", "team": 0, "bot": false, "alive": true,
      "hp": 100, "kills": 3, "deaths": 1, "assists": 2, "damage": 0, "lives": null, "missiles": 1, "mines": 1, "ship": "…"
    }
  ],
  "settings": { "mode": "dogfight", "…": "…" },
  "packs": [{ "id": "bossPack2", "name": "Boss Pack 2", "free": false, "owned": false, "bosses": ["…"], "fields": [] }]
}
```

Modes: `dogfight` (Factions), `reserves` (Fleet Reserves), `ffa`, `ffaReserves`, `boss` (every pilot against a boss,
shared lives). A saved `supernova` (removed) loads as `boss`. A boss's HP is never a number anywhere: `Good` / `Fair` /
`Low` (the thirds). A pilot's `damage` (and `dmg` in round stats) = what their hits on bosses and boss craft were
worth this round (raw hits, shields and all, nothing for a raised dome): pilots' numbers, never the boss's HP.
Calamities run in `boss` rounds too (bosses aren't pulled, pushed or hurt by them).
`phase` is `lobby` between rounds, `playing` during one. `field` is where the round is fought (Fields.ts id; in a lobby the next round's, already rolled
when the setting is `random`; also `lobby.field`).
A pilot's `ship` is its ship's key : `spaceShips_001.png`…`spaceShips_008.png` for
the 8 originals, a name like `lancer` or `valkyrie` for the newer ones.

### Settings

| Key | Values |
|---|---|
| `mode` | see Modes |
| `teamCount` | `0` (auto), `2`, `3`, `4` |
| `teamPick` | boolean, default off: `!join red\|blue\|green\|yellow` picks a team (fair-share cap) and the lobby shows the teams (`round.lobby.teams`: each team's pilot ids); off = the color is ignored, teams are balanced |
| `roundMinutes` | 1–60 |
| `fleetSize` | 1–999 (Fleet Reserves) |
| `ffaLives` | 1–20 (FFA Reserves) |
| `boss` | `random` or a boss name (BOSS mode's boss) |
| `bossLives` | 1–20 (BOSS mode: shared lives per pilot, default 5) |
| `bossMinutes` | `0` (∞), 3, 4, 5, 6, 8, 10, 15, 20 (BOSS mode's clock, default 6) |
| `bots` | 0–500 (filler bots: Bot 1, Bot 2…) |
| `windowMode` | `borderless` (default) / `fullscreen` / `windowed`: the desktop app's window |
| `soundMaster`, `musicVolume`, `lobbyMusicVolume` | 0–100 (host window: everything, battle music, menu & lobby music; defaults 100, 20, 20) |
| `seasonMusic` | `on` (default: during a season only its songs play, `music/seasons/<id>/` and its `lobby/`), `off` (the regular songs), `rotate` (both shuffled together) |
| `soundMuted` | boolean (host window: the mute; volumes are kept) |
| `muteWhenMinimized`, `muteWhenUnfocused` | boolean (host window goes silent while minimized / hidden, default on; while another window has the focus, default off) |
| `sfxLasers`, `sfxMissiles`, `sfxHits`, `sfxExplosions`, `sfxCrashing`, `sfxThunder`, `sfxJoins`, `sfxMenu` | 0–100 (sound effects by kind) |
| `namedBots` | `[{login, name, twitch, userId?, image?, color?, autoJoin}]`, up to 500, unique logins (the BOTS page roster) |
| `lobbySeconds` | 15–600 (lobby countdown, default 60) |
| `botSkill` | `easy`, `normal`, `hard`, `veryHard` (bots: +0 / +1 / +2 / +3 missiles, very hard also shields when hit; harder = more natural calamities) |
| `streamerJoins` | boolean (the streamer's ship joins every lobby, default on) |
| `camera` | `auto`, `map`, `zoom` |
| `cameraZoom` | 0.1–2 |
| `scoreboard` | `left`, `right` |
| `language` | `""` (default: the system's language) or a language code from `panels/src/i18n/languages.ts` (`es`, `ja`, `he`...): the game's text on the host and every view |
| `pieCorner` | `top-left`, `top-right`, `bottom-left`, `bottom-right` |
| `pieSize` | 80–1000 |
| `pieOffsetX`, `pieOffsetY` | -2000–2000 |
| `calamities` | boolean (natural calamities) |
| `season` | `auto` (default: by the server's date, Halloween Sep 24 – Nov 7, Christmas Nov 24 – Jan 7), `off`, `halloween`, `christmas` (forced on) |
| `field` | `random` (default: a different field every round) or `deepSpace`, `earth`, `moon`, `mercury`, `venus`, `mars`, `ceres`, `jupiter`, `saturn`, `uranus`, `neptune`, `pluto`, `sun`, `io`, `europa`, `ganymede`, `callisto`. Between rounds a change shows at once; mid-round it applies next round (`POST /field` switches now) |
| `fieldPool` | Field ids `random` rolls from (`[]` = default: every owned field). Unowned ids are skipped; none owned = every owned field; a narrowed pool never rolls a DLC teaser |
| `missileFriendlyFire` | boolean |
| `trailMinutes` | 1–600 |
| `redeemLinks` | `{ rewardId: effectId }` (prefer the `/redeems/links` routes) |

Out-of-range numbers are clamped; unknown values keep the old setting.

## Events (`GET /api/v1/events`)

Server-Sent Events: one `event: <type>` + `data: <json>` per game event, and a `: ping` comment every 25 s.
Every event has `type` and `at` (ms timestamp).

| Type | Fields |
|---|---|
| `join` | `user` |
| `kill` | `victim`, `killer` (null: crashed with nobody to blame), `friendly` |
| `round.start` | `mode`, `field`, `pilots`, `teams` (counts), `teamNames`, `roster` (`[{ name, login, bot, team }]`) |
| `round.end` | `mode`, `result`, `stats` (`[{ id, name, login, bot, won, color, team, k, d, a, f, dmg, last }]`, best first; `dmg` = damage to bosses and their craft this round, BOSS mode sorts by it; `last` = bosses finished off), `teamNames`, `winnerTeam` (name or null), `bitLeaders` (`[{ name, bits }]`), `startedAt` |
| `lobby` | `seconds` |
| `calamity` | `calamity` (type), `name`, `by` (who launched it with Bits in the extension, or null) |
| `nuke` | `by` (redeemer or null), `x`, `y`: a nuke is counting down there |
| `boss.warning` | `boss`, `role`, `pack?` (a teaser: the pack it's from): the WARNING is up, a boss is coming |
| `boss.spawned` | `boss`, `role`: its entrance is over, it fights |
| `boss.rage` | `boss`, `rage` (×2–6): it got angrier (lower HP, or its shield broke) |
| `boss.shield` | `boss`, `shield` (`down`) |
| `boss.defeated` | `boss`, `by` (the pilot id with the last hit, or null), `livesLeft` (a stock encounter's boss comes back while it has lives) |
| `boss.left` | `boss`: a Factions encounter's boss warped out when its time was up |
| `boss.cheer` | `bits`, then `team` + `lives` (stock: lives for that team) or `seconds` (timed: on the boss's clock) |
| `giant-emote` | `user`, `emote` (id): a giantified emote flew in as an asteroid |
| `cheer` | `user`, `bits`, `text` |
| `bits` | `user` (login or null), `bits`, `calamity`: a viewer bought a calamity with Bits in the Twitch extension |
| `store.bought` | `userId` (Twitch), `item` (store item id), `itemName`: a viewer bought a ship or bling with Bits in the Twitch extension |
| `redeem` | `title`, `rewardId`, `effect` (null: not a game redeem), `user`, `input` |
| `effect` | `effect`, `pilot` (applied through the API) |
| `announce` | `text` |
| `relay` | `entitlements` (DLC names; connected to the relay) |
| `extension` | `from` (`role`, `userId`, `opaqueUserId`), `message` (a viewer's extension message) |

```js
// Node 18+: no library needed
const res = await fetch('http://localhost:2087/api/v1/events', {
  headers: { Authorization: `Bearer ${process.env.SPACE_API_KEY}` },
});
const decoder = new TextDecoder();
for await (const chunk of res.body) {
  for (const line of decoder.decode(chunk).split('\n')) {
    if (line.startsWith('data: ')) {
      console.log(JSON.parse(line.slice(6)));
    }
  }
}
```

(A real client should buffer partial lines across chunks; any SSE library does.)

## MCP server (`/mcp`)

Streamable HTTP, stateless, same API key. Tools:

| Tool | Args |
|---|---|
| `get_state`, `get_leaderboards`, `list_fields`, `list_redeem_rewards` | |
| `list_effects` ⚙ | |
| `update_settings` | `settings` object |
| `open_main_menu` | |
| `open_lobby` | `mode?` |
| `show_menu_page` | `page` (or null) |
| `start_round` | |
| `hold_lobby` | `hold` |
| `start_calamity` | `type` |
| `set_field` | `field` (`random` or a field id; mid-round it changes right away) |
| `call_strikes` | `count` |
| `launch_nuke` | `by?` |
| `spawn_boss` | `boss?` (a name, or random) |
| `boss_status` | (HP as Good / Fair / Low only) |
| `place_bounty` | `target`, `stacks?` |
| `apply_effect` ⚙ | `effect`, `pilot?`, `input?` |
| `test_cosmetics` ⚙ | `kind?` (`trail` / `shield` / `explosion` / `laser`; none = all four) |
| `use_ability` | `pilot`, `ability` |
| `add_pilot` | `username` |
| `announce` | `text`, `color?`, `seconds?` |
| `show_screen` | `screen`, `seconds?` |
| `link_redeem` | `rewardId`, `effect` |
| `unlink_redeem` | `rewardId` |
| `list_scripts`, `save_script`, `delete_script`, `test_script`, `set_secret`, `delete_secret` | API key / owner only (not the extension) |
| `twitch_connect`, `twitch_connect_poll` | API key / owner only: connect Twitch (code + url, then poll) |
| `results_board` | `board`: `kills`, `out`, `damage`, `none` |
| `killlist_board` | `board`: `lives`, `kills` |
| `follow_pilot` | `id?` (none = auto camera) |
| `join_bots` | `logins` |
| `update_bot` | `login`, `name?`, `color?`, `image?` |

Claude Code:

```sh
claude mcp add --transport http space-battle http://localhost:2087/mcp \
  --header "Authorization: Bearer sbk_…"
```

Any MCP client config:

```json
{
  "mcpServers": {
    "space-battle": {
      "type": "http",
      "url": "http://localhost:2087/mcp",
      "headers": { "Authorization": "Bearer sbk_…" }
    }
  }
}
```

The MCP server only answers calls; for live events a bot reads `/api/v1/events` alongside it.

## Writing a plugin

A plugin is any program that listens to events and calls the API. Example: hype the top killer and show the
boards at the end of every round.

```js
const BASE = 'http://localhost:2087/api/v1';
const headers = {
  Authorization: `Bearer ${process.env.SPACE_API_KEY}`,
  'Content-Type': 'application/json',
};
const post = (path, body) => fetch(`${BASE}${path}`, {
  method: 'POST',
  headers,
  body: JSON.stringify(body),
});

function onEvent(event) {
  if (event.type === 'round.end' && event.stats?.length) {
    post('/overlay/announce', { text: `MVP: ${event.stats[0].name}`, color: '#ffd400', seconds: 8 });
    post('/overlay/screen', { screen: 'leaderboard', seconds: 12 });
  }
  if (event.type === 'kill' && event.friendly) {
    post('/bounties', { target: event.killer, by: 'the fleet' });
  }
}
```

Plug `onEvent` into the SSE reader above. An LLM bot works the same way: feed it events, give it the MCP tools.
