# Pocket GM — Baseball

Single-file baseball **GM simulation** game (inspired by the iOS *Pocket GM Football/Basketball* apps).
You run the front office of a fictional 30-team baseball league: roster, lineup/rotation, season
sim, trades, free agency, amateur draft, contracts/budget, playoffs, and a multi-season franchise loop.

## Run it
- It's a single static file: `index.html`. Open it directly in a browser, or serve the folder:
  `python3 -m http.server 8124 --directory ~/baseball-gm` → http://localhost:8124
- Registered in `~/.claude/launch.json` as **baseball-gm** (port 8124) for the preview tooling.
- No build step. React 18 + Babel-standalone + Tailwind, all via CDN.

## Architecture (all in `index.html`)
- **Bootstrap:** the app code lives in a `<script type="text/babel-src" id="app-src">` block. A small
  inline `<script>` at the bottom does `Babel.transform(src, {presets:[["react",{runtime:"classic"}]]})`
  then `eval`s it. The explicit **classic** runtime is required — the default `react` preset emits an
  ESM `import` (automatic JSX runtime) which breaks in a non-module inline script.
- **State:** one big `G` object held in `App` `useState`, persisted to `localStorage`
  (`pocketgm_baseball_v2`) via `commit()`. New game built by `newGame(userTeamId, opts)`.
- **Save persistence:** prefer **not** bumping `SAVE_KEY` for shape changes — instead add backfill to
  `migrate(G)` (runs in `loadGame` and on import) so existing franchises keep loading across updates.
  Header has **Save File** / **Load File** (`exportSave`/`importSave`) for manual JSON backups; bump
  `SAVE_VERSION` when you add migration steps.
  - `G.players` is an **object dict keyed by id** (not an array). Use `Object.values(G.players)` or the
    `rosterOf(G, teamId)` helper. `autoSetLineups` accepts either an array or the dict.
  - `G.seed` holds the league seed (or `null`).
- **Team id namespace:** MLB teams are `G.teams` (ids 0..29); each has a AAA affiliate in `G.aaaTeams`
  (ids 30..59, `id = 30 + parentId`). A player's `teamId` identifies both his org **and level**.
  Helpers: `isAAAId`, `aaaIdOf`, `parentIdOf`, and `teamById(G,id)` (resolves either array). Anything
  iterating `G.teams` (standings, playoffs, splash) stays MLB-only; `Leaders` filters out AAA players.
- **Seeded RNG:** `rint/pick/chance/gauss` route through a swappable `_rng` (default `Math.random`).
  `generateLeague(userTeamId, {seed, shuffle})` installs a `mulberry32(seed)` PRNG **only for the
  league build**, then restores `Math.random` so the season sim stays genuinely random. `shuffle`
  reassigns city/nickname identities across the fixed league/division slots, pinning the user's pick
  (`shuffledTeamDefs`). The Splash screen exposes both (seed accepts a number or a word via `hashSeed`).
- **Sim engine:** `simGame()` runs a real inning-by-inning, plate-appearance model. `paOutcome()`
  converts batter vs. pitcher ratings into K/BB/HR/hit/out probabilities; `advance()` moves baserunners;
  a per-PA steal check populates SB / caught-stealing. Relievers rotate ~each inning via
  `changePitcher`/`bringIn` (tracked by `segOuts`), with the highest-OVR reliever (last in `pen`)
  closing save situations — so innings distribute realistically instead of one arm soaking the game.
  Produces realistic leaders (≈ .330–.350 AVG / 45–55 HR / sub-2 ERA / ~30 SB; SP ~180–220 IP).
- **Rotation & bullpen:** every team carries a **5-man rotation** (`autoSetLineups` promotes the best
  arms if short of true SPs) and each starter goes every 5th game (≈32–33 GS). The bullpen has a
  user-editable **priority order** (`team.bullpen`, resolved live by `bullpenOrder()`; `pen[0]` = closer).
  `pickReliever` keeps the closer for 9th-inning saves, gives the 8th to the setup tier, shares earlier
  relief across the better ~70% of arms (rotated by game/inning to avoid one arm pitching every day),
  and dumps blowouts on mop-up arms — so better arms get the meaningful innings (~80–106 IP) and weak
  arms sit (~25 IP). Edit the rotation order and bullpen priority in the **Lineup** tab.
- **Season:** `buildSchedule()` = circle-method round-robin repeated to 162 games (15 games/day).
  `simDay()` advances one day, simming **both** `G.schedule` (MLB) and `G.aaaSchedule` (AAA) — so a
  full season is ~4860 game sims. At day 162 `enterPlayoffs()` fires (MLB only; AAA is regular-season).
- **AAA / minor leagues:** every org runs a AAA affiliate that plays its own 162-game season with the
  same `simGame` engine (it just operates on AAA team objects whose players carry the AAA teamId). AAA
  rosters are younger/weaker (`buildAAARoster`, `meanAdj=-6`) and prospects develop faster in the
  offseason. Roster screen has a **AAA Affiliate** section with `↑ up` / `↓ AAA` (`callUp`/`sendDown`);
  the **AAA** tab shows affiliate standings + top prospects. Draftees report to AAA.
- **Defensive positions:** `team.pos` maps playerId → assigned position (8 fielders + one DH). The
  `LineupEditor` assigns them (swapping to keep a valid permutation) and flags missing/duplicates.
  `teamDefRating` applies an out-of-position penalty and ignores the DH.
- **Draft picks:** `G.picks` is a flat list of owned, **tradeable** picks (5 per team). `pickValue`
  scales with the original team's projected finish (worse team → earlier slot → more value). The draft
  builds its selection order from owned picks (`G.draftPicks`, by round then orig-team reverse
  standings), so traded picks keep their slot. `Trades` can include players (MLB or AAA) and picks.
- **Postseason:** 6 seeds/league, Wild Card Bo3 → Division Bo5 → LCS Bo7 → World Series Bo7.
- **Offseason loop:** aging/development toward potential + decline, retirements, contract expiry →
  free agency, `buildDraft()` (5-round amateur draft), then `startNewSeason()` rolls everything over
  (resets MLB + AAA records/schedules, regenerates next year's `G.picks`).

## Player model
- Hitters rated CON/POW/EYE/SPD/DEF; pitchers STU/CTL/STM. `computeOvr()` derives OVR; `pot` is ceiling.
- `salaryFor()` scales salary by OVR + age (pre-arb discount). `playerValue()` drives trade/FA AI.
- **Talent curve:** rating means sit a bit high and ~8% of generated players get a `+8–18` "star"
  boost to their primary tools (in `makeHitter`/`makePitcher`) for a fat top end. If you bump these,
  re-check the `paOutcome` `hrP` formula — league HR is sensitive to the POW mean × `hrP` slope/cap.
- **Development:** `doProgression` is **directed toward potential** — young players climb faster the
  bigger their `pot − ovr` gap (prospects in AAA faster still), peak players hold, age-31+ decline.
  Tune the `center` ramp there if growth feels too fast/slow.
- **Roster size:** `makeTeam` builds ~30 players (16 hitters = 9 starters + 7 bench; 14 pitchers =
  5 SP + 9 RP). `ROSTER_CAP` (32) bounds offseason rosters; `startNewSeason` AI fills to ~28.

## UI notes
- `HitterTable`/`PitcherTable` take optional `badge` ("ST"/"BN" pill), `bo` (batting-order #, hitters),
  `slots` (per-row labels like SP1/CL, pitchers), `posMap` (show assigned defense, amber if out of
  position), `onName` (opens `PlayerModal`), and action callbacks `onRelease`/`onSend`/`onCallUp`
  (rendered by `RowActions`). `Roster` splits into Starting Lineup / Bench / Rotation / Closer /
  Bullpen / AAA Affiliate using these.
- `Leaders` qualifier thresholds scale with games played (`paQual`/`outQual`); AAA players excluded.
- Player movement helpers: `movePlayer(G,p,destMlbId)` (preserves level), `movePlayerToFA(G,p)`.

## Gotchas / tuning ideas
- Win leader can reach ~25 (a touch high) — tune `awardDecision()` / rotation depth if desired.
- After an auto-simmed offseason the user roster can dip into the low 20s (and rotation below 5 SP if
  starters retire/expire); fill via Free Agency. The sim tolerates short staffs.
- AI trade acceptance: accepts if value received ≥ 0.97× value given (`Trades` component).
- HR distribution is the most sensitive balance knob. Targets after a full season: AVG leader
  ~.330–.355, HR leader ~45–55 with ~6–10 hitters over 40, ERA leader ~1.8–2.2. Verify by simming a
  fresh league to playoffs and checking total W === total L (2430 per league; both MLB and AAA balance).
- Sim cost doubled with AAA (both leagues sim every day). "Sim to Playoffs" on a fresh league takes a
  few seconds. If it ever needs to be faster, AAA is the place to gate (e.g. sim AAA less often).
