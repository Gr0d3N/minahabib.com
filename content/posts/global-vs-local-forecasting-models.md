---
title: "Global vs local forecasting models"
date: 2026-11-14
draft: false
math: true
tags: ["forecasting", "machine learning", "time series"]
summary: "Local models fit one series at a time, global models learn across many at once, and foundation models forecast series they never saw. ARIMA, DeepAR, and Chronos as one example each, with the trade-offs and when each wins."
---

Suppose you have to forecast demand for thousands of products. You have two
instincts about how to build the model, and they pull in opposite directions.

The first: each product is its own thing, so fit a separate model to each one's
history. The second: products rhyme with each other, so learn one model from all
of them at once. These are the local and the global approaches. A third, newer
option skips training on your data entirely. Understanding the three, and when
each is the right call, saves a lot of wasted effort.

## Local: one model per series

A local model is fit to a single series using only that series' own past. Classic
ARIMA works this way. For each product, you estimate its own parameters from its
own history, and you get one small, self-contained model per product.

This is clean and interpretable. You can read an ARIMA model and say what it
assumes. For a handful of products with long, stable histories, it is hard to
beat.

The weakness is that it cannot borrow. A product with three weeks of history has
three weeks to learn from, full stop. A brand-new product with no history has
nothing, so a local model simply cannot start. And fitting thousands of separate
models is a lot of moving parts to maintain.

## Global: one model across all series

A global model trains a single set of parameters on many series at once. DeepAR
is a well-known example: one neural network learns from thousands of series
together, then produces a forecast for each.

The payoff is shared learning. Patterns common across products, weekly
seasonality, holiday spikes, the shape of a typical ramp, are learned once from
all the data and lent to every series. A product with three weeks of history
benefits from the thousands of other products that have years. This is why global
models tend to win when you have many series, especially many short or new ones.

The cost is that it is one bigger model to train and reason about, and it is
usually less transparent than an ARIMA you can read line by line. For a few long,
well-behaved series, a global model is often more machinery than the problem
needs.

Two honest limits. First, "handles new items" has a catch: a global model can
only lend a shared shape to a new product if that product brings *something*, a
few data points or some features (category, price, launch date). A brand-new item
with zero history and no features is still hard for anyone. Second, pooling can
backfire. If your series genuinely do not have much in common, forcing one model
across them causes **negative transfer**: unrelated series pull the shared
parameters in different directions and make each other worse. Global is a bet that
the series rhyme; when they do not, it is the wrong bet.

## Foundation models: forecast a series you never trained on

The newest option is a **foundation model** for time series: one model pretrained
on a huge collection of outside time
series, then pointed at yours with no training on your data at all. Chronos is an
example of this zero-shot approach. It treats forecasting a bit like a language
model treats text, learning general time-series structure from vast public data,
then forecasting a new series out of the box.

The appeal is obvious: no per-series fitting, no training pipeline, a forecast in
one call. It is a strong starting point, and sometimes competitive with a model
trained on your own data. Two honest caveats. It still needs *some* recent
history of your series to read as context, so "no data" is really "no training,"
not "no input." And zero-shot is not the only mode: many foundation models can
also be fine-tuned on your data, which often beats using them out of the box. It
does not know your specifics, so a model trained on your history can still win
when that history carries signal the general model never saw.

## The trade-off in one table

| | Local (ARIMA) | Global (DeepAR) | Foundation (Chronos) |
|---|---|---|---|
| Fits on | one series' own past | many series together | outside data, not yours |
| Data needed | long history per series | many series, each can be short | a little recent history as context |
| Cold start / new item | can't | good, if there are features or a little history | good, if there is some context to read |
| Interpretability | high | lower | lower |
| Compute / upkeep | many small models | one bigger model | one call, or fine-tune |
| Shines when | few long stable series | many series, many short | quick start, no pipeline |

## A note on the math

If formulas are not your thing, skip this section; the idea above is the whole
point. For those who want it stated precisely:

A local model estimates separate parameters $\theta_i$ for each series $i$ from
that series alone. A global model estimates one shared $\theta$ by minimizing the
total loss across all series at once:

$$
\theta^\star = \arg\min_\theta \sum_{i} \sum_{t} \ell\big(y_{i,t},\ \hat{y}_{i,t}(\theta)\big)
$$

In words: $\theta$ is the model's parameters, $\arg\min$ picks the parameters that
make the error smallest, the two sums add up the loss $\ell$ over every series $i$
and every time step $t$, $y$ is the actual value and $\hat{y}$ is the forecast.
That single shared $\theta$ is the whole idea: it is what lets a global model
pool strength across series.

One honesty note on this formula: a real global model is rarely *purely* one
shared $\theta$. It usually adds a little per-series machinery (a learned scale or
an embedding) so series of different sizes do not fight. And because the loss is
summed over all series, a few very high-volume series can dominate it, pulling the
model toward them unless you weight or normalize.

(The three named examples: **ARIMA** is the classic local method, a per-series
model of trend and recent values. **DeepAR** is a global neural-network model
trained across many series. **Chronos** is a foundation model, pretrained on
outside data and used on yours with little or no training.)

## Seeing the pooling pay off

A small illustration (synthetic data, not from any real business). Take 200
products, each with only 12 weeks of history, all sharing a similar weekly
seasonal shape on top of their own level. Train on the first 9 weeks of each and
forecast the final 3.

| Approach | Error (MAE) |
|---|---|
| Local: a separate fit per product | 4.52 |
| Global: one model across all 200 | 2.52 |

The global model cuts the error by about 44%. The reason is exactly the
pooling: 9 weeks is too little for any single product to learn its seasonality
alone, but across 200 products the shared shape is obvious, and every product
borrows it. This is the case global models were built for, many short series.
Flip it to a handful of long, stable series and the gap shrinks or reverses.

Score comparisons like this with
[walk-forward backtesting](/posts/walk-forward-backtesting/) so the numbers
reflect real forward performance, and with a metric like
[weighted quantile loss](/posts/weighted-quantile-loss-explained/) if you
forecast ranges rather than points.

## So which one

Newer is not automatically better. My rough guide:

- A few series, long and stable, and you want to explain the model: **local**.
- Many series, lots of them short or new, and accuracy at scale matters most:
  **global**.
- You need a forecast fast, with no training pipeline, or a strong baseline to
  judge the others against: **foundation model**.

In practice the foundation model is often the quickest way to get a number on the
board, and the global model is where a lot of real demand forecasting has landed.
But a well-specified local model on the right problem is still a perfectly good
answer, and sometimes the best one.

---

*Related reading: [weighted quantile loss](/posts/weighted-quantile-loss-explained/)
on how to measure a forecast, and
[walk-forward backtesting](/posts/walk-forward-backtesting/) on how to measure it
honestly over time. This post is about choosing the model itself; those two are
about judging it once you have one.*
