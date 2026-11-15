---
title: "Walk-forward backtesting for forecasts"
date: 2026-11-14
draft: false
math: true
tags: ["forecasting", "backtesting", "time series"]
summary: "Why a random train/test split lies to you on time series, how walk-forward backtesting fixes it, expanding vs sliding windows, and the leakage traps to watch for."
---

If you forecast demand and want to know how good your model really is, you have
to test it on data it has never seen. Everyone agrees on that. The trap is in
*how* you hold data out.

The habit most people bring from regular machine learning is a random train/test
split: shuffle the rows, keep 80% to train, 20% to test. On time series, that
quietly cheats. A random split lets the model train on June and test on May. It
learns from the future to predict the past. Your test score looks great, and
then the model falls apart in production, where the future is not available.

Walk-forward backtesting fixes this by respecting time.

## A quick illustration

Here is the gap, on a synthetic weekly demand series (made-up data, not from any
real business, just to show the effect). Take one model and score it two ways.

| Scoring method | Error (MAE) |
|---|---|
| Random train/test split | 5.15 |
| Walk-forward backtesting | 6.34 |

Same model, same data. The random split reports an error about 19% lower than
walk-forward. That lower number is a lie: it comes from letting the model peek at
future weeks during training. Walk-forward's higher number is the one you would
actually live with in production. The flattery is largest exactly when it hurts
most, when the model is flexible enough to overfit and the series drifts over
time.

## The idea

Always train on the past and test on the future that follows it. Then slide
forward and do it again.

Picture your history as a timeline. You pick a point, train on everything before
it, and forecast the stretch just after it. That stretch is your test fold. Then
you move the point forward and repeat. Each fold uses only data that existed at
the moment of the forecast.

```
Time ---------------------------------------------------->

fold 1:  [==== train ====][ test ]
fold 2:  [====== train ======][ test ]
fold 3:  [======== train ========][ test ]
fold 4:  [========== train ==========][ test ]
```

Every test window sits strictly after its training window. No fold ever sees the
future. That is the whole point, and it is the one rule a random split breaks.

## Expanding vs sliding window

There are two ways to move forward, and the choice matters.

**Expanding window.** The training set grows each fold. You keep all history and
add more as you march forward (the diagram above). Use it when older data still
helps, which is common: more data usually means a steadier fit.

**Sliding window.** The training set is a fixed length that moves forward,
dropping the oldest data as it adds new.

```
Time ---------------------------------------------------->

fold 1:  [== train ==][ test ]
fold 2:       [== train ==][ test ]
fold 3:            [== train ==][ test ]
fold 4:                 [== train ==][ test ]
```

Use a sliding window when the world changes and old patterns stop being true. If
demand behaviour three years ago no longer looks like today, training on it drags
the model backward. A sliding window forgets on purpose.

The rule of thumb: expanding when the process is stable, sliding when it drifts.
When unsure, try both and compare.

## A worked fold structure

Say you have three years of weekly data and you forecast 4 weeks ahead. A simple
expanding-window setup:

- Start with the first 2 years as training.
- Forecast the next 4 weeks. Score them.
- Add those 4 weeks to training. Forecast the following 4 weeks. Score them.
- Keep going to the end of the data.

You end up with many 4-week test folds spread across the last year. Average your
error metric over all of them. That average is a far more honest estimate of
real-world accuracy than any single split, because it is really asking: across
many different moments in time, how well would this model have done going
forward?

## The traps

Walk-forward removes the obvious leak. A few subtler ones remain.

**Leakage through features.** Respecting time on the target is not enough. Every
feature must use only information available at forecast time. A feature built
from a full-series average, or a "last 4 weeks" window that accidentally includes
the test weeks, leaks the future just as badly as a bad split. Build features
fold by fold, not once over the whole series.

**No gap between train and test.** Sometimes the most recent actuals are not
known yet when a forecast is made (reporting lag). If your model in production
will not have last week's number, your backtest should not either. Leave a gap,
sometimes called an embargo, between the end of training and the start of the
test window, matching the real lag.

**Retraining cost.** Honest walk-forward retrains at every fold, which is slow if
the model is expensive. Retraining less often is a fair compromise, but say so,
and know you are testing a slightly staler model than the per-fold ideal.

**Horizon mismatch.** Test at the horizon you actually forecast. If production
forecasts 4 weeks out, scoring 1-week-ahead folds flatters the model. Shorter
horizons are almost always easier.

## How to choose the setup

The mechanics are simple; the judgment is in the settings. Some rules of thumb:

- **How many folds.** Enough that the average is stable, not so many that each
  test window is tiny. For a few years of weekly data, 4 to 8 folds is a sensible
  range. If adding a fold swings your average a lot, you have too few.
- **How long a test window.** Match it to the horizon you actually forecast. If
  you forecast a quarter ahead, test on quarter-long windows. Testing on windows
  shorter than your real horizon flatters the model.
- **Expanding vs sliding.** Expanding when the process is stable and old data
  still helps. Sliding when behaviour drifts and old patterns mislead. When
  unsure, run both and compare; if sliding wins clearly, your series is drifting.
- **When data is short.** With little history, you cannot afford many folds.
  Use one or two, keep an expanding window so each fold trains on as much as
  possible, and treat the result as a rough read, not a precise score.

## Why it is worth the trouble

A random split answers a question nobody asked: can the model interpolate inside
a period it has already partly seen. Walk-forward answers the real one: given
only the past, how well does the model predict what comes next. The second number
is lower. It is also the one that matches what happens after you ship.

What you score inside each fold matters as much as the folds themselves. For
demand forecasts, [weighted quantile loss](/posts/weighted-quantile-loss-explained/)
is often the right error measure to average across them.
