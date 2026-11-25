---
title: "Weighted Quantile Loss, explained"
date: 2026-11-14
draft: false
math: true
tags: ["forecasting", "metrics", "time series"]
summary: "What weighted quantile loss measures, its formula, a worked example, and how it stacks up against MAE, RMSE, MAPE, WAPE, and plain quantile loss."
---

Most forecast accuracy metrics answer one question: how far off was the
prediction? Weighted quantile loss answers a different one. It asks whether your
forecast got the *uncertainty* right, and it puts big and small items on one
scale.

That makes it a good fit for demand forecasting, where you rarely want a single
number. You want a range, and you want one score that works across products of
very different size.

## The intuition

Start with a single prediction. Say you forecast demand for next week and you
want the 90th percentile, the level you'll only exceed 10% of the time. Useful
for setting stock so you rarely run out.

If the real demand comes in *above* your forecast, that's bad. You under-stocked.
If it comes in *below*, that's also not free, but it's a different kind of wrong.
Quantile loss lets you penalize those two mistakes by different amounts,
depending on which percentile you asked for.

For the 90th percentile, being too low is penalized nine times harder than being
too high. That asymmetry is the whole point: it pushes the forecast up until,
over many weeks, it lands above the truth about 90% of the time. Exactly what a
90th percentile should do.

## One formula

For a single actual value $y$, a forecast $q$ at quantile level $\tau$ (a number
between 0 and 1), the quantile loss is:

$$
\mathrm{QL}_\tau(y, q) = \max\big(\tau \cdot (y - q),\ \ (\tau - 1) \cdot (y - q)\big)
$$

Read it as two cases:

- If you under-forecast ($y > q$): the loss is $\tau \cdot (y - q)$.
- If you over-forecast ($y < q$): the loss is $(1 - \tau) \cdot (q - y)$.

At $\tau = 0.9$, under-forecasting costs 0.9 per unit and over-forecasting costs
0.1 per unit. Nine to one.

(You will also see quantile loss called **pinball loss**. Same thing: plot the
loss against the forecast and it has a kinked, pinball-table shape.)

**Weighted** quantile loss then sums this over all items and all time, and
divides by total actual volume so big and small series can sit in the same
number:

$$
\mathrm{WQL}(\tau) = \frac{\sum_i \mathrm{QL}_\tau(y_i, q_i)}{\sum_i |y_i|}
$$

The division is what people mean by "weighted", though strictly it is a
normalization. Each item's loss goes into the sum as is, with no weight
attached. Dividing by total volume just turns the total loss into a share of
total demand. Big items still end up counting more, because their misses are
bigger in units, so they put more loss into the numerator.

## A tiny worked example

One item, one week. You forecast the 90th percentile as $q = 100$.

- Actual comes in at $y = 130$ (you under-forecast by 30). Loss $= 0.9 \times 30 = 27$.
- Different week, actual comes in at $y = 70$ (you over-forecast by 30). Loss $= 0.1 \times 30 = 3$.

Same size miss in units, very different penalty. The metric is telling you that
for a 90th percentile, running short is the expensive mistake.

## Why the "weighted" part matters

Now two items, to see the weighting work (again, illustrative numbers). Both
under-forecast by the same 10%, at $\tau = 0.9$:

- **High-volume item:** actual $y = 1000$, forecast $q = 900$. QL $= 0.9 \times 100 = 90$.
- **Low-volume item:** actual $y = 100$, forecast $q = 90$. QL $= 0.9 \times 10 = 9$.

If you just averaged the two losses, you would get $(90 + 9)/2 = 49.5$, treating
both items as equals. WQL instead divides the total loss by the total volume:

$$
\mathrm{WQL} = \frac{90 + 9}{1000 + 100} = \frac{99}{1100} = 0.09
$$

The high-volume item contributes 90 of the 99 total, about 91%. Same percentage
error on both, but the big item drives almost the entire score. Nothing weighted
it up. Its miss is simply ten times bigger in units. In practice that means WQL
mostly reflects how well you forecast the high-volume items, which is usually
what the business cares about too.

## How it compares to the usual metrics

- **MAE** (mean absolute error) averages the size of the miss. Simple and steady,
  but it only scores a single point forecast, and it treats over and under the
  same. No notion of a quantile.
- **RMSE** (root mean squared error) squares the errors first, so a few big
  misses dominate. Good when large errors are especially costly, but sensitive to
  outliers and, again, point-only and symmetric.
- **MAPE** (mean absolute percentage error) divides each error by the actual.
  Easy to explain as a percentage, but it blows up when actuals are near zero,
  and it quietly over-weights low-volume items. In intermittent demand it can be
  useless or undefined.
- **WAPE** (weighted absolute percentage error) fixes MAPE's main flaw by
  dividing total error by total actual volume, so it's volume-weighted and stable
  near zero. It's basically WQL's median sibling: WAPE lines up with WQL at
  $\tau = 0.5$ up to a scale factor. But it still scores one point forecast, not
  a range.
- **Quantile loss** (unweighted) has the asymmetry WQL has, but without the
  volume normalization, so a handful of large series can swamp it and you can't
  compare across datasets of different scale.

The short version. MAE, RMSE, MAPE and WAPE all grade a single number. WQL grades
a *distribution*. It checks whether your forecast's percentiles are honest, and it
does that in a volume-aware way.

## When to use it

Reach for WQL when all three are true:

1. You produce probabilistic forecasts (percentiles or a full distribution), not
   just one number.
2. The cost of being wrong is asymmetric. Running out hurts more than
   overstocking, or the reverse.
3. Volume varies a lot across items and you want one score that is stable
   across them and comparable between datasets.

If you only have a point forecast, WQL has nothing extra to offer, so use WAPE or
MAE. For when a range is worth producing in the first place, and how to read
one, see [point vs distributional forecasts](/posts/point-vs-distributional-forecasts/).

## Choosing tau, and two pitfalls

**Pick tau from the cost, not the default.** The quantile you score should match
the decision. If running out is much worse than overstocking, you care about a
high quantile like the 90th, so score there. A symmetric decision scores the
50th. Do not score the 90th out of habit if your actual trade-off is balanced.

**WQL alone does not prove good coverage.** A low WQL says the forecast is
sharp and roughly right, but it does not guarantee that your "90th percentile"
actually sits above the truth 90% of the time. Check calibration separately:
over many weeks, how often did actuals fall below each quantile? A metric and a
coverage check answer different questions.

**Track more than one quantile.** Score at least the 50th and the 90th. The 50th
tells you if the forecast is centered; the 90th tells you if the tail is
calibrated. A model can look fine at the median and badly miss the tail that
drives your safety stock.

One more thing: a single WQL number is only as honest as the way you measured it.
Score it with [walk-forward backtesting](/posts/walk-forward-backtesting/),
averaging over many time folds, rather than on one lucky slice of history.
