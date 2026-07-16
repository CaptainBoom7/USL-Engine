# USLC Match Engine — auto-updating

A self-contained scouting + strategy terminal for USL Championship fixtures.
Ratings (Elo, xGe, Poisson projection) come from baked results data. Tactical
diagnostics (formation, 1v1, set-piece, aerial, press, finishing vs xG) come
from a **nightly feed** that a GitHub Action regenerates at 08:00 America/Detroit.

## How the data flows

```
08:00 daily ─► GitHub Action ─► update_scout.py ─► data/scout.json ─► index.html fetches it
                                   │
                     Sofascore (primary) + FBref (enrich if reachable)
```

- **Sofascore** is primary: reliable from cloud runners, provides possession,
  shots, SoT%, finishing vs xG, tackles, aerials, errors, and formation.
- **FBref** enriches when reachable (pressures, dribbled-past, keeper save%).
  FBref rate-limits datacenter IPs, so on GitHub's runners it often 403s —
  **this is expected and handled**; the run continues with Sofascore data.
- The app **degrades gracefully**: any field a source can't supply is filled
  from the model so no panel ever breaks. A totally empty feed → the app runs
  in placeholder mode. It can never hard-fail.

## One-time setup (~5 min)

1. **Create a GitHub repo** and push these four things:
   - `index.html`
   - `update_scout.py`
   - `data/scout.json` (ships empty; the Action fills it)
   - `.github/workflows/update.yml`

2. **Enable GitHub Pages**: repo → Settings → Pages → Source = "GitHub Actions".

3. **Enable Actions write access**: Settings → Actions → General →
   Workflow permissions → "Read and write permissions" → Save.
   (The workflow commits the refreshed `scout.json` back to the repo.)

4. **Add USL Championship to soccerdata** (only needed for FBref enrichment).
   soccerdata doesn't ship USLC by default. Create a
   `league_dict.json` and point the `SOCCERDATA_LEAGUE_DICT` env var at it,
   or skip FBref entirely — Sofascore covers most fields. To skip, the
   workflow already tolerates FBref failing; nothing to do.

5. **Trigger the first run**: Actions tab → "Update USLC scout data" →
   "Run workflow". After it finishes, your site is live at
   `https://<you>.github.io/<repo>/` and `data/scout.json` is populated.

## Verify locally first (recommended)

FBref and Sofascore both block many cloud IPs but work from a home IP.
Run the updater on your own machine to confirm the feed populates:

```bash
pip install requests soccerdata
python update_scout.py --verbose          # watch it hit Sofascore, fill teams
python -m http.server 8000                # serve the folder
# open http://localhost:8000 — pill should read "LIVE FEED · N/25 teams"
```

If `--verbose` shows `[403]` on Sofascore from your home machine too, add a
longer pause (raise `pause=` in `jget`) or run behind a residential proxy.
`--no-fbref` skips FBref if you only want the Sofascore fields.

## Schedule

The workflow runs at both 12:00 and 13:00 UTC and no-ops on whichever isn't
08:00 in Detroit, so **08:00 local holds year-round** across daylight saving
with no manual change. Edit the `TZ=America/Detroit` in `update.yml` to move it.

## Files

| file | role |
|---|---|
| `index.html` | the app — fetches `data/scout.json`, falls back to model if absent |
| `update_scout.py` | the nightly updater (Sofascore + FBref) |
| `data/scout.json` | generated feed the app reads |
| `.github/workflows/update.yml` | 08:00 cron + Pages deploy |

## Adjusting field mappings

If Sofascore renames a JSON key (rare, but scraping is scraping), the affected
field just goes null and the app falls back — nothing breaks. To fix it, update
the `dig(s, "…")` path in `sofascore_team_stats()`. Every extraction is
isolated, so one broken field never touches the others.
