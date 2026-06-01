---
layout: post
title: "Making LLM Judges Better Without Fine-Tuning: A Writeup of Our ICML Paper"
date: 2026-05-29
tags: [research, llm-as-a-judge, evaluation, icml]
math: true
description: >-
  A writeup of "On Cost-Effective LLM-as-a-Judge Improvement Techniques" — which
  drop-in tricks actually move the needle on RewardBench 2, and which don't.
---

I'm excited to share that our paper, **"On Cost-Effective LLM-as-a-Judge
Improvement Techniques"** (joint work with Luke Markham at Composo AI), is
heading to **ICML**. This post is a writeup: the short version up top, and a
more technical walkthrough below for anyone who wants the details.

- 📄 Paper: [arXiv:2604.13717](https://arxiv.org/abs/2604.13717)
- 💻 Code & data: [composo-ai/llm-judge-criteria-ensembling](https://github.com/composo-ai/llm-judge-criteria-ensembling)

---

## The short version

Using a language model to *score* or *rank* other models' outputs —
"LLM-as-a-Judge" — has quietly become the dominant approach to automated
evaluation. It shows up in RLHF reward signals, in benchmarking, in offline
test suites that gate releases, and in real-time monitors that watch for
regressions in production. The catch is that an LLM judge is a **noisy,
stochastic** instrument: ask it twice and you can get two different scores.

There's a long menu of folk techniques for making judges more reliable —
ensembling, fancier prompts, calibration, routing hard cases to bigger models.
But which actually help, and which are just expensive? We ran a controlled study
on **RewardBench 2** to find out, testing four drop-in techniques (no
fine-tuning, no new models) across model tiers and two providers.

The headline result: **two cheap, drop-in techniques do almost all the work.**

- **Task-specific criteria injection** — telling the judge what to look for on
  *this* category instead of generic "is it helpful?" prompts — gives
  **+3.0pp at essentially zero extra cost**.
- **Ensemble scoring** — averaging several independent judgments — gives
  **+9.8pp** (at the full model, ~5× cost).
- Together they reach **up to 85.8% accuracy, +13.5pp over a 71.7% baseline —
  and, crucially, at just 1.3× baseline cost**, because the peak comes from a
  *small* model that's ensembled and given good criteria.

The slightly contrarian finding: **calibration context and adaptive model
escalation also beat the baseline, but they're dominated** by criteria +
ensembling on the cost–accuracy frontier. Once you're averaging enough samples,
the residual noise is small enough that these extra interventions stop adding
signal. The practical recipe is boring and cheap: **write task-specific
criteria, ensemble a handful of samples, and prefer a small model with more
samples over a big model with fewer.**

---

## The technical version

### Framing: a judge is a noise source

The unifying lens of the paper is to treat the judge as a **stochastic scoring
function** and read each technique as a form of *noise control*:

- **Ensembling** is Monte Carlo averaging over per-call noise. If a single
  judgment is the true score plus zero-mean noise, averaging $k$ independent
  calls shrinks the variance of the estimate as $1/k$:

  $$\operatorname{Var}\!\left(\frac{1}{k}\sum_{j=1}^{k} s_{ij}\right) = \frac{\sigma_i^2}{k}.$$

  That's why returns diminish — most of the gain is captured by $k=3$; beyond
  that you pay linearly for little.

- **Criteria injection** doesn't change per-response variance (mean
  $\sigma_i$ = 0.31 vs 0.32; KS $p$ = 0.88) — instead it sharpens *between-response
  discrimination*, so the same noise is less likely to flip the ranking.

- **Per-response score variance** is itself a weak but measurable *uncertainty
  signal* (AUC ≈ 0.60 for predicting an incorrect judgment).

### Setup

We evaluate on **RewardBench 2 (RB2)**, on the 1,753 examples spanning five
categories — Factuality, Focus, Math, Precise IF, and Safety — after excluding
the Ties subset. Each example gives a query and **four** candidate responses
(response 0 is always correct); the judge assigns each an integer score from 1
to 10, and we count an example correct only if the *unique* highest-mean
response is the right one (any tie counts as wrong — a deliberately conservative
rule that punishes judges that can't discriminate).

We test across three capability classes — **full** (Claude Sonnet 4.6, GPT-5.4),
**mini** (Claude Haiku 4.5, GPT-5.4 mini), and **nano** (GPT-5.4 nano) — and
report **cost as a ratio** to the baseline (full model, $k$=1), since absolute
API prices are vendor- and time-specific.

### What worked

| Condition | Best model | Accuracy | Cost | Δ vs base |
|---|---|---|---|---|
| Baseline (full, k=1) | GPT-5.4 | 71.7% | 1.0× | — |
| + Criteria (full, k=1) | GPT-5.4 | 74.7% | 1.1× | **+3.0pp** |
| + Ensemble (full, k=8) | GPT-5.4 | 81.5% | 5.0× | **+9.8pp** |
| + Criteria + ensemble (full, k=8) | GPT-5.4 | 83.6% | 5.3× | +11.9pp |
| **+ Criteria + ensemble (mini, k=8)** | **Haiku 4.5** | **85.8%** | **1.3×** | **+13.5pp** |

Two things stand out:

1. **Criteria injection is nearly free.** It only adds a one-sentence,
   category-aware instruction to the prompt (e.g. for Math: *"Focus on whether
   the mathematical reasoning is logically valid, the steps are correct, and the
   final answer is accurate"*). On the full model that's +3.0pp at $k$=1 — and
   the criteria were committed to the repo *before* the first data-collection run
   to rule out post-hoc tuning. The gains concentrate in Math (+12.0pp) and
   Safety (+3.3pp).

2. **Small models benefit disproportionately from ensembling.** The absolute
   $k$=1 → $k$=8 gain grows as base capability falls: +9.8pp (full), +14.4pp
   (mini), +19.1pp (nano). The upshot is that **mini + criteria at $k$=8 matches
   full-model ensembling at roughly one-quarter the cost**, and the overall peak
   (85.8%) is a *mini*-class result. Ensembling raises a weak model's floor, not
   its ceiling, though — nano $k$=8 still trails mini $k$=8 by ~8pp.

The tie rate tells the variance-reduction story cleanly: ensembling collapses it
from 20.4% ($k$=1) to 4.5% ($k$=8), and adding criteria pushes it to 3.2%.

### What didn't (reliably) work

This is the part worth internalizing. Several popular ideas beat the raw
baseline but were **dominated** by criteria + ensembling at comparable cost:

- **Calibration context** — injecting a previously-scored reference example to
  anchor the judge's scale. Helps +1–2pp at $k$=1 (the "low" anchor, showing a
  known-*bad* example, slightly beats "high"; cross-category works as well as
  within-category, so it's general scale-anchoring, not transfer). But at $k$=8
  it's a **no-op** (within ±0.2pp of plain ensembling): the ensemble has already
  removed the noise that calibration was suppressing.
- **Adaptive model escalation** — routing high-variance responses to the big
  model. Hard variance routing has a large "dead zone" (escalating *some* but
  not all responses rarely changes the four-way winner), so useful operating
  points collapse to the cheap or expensive extremes. **Soft blending** looked
  good in-sample (83.2%) but failed to generalize (80.2% on held-out test, below
  full $k$=8's 81.5% — midpoint overfitting). Variance-informed ensembling at a
  low budget (74.9%) is dominated by mini $k$=8 (79.2% at 1.2×).
- **Stacking everything** — the combined condition (criteria + calibration +
  dual-model ensembling) reached 82.6% at 6.8× cost, *below* criteria $k$=8 alone
  (83.6% at 5.3×). The interventions are substitutes, not complements, once
  variance is already low.

A nice practitioner aside: even at **temperature 0**, $k$=8 beats $k$=1 by
+4.6pp (CIs non-overlapping) — temp-0 isn't actually deterministic in practice
(GPU floating-point non-determinism, no seed parameter), so even "deterministic"
deployments benefit from ensembling.

### Generalization

The findings replicate across both **OpenAI GPT** and **Anthropic Claude**
families, which suggests these are properties of the LLM-as-a-Judge setup itself
rather than quirks of one provider.

---

## Takeaways for practitioners

If you run LLM judges in a research pipeline or in production:

1. **Write task-specific criteria.** Highest return-on-effort change available,
   essentially free.
2. **Ensemble a few samples** ($k$≈3–8 — most of the gain is at $k$=3).
3. **Prefer a small model with more samples** over a big model with fewer; it's
   often cheaper *and* more accurate.
4. **Be skeptical of calibration and routing** as accuracy boosters — measure
   them against this simple baseline before paying for them.

If you'll be at **ICML**, come say hi — I'd love to talk about evaluation, judge
reliability, and where this goes next.

*Full ablations, per-category CIs, prompts, and escalation analyses are in the
[paper](https://arxiv.org/abs/2604.13717) and
[repository](https://github.com/composo-ai/llm-judge-criteria-ensembling).*
