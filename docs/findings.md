# Findings

This document holds the evidence behind the README: the full results, how they were checked, the data
leaks I found and fixed, and which ideas worked and which did not. The results in 1.1 are for the
models serving today. The sizes in 1.3 to 1.6 were measured when each decision was made, and any that
predate the feature regeneration say so.

Note: I drafted this document with the help of an agent, working from the project's notes and decision
records (about 50 files), and it replaces those files together with `data-and-limits.md`. Every number in
1.1 was checked against the evaluation outputs, and a sample of the numbers in the other sections was
checked against those records.

## Summary

The model's typical miss is lower than a player's average and than their last five games on every
stat, by about 12% on points and under 1% on blocks, and its 80% and 95% ranges land within 1.2 points
of their targets ([1.1](#11-results), with the score explained in [1.2](#12-why-crps)). The scores are
on a held-out season that is not strictly later than the training data, so decisions rest on
walk-forward windows ([1.3](#13-how-it-is-validated)). I found and fixed several data leaks. Fixing
them made every model about 2% worse and reversed no decision ([1.4](#14-leaks-and-mismatch)). One model
per stat beat my earlier two-layer stack, and recent-form features, three-point attempt volume and an explicit
zero for assists and turnovers each helped ([1.5](#15-what-worked)). Several ideas did not
([1.6](#16-what-did-not-work)), and the references are in [1.7](#17-references).

## 1.1 Results

All figures are on the 2024-25 season, which has 28,097 player-games (players who played). No final model was trained,
tuned or calibrated on it. The models train on 2022-23, 2023-24 and the 2025-26 regular season, so the
season is unseen but not strictly later than the training data. By Kapoor and Narayanan's criterion that
test data should always be later than training data [1], that makes this fixed holdout temporally leaky.
The walk-forward windows in section 1.3, where each window trains only on earlier games, carry the
strictly chronological checks. They also decided what I adopted, and three of the five test windows
overlap 2024-25 (the fifth trains on it), so the holdout is not blind to my design choices.

The first table gives the typical miss and the distribution score (lower is better for both). The
second gives how the ranges performed.


| Stat           | Distribution     | MAE   | MAE, player average | MAE, last 5 games | CRPS  |
| -------------- | ---------------- | ----- | ------------------- | ----------------- | ----- |
| Points         | Normal           | 4.586 | 5.186               | 4.827             | 3.220 |
| Rebounds       | Normal           | 1.905 | 2.079               | 2.009             | 1.334 |
| Assists        | Hurdle LogNormal | 1.341 | 1.461               | 1.391             | 0.918 |
| Turnovers      | Hurdle LogNormal | 0.888 | 0.933               | 0.933             | 0.586 |
| Three-pointers | Hurdle LogNormal | 0.888 | 0.932               | 0.933             | 0.588 |
| Steals         | Hurdle LogNormal | 0.711 | 0.727               | 0.748             | 0.451 |
| Blocks         | Hurdle LogNormal | 0.521 | 0.524               | 0.523             | 0.302 |


Points, rebounds and assists show the clearest gain: the model's typical miss is about 8% to 12% below
the player average. Turnovers and three-pointers are about 5% below it, steals about 2%, and blocks
are a tie. The model also beats the last-5-games baseline on every stat, by 0.003 or less on blocks. Resampling players 5,000 times (a player-clustered bootstrap on the holdout) puts a 95% interval of 10.1% to 13.2% on the points gain over the player average, which is 0.52 to 0.69 points of MAE.


| Stat           | 80% hit rate | 80% mean width | 95% hit rate | 95% mean width |
| -------------- | ------------ | -------------- | ------------ | -------------- |
| Points         | 78.8%        | 12.42          | 94.6%        | 19.29          |
| Rebounds       | 79.5%        | 4.55           | 94.6%        | 7.54           |
| Assists        | 79.9%        | 2.79           | 95.2%        | 9.08           |
| Turnovers      | 79.8%        | 1.49           | 95.1%        | 2.89           |
| Three-pointers | 79.0%        | 1.43           | 94.7%        | 2.82           |
| Steals         | 79.8%        | 0.92           | 94.9%        | 2.04           |
| Blocks         | 79.4%        | 0.35           | 94.7%        | 1.30           |


Every 80% hit rate is within 1.2 points of its target, and every 95% hit rate within half a point.
Widths scale with the stat: an 80% range for points is about 12 points wide, while for blocks it is
well under one.

The player average is each player's average over the training seasons, one fixed number per player
(the league average for a player not seen in training). It is built from the same seasons the models
see, so the comparison is like for like. The last-5-games baseline uses each player's previous five
games in the same season.

An 80% range should contain the actual result about 80% of the time, and a 95% range about 95%. The
hit rates run from 78.8% to 79.9% and from 94.6% to 95.2%. The ranges are read off the same
whole-number probability table that each forecast publishes, and their ends are fitted on a
calibration slice that is never part of the evaluated season.

A 50% range exists only for points. A range of whole numbers grows in steps, so for the other six
stats none lands near 50%. Blocks are zero in 69% of games, and the narrowest range around zero
already holds 78% of results.

The assists 95% range is wider than it needs to be. It averages 9.08 wide, which is about the width of
the narrowest range from 0 that holds 95% of all assist totals in the season (0 to 8), so it says little about the
individual player. It was 5.05 wide, with 94.9% coverage, before assists moved to the hurdle form. The
fit that picks the range ends ranks candidates by how close coverage lands to the target and uses width
only to break exact ties, so on assists it chose ends that cover 95% but are far wider than they need
to be. Coverage, MAE and CRPS are not affected. I plan to change the rule to pick the narrowest range
within a small coverage tolerance and refit.

For a concrete forecast, here is the first row of the evaluation file: LeBron James in the 2024-25
opening game, Lakers against Timberwolves. For three-pointers made, the model gave an expected value of
1.86 and these probabilities (rounded, so they add up to slightly over 100%):

| Made | 0 | 1 | 2 | 3 | 4 | 5 | 6 or more |
|---|---:|---:|---:|---:|---:|---:|---:|
| Probability | 22% | 22% | 29% | 16% | 7% | 3% | 3% |

The 80% range is 0 to 2 made, and he made 1. That range holds 73% of this forecast's own probability,
not 80%. Results come in whole numbers, so a range grows in chunks (0 to 3 would hold 89%), and the
cut points are fitted so that over many games about 80% of results land inside. The 22% for zero is its
own probability, which is what the hurdle form is for. For assists the same row gave an expected value
of 8.2 and an 80% range of 4 to 10, and he had 4.

The numbers come from `scripts/eval_forecast_calibration.py`, run on the serving models. It writes two
CSV files under `data/eval/`, which is not committed.

The 2026-27 season has not started, so no live forecast could be scored yet. The nightly pipeline has
been rehearsed end to end on a copy of the database, including a replay of the 2024-25 opening night,
which produced 448 forecasts (64 players, seven stats) with no missing expected values.

## 1.2 Why CRPS

MAE scores one number per forecast, the expected value. Each forecast here is a whole distribution, so
it also needs a score that judges the distribution. The continuous ranked probability score (CRPS) is
the standard choice for that [2]. It compares the forecast's cumulative probability with what actually
happened, and it is lowest when the distribution is centered on the result and no wider than it needs
to be. For a forecast that is a single number it reduces to the absolute error, so MAE and CRPS are on
the same footing. Lower is better here. The original paper uses the opposite sign.

For scale, a Normal distribution centered on a player's average scores a CRPS of 3.64 on points, and
one centered on their last five games scores 3.42, against 3.220 for the model. For rebounds the
figures are 1.47 and 1.42, against 1.334. I give these only for points and rebounds, because the same
baseline for the other five stats cannot predict a zero (see the last paragraph below).

Results are whole numbers, so I compute CRPS on whole numbers: the sum over every k of the squared
difference between the forecast's cumulative probability at k and 1 if the actual result is at most k
(0 otherwise). The published table uses this version.

Older notes in the project use the continuous version, which differs by 0.47% to 4.74% (points 3.2354
against 3.2204, for example). The gap is mechanical: it appears for any continuous forecast of a whole
number, so it says nothing about the model. It also reorders nothing. No comparison between two models
moved by more than 0.069 percentage points, and none changed sign.

I fit the range ends rather than reading them off the distribution. A range of whole numbers moves in steps, so the nominal 10% and 90%
cut points over-cover the hurdle-modeled stats by 8 to 14 points. The two ends of each range are searched
separately, because a fixed-width window cannot reach every coverage level.

I do not quote a skill score (the percent improvement in CRPS over a baseline) against a player-average baseline for the five hurdle-modeled stats.
That baseline is built in the model's own family, a LogNormal with no probability at zero, so it
cannot predict a zero and flatters any model that can. Assists' skill score rose from 9.8% to 11.3%
when CRPS moved by only 0.73%. A baseline of the same family is planned, and until it exists I compare
by MAE and CRPS.

## 1.3 How it is validated

Most of my testing time went into checking that the scores were not flattering the model. These are
the checks that changed a decision or caught a real defect. The walk-forward design follows what the
forecasting literature calls time-series cross-validation [3].


| Check                                  | What it guards against                                    | What happened                                                                                                                                                                                                                                                                                                                                          |
| -------------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Held-out season                        | Scores that reflect fitting to the evaluated data         | No final model is trained or calibrated on 2024-25, and the step that fits the range ends refuses to run if holdout rows reach its calibration slice. The walk-forward windows used to decide what to adopt do overlap it (section 1.1).                                                                                                                                                                                               |
| Walk-forward validation                | One window helping or hurting a change by luck            | Five test windows across three seasons. Each trains only on games before it, leaves a 3-day gap, and calibrates on the last 20% of its training rows. One window is a screen; a change needs the windows to agree.                                                                                                                                     |
| One scoring function                   | Models scored by different rules                          | Every model and baseline goes through one function, on the same whole-number distribution. Three copies of the scoring code had drifted. Moving one onto the shared rule changed points 80% coverage from 85.4% to 78.9%, because it had only calibrated one end of the range.                                                                         |
| A minimum effect size                  | Results that are significant but too small to matter      | At about 28,000 rows a 51% win rate gives a p-value near zero, so significance decides nothing. A change must improve CRPS by at least 0.5%.                                                                                                                                                                                                           |
| Uncertainty clustered by player        | Overconfident intervals from treating rows as independent | A player's games are not independent. A row-level bootstrap understates the interval. In the first release gate, before the feature regeneration, clustering widened the intervals by 10% to 120%. I have not re-measured that figure since.                                                                                                                                                                                                     |
| Matched controls                       | Crediting a cheap change to an expensive model            | A challenger that changes two things is also tested against an arm that changes only the cheap one. Three-pointers as attempts times make rate read as a 1.94% gain until this control showed 0.28% (section 1.6).                                                                                                                                     |
| Model family read from the saved model | Scoring a model as something it is not                    | An evaluator once read the family from an environment setting that defaults to off, so it scored the hurdle stats as Normal and turned a large gain for steals into a large loss. The tell was that raw and calibrated coverage matched exactly.                                                                                                       |
| Nightly build equals full build        | Nightly updates drifting from a full rebuild              | Replaying a night's incremental updates must reproduce a full rebuild exactly. A check before the first live slate found six places where it did not: two feature modules and four in the step that updates the time-series models. After the fixes, all 142 columns of the full feature build matched on four dates and all 34 time-series columns matched on 234 rows. |
| Rehearsal                              | A nightly run that fails or touches production            | The real nightly pipeline runs against a copy of the database, and the production database is checked before and after, so any change fails the run. It caught a real failure, the injury feed refusing requests, and replayed the 2024-25 opening night: 448 forecasts for 64 players across seven stats, with no missing expected values.                             |
| Serving matches evaluation             | The served numbers differing from the evaluated ones      | Replaying four past slates through the serving code left 38 other columns unchanged, and every range agreed with its row's own probability table (400 rows at three levels). A parity audit compares served rows with training rows: served CRPS is within 0.02% on every stat, and 64 of the 68 columns it compares match exactly.                            |
| Leakage tests                          | Features reading later games                              | 15 test files target leakage. The pattern is to change a later game and assert that earlier features do not move. A guard fails the whole suite if any test writes to the model directory.                                                                                                                                                    |
| Training is not bit-reproducible       | Reading noise as signal                                   | Two identical NGBoost fits differ row by row, by about 0.014% CRPS in aggregate. So arms are compared within one run, and rows are never paired across runs.                                                                                                                                                                                           |


On the 5,684 settled rows from those replays (812 per stat), the 80% ranges held between 77.8% and
81.7% of the time and the 95% ranges between 93.6% and 96.4%. With 812 rows per stat an 80% rate
carries about 1.4 points of sampling noise, and these rows are about 3% of the holdout season (812 of its 28,097 player-games), so
they check the serving path rather than add independent evidence. The replay was run before assists
and turnovers moved to the hurdle form.

## 1.4 Leaks and mismatch

A leak is a feature that uses information from after the game being predicted. It makes offline
scores look better than live ones, and it is a documented cause of over-optimistic results in machine
learning [1]. In that paper's taxonomy, most of the leaks below are the type where a model uses features
that would not be available at prediction time, and the whole-table averages resemble
pre-processing on the full dataset. The table lists the ones I confirmed, roughly in the order I found them, and sizes are
those measured when each was found. One entry, the stale live rows, is a mismatch rather than a leak:
it used older information, not later information, and made live scores worse instead of flattering
offline ones.


| Leak                                          | What went wrong                                                                                                                             | How it was found                                                                                      | Size when found                                                                                                                      | Fix                                                                                                                      |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| Starter flag                                  | The flag marking a player as a starter was ranked on minutes played in the game being predicted                                             | One flag topped an ablation; removing it moved median skill by an implausible 2.78% (7.4% for points) | About a third of the headline points skill (32%); rebounds 25%, assists and three-pointers 16%, turnovers 11%, steals 10%, blocks 4% | The label is the share of the player's previous five games that they started                                             |
| Whole-season and whole-table averages         | Four normalization steps (usage, opponent defense, pace, referee) used a mean or fit over a whole season or table, so a row saw later games | Writing leak tests for nine feature modules that had none; five of the nine leaked                    | Each moved aggregate scores by at most about 2%, in both directions                                                                  | Expanding averages, shifted by one date                                                                                  |
| Time-series models fitted on the full history | The count model and the regime model were fitted once on a player's entire series, so early rows depended on later games                    | Changing a later game's value changed earlier rows' features                                          | With the trained models held fixed, causal values raised CRPS 1.75% on points, 1.86% on rebounds and 2.62% on assists                | The count model is refit every five games on earlier games only; each regime-model row uses its own expanding fit        |
| Kalman filter start                           | A player's first 15 filter rows used a start state and noise scale computed from their own first 15 games                                   | A new test that perturbs every observation; the old one only moved game 20                            | Not sized on its own                                                                                                                 | The first 15 rows are left empty                                                                                         |
| Usage weight                                  | The weight that spreads a missing teammate's minutes was estimated from later box scores                                                    | Cutting the box scores off at a date changed 51 of 187 rows                                   | 51 of 187 rows moved; none after the fix                                                                                             | A player enters the shares only from their first game with earlier data                                                  |
| Same-night ordering                           | Games on the same night entered each other's opponent averages                                                                              | Reading the code                                                                                      | A few rows inside averages built from hundreds                                                                                       | Averages use earlier dates only                                                                                          |
| Live rows one game stale                      | The live pipeline carried features forward from each player's previous row instead of recomputing them                                      | A read-only probe that swapped the previous row's values into the live path                           | CRPS 1.17% worse on points, 1.02% on rebounds, 1.05% on assists; only 5 of 63 feature columns matched training                       | Live rows are built by running the training feature code on pre-game data plus placeholder rows for games not yet played |
| Failed fit blanked a player                   | One failed regime-model fit left every row empty for that player, so a row depended on later fits                                           | The feature regeneration                                                                              | 41 players                                                                                                                           | Only the failed row is left empty                                                                                        |


More on the starter flag. The model had a feature saying whether a player was a
starter, and I had built it by ranking players on minutes played in the game being predicted. Players
it flagged averaged 31.7 minutes against 14.0 for the rest, and the flag correlated 0.81 with that
game's minutes, which made it close to the answer. I caught it because an ablation looked too good.
One boolean beat every feature I expected to matter, and dropping it cost 7.4% on points. The fix
changes the label to the share of the player's previous five games that they started; that changed
18.8% of labels and dropped the correlation with same-game minutes from 0.81 to 0.65.  Five defects turned up
in seven days around then, and none came from an error message.

After the fixes I regenerated every feature and re-ran every earlier decision on leak-free features.
Every model got about 2% worse, as expected once a leak is gone. Points
and rebounds lost only 0.6% to 0.7%, because their recent-form features gave them another source of
the same signal. No earlier decision reversed; only the sizes changed. Live forecasts moved
to the training feature code in the same round, which closed a gap of up to 1.24% between offline and
live quality.

Two cases are known and not fixed. A team-level feature conditions on who actually played the game.
It feeds only an experimental path that is not served, and a test marked as expected to fail keeps it
visible. The usage features use the roster that actually played in training and the projected roster
when serving; the two differ on 145 of the 231 rows compared, with no measurable forecast cost. Fixing
either needs a pre-game definition of the expected roster, which is planned for the next full retrain.

## 1.5 What worked

Each change was kept only if it improved CRPS by at least 0.5% across the walk-forward windows. The
one exception is described in the second row. Negative numbers mean CRPS fell, which is better.
The flat-versus-stack margins come from the gate run on the continuous convention; moving to whole
numbers changed none of them by more than 0.069 points (section 1.2).


| Change                                                                                    | Stats                                     | Effect on CRPS                                                 | Windows ahead                    | Why it helps                                                                                                                                  |
| ----------------------------------------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------- | -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| One NGBoost model per stat on all features [4] instead of a two-layer stack (3 XGBoost models per stat on separate feature groups, combined by an NGBoost model) | Points, rebounds, assists                 | Pooled −2.33%, −1.96%, −1.11%; holdout −2.45%, −2.00%, −1.45%  | 5 of 5 each                      | The single model was more accurate and simpler. Two contributors are documented but neither was isolated: the stack's per-model tuning was coupled to its feature set, and it over-predicted when playoff rotations shortened                                                     |
| The same change                                                                           | Turnovers, three-pointers, blocks, steals | −0.25%, −0.23%, −0.21%, −0.13%, all under the bar              | 3, 5, 5 and 5 of 5               | Adopted as a simplification, because the single model is never significantly behind                                                                         |
| Recent-form features, restored                                        | Points, rebounds                          | −1.55%, −1.31%, against a model lacking exactly those features | 5 of 5 each                      | Without them the model's only recent-form input is the count model's rate; the restored features give it independent ones. The same reason explains why points and rebounds lost only 0.6% to 0.7% when leaks were removed. They measured under the bar for the other five stats, so they are not used there       |
| Three-point attempt volume                                                                | Three-pointers                            | −1.26% (interval −1.47% to −1.05%); holdout −1.41%             | 5 of 5                           | Without it the model forecasts makes without knowing how many shots a player takes. The largest per-row gain is for players who rarely shoot threes (−12.48%), though they are few; players for whom threes are 45% or more of their shot attempts carry half of the total gain. The gate was written down before the run                  |
| An explicit zero (the hurdle model)                                                       | Assists, turnovers                        | Walk-forward −0.90% and −1.24%; holdout −0.73% and −0.95%      | 4 of 5 (none against) and 5 of 5 | In the fifth of rows with the lowest expected value, assists are zero 59% of the time and turnovers 69%; the bell curve predicted 43% and 54% |


Two things that look like wins are not listed. Scoring on whole numbers is a mechanical gain
(section 1.2), and running the training feature code when serving is the fix for a
training and serving mismatch (section 1.4).

## 1.6 What did not work

Each idea below was measured the same way as the ones that worked, and each carries one of six
labels, so the reason it was dropped is on the page.

- Below the bar: the measured gain was under 0.5%, or its interval included zero.
- Lost to a matched control: the apparent gain disappeared against an arm that shares the
challenger's structure.
- Unreachable ceiling: the best case looked large but depends on something that cannot be known
before the game.
- No value after a leak fix: the apparent signal was the leak.
- Unstable: the method behaved badly on some rows or overfit.
- Not funded: the best case was too small for the cost, or the data was not available, so it was
never built.


| Idea                                          | Verdict                   | Result                                                                                                                                                                                                                  |
| --------------------------------------------- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Three-pointers as attempts times make rate    | Lost to a matched control | 0.28% against 1.94% versus the model then serving; worse on 56% of player-games                                                                                                                                         |
| Opponent and matchup features from box scores | Unreachable ceiling       | Even a perfect opponent measure is worth only 0.17% to 0.73%; the one apparent pass was a player-type effect, not an opponent effect                                                                                    |
| An expected-minutes model                     | Unreachable ceiling       | Minutes are only 63.6% predictable before the game; used as an input to the zero channel it gave steals −0.03% and blocks −0.02%. A bound of 1% to 5% from perfect minutes overstated the reachable gain about 30 times |
| Negative Binomial distribution                | Unstable                  | Normal won or tied on the restored features, and low-count rows drifted to expected values near a million; code removed                                                                                                 |
| Referee tendencies                            | No value after a leak fix | No gain on any of the seven stats once its leak was fixed; the block and its ingestion were removed                                                                                                                     |
| Recent-form features on the old stack         | Lost to a matched control | They hurt points there, though they help the single model; the stack's tuning was set when the recent-form block held a single feature                                                                                                                   |
| Season or month drift correction              | Below the bar             | At most 0.09 points over a single level on every stat, even when fitted with the answers known                                                                                                                          |
| Shot-volume features on other stats           | Below the bar             | Points −0.03%, assists −0.09%, steals −0.06%; the gain sits on the three-point axis                                                                                                                                     |
| Own-team pace features                        | Below the bar             | Three-pointers +0.06% (no gain), points −0.12% (under the bar even at four times the effect)                                                                                                                            |
| Shot-chart zone profiles                      | Not funded                | Reliability falls from 0.57 to 0.22 as zones get finer; best case about −0.4% to −0.5% at a high ingestion cost                                                                                                         |


Also tried or scoped and set aside: a hurdle model for points (−0.09%, interval including zero),
fitting the Kalman filter's noise terms by EM (overfit), calibrating sigma for the hurdle stats
(near-constant and harmful), a playoff-specific model (too few playoff rows), per-population routing
(about 0.1%), position features (worth at most about 0.05%, and constant by design) and
player-versus-defender tracking data (incomplete coverage and needs a license).



Three-pointers made equals attempts times make rate, so a model that forecasts the two separately
looked like the natural design. Against the model then serving it improved CRPS by 1.94%. The
comparison was unfair: 86% of that gain came from putting the forecast on whole numbers, which the
existing model could do for free. Against the same model on whole numbers, the factorized version
gained 0.28% (interval 0.09% to 0.47%), under the 0.5% bar, and did worse on 56% of player-games. The
ceiling looked enormous, since knowing the real attempt count would be worth 33%. But attempts are
only partly predictable before the game (R² of 0.57, an average miss of 1.5 attempts against a
spread of 3.2), and feeding the predicted attempt distribution through the same model recovered
about 1% of that ceiling. The same thing had happened with minutes and with matchups, so best-case bounds overstated the gain every time, and I now discount them.

Negative Binomial is a standard choice for overdispersed counts, and in an early comparison it beat
the Normal on CRPS for points, assists and turnovers. That comparison ran before the recent-form
features were restored. On the restored feature set the Normal won or tied, and a handful of
low-count rows drifted to expected values around a million during training, a sign the fit was
unstable. I left the option switched off and planned a variant that clipped the parameters, which I
never built. Later I removed the code, about 740 lines, for a different reason: a setting that was
off by default could still reach serving. With it on, the flat model would have trained points and
rebounds as Negative Binomial and served distributions built from Normal-trained parameters, which
would have changed the published family without anyone choosing it.

## 1.7 References

[1] Kapoor, S. and Narayanan, A. (2023). Leakage and the reproducibility crisis in machine-learning-based
science. Patterns 4(9), 100804. doi:10.1016/j.patter.2023.100804

[2] Gneiting, T. and Raftery, A. E. (2007). Strictly proper scoring rules, prediction, and estimation.
Journal of the American Statistical Association 102(477), 359-378. doi:10.1198/016214506000001437

[3] Hyndman, R. J. and Athanasopoulos, G. (2021). Forecasting: Principles and Practice, 3rd edition,
section 5.10, Time series cross-validation. OTexts. https://otexts.com/fpp3/tscv.html

[4] Duan, T. et al. (2020). NGBoost: Natural gradient boosting for probabilistic prediction.
Proceedings of the 37th International Conference on Machine Learning, PMLR 119. arXiv:1910.03225
