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
  `pickReliever` is **role + workload based**: the closer (`pen[0]`) only enters to finish a 9th+ save of
  1–3; the **setup tier** (best ~55% of non-closers) carries normal relief; the fringe mop up blowouts
  (|lead|≥7). Within a role it hands the ball to whoever's thrown the least — first **this game**
  (`gOuts`), then **this season** (`stats.OUT`) — via `loadKey`/`leastLoaded`, and it *excludes the
  current arm* so nobody pitches consecutive innings or runs away with the workload. This replaced an
  older "fresh/unused" model whose exhaustion fallback let one arm soak 600+ IP (esp. in AAA). Result:
  good arms get the meaningful innings (~85–105 IP), fringe arms sit (~10–17 IP), no reliever outlier.
  Edit the rotation order and bullpen priority in the **Lineup** tab. NOTE: a deep enough pen matters —
  AAA carries 8 RP so its setup tier has ~5 arms like the bigs (too few → per-arm IP balloons).
- **Season / schedule:** `buildSchedule(teams)` builds a 162-game slate **with off days** — it spreads
  the games over ~224 days (~11 games/day, each team idle ~62 days), so not everyone plays daily.
  Matchup counts come from `gamesBetween` — **18 per division rival, 8 per same-league team, 2 per
  mirror-division interleague team, 0 other interleague** (72 + 80 + 10 = 162). Games are packed greedily
  into the emptiest valid day (no team twice/day). `padSchedules` keeps MLB and AAA calendars the same
  length. `simDay()` plays both `G.schedule` and `G.aaaSchedule` (~4860 sims/season); `enterPlayoffs()`
  fires at `G.day >= G.schedule.length`. Anything that referenced "162 days" now uses `G.schedule.length`.
- **AAA / minor leagues:** every org runs a AAA affiliate that plays its own 162-game season with the
  same `simGame` engine (it just operates on AAA team objects whose players carry the AAA teamId). AAA
  rosters are younger/weaker (`buildAAARoster`, `meanAdj=-6`; **14 hitters + 5 SP + 8 RP**) and prospects
  develop faster in the offseason. Roster screen has a **AAA Affiliate** section with `↑ up` / `↓ AAA`
  (`callUp`/`sendDown`); the **AAA** tab shows affiliate standings + top prospects. Draftees report to AAA.
- **AI roster management (`aiAlignOrg`):** keeps each org's BEST talent on the big-league club instead of
  buried in AAA. Promotes by OVR while guaranteeing a natural fielder at every position and a 5-man-capable
  staff (≤16 hitters + ≤14 pitchers up, the rest to the affiliate). Called at league gen (all clubs) and in
  `startNewSeason` for AI clubs (after their FA fill, before `trimRoster`/`autoSetLineups`); the user's club is
  left alone so manual call-ups stand. `migrate` runs it once on old saves (guarded by `G._aiAligned`,
  skipping the user). Without it, randomly-generated/aged orgs stranded 70+ OVR players behind MLB scrubs.
- **Defensive positions:** `team.pos` maps playerId → assigned position (8 fielders + one DH). The
  `LineupEditor` is a **drag-and-drop field diagram** (tap a player then a spot, or drag) backed by
  `placeAtPosition`, which glues the position to the player and keeps the lineup a valid permutation
  (bench bat dropped on a fielder takes his batting-order slot). It flags missing/duplicates;
  `teamDefRating` applies an out-of-position penalty and ignores the DH. The Hub shows an end-of-season
  reminder linking to Finances → Contracts when players are in a contract year (`years<=1`).
- **Draft picks:** `G.picks` is a flat list of owned, **tradeable** picks (5 per team). `pickValue`
  scales with the original team's projected finish (worse team → earlier slot → more value). The draft
  builds its selection order from owned picks (`G.draftPicks`, by round then orig-team reverse
  standings), so traded picks keep their slot. `Trades` can include players (MLB or AAA) and picks.
  When a prospect is drafted (`assign`), `p.draftInfo = {year, round, overall, by}` is stamped on him
  (shown in `PlayerModal`).
- **Trade market:** a **deadline** (`G.tradeDeadline` ≈ 58% through the calendar) closes the market —
  `Trades` disables proposing past it (banner + disabled button) and the Hub shows a countdown.
  **AI-initiated offers** (`makeAITradeOffer`): each sim-day before the deadline an AI club may pitch the
  user a deal (`chance(0.045)`, ≤2 pending, expire after 12 days), coveting a useful user MLB player and
  assembling a comparable-value package (no single piece worth ≫ the target; total ~0.9–1.15× — fair,
  user decides). Offers live in `G.tradeOffers`; the **Hub** renders Accept/Reject cards. Accept runs
  `executeOffer` (preserves MLB/AAA level, refreshes lineups, logs to the news wire). `startNewSeason`
  recomputes the deadline and clears pending offers.
- **Per-season history:** every game tags the team a player suited up for into `p._yrTeams`
  (`tagTeam` in `markGamesPlayed` for hitters and at the end of `simGame` for the pitchers who threw —
  so mid-season trades record *both* teams). At the offseason rollover `recordSeasonHistory(p, season)`
  (called at the top of `doProgression`, before aging) pushes `{season, age, ovr, teams[], stats}` onto
  `p.history` and clears `_yrTeams`; `mergeCareer` then folds the line into career totals. `PlayerModal`
  renders a **Year by Year** table from `p.history`.
- **Contracts:** each deal is `p.contract` — a per-year array of guaranteed salaries (cap hits);
  `p.salary` mirrors `contract[0]` (the current-year hit everything reads), `p.years = contract.length`.
  `p.incentives` is a list of off-cap, stat-based bonuses `{stat, need, amount}` (multiple per deal;
  hitting stats for hitters, pitching for pitchers, via `STAT_INCENTIVES`). Helpers: `setContract`
  (install a flat schedule), `ensureContract` (migrate/backfill — also heals empty contracts),
  `requiredAAV`/`willSign` (acceptance; incentives count **½** toward the ask, with a price `factor`
  for hometown discounts), `contractLabel`, `payIncentives`.
  - **Negotiation** runs through `ContractModal` (sign a FA or extend your own: set years + base AAV +
    add incentives, live accept/“wants more” verdict) and `RestructureModal` (defer current-year money
    to later years for a 10% premium). Finances → Contracts has Extend/Restructure; Free Agency has
    Negotiate. `payroll` counts only the base (`p.salary`).
  - **Money:** incentives are off-cap but real — `payIncentives` (called in `enterPlayoffs`) settles
    earned bonuses and **deducts them from the team's budget** (`t.incentivesPaid`), so the growth
    formula in `startNewSeason` builds on the reduced figure. Each offseason a rostered contract
    `shift()`s a year and expires to FA when empty; **free agents are skipped** (their contract is an
    ask, not a deal in progress).
- **Postseason:** 5 seeds/league (3 division winners + 2 wild cards). Wild Card is a **single play-in
  game** (`SERIES_LEN.wc=1`) between seeds 4 & 5 (higher hosts); the winner becomes the 4th team. Then
  Division Bo5 (**#1 vs play-in winner, #2 vs #3**) → LCS Bo7 → World Series Bo7. Played **game by game** —
  each series is a `mkSeries` object (`hw/lw/games[]`); `playoffSeriesGame` plays one game (2-2-1 home
  pattern), `activeSeriesList` is the round's live series, `buildNextRound` builds the next round once
  they're all decided. The `Playoffs` UI offers **Sim Next Game** (one game per active
  series, with line scores) / **Sim Whole Round** / **Advance**, then crowns the champion → offseason.
- **Economy:** **revenue/budget evolves** at season rollover (`startNewSeason`) — `t.budget` shifts with
  win% (`(wp-.5)*0.45`) plus playoff/championship bonuses, then regresses 6%/yr toward a ~160 market mean
  (clamped 95–290) so a dynasty can't run payroll away forever. `t.budgetDelta` is stored and shown on the
  Finances budget card; the user's change is logged to the news wire. **Waiver wire** (`G.waivers`): cut
  players (in-season user releases via `placeOnWaivers`, plus occasional AI churn via `maybeAIWaive`) sit on
  waivers ~2 days (user has first claim from the Free Agency tab) then `processWaivers` lets AI clubs claim
  in reverse-standings order; unclaimed after ~4 days they clear to free agency. Cleared/flushed each new
  season. ROSTER_CAP and cap space gate claims.
- **Offseason loop:** aging/development toward potential + decline, retirements, contract expiry →
  free agency, `buildDraft()` (5-round amateur draft), then `startNewSeason()` rolls everything over
  (resets MLB + AAA records/schedules, regenerates next year's `G.picks`). Before resetting records,
  `startNewSeason` pushes the finished season into each `team.history` (`{season,w,l,made,champ}`).
- **BA Cup:** a single-game knockout (FA-Cup style) open to **all 60 clubs** — every MLB team AND its AAA
  affiliate — whose rounds are **sprinkled through the regular season** and play automatically as you sim
  (not a manual event). `startBACup` shuffles the 60 ids into a 64-bracket (4 random byes) and assigns each
  of the 6 rounds a target day (`roundDays`, fractions of `schedule.length`: R64→R32→R16→QF→SF→Final).
  `simDay` calls `ensureSeasonCup` + `tickBACup` (plays any round whose day has arrived, catching up if
  several are overdue); `startNewSeason` starts a fresh bracket each year; `enterPlayoffs` calls
  `finishBACup` as a safety so the Cup always crowns before the season closes. Games are scored by a
  lightweight ace-weighted strength model (`cupGame`/`cupPower`) so **cup play never touches league
  stats/records** (career BA Cup stats may come later). State on `G.baCup = {season, roundIdx, matches,
  past[], champion, roundDays}`; winners roll into `G.baCupChampions`. The **BA Cup** tab is informational
  (auto-inits via `useEffect`): current/upcoming round, your club's matchup or knockout round, past rounds,
  past winners. NOTE: champion is a team id (0..59) — all "has a champion" checks use `!= null` (id 0 is falsy).
- **Rivalries:** `assignRivals` pairs teams up (two pairs per division + the odd team gets a cross-league
  mirror-division rival), stored as `team.rivalId`. Shown on the Hub (season series + next meeting) and
  highlighted in the Schedule. Migrate backfills rivals for old saves.
- **History / Hall of Fame:** `G.champions` is the title roll (pushed in `buildNextRound`'s WS case);
  `team.history` holds per-season records. **Championship rings:** when a team wins the WS, every player on
  its MLB roster gets a ring (`p.rings` = array of seasons) in `buildNextRound`. `p.peakOvr` tracks
  career-best OVR; on retirement `inductIfWorthy` adds a snapshot to `G.hof` when `hofScore(p) >=
  HOF_THRESHOLD` (175). **`hofScore` weighs counting stats + a peak-greatness bonus + hardware**:
  `accCount(p,award)` (MVP ×28, Cy Young ×24, All-Star ×7) and `ringCount(p)` (×9) — so a decorated,
  ring-laden career clears the bar that a mere compiler wouldn't. The hof entry stores `mvp/cy/allStar/rings`
  counts for display. `PlayerModal` shows a 💍 ring badge next to MVP/All-Star honors.
- **Records:** `recordFranchiseRecords` (per-club, `team.records`, single-season) runs in `enterPlayoffs`.
  League-wide records live on **`G.records = {game, season, career}`** (each entry `{value, holder, team,
  season}`): `recordSeasonRecords` (single-season bests, PA≥300/OUT≥120 qualifiers) + `recordCareerRecords`
  (best career totals = `career`+live-season, persistent high-water mark so it survives pruning) run in
  `enterPlayoffs`; **game records** are fed live by `updateGameRecords` from `simGame`'s `out.gameLines`
  (a per-player diff vs a start-of-game snapshot `gSnap`) — MLB regular-season (`simDay`) + playoff
  (`seriesGame`) games only, so they fill in **going forward** (no retro). `migrate` carries the old flat
  `G.leagueRecords` into `G.records.season` and seeds `.career` from current players (once, when `G.records`
  is first created). The **Hall of Fame** tab's League Records card has Game/Season/Career toggle tabs.
- **League news / transaction wire (`logNews`, `G.news`):** a persistent, newest-first, capped-250 log of
  notable moves — FA **signings** + extensions, **trades**, **roster** moves (call-up/option/release),
  the user's **draft** picks, end-of-season **awards** (MVP/Cy Young), and World Series **titles**. Each
  entry is `{season, day, phase, type, msg, teamId}`. Hooked in at each action site (FreeAgency, Finances,
  Trades, Roster, Draft, `computeAwards`, `buildNextRound`). The **News** tab (`NewsFeed`, type-filtered via
  `NEWS_META`) renders the full wire; the Hub shows the latest five. Backfilled empty in `migrate`.

## Player model
- **Tracked stats:** hitters `G/PA/AB/H/2B/3B/HR/R/RBI/BB/SO/SB/CS`; pitchers
  `G/GS/QS/OUT/H/ER/BB/SO/W/L/SV/HLD/HRA`. **CS** is logged in the steal-fail branch; **QS** (6+ IP,
  ≤3 ER) is credited post-game by diffing the starter's OUT/ER against a pre-game snapshot (`startSnap`);
  **HLD** is credited in `awardDecision` to used relievers in a ≤3-run win other than the winner/saver
  (an approximation — it runs a bit high because relievers rotate ~per inning). Shown in
  `HitterTable`/`PitcherTable`/`PlayerModal` and as **Quality Starts** / **Holds** leaderboards.
- **Franchise records (`recordFranchiseRecords`, `team.records`):** each club's all-time single-season
  bests (team wins; HR/RBI/H/R/SB/AVG by a hitter; W/K/SV/ERA by a pitcher), each `{value, holder, season}`.
  Updated in `enterPlayoffs` (after `computeAwards`, before stats reset) with PA≥300 / OUT≥120 qualifiers.
  Rendered on the **Hall of Fame** tab with a team selector.
- **Prospect rankings (`prospectGrade`):** AAA players + young (≤22) MLB rookies ranked by a scouting grade
  (POT-weighted + current ability + youth), with an `prospectETA` estimate, on the **AAA** tab (top 30).
- Hitters rated CON/POW/EYE/SPD/DEF; pitchers STU/CTL/STM. `computeOvr()` derives OVR; `pot` is ceiling.
- `salaryFor()` scales salary by OVR + age (pre-arb discount).
- **Trade/FA value (`playerValue`):** built on `abilityValue(ovr)` — a steeply **convex** curve
  (≈ 50→3, 65→18, 80→42, 85→67, 92→95) so stars are scarce and you can't bundle scrubs to equal one
  (an 85 ≈ 22× a 50). Value blends current ability with **projected peak** (`ovr + (pot−ovr)·youth`,
  `youth = (28−age)/13` clamped), weighted toward projection for the young — so high-ceiling prospects
  are worth a lot (a 19yo pot-90 ≈ a mid-30s star). Past-prime players take a decline tax; salary is a
  small drag. `pickValue` is tuned to match (a top pick ≈ a blue-chip prospect).
- **Extensions:** `extensionTerms(G,p)` supplies the price `factor` + `refuses` flag for the extend
  `ContractModal` (Finances → Contracts, eligible when `years<=2`). Winning clubs get a hometown
  discount, losing clubs pay a premium, and a star (`ovr>=75`) on a sub-.400 team refuses (wants to
  test FA). Uses a neutral .500 baseline until the team has played ≥20 games. The factor is threaded
  into `requiredAAV`/`willSign` so the displayed discount actually drives acceptance.
- **Talent curve:** rating means sit a bit high and ~8% of generated players get a `+8–18` "star"
  boost to their primary tools (in `makeHitter`/`makePitcher`) for a fat top end. If you bump these,
  re-check the `paOutcome` `hrP` formula — league HR is sensitive to the POW mean × `hrP` slope/cap.
- **HR formula (`paOutcome` `hrP`):** `0.019 + 0.074*(pow−.5) + 0.040*(con−.5) − 0.030*(pitStu−.5)`.
  The **contact term is deliberate** — HR needs power *and* some contact, so a one-dimensional slugger who
  can't make contact no longer leads the league in homers (the old power-only slope let a 58 OVR mash 36 HR).
  Tuned targets (3-seed check): HR leader ~48–53, ~3–6 hitters over 40, no sub-60 OVR sluggers over ~32.
- **Potential & boom prospects (`setPotential`):** `pot = ovr + age-scaled room` (room up to ~24). On top,
  young players (≤22) have a ~3.5% chance to be a **boom prospect** — a raw, low-OVR youngster with a
  huge ceiling (e.g. 55 OVR / 85 POT, gaps up to ~45). Rare, and most never fully realize it, but a
  long-grind star can emerge from the draft/AAA.
- **Retirement (`doProgression`, offseason):** retirement odds are driven by **both age and quality** —
  base ramps with age (`(age-31)*0.06` past 31), reduced for good players (`(ovr-60)/100*1.4`, so a 35+
  vet still ~65 OVR keeps playing) and increased for genuinely bad aging players (`ovr<55` adds
  `(age-33)*0.05`). A stable per-player longevity factor (~0.78–1.22 from the id) adds variety; 44+ almost
  always retire. Replaced the old flat age-bucket table.
- **Development & regression (`doProgression`, offseason):** each attribute shifts by `gauss(center,1.2)`.
  `center` ramps with age — young players climb toward potential (≤23 fastest, growth fades to a ~26-28
  peak), 29-30 gentle decline, 31-33 steeper, 34+ falls off a cliff. Modifiers: **AAA prospects (≤25)
  get +1.2** (develop faster in the minors); a **performance term** (`seasonPerf(p)` → score×weight)
  adds growth for productive young players (so a kid promoted early who rakes keeps developing fast) and
  lets veterans who still rake age gracefully. Young breakouts can raise their `pot`; 29+ erodes it.
  `p._lastDelta` records the OVR change and feeds the **Player Development risers/decliners report**
  shown on the offseason Hub. Tune the `center` ramp / performance multipliers there.
- **Awards (`computeAwards`, run in `enterPlayoffs` before stats reset):** per league it picks a
  **Player of the Year** (`mvpScore`), **Cy Young** (`cyScore`), and **All-Star** team (best at each of
  9 lineup slots + top 6 pitchers). Stored on `G.awards` (current) and unshifted into `G.awardHistory`;
  each honor is also appended to the player's `p.accolades` (shown in `PlayerModal` and surviving
  retirement because award records carry name/line snapshots). The **Awards** tab renders it all.
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
- **`PlayersBrowser`** (Players tab): league-wide table of every player with filters (name search,
  team, level MLB/AAA/FA, position, age min/max) and sort (OVR/POT/trade value/age/salary/HR/AVG/ERA).
  Shows a `Val` column (`playerValue`) and a key stat line; names open `PlayerModal`. Capped to 400 rows.
- **`Awards`** tab: current-season MVP/Cy Young/All-Stars per league + an award-history table.

## Gotchas / tuning ideas
- Win leader can reach ~25 (a touch high) — tune `awardDecision()` / rotation depth if desired.
- After an auto-simmed offseason the user roster can dip into the low 20s (and rotation below 5 SP if
  starters retire/expire); fill via Free Agency. The sim tolerates short staffs.
- AI trade acceptance: accepts if value received ≥ 0.97× value given (`Trades` component).
- HR distribution is the most sensitive balance knob. Targets after a full season: AVG leader
  ~.330–.355, HR leader ~45–55 with ~6–10 hitters over 40, ERA leader ~1.8–2.2. Verify by simming a
  fresh league to playoffs and checking total W === total L (2430 per league; both MLB and AAA balance).
- **Sim performance:** `simDay` builds a `rosterIndex(G.players)` (team→players) **once per day** and
  threads it (`idx`) into `simGame`/`getPitchers`/`bullpenOrder`/`lineupHitters`/`distributeRuns`/
  `markGamesPlayed` so games don't each re-scan the whole player dict. Sim cost is now ~flat in total
  player count (≈0.5–2 ms/day) instead of O(players × games). If you add a hot per-game helper that needs
  a team's roster, take the optional `idx` and use `teamRoster(players, teamId, idx)`.
- **Unfieldable-roster guard (`refillRoster`, in `simDay`):** over many seasons, call-ups/trades/lopsided
  draft classes can drain a roster (esp. AAA) below what a game needs — a team with no pitcher or <9 hitters
  used to **crash `simGame` mid-day, throwing out of the whole Sim loop and hanging the button forever**
  (this was the real "days take forever / stuck" report, not slowness). `simDay` now checks every MLB/AAA
  club against a playable minimum and refills depleted ones before any game (then rebuilds the index).
- **Player-pool is bounded** so sim speed stays flat across seasons: `startNewSeason` caps AAA rosters
  (`trimRoster(t, AAA_CAP=38)`), sweeps orphaned teamless players (e.g. leftover draft class) into the FA
  pool, then `pruneFreeAgents` (keep best ~220, delete the rest from `G.players`). `migrate` runs a
  one-time cleanup on already-bloated saves (>2800 players). Without this the dict grew +150/yr forever.
