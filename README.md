# Football Calendar

A Telegram Web App showing a football fixture calendar with per-match forecasts
and a running accuracy counter.

> **The live demo does not work.** The backend that serves the data is not
> published, so the app loads and then reports that it could not fetch matches.
> This repository is here for the front-end code, not as a working demo.
>
> All forecast figures and accuracy numbers visible in the front end are
> **synthetic placeholders**.

## What the front end does

- Month tabs, with data reloaded per month
- Filtering by league — Premier League, Bundesliga, La Liga, Serie A, Ligue 1
- A match card per fixture: kick-off time, teams, forecast probability
- Cards colour-code themselves by confidence, and by whether a finished match
  matched its forecast
- Expandable detail per match: predicted winner, first-goal player, minute and
  team, injury notes for both sides
- A header badge with overall accuracy
- Loading and empty states, and a failure state — which is what you will
  actually see on the live link

Rendering is a single `render()` pass over the current state, so filtering and
expanding a card never touch the network.

---

## The system behind it

The front end is the thin part. Behind these two endpoints sits a data pipeline
and a model ensemble.

### Ensemble

Five classifiers produce a match result, and N5 is an aggregator stacked on top
of the other four rather than another view of the raw features:

| | Model | Reads |
|---|---|---|
| N1 | Form | rolling team form |
| N2 | Head-to-head | historical meetings between the two clubs |
| N3 | Market | bookmaker-implied probabilities |
| N4 | Combined | the full feature set |
| N5 | **Aggregator** | the outputs of N1–N4 |

Two further regressors estimate goals scored by each side, which feeds the
scoreline model below.

Training set: roughly **16.9K matches** across five domestic leagues plus seven
seasons of UEFA club competition.

### Validation, honestly

Cross-validation uses **5-fold `TimeSeriesSplit`** — folds respect chronology,
so the model is never validated on matches that precede its training data.
Ordinary k-fold would leak the future into the past and inflate every number
here.

The aggregator's cross-validated accuracy is **58.7%** on three-way outcomes
(home / draw / away). That is a cross-validation figure on historical data, not
a claim about future matches, and the system makes no claim about betting
returns of any kind.

### Features

- **Form** — points per match, win/draw/loss rates, goals for and against,
  goal difference, shots on target, and a recency-weighted form score
- **Head-to-head** — historical record and goal averages between the two clubs
- **Motivation** — table position, distance to the title race and to the
  relegation zone, days of rest since the last fixture, and separate home and
  away win rates
- **UEFA club coefficients** — and the difference between the two sides
- **Expected goals (xG)** — rolling averages
- **Availability** — injuries and suspensions

### Scoreline and first goal

Goals for each side are modelled as independent Poisson processes with rates
λ_home and λ_away from the goals regressors. The probability of any exact
scoreline follows from the pair, which also yields an estimate of when the first
goal falls and which side scores it.

### Market as a sanity check, not as an oracle

Odds are aggregated across roughly **18 bookmakers** per fixture. The final
probability is a weighted blend — the ensemble carries the larger share, the
market the rest — so the market can pull a prediction back but cannot originate
one.

Publication is gated:

- a forecast is published only at **≥60% ensemble confidence** confirmed by the
  odds
- **away favourites face a stricter bar of 65%**, because an away side favoured
  by the model but not by the market is the shape that fails most often
- fixtures below the threshold are still **listed, but without a prediction** —
  the absence of a forecast is itself information, and hiding those matches
  would quietly flatter the hit rate

A home-favourite flag feeds the same gate.

### Pipeline

Collectors pull fixtures, results, lineups, injuries, xG and odds into SQLite
(sixteen tables, with migrations). A feature build turns raw matches into the
model's inputs; training produces the ensemble; scheduled runs generate and
publish forecasts; a result watcher reconciles each published forecast against
the final score, which is where the accuracy counter comes from.

An LLM step reads match news and adjusts the forecast where something material
— a manager change, a late fitness update — has not reached the structured
feeds yet.

> **The backend and pipeline are not published yet.**

---

## Backend contract

The app expects two endpoints:

```
GET /api/matches?month=<current|next|prev>
GET /api/stats
```

`/api/matches` returns days, each with its matches:

```jsonc
{
  "days": [
    {
      "date": "2026-08-10",
      "matches": [
        {
          "id": "...",
          "time": "19:00",
          "league": "🏴 Premier League",
          "home": "...", "away": "...",
          "prob": 72,              // ensemble confidence, %
          "winner": "...",
          "fg_player": "...", "fg_minute": 23, "fg_team": "...",
          "injuries_home": "...", "injuries_away": "...",
          "result": null           // or { "score_h": 2, "score_a": 1, "ok": 1 }
        }
      ]
    }
  ]
}
```

`/api/stats` returns `{ "total": 0, "correct": 0, "accuracy": 0 }`.

## Stack

Front end: vanilla JavaScript, HTML and CSS in one file, Telegram Web App SDK.
No framework, no build step.

Backend: Python, Flask, SQLite, scikit-learn.

Point the front end at a backend by setting `window.API_URL` before the script
runs; it falls back to `http://localhost:5000`.
