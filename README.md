# Football Calendar

A Telegram Web App showing a football fixture calendar with per-match forecasts
and a running accuracy counter.

> **The live demo does not work.** The backend that serves the data is not
> published, so the app loads and then reports that it could not fetch matches.
> This repository is here for the front-end code, not as a working demo.
>
> All forecast figures and accuracy numbers are **synthetic**. Nothing here
> implements or claims a real predictive model.

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
          "prob": 72,              // forecast confidence, %
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

Vanilla JavaScript, HTML and CSS in one file. Telegram Web App SDK for the
in-Telegram chrome. No framework, no build step.

Point it at a backend by setting `window.API_URL` before the script runs;
it falls back to `http://localhost:5000`.
