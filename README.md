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

Your franchise saves automatically to the browser (localStorage).

## Features

- Pick from 30 fictional teams in two leagues, then play a 162-game season
- Sim controls: day / week / month / straight to the playoffs
- Roster management + editable batting order and starting rotation
- Inning-by-inning, plate-appearance sim engine driven by player ratings — produces realistic stat lines
- Standings and full stat leaderboards (AVG, HR, RBI, OPS, ERA, K, W, SV, WHIP, …)
- Trades with value-based AI, free agency with a salary cap, contracts & budget view
- Full postseason: Wild Card → Division Series → League Championship → World Series
- Multi-season franchise loop: player aging/development, retirements, expiring contracts,
  a 5-round amateur draft, and season rollover

## Tech

React 18 + Babel-standalone + Tailwind, all via CDN. Everything lives in `index.html`.
See [CLAUDE.md](CLAUDE.md) for architecture notes.

## License

MIT
