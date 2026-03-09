# OSL Player Cards — Draft Season 1

A web-based FIFA-style player card generator for the Oceanic Slapshot League (OSL) Draft Season 1 in Slapshot Rebound.

Players can look up their name, upload a screenshot of their in-game character (with automatic background removal), and generate a downloadable player card with their calculated stats.

**Live site:** (add your GitHub Pages URL here)

---

## How It Works

### The Card
Each player gets a card styled after FIFA Ultimate Team cards, built entirely in HTML/CSS (no Photoshop template). Cards include:
- **Overall rating** (50–99) in the top left
- **Season label** ("Draft S1") and OSL logo below the rating
- **Player image** (uploaded by the user, background auto-removed)
- **Player name and team**
- **4 individual stat ratings** (50–99 each): SHO, AST, PAS, DEF
- **Rarity tier**: Gold (80+), Silver (65–79), Bronze (50–64) — each with a distinct colour scheme

### The Stats

Stats are sourced from https://oslstats.com/seasons/DS1/leagues/Draft/stats and baked into the site as a JSON object. There are 4 stat categories, all calculated as **per-period averages** to normalise for players who've played different amounts:

| Card Stat | Label | What It Measures | Raw Calculation |
|-----------|-------|-----------------|-----------------|
| SHO | Shooting | Goal scoring output | Goals / Periods Played |
| AST | Assists | Playmaking ability | Assists / Periods Played |
| PAS | Passing | Passing volume | Passes / Periods Played |
| DEF | Defending | Defensive contribution | (Saves×2 + Blocks) / Periods Played |

**Why saves are weighted double in DEF:** Saves require more skill/positioning than blocks and are rarer, so they're given extra weight to properly value goalkeeper-style play.

### The Scoring Algorithm

#### Step 1: Per-Period Averages
All raw stats are divided by periods played (PP) to normalise across players with different amounts of game time.

#### Step 2: Percentile Ranking (Individual Stats)
For each of the 4 stats, all 40 players are ranked against each other. Each player's percentile position is mapped to the 50–99 range:

```
score = 50 + round(percentile × 49)
```

So the best player in a stat gets 99, the worst gets 50, and everyone else is proportionally distributed. Ties are handled by averaging ranks.

This approach was chosen over z-scores/bell curves because with only ~40 players, the sample is too small for a normal distribution to be reliable, and outliers would compress everyone else into a narrow band.

#### Step 3: Weighted Base Score
Individual stats are combined using weights:

| Stat | Weight |
|------|--------|
| SHO  | 30%    |
| AST  | 20%    |
| PAS  | 20%    |
| DEF  | 30%    |

Shooting and defending are weighted higher as they represent the core offensive and defensive outputs.

#### Step 4: Specialist Boost
If a player's highest stat is 15+ points above the average of their other stats, they receive a boost to their weighted score. This prevents the system from undervaluing specialists (e.g., a player with 99 DEF but average offence).

The boost blends a portion of their top stat into their overall, scaled by how extreme the specialisation is:

```
gap = topStat - avgOfOtherStats
if gap >= 15:
    bonusStrength = min((gap - 15) / 20, 1) × 0.3
    boostedScore = baseScore + (topStat - baseScore) × bonusStrength
```

This means Eagle (99 DEF, ~68 avg in other stats) gets a meaningful bump, while a well-rounded player like Moses (all stats 85+) sees no change.

#### Step 5: Percentile-Ranked Overall
The boosted weighted scores are then **percentile-ranked again** among qualified players only, mapped to 50–99. This ensures the #1 player hits 99 and the spread fills the full range, rather than clustering in the 80s.

#### Step 6: Minimum Periods Threshold
Players with fewer than 10 periods played are flagged as "unqualified". Their overall is calculated relative to the qualified pool but **capped at 69** (Silver maximum). This prevents players with tiny sample sizes from getting inflated ratings from a few lucky periods.

#### Step 7: Rarity Assignment
- **Gold**: OVR 80–99
- **Silver**: OVR 65–79
- **Bronze**: OVR 50–64

### Background Removal
The site uses [@imgly/background-removal](https://github.com/niceBoys/background-removal-js), a client-side ML model that runs entirely in the browser (~40MB model, downloads on first use). No server or API key required. If the library fails to load, images are used as-is.

### Card Export
Cards are rendered as HTML/CSS and exported to PNG using [html2canvas](https://html2canvas.hertzen.com/). The download button snapshots the card div at 2× resolution.

---

## Project Structure

```
osl-cards/
├── index.html          # Everything: HTML, CSS, JS, stats data (single-file app)
├── assets/
│   └── osl-logo.png    # OSL league logo (used on cards + header)
└── data/
    └── stats.json      # Player stats as standalone JSON (for reference/updates)
```

The site is a single HTML file with no build step — just open `index.html` or deploy to GitHub Pages.

---

## Deploying to GitHub Pages

1. Create a new GitHub repository
2. Push the project files (index.html, assets/, data/)
3. Go to Settings → Pages → Source: "Deploy from a branch" → Branch: main, folder: / (root)
4. Your site will be live at `https://yourusername.github.io/repo-name/`

---

## Updating Stats

Stats are currently hardcoded in the `STATS_DATA` object inside `index.html`. To update:

1. Visit https://oslstats.com/seasons/DS1/leagues/Draft/stats
2. Copy the "All Stats" table data
3. Update the `players` array in `STATS_DATA` with new values
4. The scoring engine recalculates everything automatically on page load

**Future improvement:** If API access to oslstats.com becomes available, the site could fetch stats dynamically instead of requiring manual updates.

---

## Tuning the System

Key constants at the top of the scoring engine in `index.html`:

```javascript
const WEIGHTS = { SHO: 0.30, AST: 0.20, PAS: 0.20, DEF: 0.30 };
const MIN_PERIODS = 10;           // Minimum periods for full rating
const SPECIALIST_THRESHOLD = 15;  // Gap needed for specialist boost
const SPECIALIST_BLEND = 0.3;     // Strength of specialist boost
```

Rarity thresholds are in the `computeScores` function:
```javascript
if (p.OVR >= 80) p.rarity = 'gold';
else if (p.OVR >= 65) p.rarity = 'silver';
else p.rarity = 'bronze';
```

---

## Tech Stack

- **HTML/CSS/JS** — no frameworks, no build step
- **Google Fonts** — Bebas Neue, Teko, Rajdhani
- **html2canvas** — card-to-PNG export (loaded from CDN)
- **@imgly/background-removal** — client-side background removal (loaded from CDN)
- **GitHub Pages** — free static hosting

---

## Credits

- Stats from [OSL Stats](https://oslstats.com) by Haelnorr
- Built for the Oceanic Slapshot League community
