# ⚾ Pocket GM — Baseball

A single-file baseball **general-manager simulation** game, inspired by the iOS *Pocket GM Football /
Basketball* apps. Run the front office of a fictional 30-team league across multiple seasons.

**[▶ Play it](index.html)** — it's one static HTML file. No build, no install.

## Run

Just open `index.html` in a browser. Or serve the folder:

```bash
python3 -m http.server 8124 --directory .
# → http://localhost:8124
```

Your franchise autosaves to the browser (localStorage), and you can **Save File / Load File** from the
header to keep a JSON backup that survives updates.

## Features

- Pick from 30 fictional teams in two leagues, then play a 162-game season
- **Freshly randomized league every new franchise** — optionally shuffle cities across
  divisions, or enter a **seed** (number or word) to reproduce/share an exact league
- Sim controls: day / week / month / straight to the playoffs
- Deeper rosters (~30 players: a full lineup + bench and a 5-man rotation + bullpen), with a
  **fatter talent curve** so star players genuinely stand out
- Roster screen split into **Starting Lineup / Bench / Rotation / Closer / Bullpen** with ST/BN
  badges; editable batting order **and a defensive alignment** (assign all 8 field positions + a DH;
  out-of-position players take a fielding penalty)
- Editable **5-man rotation and bullpen priority** — set your closer & setup man; your best arms work
  the high-leverage innings while mop-up arms take blowouts
- Players **develop toward their potential** each offseason (young prospects fastest)
- **AAA affiliates** — every club runs a minor-league team that plays its own full season (with its
  own standings and stat lines). Develop prospects there, then **call them up / send players down**
- **Tradeable draft picks** — deal future picks whose value scales with each team's projected finish,
  alongside players (big-leaguers or prospects)
- Inning-by-inning, plate-appearance sim engine driven by player ratings — including stolen
  bases and a rotating bullpen — produces realistic stat lines
- Rich box-score stats everywhere: hitters show G/AB/R/H/HR/RBI/BB/SO/SB and AVG/OBP/SLG/OPS;
  pitchers show GS/IP/H/BB/SO/ER and ERA/WHIP/SV
- Click any player for a detail card with season + career splits
- Standings and 17 stat leaderboards (AVG, HR, RBI, OBP, SLG, OPS, Hits, Runs, BB, SB, ERA,
  Innings, K, K−BB, W, SV, WHIP) with qualifiers that scale through the season
- Trades with value-based AI, free agency with a salary cap, contracts & budget view
- **Re-sign your own players** to early extensions before they hit free agency — winning earns a
  hometown discount, but a struggling club pays a premium (and its stars may walk)
- **Play the postseason game by game** — sim one game at a time and watch each series' line scores,
  or fast-forward a whole round
- Full postseason: Wild Card → Division Series → League Championship → World Series
- Multi-season franchise loop: player aging/development, retirements, expiring contracts,
  a 5-round amateur draft, and season rollover

## Tech

React 18 + Babel-standalone + Tailwind, all via CDN. Everything lives in `index.html`.
See [CLAUDE.md](CLAUDE.md) for architecture notes.

## License

MIT
