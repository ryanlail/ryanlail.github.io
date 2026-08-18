---
layout: post
title: "Quantifying blind spots of LLM Evaluators"
date: 2026-06-21
tags: [research, llm-as-a-judge, evaluation, ai-safety, uncertainty]
description: >-
  Most LLM-judge errors survive resampling, so a disagreement-based detector
  cannot see them. That ceiling on detection tightens as judges get stronger
  and is tightest on Safety.
subtitle: "When an LLM judge is wrong, does its own resample disagreement know?"
math: true
---

- 💻 Code & data: [ryanlail/does-the-judge-know](https://github.com/ryanlail/does-the-judge-know)

---

When an LLM judge makes a wrong safety decision, does its own uncertainty know? Under the bias–variance decomposition, errors are either variance errors that are detectable by resampling, or bias errors where the judge is confidently and repeatably wrong. We find it is mostly the latter on frontier judges. Wrong answers are usually stable under resampling, the same wrong winner returns when you re-draw the scores. GPT-5.4's errors are 70% stable across all categories and 86% stable on Safety, so a disagreement-based detector can catch at most 14% of its Safety errors. We show there is a ceiling on what single-judge disagreement can catch, and this ceiling falls as judge models get stronger, with the effect strongest on Safety.

## Why this matters

LLM Judges are increasingly deployed as safety monitors, the backstop that is supposed to stop a model from producing unsafe outputs, where the tolerance for error is close to zero. When Anthropic released [Fable 5](https://www.anthropic.com/news/claude-fable-5-mythos-5), it was implemented as Mythos 5 with safety guardrails on top to prevent unsafe use, however, these are [not guaranteed to prevent unsafe use in all scenarios](https://x.com/elder_plinius/status/2064776322979676227?s=20). Therefore, a key problem in AI Safety research is how we can improve the safety monitors to control AI models. As we recommended in [our ICML workshop paper](https://arxiv.org/abs/2604.13717), a cheap improvement to standard LLM as a Judge is to resample the same model to reduce variance errors. Another approach may be to sample independent models. [Kohli](https://arxiv.org/abs/2605.29800) found that sampling different models shares bias errors too, concluding that nine judges are worth roughly two independent votes. A detector built only on disagreement can reduce errors, but can give false reassurance precisely on the inputs an adversarial model would choose.

## How we measure it

On RewardBench 2 each example has four candidate responses, with one being the correct one, and a judge scores all four. We collected eight independent score samples per candidate response, which gives an estimation of the distribution of the judges' rating. We run this for two frontier judges, GPT-5.4 and Sonnet 4.6, on the same set of around 1,750 examples, plus three weaker judges: GPT-5.4-mini, Haiku 4.5, and GPT-5.4-nano. A judge's predicted winner is the response with the highest mean score across its eight samples; if that is not the correct candidate, the judge has made an error.

What we measure is whether an error survives resampling. For each error we bootstrap the eight existing samples: resample the scores per response with replacement, recompute the winner, and repeat 1,000 times. We call the fraction of those draws that reproduce the same wrong winner $p_\text{stable}$. This deliberately resamples the eight scores we already have rather than making fresh judge calls, because those eight samples are exactly the information an ensemble or disagreement detector has at deployment. A result about their stability is therefore a result about what any such detector can ever see. We label a non-tie error bias-dominated when $p_\text{stable} \geq 0.9$, with sensitivity reported at 0.75 and 0.95 (ties, where no single response wins, count as errors but are analysed separately). The bias share is the fraction of a judge's non-tie errors that are bias-dominated. A disagreement-based detector can only flag an error it sees move under resampling, so the most it can ever catch is the variance-dominated fraction. We call 1 − bias share the recall ceiling, an upper bound on the recall of any detector built from single-judge disagreement.

## Results

### Most errors are bias-dominated, and this peaks on safety tasks

For both frontier judges, most errors survive resampling.

| Judge | Errors that are *stable* (bias share) with 95% CI | Detector ceiling (max errors catchable) |
| --- | --- | --- |
| GPT-5.4 (overall) | 70% [64–75] | 30% |
| **GPT-5.4 (Safety)** | **86% [72–97]** | **14%** |
| Sonnet 4.6 (overall) | 58% [51–64] | 42% |
| Sonnet 4.6 (Safety) | 63% [48–78] | 37% |

![Bias share by RewardBench 2 category for both frontier judges (fraction of errors that are stable under resampling, $p_\text{stable} \geq 0.9$), with 95% confidence intervals. Safety is highlighted; the dashed line marks 25%. The recall ceiling of any disagreement detector is one minus the bar height.]({{ "/assets/images/fig1_bias_share_by_category.png" | relative_url }})

The bias share is not uniform across the benchmark, it runs from 56% on Math to 86% on Safety for GPT-5.4. It is highest on Safety, the category where we would least want a blind monitor.

### Stronger judges have a larger bias share

Within a model family, the bias share rises with model size.

| GPT judge | Errors that are *stable* (bias share) with 95% CI | Detector ceiling |
| --- | --- | --- |
| GPT-5.4-nano | 28% [23–32] | 72% |
| GPT-5.4-mini | 50% [44–56] | 50% |
| **GPT-5.4** | **70% [64–75]** | **30%** |

From nano to mini to the full model the detector ceiling falls, 72% to 50% to 30%. This does **not** mean larger models make worse judges, as they are still less likely to make errors in the first place. We highlight these results to emphasise that the shape of errors are very different as model capability increases.

### Different models share the same blind spot

We observe when a frontier judge is wrong the other is also wrong more than half the time, roughly three times its base rate. A panel of judges does not buy as many independent votes as its size suggests as the judges tend to fail on the same inputs.

![How often both frontier judges are wrong on the same example, split by difficulty (how many of the three weaker judges also get it wrong). Within each band the observed rate (dark) sits above what independence predicts (light), and that gap is shared bias. Difficulty drives most of the overlap, but the gap persists at every level.]({{ "/assets/images/fig2_cross_judge_overlap.png" | relative_url }})

Some of this can be attributed to task difficulty, but the figure shows this isn't entirely correlated with difficulty. Within every difficulty band they are wrong together more often than independence predicts. Independence is the two judges' error rates within the band multiplied together, the rate you would expect if one of them being wrong told you nothing about the other. We measure difficulty by how many of the three weaker judges (GPT-5.4-mini, Haiku 4.5, GPT-5.4-nano) also get it wrong.

![The same difficulty-adjusted overlap for three pairs. Each bar is how much more often the two judges are both wrong than you would expect if they failed independently at each difficulty level (difficulty is set by judges outside the pair, so it cannot absorb the shared bias). Two judges from different providers overlap a little beyond difficulty, 1.2x. Two from the same family overlap far more, 1.6 to 1.7x, they inherit the same blind spots.]({{ "/assets/images/fig_family_overlap.png" | relative_url }})

Comparing difficulty-adjusted errors across model families shows that blind spots overlap the least across model providers.

### A ceiling, not a weak signal

Our earlier paper found that a judge's score variance predicts incorrectness with an AUC of about 0.60, a weak but positive signal. A low AUC could mean we just have a noisy estimator, however, the bias share shows that for most errors there is nothing in the disagreement to detect. Even the best resample signal we could build, the winner's instability ($1 - p_\text{stable}$), only reaches AUC 0.78 to 0.81, and it is capped the same way; it cannot separate a stable wrong winner from a confidently correct one. Therefore, we present a limit to the class of disagreement solutions.

## What this means for safety

Averaging more draws has limits, and polling different judges still has limits beyond capability. Two judges from the same family share their blind spots far more than two different providers do. So escalating from mini to full within a family is the worst case, it sends the monitor's hardest cases to the judge whose blind spots line up most closely with its own. The trigger is weak also, escalating the most uncertain fifth of cases catches only about a third of the frontier judge's bias errors.

An adversary who can push an input into the stable region can exploit a confidently-wrong monitor for free. It gives the most reassurance on exactly the inputs an adversary would choose. This is a risk that should be considered when building safety monitors.

The same shape of claim runs through what we think is the natural sequel. The uncertainty signal you would most want to trust is blind exactly where an adversary would operate. A judge that is confidently wrong on a steered input, and a model that sandbags when it senses it is being evaluated, are the same problem, the behaviour you care about hides where your detector sees calm.
