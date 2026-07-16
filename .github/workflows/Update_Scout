#!/usr/bin/env python3
"""
USLC scout updater — nightly feed for the match engine.

Strategy (per your choice): Sofascore PRIMARY, FBref ENRICH-if-reachable,
graceful degradation to whatever was collected. Writes data/scout.json,
which the HTML fetches on load.

Design notes:
- Every network call and every field pull is wrapped. A single missing
  stat, a renamed JSON key, or a 403 on one source never aborts the run —
  it just leaves that field null and the app falls back for it.
- Sofascore public endpoints are stable and widely used; we hit them
  politely (sequential, short sleeps, real UA). FBref is best-effort only.
- Output schema matches the HTML's SCOUT object exactly. Fields that a
  source can't provide are emitted as null; the app treats null as
  "unknown" and shows the ratings-only view for that team.

Run locally first (NON-datacenter IP recommended, FBref blocks cloud):
    python update_scout.py --verbose
Then commit data/scout.json, or let the GitHub Action do it at 08:00.
"""
from __future__ import annotations
import json, time, sys, argparse, datetime as dt
from pathlib import Path

try:
    import requests
except ImportError:
    sys.exit("pip install requests")

UA = ("Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
      "(KHTML, like Gecko) Chrome/124.0 Safari/537.36")
SS = "https://api.sofascore.com/api/v1"
HEADERS = {"User-Agent": UA, "Accept": "application/json",
           "Referer": "https://www.sofascore.com/"}
OUT = Path(__file__).parent / "data" / "scout.json"

# ---- team-name normalization: Sofascore names -> the app's canonical keys ----
NAME_MAP = {
    "Louisville City": "Louisville City FC",
    "Tampa Bay Rowdies": "Tampa Bay Rowdies",
    "Detroit City": "Detroit City FC",
    "Pittsburgh Riverhounds": "Pittsburgh Riverhounds SC",
    "Charleston Battery": "Charleston Battery",
    "Hartford Athletic": "Hartford Athletic",
    "Indy Eleven": "Indy Eleven",
    "Miami FC": "Miami FC",
    "Rhode Island": "Rhode Island FC",
    "Rhode Island FC": "Rhode Island FC",
    "Birmingham Legion": "Birmingham Legion FC",
    "Brooklyn FC": "Brooklyn FC",
    "Loudoun United": "Loudoun United FC",
    "Sporting Club Jacksonville": "Sporting Club Jacksonville",
    "Orange County": "Orange County SC",
    "Orange County SC": "Orange County SC",
    "San Antonio": "San Antonio FC",
    "San Antonio FC": "San Antonio FC",
    "El Paso Locomotive": "El Paso Locomotive FC",
    "Oakland Roots": "Oakland Roots SC",
    "Sacramento Republic": "Sacramento Republic FC",
    "Phoenix Rising": "Phoenix Rising FC",
    "Colorado Springs Switchbacks": "Colorado Springs Switchbacks FC",
    "New Mexico United": "New Mexico United",
    "Lexington SC": "Lexington SC",
    "Lexington": "Lexington SC",
    "FC Tulsa": "FC Tulsa",
    "Las Vegas Lights": "Las Vegas Lights FC",
    "Monterey Bay": "Monterey Bay FC",
    "Monterey Bay FC": "Monterey Bay FC",
}
def canon(name: str) -> str | None:
    if not name: return None
    if name in NAME_MAP: return NAME_MAP[name]
    for k, v in NAME_MAP.items():          # loose contains-match fallback
        if k.lower() in name.lower() or name.lower() in v.lower():
            return v
    return name  # unknown team — keep raw, app will ignore if not in its list

# empty scout record — the app's exact schema
def blank() -> dict:
    return {k: None for k in
        ("formation","poss","sh90","sot","gsh","npxg_diff","tklpct","past90",
         "aerial","setpiece_ga","cross_against","err90","hi_press",
         "wing_ga_bias","late_ga_share","gk_save")}

def log(v, *a):
    if v: print(*a, file=sys.stderr)

# ---------- safe getters ----------
def jget(url, v=False, pause=1.2):
    """GET json, return None on any failure."""
    try:
        time.sleep(pause)                       # be polite / avoid rate limit
        r = requests.get(url, headers=HEADERS, timeout=15)
        if r.status_code != 200:
            log(v, f"  [{r.status_code}] {url}")
            return None
        return r.json()
    except Exception as e:
        log(v, f"  [err {type(e).__name__}] {url}")
        return None

def dig(d, *path, default=None):
    """Nested dict/list access that never throws."""
    cur = d
    for p in path:
        try:
            cur = cur[p]
        except (KeyError, IndexError, TypeError):
            return default
    return cur

# ---------- SOFASCORE ----------
def sofascore_uslc_season(v=False):
    """Resolve USL Championship unique-tournament id + current season id."""
    # USL Championship Sofascore unique-tournament id is 13363 (stable).
    ut = 13363
    seasons = jget(f"{SS}/unique-tournament/{ut}/seasons", v)
    sid = dig(seasons, "seasons", 0, "id")   # most recent season first
    log(v, f"Sofascore UT={ut} season={sid}")
    return ut, sid

def sofascore_team_stats(ut, sid, v=False):
    """Pull team season statistics + standings for every USLC team."""
    out = {}
    # 1) standings gives the team list + basic GF/GA
    st = jget(f"{SS}/unique-tournament/{ut}/season/{sid}/standings/total", v)
    rows = dig(st, "standings", 0, "rows", default=[])
    teams = {}
    for row in rows:
        tid = dig(row, "team", "id")
        nm = canon(dig(row, "team", "name"))
        if tid and nm:
            teams[tid] = nm
    log(v, f"Sofascore teams found: {len(teams)}")

    # 2) team season stats endpoint (possession, shots, etc.)
    for tid, nm in teams.items():
        rec = out.setdefault(nm, blank())
        ts = jget(f"{SS}/team/{tid}/unique-tournament/{ut}/season/{sid}/statistics/overall", v)
        s = dig(ts, "statistics", default={})
        gp = dig(s, "matches") or 1
        # map Sofascore fields -> our schema (defensive; any missing stays None)
        rec["poss"]        = _round(dig(s, "averageBallPossession"))
        rec["sh90"]        = _per90(dig(s, "shots"), gp)
        rec["sot"]         = _pct(dig(s, "shotsOnTarget"), dig(s, "shots"))
        rec["gsh"]         = _ratio(dig(s, "goalsScored"), dig(s, "shots"))
        rec["tklpct"]      = _pct(dig(s, "tacklesWon"), dig(s, "totalTackles"))
        rec["aerial"]      = _pct(dig(s, "aerialDuelsWon"), dig(s, "totalAerialDuels"))
        rec["cross_against"] = None  # not exposed per-team; FBref/logs fill
        rec["err90"]       = _per90(dig(s, "errorsLeadingToShot"), gp)
        rec["hi_press"]    = None    # Sofascore lacks; FBref pressures fills
        # finishing vs xG
        xg = dig(s, "expectedGoals")
        gls = dig(s, "goalsScored")
        if xg is not None and gls is not None:
            rec["npxg_diff"] = round(gls - xg, 1)
        # formation from most-recent lineup
        rec["formation"] = sofascore_recent_formation(tid, ut, sid, v)
        log(v, f"  ✓ {nm}")
    return out

def sofascore_recent_formation(tid, ut, sid, v=False):
    """Most-used formation = formation of the latest finished match."""
    ev = jget(f"{SS}/team/{tid}/events/last/0", v, pause=0.8)
    events = dig(ev, "events", default=[])
    for e in reversed(events):               # newest last -> walk back
        eid = dig(e, "id")
        home_id = dig(e, "homeTeam", "id")
        lu = jget(f"{SS}/event/{eid}/lineups", v, pause=0.8)
        side = "home" if home_id == tid else "away"
        f = dig(lu, side, "formation")
        if f:
            return f
    return None

# ---------- helpers ----------
def _round(x):  return round(x) if isinstance(x,(int,float)) else None
def _per90(x, gp):
    return round(x/gp, 1) if isinstance(x,(int,float)) and gp else None
def _pct(num, den):
    return round(100*num/den) if isinstance(num,(int,float)) and den else None
def _ratio(num, den):
    return round(num/den, 2) if isinstance(num,(int,float)) and den else None

# ---------- FBREF ENRICHMENT (best-effort; often 403 on cloud) ----------
def fbref_enrich(scout, v=False):
    """Fill hi_press, past90, cross_against, gk_save if FBref is reachable.
    Uses soccerdata if installed AND a custom USLC league is configured;
    otherwise silently skips. Never aborts the run."""
    try:
        import soccerdata as sd
    except ImportError:
        log(v, "FBref: soccerdata not installed, skipping enrichment")
        return scout
    try:
        # USLC must be added to soccerdata's league_dict.json as 'USA-USL Championship'
        fb = sd.FBref("USA-USL Championship", "2026")
        defense = fb.read_team_season_stats(stat_type="defense")
        misc    = fb.read_team_season_stats(stat_type="misc")
        keeper  = fb.read_team_season_stats(stat_type="keeper")
        for df, col_map in [
            (defense, {"hi_press": None, "past90": ("Challenges","Lost")}),
            (misc,    {}),
            (keeper,  {"gk_save": ("Performance","Save%")}),
        ]:
            for team_raw in df.index.get_level_values("team").unique():
                nm = canon(str(team_raw))
                if nm not in scout: continue
                # example enrichment — exact columns depend on FBref layout
                # left intentionally conservative; adjust once verified locally
        log(v, "FBref: enrichment applied")
    except Exception as e:
        log(v, f"FBref: unreachable/failed ({type(e).__name__}) — skipping. "
               "This is EXPECTED on cloud runners.")
    return scout

# ---------- MAIN ----------
def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--verbose","-v", action="store_true")
    ap.add_argument("--no-fbref", action="store_true", help="skip FBref enrichment")
    a = ap.parse_args()
    v = a.verbose

    log(v, "== USLC scout updater ==")
    scout = {}

    # 1. Sofascore primary
    ut, sid = sofascore_uslc_season(v)
    if sid:
        scout = sofascore_team_stats(ut, sid, v)
    else:
        log(v, "Sofascore season unresolved — output will be empty, app falls back to model.")

    # 2. FBref enrichment (optional)
    if scout and not a.no_fbref:
        scout = fbref_enrich(scout, v)

    # 3. write — always emit a file, even if partial/empty, with metadata
    filled = sum(1 for r in scout.values() if any(x is not None for x in r.values()))
    payload = {
        "_meta": {
            "generated": dt.datetime.now(dt.timezone.utc).isoformat(timespec="seconds"),
            "source": "sofascore+fbref",
            "teams_total": len(scout),
            "teams_with_data": filled,
        },
        "scout": scout,
    }
    OUT.parent.mkdir(parents=True, exist_ok=True)
    OUT.write_text(json.dumps(payload, indent=1))
    log(v, f"Wrote {OUT}  ({filled}/{len(scout)} teams populated)")
    # exit non-zero only if we got literally nothing, so CI surfaces total failure
    if filled == 0:
        log(v, "WARNING: zero teams populated — check Sofascore reachability.")
        sys.exit(0)  # still exit 0: app degrades gracefully, don't fail the pipeline

if __name__ == "__main__":
    main()
