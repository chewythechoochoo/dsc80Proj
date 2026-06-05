# First Dragon, First Win? 🐉

**A DSC 80 @ UCSD Final Project**
By Chu Pei

This project investigates professional *League of Legends* esports matches from the
2022 season (data from [Oracle's Elixir](https://oracleselixir.com/tools/downloads))
to answer one question:

> **Does securing the first dragon of the game give a team a higher chance of winning?**

Dragons are early neutral objectives that grant lasting, stacking buffs, so understanding
their impact helps players, coaches, and analysts decide how aggressively to contest the
early game. The full analysis lives in [`notebook.ipynb`](notebook.ipynb).

## Introduction

The Oracle's Elixir dataset records one row per player and per team for every professional
game tracked in 2022. We focus on the **team-level rows** (`position == "team"`), giving us
about **25,000 team-games** to analyze. The columns most relevant to our question are:

| Column | Description |
| --- | --- |
| `result` | 1 if the team won the game, 0 if it lost. |
| `firstdragon` | 1 if the team secured the first dragon of the game. |
| `firstblood` | 1 if the team got the first kill of the game. |
| `firsttower` | 1 if the team destroyed the first tower. |
| `golddiffat15` | The team's gold lead/deficit at 15 minutes. |
| `xpdiffat15` | The team's experience lead/deficit at 15 minutes. |
| `csdiffat15` | The team's creep-score (farm) lead/deficit at 15 minutes. |
| `league` | The professional league/competition the game belongs to. |

## Data Cleaning and Exploratory Data Analysis

We kept only team rows, dropped games missing `result` or `firstdragon`, and cast the
0/1 flag columns (`result`, `firstdragon`, `firstblood`, `firsttower`) to integers so they
can be averaged into win rates and fed to models. The 15-minute differentials were left
numeric. Here is the head of the cleaned team-level data:

| league   | teamname                 |   result |   firstblood |   firstdragon |   firsttower |   golddiffat15 |   xpdiffat15 |
|:---------|:-------------------------|---------:|-------------:|--------------:|-------------:|---------------:|-------------:|
| LCKC     | HANJIN BRION Challengers |        0 |            1 |             0 |            1 |            107 |        -1617 |
| LCKC     | Nongshim Esports Academy |        1 |            0 |             1 |            0 |           -107 |         1617 |
| LCKC     | T1 Esports Academy       |        0 |            0 |             0 |            0 |          -1763 |         -906 |
| LCKC     | Liiv SANDBOX Youth       |        1 |            1 |             1 |            1 |           1763 |          906 |
| LCKC     | KT Rolster Challengers   |        1 |            0 |             1 |            1 |           1191 |         2298 |

### Univariate Analysis

<iframe src="assets/univariate_golddiff.html" width="800" height="500" frameborder="0"></iframe>

The gold difference at 15 minutes is roughly symmetric and centered near zero: in any game
one team's lead is the other's deficit, so the team-level distribution is nearly mirror
symmetric about 0. The wide spread shows that sizable early advantages are common.

### Bivariate Analysis

<iframe src="assets/bivariate_dragon_winrate.html" width="800" height="500" frameborder="0"></iframe>

Teams that secure the first dragon win clearly more often than teams that do not. This is
the association we formally test in the Hypothesis Testing section below.

<iframe src="assets/bivariate_gold_result.html" width="800" height="500" frameborder="0"></iframe>

Teams that eventually win tend to already hold a gold lead at 15 minutes, suggesting that
early-game state is predictive of the final result and motivating our prediction problem.

### Interesting Aggregates

Win rate broken down by first dragon (rows) and first blood (columns):

|                 |   No First Blood |   First Blood |
|:----------------|-----------------:|--------------:|
| No First Dragon |            0.324 |         0.533 |
| First Dragon    |            0.468 |         0.675 |

Win rate rises as a team secures *more* early objectives: teams with neither advantage win
least often (~32%), teams with both win most often (~68%), and either objective on its own
lands in between, so the two appear to contribute roughly additively.

## Assessment of Missingness

**NMAR analysis.** We do **not** believe `golddiffat15` is NMAR. Its missingness is not
explained by the gold value itself; instead, some leagues and tournaments simply never had
the detailed timeline tracking that produces 15-minute snapshots. Because whether the value
is recorded depends on an *observed* column (`league`), the data are **MAR**, not NMAR. A
more plausible NMAR example would be a voluntary post-game survey, where players who
performed poorly might be less likely to respond; obtaining each player's measured
performance would help explain that propensity and make it MAR.

**Missingness dependency.** Using permutation tests, the missingness of `golddiffat15`
**depends strongly on `league`** (observed TVD = 0.987, p-value ≈ 0; entire leagues such as
LPL and LDL never recorded 15-minute data) but is **independent of `result`** (observed
difference in mean result ≈ 0.0000, p-value ≈ 1.0).

<iframe src="assets/missingness_result.html" width="800" height="500" frameborder="0"></iframe>

The observed difference (red line) sits right in the middle of the null distribution,
confirming that whether a team won is unrelated to whether its 15-minute gold was recorded.

## Hypothesis Testing

- **Null hypothesis:** Teams that secure the first dragon have the same win rate as teams that do not; any observed difference is due to random chance.
- **Alternative hypothesis:** Teams that secure the first dragon have a *higher* win rate than teams that do not.
- **Test statistic:** Difference in win rates (win rate with first dragon minus win rate without first dragon). This directed statistic matches our one-sided alternative.
- **Significance level:** 0.05.

Using a permutation test with 10,000 shuffles, teams that secured the first dragon won
**57.9%** of games versus **42.1%** for teams that did not, an **observed difference of 15.7
percentage points**. No shuffle under the null came close (the largest simulated difference
was about 2.3 percentage points), giving a **p-value ≈ 0.0000**.

<iframe src="assets/hypothesis_perm.html" width="800" height="500" frameborder="0"></iframe>

Because the p-value is far below 0.05, we **reject the null hypothesis**: there is strong
evidence that securing the first dragon is associated with a higher win rate. As this is a
statistical test on observational data and not a controlled experiment, we cannot conclude
that the first dragon *causes* wins or prove the result with certainty.

## Framing a Prediction Problem

We frame a **binary classification** problem: predict whether a team **wins** a game
(`result`, where win = 1 and loss = 0) from its **early-game state**. We chose `result` as
the response because it is the outcome of central interest and extends the question above.

To respect the **time of prediction**, we use only information available by the 15-minute
mark: the early-objective flags `firstblood`, `firstdragon`, `firsttower`, and the
15-minute differentials `golddiffat15`, `xpdiffat15`, `csdiffat15`. We exclude end-of-game
information that would not be known mid-game.

We evaluate with **accuracy**. The classes are essentially balanced (each game has exactly
one winner and one loser, so ~50% of team-rows are wins), which makes accuracy easy to
interpret and not misleading the way it can be on imbalanced data; we prefer it over F1 here
precisely because there is no rare positive class to protect.

## Baseline Model

Our baseline is a **logistic regression** in a single `sklearn` Pipeline using **two
features**: `firstblood` (**nominal**, binary, passed through) and `golddiffat15`
(**quantitative**, standardized with `StandardScaler`). That is 1 quantitative, 1 nominal,
and 0 ordinal features.

On a 75/25 train-test split it achieves **73.7% test accuracy** (74.4% train), well above
the ~50% you would get by guessing. That is a reasonable start, but it ignores the other
early objectives and resource leads, leaving clear room to improve.

## Final Model

The final model uses all six early-game features, **two engineered features**, and a
**Random Forest** tuned with `GridSearchCV`:

1. **`early_objectives`** — the count of early objectives secured
   (`firstblood + firstdragon + firsttower`, 0–3). Securing several early objectives
   reflects compounding map control, which should be more predictive together than any
   single flag alone.
2. **`gold_xp_sum`** — the combined gold and experience lead at 15 minutes
   (`golddiffat15 + xpdiffat15`), which summarizes a team's overall resource advantage
   better than either component on its own.

We tuned `max_depth` (controls overfitting) and `n_estimators` (number of trees) with
5-fold cross-validation on the training set only. The best hyperparameters were
**`max_depth = 4`, `n_estimators = 200`**. The final model reaches **74.3% test accuracy**,
an **improvement over the baseline's 73.7%** on the same train/test split. We attribute the
gain to the richer, well-motivated features and the forest's ability to capture non-linear
interactions (e.g., a gold lead matters more when paired with objective control), rather
than to chance.

<iframe src="assets/confusion.html" width="650" height="500" frameborder="0"></iframe>

## Fairness Analysis

We ask whether the final model predicts equally well across league tiers.

- **Group X — major leagues:** LCK, LCS, LEC.
- **Group Y — all other leagues** in the modeling data.
- **Evaluation metric:** accuracy.
- **Null hypothesis:** The model is fair; its accuracy is the same for both groups and any difference is due to chance.
- **Alternative hypothesis:** The model is unfair; its accuracy differs between the groups.
- **Test statistic:** |accuracy(major) − accuracy(other)| (two-sided). **Significance level:** 0.05.

The model achieved **69.1% accuracy on major leagues** versus **74.8% on other leagues**, an
observed gap of **5.7 percentage points**. A permutation test (10,000 shuffles of the group
label) gave a **p-value ≈ 0.007**.

<iframe src="assets/fairness_perm.html" width="800" height="500" frameborder="0"></iframe>

Because the p-value is below 0.05, we **reject the null hypothesis**: there is evidence that
the model performs worse on major-league games than on other games. A likely explanation is
that elite teams play more even, drawn-out games where an early 15-minute lead is less
decisive, making outcomes harder to predict from early state alone. As always, this is a
statistical conclusion on observational data and does not prove unfairness with certainty.
