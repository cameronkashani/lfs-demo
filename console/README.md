# The admin console, as a working prototype

**Open `console.html` in any browser. No install, no server, no network.**

Then press **BUILD NOTES** in the top bar. Every stage gets an annotation panel underneath it
explaining where its data comes from, what is faked in this file, and what you have to build.

---

## What this is

The six stage console from the earlier spec pack, running. Cameron and I built a card with it
against live data for Thursday 20 August, and it is loaded with that night's real slate.

Click a matchup on stage 1, pick a wrinkle on stage 2, and watch stages 3 and 4 rewrite
themselves. That is the whole idea: **the operator makes two choices and the system does the rest.**

---

## What is real and what is not

**Real.** Every number came back from a live BALLDONTLIE call on 19 August 2026.

- The two preseason games, their venues and kickoff times
- Both rosters, from the DraftKings slate, with per player status flags
- Every player's 2025 season line
- The 2025 defensive rankings, computed from 17,777 player game rows
- The six league wide effects in the wrinkle scoring, measured the same way
- The betting totals, which were updating while we tested

**Faked.** Two things only.

- The candidate list is frozen. In production it is rebuilt at load from three calls.
- The effects table has six rows instead of a few hundred.

**Deliberately broken.** The art gate fails on every row, because it should. The football world
and the weather scene are the only two pieces of art that exist. There is no action sprite, so
beats 1 and 2 have nothing to scrub. That is not a bug in the prototype, it is the actual state
of the project and the console is right to block it.

---

## The endpoints behind each stage

Base URL is `https://api.balldontlie.io`, auth is `Authorization: Bearer <key>`.

| Stage | Call | Note |
|---|---|---|
| 1 | `GET /nfl/v1/games?seasons[]=2026` | Filter to tomorrow **in code** |
| 1 | `GET /nfl/v1/dfs/slates` then `/dfs/draftables?slate_id=` | The only NFL roster source |
| 1 | `GET /nfl/v1/season_stats?season=2025&player_ids[]=` | Max 8 ids per call |
| 1 | `GET /nfl/v1/odds?season=&week=` | Internal only, never shown |
| 1 | `GET /nfl/v1/player_injuries` | The availability gate |
| 2 | `GET /nfl/v1/stats?seasons[]=` | The game log, for the effects job |
| 2 | `GET /nfl/v1/team_stats?season=` | Team level per game rows |
| live | `GET /nfl/v1/plays?game_id=` | Singular, no brackets. Runs live during a game. |
| MLB | `GET /mlb/v1/lineups?game_ids[]=` | The only home of `is_probable_pitcher` |

---

## Six traps that cost me time

1. **`dates[]` does not find NFL preseason games.** The Raiders at Texans game exists on
   2026-08-21 and loads fine by id, but `dates[]=2026-08-21` returns zero rows. `weeks[]=3`
   returns the sixteen regular season week 3 games instead.

2. **`per_page` over 100 returns an empty array with a 200.** Not an error. Cap at 100.

3. **`season_stats` silently drops players on large id batches.** 20 ids returned 65 of 126
   players. Use 8.

4. **Parameter style is not consistent.** Most filters want the bracket array form,
   `team_ids[]` and `game_ids[]` and `dates[]`. But `plays` wants `game_id` singular, and
   NFL odds wants `season` and `week`. **A 400 means the endpoint exists and you called it
   wrong. Read the body, it names the parameter.** I wrote off four working endpoints as
   missing before I learned this.

5. **Innings pitched is not a decimal.** MLB only. `6.1` means six and one third. Naive float
   maths gave me a 3.96 ERA for a 3.77 pitcher. Convert once, at ingest.

6. **The game log includes in-progress games, season stats do not.** At 20:00 ET games are
   running, so any season average computed naively from the log is polluted by tonight's
   half finished lines. Filter on game status before aggregating.

---

## What does not exist and never will

Confirmed absent for both NFL and MLB, under every path variant including v2 and the opening
forms: box scores, leaders, player props, depth charts, advanced stats, season averages by
category, contracts. All of them exist for NBA only. None is load bearing for us.

NFL has **no lineups endpoint**, so there is no equivalent of `is_probable_pitcher` and no
inactives list. Rosters come from the DFS draftables instead.

---

## The three things to build first

1. **The effects table and its weekly job.** Everything on stage 2 reads from it. Nothing
   statistical should run while an operator waits.

2. **The asset registry.** Three states per asset: DRAFT, APPROVED_FOR_BUILD, SHIPPED_IN vN.
   Only SHIPPED counts for the gate, because art lives in the app binary and needs a release.

3. **The re-check and void path.** Between publish and lock a named participant can disappear.
   Sportsbooks solve this with a rule rather than with timing: a player prop on a pitcher who
   does not start is voided. Our card is the same shape. A void preserves the streak and issues
   no receipt.

---

## Open questions for you

- Where does the asset registry live, and who updates it when a build ships?
- Is field level locking worth it in v1, or do we accept that changing the wrinkle wipes edits?
- Do we record the betting line nightly starting now? There is no historical odds data in this
  API at all, so every night we skip is a night we cannot backtest later. It is one small job
  and it is impossible to recover.
