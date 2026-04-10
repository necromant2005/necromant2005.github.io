---
title: Adaptive Allocation For A/B Testing For Products
permalink: /analytics/adaptive-allocation-ab-testing-products/
---

# Adaptive Allocation For A/B Testing For Products

In classic A/B testing, we split traffic 50/50 and wait until the end. It is simple, but it can be expensive. If one variant is clearly better, we still keep sending half of users to the weaker version.

Adaptive allocation solves this. The idea is simple: while test is running, we gradually send more traffic to the variant that looks better.

![Adaptive allocation idea](/assets/images/adaptive-allocation-overview.svg)

## Why Product Teams Use It

1. Less lost revenue during experiments.
2. Faster learning in real market conditions.
3. Better user experience because more users see stronger variant earlier.

But there is a tradeoff: if we adapt too aggressively, we can make wrong decisions from noisy early data.

## Static vs Adaptive

Static allocation:

- Traffic split does not change.
- Easy to analyze and explain.
- More users may see weaker variant for longer.

Adaptive allocation:

- Traffic split changes over time.
- Usually better short-term business outcome.
- Needs stronger experiment discipline.

## Practical Methods

### 1) Epsilon-Greedy (easy to start)

At each step:

- With probability `epsilon` (for example 0.1), explore: send traffic randomly.
- With probability `1 - epsilon`, exploit: send traffic to current best variant.

Good first step for teams that want a simple implementation.

### 2) Thompson Sampling (usually better balance)

For each variant, keep a probability model for conversion rate. Then:

- Sample one possible conversion rate from each variant model.
- Pick variant with the highest sampled value.
- Update model after new data.

This naturally balances exploration and exploitation.

![How allocation shifts over time](/assets/images/adaptive-allocation-traffic-shift.svg)

## Rules That Keep It Safe

1. Start with an exploration floor: keep at least 10% traffic for each active variant.
2. Define guardrails before test starts: crash rate, latency, refund rate, complaints.
3. Use a minimum sample size before strong reallocation.
4. Stop test when both are true:
   - Business impact is meaningful.
   - Statistical confidence threshold is reached.
5. Run post-test validation on a short fixed split to confirm no hidden bias.

## Example Rollout Plan

1. Week 1: Run normal 50/50 A/B test, collect baseline variance.
2. Week 2: Launch adaptive policy with conservative limits (for example max 70% traffic to one variant).
3. Week 3+: Increase max allocation only if guardrails are stable.

## When Not To Use Adaptive Allocation

- Very small traffic products.
- Experiments with long outcome delay (for example 30-day retention only).
- High-risk flows where one wrong shift can cause serious damage.

## Final Thought

Adaptive allocation is not magic. It is a control system for uncertainty. If you define safety rules first and adapt gradually, you usually get both better learning and better business results than fixed 50/50 tests.
