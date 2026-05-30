# First Dragon, First Win? 🐉

**A DSC 80 @ UCSD Final Project**
By Chu Pei

This project investigates professional *League of Legends* esports matches from the
2022 season (data from [Oracle's Elixir](https://oracleselixir.com/tools/downloads))
to answer one question:

> **Does securing the first dragon of the game give a team a higher chance of winning?**

Dragons are neutral objectives that grant lasting buffs, so understanding their impact
helps players, coaches, and analysts decide how aggressively to contest early-game
objectives.

## Introduction

The dataset contains row-level statistics for every team and player across thousands of
professional matches in 2022. After filtering to **team-level rows only** (`position == "team"`),
we analyze **22,872 team-games**. The columns most relevant to our question are:

| Column | Description |
| --- | --- |
| `result` | Whether the team won (1) or lost (0) the game. |
| `firstdragon` | Whether the team secured the first dragon of the game. |
| `firstblood` | Whether the team got the first kill of the game. |
| `golddiffat15` | The team's gold lead/deficit at 15 minutes. |

## Data Cleaning and Exploratory Data Analysis

We kept only team rows, dropped games missing `result` or `firstdragon`, and cast both
to integers. Exploratory analysis shows that teams securing the first dragon win
**57.9%** of the time, compared to **42.1%** for teams that do not, a clear early-game
advantage that motivates a formal hypothesis test.

## Assessment of Missingness

Columns such as `golddiffat15` are missing for some games. We assess whether this
missingness depends on other columns (e.g., league or game length) using permutation
tests. *(Full analysis in the notebook.)*

## Hypothesis Testing

- **Null Hypothesis:** Teams that secure the first dragon have the same win rate as teams that do not. Any observed difference is due to random chance.
- **Alternative Hypothesis:** Teams that secure the first dragon have a *higher* win rate than teams that do not.
- **Test Statistic:** Difference in win rates (win rate *with* first dragon − win rate *without* first dragon).
- **Significance Level:** 0.05.

Using a permutation test with 10,000 shuffles, the **observed difference of 15.7
percentage points** was never exceeded under the null (largest simulated difference ≈ 2.3
percentage points), giving a **p-value ≈ 0.0000**. We reject the null hypothesis: the data
provide strong evidence that securing the first dragon is associated with a higher win rate.

## Framing a Prediction Problem

We frame a **binary classification** problem: predict whether a team wins a given game
(`result`) using only information available partway through the game. We evaluate with
**accuracy**, since the classes are roughly balanced (each game has exactly one winner
and one loser).

## Baseline Model

Our baseline is a **logistic regression** built inside a single `sklearn` Pipeline using
two features: `firstblood` (binary/nominal, passed through) and `golddiffat15`
(quantitative, standardized with `StandardScaler`). Trained on a 75/25 train-test split,
it achieves **~73.7% test accuracy**, comfortably above the 50% baseline of guessing.

## Final Model

*(In progress.)* We plan to engineer additional features, such as early objective
counts and gold/XP differentials at multiple timestamps, and tune hyperparameters with
`GridSearchCV` to improve over the baseline.

## Fairness Analysis

*(In progress.)* We will test whether the final model performs equally well across groups,
for example across different competitive regions/leagues.
