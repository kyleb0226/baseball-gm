# Pocket GM — Baseball — Next Session Work Order

Hand this file to a fresh Claude Code session. It is self-contained: it explains the
codebase, the exact changes wanted, where to make them, and how to verify. Read
`CLAUDE.md` in this same folder first for architecture, then do the work below.

## Project at a glance
- **One file:** `~/baseball-gm/index.html` (React 18 + Babel-standalone + Tailwind, all CDN, no build).
- App code lives in `<script type="text/babel-src" id="app-src">`; a bottom inline `<script>` does
  `Babel.transform(src, {presets:[["react",{runtime:"classic"}]]})` then evals it.
  **Keep the `runtime: "classic"`** — the default preset emits an ESM `import` that breaks inline.
- State is one `G` object in `App` `useState`, saved to `localStorage` key `pocketgm_baseball_v1`
  via `commit()`. `G.players` is an **object dict keyed by id**, NOT an array — use
  `Object.values(G.players)` or the `rosterOf(G, teamId)` helper.
- On GitHub: https://github.com/kyleb0226/baseball-gm (push when done — see "Wrap up").

## How to run / verify (use the preview tooling, not manual)
- A launch config named **baseball-gm** exists in `~/.claude/launch.json` (port 8124,
  `python3 -m http.server 8124`). Start it with `preview_start`, then drive the page with
  `preview_eval` / `preview_click` / `preview_screenshot` and check `preview_console_logs` (errors).
- IMPORTANT: runtime errors in the eval'd script surface to `window.onerror` as a masked
  **"Script error."** with no line info. To debug, wrap suspect logic in try/catch and log, or
  reason from the source. (This is how the original author found a dict-vs-array bug.)
- After code edits, `preview_eval: window.location.reload()` (python http.server may cache; a hard
  reload via cache-busting query is safest).
- Smoke test the full loop after changes: new game → Sim to Playoffs → run postseason → Hub
  "Process Offseason" → Draft "Sim Rest" → "Start Season" → Sim 1 Month. Confirm zero console errors
  and balanced league record (total W === total L).

---

## Requested changes

### 1) Track & display more stats (incl. innings pitched)
**Goal:** richer box-score stats, and show IP for pitchers everywhere.

What already exists:
- Stat shells: `emptyHitStats()` and `emptyPitStats()`. Pitchers already accumulate `OUT` (outs
  recorded); innings pitched is derived by the existing `ip(s)` helper (`OUT/3`, formatted x.1/.2).
  Hitters already track `G, PA, AB, H, _2B, _3B, HR, R, RBI, BB, SO, SB`.
- Derived helpers already defined: `avg, obp, slg, ops, ip, era, whip, f3, f2`.
- The sim (`simGame`) already mutates these per plate appearance.

Do this:
- **Add an `IP` column to `PitcherTable`** (use `ip(p.stats)` → format with `f1`, e.g. `7.2`). Also add
  `GS`, `H`, `BB`, `ER` columns. Add a helper `const f1 = v => v.toFixed(1);` near the other formatters.
- **Add an `IP` (or innings) leaderboard** and a `BB`/`K-BB` board to the `Leaders` component; also add
  hitter boards for `OBP`, `SLG`, `SB`, `BB`, `R` if not present. Keep the qualifier thresholds
  (`PA>=20` / `OUT>=30`) but consider scaling them with games played so late-season boards read right.
- **Expand `HitterTable`** to show `G, AB, R, H, BB, SO, SB, OBP, SLG` in addition to current
  AVG/HR/RBI/OPS. The table is horizontally scrollable already (`overflow-x-auto`) so width is fine.
- **Player detail (nice-to-have):** clicking a player name opens a modal/panel with full season +
  career splits (career already accumulates in `p.career` via `mergeCareer` at season rollover).
- Verify: after simming a month, IP for a starter ≈ 30–45, and `OUT/3` matches the displayed IP.

### 2) Rosters: show Starters vs Bench (and rotation vs bullpen)
**Goal:** the Roster screen should make clear who starts and who is benched.

Current behavior: `Roster` just lists all hitters then all pitchers sorted by OVR. Starters are implied
only by `team.lineup` (9 hitter ids, batting order) and `team.rotation` (5 SP ids); relievers are chosen
automatically (highest-OVR reliever closes).

Do this in the `Roster` component (and reuse the existing `HitterTable`/`PitcherTable`):
- Split hitters into **Starting Lineup** (ids in `team.lineup`, shown in batting-order sequence with the
  batting-order number and fielding position) and **Bench** (the rest).
- Split pitchers into **Rotation** (`team.rotation`, labeled SP1–SP5), **Closer** (current logic = the
  highest-OVR reliever; surface that), and **Bullpen** (remaining RPs).
- Add a small badge/pill on starters (e.g. green "ST" vs grey "BN"). A `starters` Set built from
  `team.lineup`/`team.rotation` ids makes this easy to pass into the table rows.
- Keep the existing `LineupEditor` working (it edits `team.lineup` order and `team.rotation`).

### 3) More players, and better players
**Goal:** deeper rosters and a higher talent ceiling so stars feel like stars.

Where talent/roster size is set:
- `makeHitter(pos, ageRange)` and `makePitcher(role, ageRange)` — rating means via `rateGauss(mean, sd)`.
- `makeTeam(def, idx)` builds each roster: currently 9 position starters + 4 bench = 13 hitters, and
  5 SP + 8 RP = 13 pitchers (26 total).
- `computeOvr`, `setPotential`, `salaryFor` derive OVR/POT/salary from ratings.

Do this:
- **Deeper rosters:** bump bench hitters from 4 → 6–7 and relievers as needed so each team carries
  ~30–34 players (a 26-man active roster + a small "reserve"/minors feel). If you add a reserve concept,
  keep it simple — extra players that can be subbed in / called up, not a full minors system, unless you
  want to build that (optional; note it in `CLAUDE.md` if you do).
- **Better players / more stars:** raise the rating distribution so there's a fatter top end. Two good
  knobs: (a) nudge the gaussian means up (e.g. hitters CON ~58, POW ~55; SP STU ~58) and (b) give a
  small fraction of generated players a "star" boost (e.g. ~8% of players get +8–18 to their primary
  ratings before `computeOvr`). Make sure OVR still clamps at 99 and the curve isn't flat — you want a
  spread, not everyone at 80.
- **Keep balance:** re-run a full-season sim and check the leaderboards stay believable
  (rough targets: AVG leader ~.330–.360, HR leader ~45–60, ERA leader ~1.8–2.6, ~3–6 hitters above 40 HR
  league-wide). Tune `salaryFor` if payrolls blow past budgets after the talent bump.
- Re-check the AI free-agency fill in `startNewSeason` and `trimRoster` (roster cap currently 30) so the
  new, larger rosters don't get over-trimmed.

### 4) Randomly generated league on each new game (not fixed defaults)
**Goal:** every new franchise gets a freshly randomized league; current rosters feel like fixed defaults.

Context: `newGame(userTeamId)` already calls `generateLeague()` which uses `Math.random()`, so each NEW
game IS randomized. The reason it "seems set" is that the league **persists in localStorage** — reopening
the page reloads the same saved franchise (intended). The 30 team identities (city/nickname/division) come
from the fixed `TEAM_DEFS` table.

Do this (confirm intent against these — the user said "random generated on each opening as well as just
the defaults that seem to be set"):
- **Keep `TEAM_DEFS` team identities as the default**, but ensure **players/ratings are freshly random
  each new franchise** (verify two new games produce different rosters — they should already; if not,
  fix the RNG).
- **Add a "Randomize" affordance** on the Splash screen and/or near the existing "New" button: e.g. a
  "🎲 Randomize League" toggle/button that regenerates the player pool (and optionally shuffles which
  cities land in which division) without changing the user's selected franchise identity.
- Optional: add a numeric **seed** input so a league can be reproduced/shared. If you add seeding, replace
  raw `Math.random()` with a seeded PRNG (e.g. mulberry32) wired through the `rint/pick/chance/gauss`
  helpers, and store the seed in `G`.
- Make sure the "New" button flow (top-right, `clearSave()` then back to Splash) still works and clearly
  produces a different league.

---

## Constraints & style
- Stay **single-file**; no new dependencies or build step.
- Match the existing visual style (Tailwind dark theme, `Card`, `OvrPill`, emerald accents, `tnum`
  tabular numbers for stat columns).
- Don't break `localStorage` saves casually. If you change the shape of `G` or the stat objects,
  bump the save key (`SAVE_KEY`) or add a migration so old saves don't crash on load.
- Keep `autoSetLineups` tolerant of being passed either an array or the `G.players` dict (it already is).

## Wrap up
- Update `CLAUDE.md` (and `README.md` feature list) to reflect the new stats, roster sections, deeper/
  better rosters, and randomize/seed feature.
- Verify the full loop is clean (see "How to run / verify"), then commit and push:
  ```bash
  cd ~/baseball-gm && git add -A && git commit -m "<summary>
  
  Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>" && git push
  ```
