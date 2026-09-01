# odds-analysis

Tools for ingesting historical NFL betting odds from
[sportsbookreviewsonline.com](https://www.sportsbookreviewsonline.com/) and
reshaping them into one row per game.

## Setup

```bash
pip install requests pandas lxml
```

`lxml` is required by `pandas.read_html()` to parse the odds tables — without
it, `fetch_season_table` raises `ImportError: lxml not found`.

## Ingest pipeline (`raw_data/ingest.ipynb`)

1. **`fetch_season_table(url)`** — downloads a season's archive page and
   extracts the largest HTML table on it (the odds table). The site renders
   its header as a plain data row instead of `<th>` cells, so the function
   promotes row 0 to column headers when pandas hasn't picked them up.
2. **`parse_games(raw, season_start_year)`** — the raw table has one row per
   *team* (away and home rows stacked back to back); this pairs consecutive
   rows into one row per *game*, splits the dual-purpose Open/Close columns
   into separate spread and total values, and resolves the Date column into
   a real `datetime` (see below).
3. **Pulling data** — loops `season_url()` + `fetch_season_table` +
   `parse_games` over every season in `SEASON_START_YEARS` (2007-08 through
   2021-22), tags each game with its `season` label, concatenates everything,
   and saves it to `nfl_odds_2007-2022.csv` next to the notebook.

### Multi-season dates

The site encodes Date as an integer `mmdd` with **no year** (e.g. `906` =
Sep 6, `1216` = Dec 16, `203` = Feb 3) — month/day fall out of `divmod(x,
100)` regardless of digit count. Since the year isn't in the data at all,
`parse_games` infers it from `season_start_year`: Sep–Dec dates belong to the
season's start year, Jan–Jun dates (playoffs / Super Bowl) belong to the
following year. Verified against actual Super Bowl dates (e.g. 2007-08 season
correctly ends 2008-02-03, the date of Super Bowl XLII).

## Raw column reference

The source table has one row per team per game (an away row immediately
followed by its matching home row):

| Column | Meaning |
|---|---|
| **Date** | Game date as `mmdd`, no year (e.g. `909` = Sept 9) — year is inferred during parsing, see [Multi-season dates](#multi-season-dates) |
| **Rot** | Rotation number — a unique ID per team-in-a-game. Away/home rows are consecutive (e.g. 451/452), which is how rows get paired into a game |
| **VH** | `V` = visiting (away) team's row, `H` = home team's row |
| **Team** | Team name/abbreviation |
| **1st / 2nd / 3rd / 4th** | Points scored by that team in each quarter |
| **Final** | Final score for that team |
| **Open** | Opening betting line. **Dual-purpose column**: for an away/home pair, one row holds the point spread and the other holds the game total (over/under). Disambiguated by magnitude — the larger absolute value is the total, the smaller is the spread |
| **Close** | Same dual-purpose spread/total idea as Open, but the closing line (right before kickoff) |
| **ML** | Moneyline odds, American format: negative = favorite (amount you must bet to win $100), positive = underdog (amount you win per $100 bet) |
| **2H** | Second-half line — same dual-purpose spread/total format, but for second-half-only betting. Renamed to `second_half` during parsing but not currently split into its own spread/total |

Special values (handled by `_to_num`):
- `pk` → `0.0` (pick'em / no favorite)
- blank or `NL` → `NaN` (no line offered)

## Parsed output (`parse_games`)

Each row of the returned DataFrame is one game:

| Column | Meaning |
|---|---|
| `date` | Game date as a real `datetime` (year resolved from `season_start_year`) |
| `away_team` / `home_team` | Team names |
| `away_score` / `home_score` | Final scores |
| `away_ml` / `home_ml` | Moneyline odds for each team |
| `open_spread` / `close_spread` | Point spread at open/close |
| `open_total` / `close_total` | Over/under total at open/close |
| `season` | Season label (e.g. `2021-22`) — added by the Pulling data loop, not by `parse_games` itself |
