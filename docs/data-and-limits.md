# Data and limits

This document covers what data the project runs on, how it is handled, and where it falls short.
Counts are from the database as it stands in October 2026. The last game in it was played in June
2026, and none have been played since. The models train on 2022-23, 2023-24 and the 2025-26 regular
season and are scored on 2024-25 (section 1.1 of the findings). Fixes marked "proposed" are my suggestions and are not scheduled work.

## Short version

- No live forecast has been scored yet. Every result comes from past games, and the first live slates
  are the first real test.
- Box scores are missing for 154 of the 5,270 games, including the whole 2025-26 playoffs, so the
  models never see those games.
- The official inactive lists are empty at the source for 20 games of the 2024-25 playoffs. The
  holdout includes 1,804 playoff rows where absence features under-count, and I have not measured the
  effect on the scores.
- The served models cannot be recreated by retraining, and the backups share a disk with the
  originals.
- Two operational traps: a clean rebuild yields three seasons, and a missing `.env` quietly changes
  which model is served.

What I would trust and what I would not: the accuracy and range numbers in the findings are measured
on games the models never saw, and I would rely on them as they stand. I would not yet rely on live
behavior, which is unproven, on absence-based features for playoff games, or on the width of the
assists 95% range.

## 2.1 What data I use

| Source | What it provides | How it arrives | Coverage and gaps |
|---|---|---|---|
| Game schedule and results | Teams, dates, scores, season and game type | The league game-log endpoint of [nba_api](https://github.com/swar/nba_api) | 5,270 games from October 2022 to June 2026: regular season, play-in and playoffs, plus the 2023-24 in-season tournament games, which are tagged separately |
| Box scores | Minutes and box-score stats for every player in every game | The player game-log endpoint | 132,685 player-game rows for 882 players across four seasons. 154 games have none: the 67 in-season tournament games of 2023-24, all 85 games of the 2025-26 playoffs, and two 2024-25 regular-season games that have no result in the schedule. The per-player pull defaults to the regular season, which probably explains the 2025-26 gap |
| Official inactive lists | Which players were ruled inactive for each game, including healthy scratches and two-way players | The box-score summary endpoint: version 2 before 10 April 2025, version 3 from that date | 42,002 rows from October 2022 to May 2026. 34 playoff games list nobody (20 in 2024-25 and 14 in 2025-26) and 3 regular-season games have no list (see 2.2 and 2.3) |
| Injury reports | A status such as "Out" for players in upcoming games | A public injury feed, pulled each night before forecasting | 2,906 rows, starting March 2026. It covers tonight's games, which have no inactive list yet |
| Preseason appearances | Which players appeared for which team in preseason games | The game-log endpoint, limited to September and October | 7,850 rows over four seasons. Used to build a roster at the start of a season, never as a feature |
| Player information | Team, active flag, height, weight, draft details | The player-info endpoint | 882 players. Position is missing for 318 of them |

The game data is NBA.com's, accessed through nba_api, and is used here for a personal, non-commercial
project. Both sources are unofficial, undocumented endpoints that can change or stop responding
without notice, and none of the data is redistributed.

Training uses the official inactive lists to know who was out. For tonight's games there is no
inactive list yet, so serving uses the injury report's "Out" players. The two do not mean exactly the
same thing, and that difference is listed in 2.3.

Several things are deliberately left out.

- Player tracking data. Coverage for 2022 to 2024 is incomplete, and it is licensed separately, as I
  understand it.
- Shot-chart zone profiles. Reliability falls as zones get finer, the best case is under the 0.5%
  improvement I require (section 1.3 of the findings), and ingestion would be expensive.
- Referee assignments. The feature block built on them gave no gain once its leak was fixed, so the
  block and its ingestion were removed. The table is still in the database.
- Preseason minutes as features. They are unrepresentative by design, and a feature-invariance test
  checks that they never change a feature.
- Anything inferred about absences from games played later. A function that did this used future
  games and was deleted.
- Seasons before 2022-23. The pulls and loaders assert this floor.

## 2.2 How it is handled

Pulls go through nba_api with retries and exponential backoff. Each run has a cap on the number of
requests, and a `pull_status` table records every endpoint, entity and season, so an interrupted pull
resumes where it stopped (the table holds 6,734 entries: 6,697 complete, 34 complete but empty, and 3
errors). Everything lands in one local [DuckDB](https://duckdb.org) file, and every write is checked
against a schema first. After a bulk load, an audit script checks the raw tables, including that date
filters return the same rows as a full scan.

The nightly run pulls new games and injury reports, rebuilds features with the same code used for
training, and forecasts the games that night. One rehearsal on my laptop took about 16 minutes, most of
it the feature rebuild. The run always does the full feature build, because an incremental build once
disagreed with it (section 1.3 of the findings). For tonight's games, the roster is the players who
appeared for the team in the previous 30 days, minus anyone listed out. A team with no
regular-season player in that window is filled from preseason appearances.

Models and the database are never committed. Neither are the evaluation outputs. A script archives
the model files outside the repository with a checksum manifest and refuses a destination inside it;
the set is about 120 files, 96 MB, 49 MB compressed, counting the live model files and not older backup copies. Retraining cannot recreate the served models exactly, because
NGBoost training is not bit-reproducible, so those backups matter. The project needs no credentials
(nba_api takes no key), and a test fails if a credential-shaped setting or an API-key literal ever
appears.

Three incidents shaped how the pipeline works.

The inactive lists went silently empty. The box-score summary endpoint stopped returning inactive
players for games from 10 April 2025, while its newer version still did. My pull parsed the empty
response, stored nothing, and marked each game complete. Nothing failed. By the time I looked, 1,036
games of 2025-26 were marked complete with no data. I saw it in a feature, not in an error: from
November to February the share of player rows with at least one teammate out was between 0.97 and 1.00
in the three earlier seasons, and between 0.00 and 0.12 in 2025-26. I confirmed the cause by pulling
one game from both versions: the old one returned no inactive players and the new one returned 12.
The pull now uses the newer version from that date, has three explicit statuses (complete, complete
but empty, error), and no longer fails open. The repair reset 1,161 games (the 1,036 from 2025-26 and the 125 played from 10 April 2025 on in 2024-25) and wrote 12,272 inactive rows, after which the absence share for 2025-26 regular-season months is 0.96 to 1.00 and all 96
integrity checks pass. Gaps remain, all at the source: 34 playoff games list nobody (20 in 2024-25 and
14 in 2025-26, including every game after 30 May 2026), and three 2025-26 regular-season games return
no payload. None of the 14 in 2025-26 has box scores, so no training row depends on them. The models now serving were trained on features rebuilt after the repair. The fixed
holdout is not clean on this point: 1,804 of its 28,097 rows are 2024-25 playoff games, where absence
features under-count, and I have not measured the effect on the scores. 

Opening night would have produced no forecasts. The roster for a night is built from appearances in
the previous 30 days, and the offseason gap is 121 to 191 days, so on opening night every team came back
empty. The pipeline logged one warning and returned an empty table, which looks the same as a night
with no games. I found it while pinning down the 2026-27 calendar. The fix fills a team from preseason
appearances only when it has no regular-season player in the window; an unconditional union would have
changed one of the parity dates by 138 players. Replaying the 2024-25 opening night through the serving
path now gives 448 forecasts (64 players, seven stats, no missing expected values). An early-warning check also
flags an empty upcoming slate.

The injury feed started refusing requests. On the first run of a new rehearsal script, the feed
answered my Chrome/120 User-Agent header with HTTP 403 and the whole nightly run aborted at that step.
I tried other strings: only the default client agent got 200, while Chrome 120, Chrome 140 and a
project-named string all got 403, so a newer Chrome string would not have helped. Other headers made no
difference. The pull now sends no custom User-Agent, and a test pins that. Injury reports drive the
absence features, so this would have broken the first live night.

## 2.3 Current limits and possible fixes

The tables group the limits by what they touch. "Cost" is what the limit is measured to cost today.

### Data coverage

| Limit | What it affects | Cost | Possible fix |
|---|---|---|---|
| 154 games have no box scores | Training rows, and any analysis of those games | The 2023-24 in-season tournament games, the whole 2025-26 playoffs, and two 2024-25 regular-season games with no result. 2025-26 training rows are regular season only | Pull the 2025-26 playoff logs, since the per-player pull defaults to the regular season, and check the two games with no result (proposed) |
| Official inactive lists are empty for some playoff games | Absence features | 34 playoff games list nobody. The 20 in 2024-25 have box scores and feed the holdout. The 14 in 2025-26 have none, so no row depends on them. An average of 4.0 inactive rows on a game's date in the 2024-25 playoffs, against 6.0 and 6.2 in earlier ones; 3 regular-season 2025-26 games have no list | Nothing better exists at the source. Regular seasons are complete, and playoff absence features are known to under-count |
| The inactive list and the injury report mean different things | Absence features in training against serving | Not measured | Measure the overlap on games that have both, then decide whether to map the report onto the list's definition (proposed) |
| Position is missing for many players | Position features | Missing for 318 of 882 players, which is 18.7% of game-log rows. The position columns are constant. A correct position signal measured worth about 0.05% at most | Derive player types from box-score history instead of the position field. Low priority given the size |

### Features and models

| Limit | What it affects | Cost | Possible fix |
|---|---|---|---|
| Usage features use different rosters in training and serving | Four usage columns, the only four of 68 that differ between training and serving rows | No measurable cost on the 231 rows compared (findings 1.4). The projected roster differs from who played on 145 of them, by up to 27.9 minutes | Define an expected roster before the game and use it in both paths. Planned for the next full retrain; the accuracy gain is expected to be about zero |
| Usage-column checks enforce only a lower bound | Data validation | One column is a team total despite its name, with a value of 343 minutes when 14 teammates were out. The 0 to 240 bound first proposed would have failed 189 rows | Set real bounds and rename the column, which needs a retrain because the name is stored in the model files |
| The earlier model design has not been re-evaluated | Whether the frozen two-layer stack (three XGBoost models per stat feeding one NGBoost model) would be competitive if retrained on 2026-27 data | Not established | Score it again in the stack arm of the walk-forward tests, and remove its serving path if it no longer helps |

### Evaluation

| Limit | What it affects | Cost | Possible fix |
|---|---|---|---|
| No skill baseline in the same family | Any skill score on the five hurdle-modeled stats | Assists' skill rose from 9.8% to 11.3% while CRPS moved by 0.73%, because that baseline cannot predict a zero | A baseline built in the same family (planned, with no design yet). Until then I compare by MAE and CRPS |
| The bootstrap widening figure is old | The interval claim in findings 1.3 | Measured once, in the first release gate, before the feature regeneration | Have the gate print clustered and row-level widths on every run (planned) |
| The assists 95% range is wider than it needs to be | The 95% range for assists | Averages 9.08 wide, about the range holding 95% of all assist totals, against 5.05 before assists moved to the hurdle form. Coverage (95.2%), MAE and CRPS are not affected | Change the fit so it picks the narrowest range within a small coverage tolerance, then refit and re-run the holdout evaluation (planned) |
| A 50% range exists only for points | Any 50% claim | On replayed slates the nearest whole-number 50% ranges covered about 51% for points and 60% or more for the other six stats | Inherent to whole-number ranges, so the fix is to publish only 80% and 95% for the other six |
| No live forecast has been scored | The monitor's settled-error, coverage and zero-rate checks | They have only run on synthetic rows | Wait for the first live slates, then update findings 1.1 |

### Operations

| Limit | What it affects | Cost | Possible fix |
|---|---|---|---|
| A clean rebuild yields three seasons, not four | The backfill defaults and its valid-season check | The default pull stops at 2024-25, and 2025-26 needs `CURRENT_SEASON` set | Add 2025-26 to the defaults and the valid set, or stop deriving the valid set from today's date |
| Missing settings change results quietly | Retraining and serving | An empty routing list serves the older model, and a retrain with the hurdle flag off drops the hurdle model. Serving already refuses to start if hurdle classifiers exist and the flag is off, and the Dockerfile sets both values (the image has never been built, so that is untested) | Planned: make retraining fail loudly and warn on an empty routing list. The defaults stay, because an empty list is the tested rollback |
| Retraining is not bit-reproducible | Recreating served models | Identical fits differ by up to 0.054 in expected value and 0.087 in spread per row, about 0.014% CRPS in aggregate | Keep the model backups, and compare arms within one run |
| Backups share the disk by default | Loss of the disk loses originals and backups | The archive is about 120 files and 96 MB | Point the backup setting at another disk. I decided against an off-disk copy for now, and the nightly backup job exists only when the setting is set |
| External feeds can change without notice | The injury pull and the nightly run | The injury feed refused one header and aborted a nightly run. The library's issue tracker also has reports of timeouts and blocks from cloud-hosted addresses such as AWS, which matters if I run this on an instance | I cannot prevent it; the rehearsal and a header test would flag it. For the injury pull I am looking for a source with clear terms for automated use (the official NBA injury report and a paid API tier are the candidates) |
| A DuckDB date filter once returned nothing | Reads on some tables | Silently returned no rows for some row groups; fixed by rebuilding the tables, and the root cause is unknown | Run the integrity audit after bulk writes, since it checks date-filter parity |
| The opening-night roster path has not run on 2026-27 data | The first live slate | Untested | Run a rehearsal once the 2026-27 preseason games are in |
| The container image has never been built | Deployment | Untested | Build it on a machine with Docker, which matters only if I deploy it |

### Minor

- A team-level feature conditions on who played the game. It feeds only an experimental matchup path
  that is not served, so it does not affect forecasts. The same expected-roster definition would fix it.
- The stored 50%, 80% and 95% ranges are read by nothing yet. A proposed fix is to have the API return
  them.
- Two small time-series edge cases: pace on the first date of a season uses the previous season's full
  average, and the nightly regime model restarts a streak at 1 when it has no prior state. No fix is
  planned.
