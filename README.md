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
   rows into one row per *game*, splits the dual-purpose Close column into a
   spread value and a total value, uses the moneylines to attribute the
   spread to whichever team is favored (`away_spread` / `home_spread`), and
   resolves the Date column into a real `datetime` (see below). The Open
   column is fetched but currently dropped — it isn't carried into the
   parsed output.
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
| **Open** | Opening betting line. Fetched but currently dropped during parsing — not carried into the parsed output |
| **Close** | Closing betting line (right before kickoff). **Dual-purpose column**: for an away/home pair, one row holds the point spread and the other holds the game total (over/under). Disambiguated by magnitude — the larger absolute value is the total, the smaller is the spread |
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
| `away_spread` / `home_spread` | Closing point spread, attributed to whichever team it favors: the favored team (the one with the more negative moneyline) gets the spread value, the other team gets `0.0`. A tied moneyline (no favorite by ML) defaults the spread to `away_spread`. Values are always positive (no sign) — the smaller the number, the more favored that team is |
| `close_total` | Over/under total at close |
| `season` | Season label (e.g. `2021-22`) — added by the Pulling data loop, not by `parse_games` itself |
| `home_win` | Added by `add_home_win()`: `1` if `home_score > away_score`, `0` if the home team lost, `NaN` on a tie or missing score |

## Analysis (`analysis/conversions.ipynb`)

Reads `nfl_odds_2007-2022.csv` and derives implied probabilities/scores from
the raw lines. Each function mutates the shared `raw_data` DataFrame in
place, adding its own columns:

| Function | Columns added | What it does |
|---|---|---|
| `ml_vig_prob()` | `vig_prob`, `home_ml_prob`, `away_ml_prob` | Converts each team's moneyline to a raw implied probability via `ml_implied_prob()` (favorite and underdog moneylines use different formulas, picked by the odds' own sign — not by home/away), then normalizes `home_ml_prob`/`away_ml_prob` to remove the vig. `vig_prob` is the bookmaker's overround before normalizing |
| `spread_prob()` | `home_spread_prob`, `away_spread_prob` | Fits a logistic regression of `home_win` on `away_spread` + `home_spread` and predicts each team's win probability from the closing spread. Using both columns (rather than a single unsigned spread) lets the model learn which team a given spread favors |
| `implied_score()` | `h_impScore`, `a_impScore` | Backs out each team's implied score from `close_total` and the spread: `(total + your_spread - opponent_spread) / 2`. The two always sum back to `close_total` |

### Does the market agree with itself? (`home_prob_diff`)

`home_prob_diff = home_ml_prob - home_spread_prob` measures how much the
moneyline and the closing-spread model disagree about the home team's win
probability. `dis_model` fits `Logit(home_win ~ home_prob_diff)` to test
whether that disagreement itself predicts the outcome (H0: no relationship —
a fully efficient market should show none).

Result: the coefficient is negative and statistically significant
(p = 0.002) — larger disagreement (moneyline more bullish on the home team
than the spread model) is associated with a lower actual home win rate — but
the effect is small (Pseudo R² = 0.0017) and the disagreement itself is tiny
in practice: mean |`home_prob_diff`| is 0.019, and the two models only pick
different favorites in 26 of 4025 games. Splitting games into 8 equal-sized
buckets by `home_prob_diff` and plotting each bucket's actual home win rate
with a 95% Wilson confidence interval against the baseline home win rate
shows every bucket's interval overlapping both the baseline and its
neighbors — i.e. not something the eye can distinguish bucket-by-bucket at
this sample size, consistent with the tiny Pseudo R².

| Function | Output | What it does |
|---|---|---|
| `plot_disagreement_bins(n_bins=8)` | `disagreement_bins.png` | Bins games by the **rank** of `home_prob_diff` (not its raw value) into `n_bins` equal-sized buckets — a plain value-based `qcut` produces uneven bucket sizes here since `home_prob_diff` is rounded to 2 decimals and many games tie at the same value. Plots each bucket's actual home win rate, with a 95% Wilson confidence interval, against the overall baseline home win rate |

A `seaborn.histplot` of `home_prob_diff` (no dedicated function, inline in
the notebook) shows the distribution is tightly clustered around 0 (std
≈ 0.025) with a long, sparse tail out to ~0.44.
