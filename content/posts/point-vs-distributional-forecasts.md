---
title: "Point vs distributional forecasts"
date: 2026-11-24
draft: false
math: true
tags: ["forecasting", "uncertainty", "time series"]
summary: "What a point forecast hides, what a distributional forecast adds, how to read P10, P50 and P90, how to pick the quantile that matches a decision, and when a single number is enough."
---

Ask for a demand forecast and you usually get one number. Next week: 3 units.
It looks precise, and it is easy to plug into a plan. The trouble is everything
it leaves out.

Three units could mean "almost surely between 2 and 4". It could also mean
"probably nothing, maybe 15". Those are very different weeks to plan for, and a
single number cannot tell them apart. A distributional forecast can.

## Point forecasts

A point forecast is one value per item per period. Under the hood it is almost
always an estimate of the mean or the median, even when nobody says which.

That is fine when the decision only needs the typical case. It breaks down when
the decision is about risk: how much stock to hold, how many people to staff,
how much capacity to book. Those decisions depend on how bad things can get, and
a point forecast says nothing about that.

It also hides the fact that the mean and the median are not the same thing. More
on that below, because it bites hardest on exactly the items people struggle
with.

## Distributional forecasts

A distributional forecast describes the whole range of outcomes. In practice it
comes in one of two forms:

- **Quantiles.** A few chosen levels, typically P10, P50 and P90. This is the
  most common form in demand forecasting.
- **A full distribution.** Either a named one with parameters (Poisson, negative
  binomial, gamma) or a set of sample paths you can summarize however you like.

Quantiles are easier to store and explain. A full distribution is more flexible,
because you can read off any quantile later, or compute things like the chance
of selling zero.

## How to read one

Take P90. It is the level that actual demand should stay at or below 90% of the
time. Over many weeks, demand lands above it in about 1 week in 10. P50 is the
middle: half the weeks come in above, half below. P10 is the low end.

A few habits help:

- **Read the width, not just the middle.** The gap between P10 and P90 is how
  unsure the model is. A narrow band means a steady item. A wide one means plan
  for surprises.
- **Read each quantile one period at a time.** The P90 for each week is a
  statement about that week alone. Adding the weekly P90s does not give you the
  P90 of the month; it gives you something higher, because a bad week and a
  good week tend to cancel out.
- **Expect the range to grow with the horizon.** Four weeks out should be wider
  than one week out. If it is not, be suspicious.

## Mean, median, and skew

Here is a made-up example (synthetic, not from any real business). Two items,
weekly demand, both with clean distributions:

| Item | Mean | P10 | P50 | P90 |
|---|---|---|---|---|
| Steady seller | 100 | 87 | 100 | 113 |
| Sparse seller | 3 | 0 | 2 | 8 |

For the steady seller, the mean and the median are the same, and the band is
tight. A point forecast of 100 tells you nearly everything.

The sparse seller is different. It sells nothing in about 30% of weeks, and every
so often it sells a lot. The mean is 3, but the median is 2, and P90 is 8. The
long right tail pulls the mean above the typical week. Stock it at the mean of 3
and you run out in about 30% of weeks. Stock it at P90 and you run out in under
10%.

Most catalogs are full of items like the second one. That is where a point
forecast does the most damage, because it looks just as confident as it does for
the steady seller.

## Picking the quantile from the cost

Which quantile should drive a decision? The one that balances the two ways of
being wrong. If running short costs $c_u$ per unit and having too much costs
$c_o$ per unit, the best level to plan at is

$$
\tau^* = \frac{c_u}{c_u + c_o}
$$

This is the classic newsvendor result. If a lost sale costs nine times what an
extra unit on the shelf costs, $\tau^* = 9 / (9 + 1) = 0.9$, so you plan at P90.
If the two costs are equal, you plan at P50.

That nine to one ratio is the same one behind the 90th percentile in
[weighted quantile loss](/posts/weighted-quantile-loss-explained/). The metric
and the decision rule are two views of the same trade-off, which is why scoring
a forecast at the quantile you will actually use makes sense.

In practice, different teams read different quantiles off the same forecast:

- **Safety stock and service levels:** a high quantile, like P90.
- **Capacity plans (space, sites, peak):** a high quantile, often P70 to P90,
  since running short is slow and costly to fix.
- **Labor and transport plans:** often P50, the typical week, when gaps can be
  patched with overtime or extra trucks.
- **Financial plans and revenue:** the mean or P50, depending on the convention.
- **Downside and risk planning:** a low quantile, like P10.

One forecast, several users, each picking the level that fits their cost.

## When to use which

My rough guide:

- The decision is symmetric, demand is steady, and you only need the typical
  value: **a point forecast is enough**.
- The cost of being short and being over are different: **use quantiles**, and
  plan at the level the cost ratio gives you.
- Demand is sparse, lumpy, or skewed: **use quantiles or a full distribution**.
  The mean alone will mislead you.
- You need things like the chance of zero sales, or you want to roll forecasts up
  across weeks or items: **use a full distribution or sample paths**, since
  quantiles do not add.
- Several teams will use the same forecast for different decisions: **produce
  quantiles**, and let each team read its own level.

## How to judge one

A distributional forecast needs a metric that grades the whole range, not just
the middle. MAE and WAPE only see one number.
[Weighted quantile loss](/posts/weighted-quantile-loss-explained/) is built for
this: it scores each quantile with the right asymmetry and puts them on a common
scale.

Pair it with a coverage check. Over many weeks, how often did actuals fall at or
below each quantile? A good P90 should cover about 90% of them. If it covers 70%,
the tail is too thin, and your safety stock is too low, no matter how good the
P50 looks.

And as always, measure both with
[walk-forward backtesting](/posts/walk-forward-backtesting/), so the numbers
reflect how the forecast will do on weeks it has not seen.

---

*Related reading: [weighted quantile loss](/posts/weighted-quantile-loss-explained/)
on how to score a distributional forecast,
[walk-forward backtesting](/posts/walk-forward-backtesting/) on how to score it
honestly over time, and
[global vs local forecasting models](/posts/global-vs-local-forecasting-models/)
on choosing the model that produces it.*
