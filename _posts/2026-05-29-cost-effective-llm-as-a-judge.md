---
layout: post
title: "Making LLM judges more reliable without fine-tuning"
date: 2026-05-29
tags: [research, llm-as-a-judge, evaluation, icml]
description: >-
  How far you can push an LLM judge with drop-in, inference-time changes and no
  fine-tuning. What worked on RewardBench 2, what surprised me, and where it
  goes next.
---

- 📄 Paper: [arXiv:2604.13717](https://arxiv.org/abs/2604.13717)
- 💻 Code & data: [composo-ai/llm-judge-criteria-ensembling](https://github.com/composo-ai/llm-judge-criteria-ensembling)

---

It is intuitive that an LLM writing prose is sampling from a distribution of possible outputs. It is less intuitive that an LLM scoring a response is doing the same thing, but it is. Ask the same judge the same question twice and you will often get two different scores. Once you see it that way, a lot of what gets called judge unreliability starts to look like ordinary noise, and the question becomes how much of it you can average away.

Reliable evaluation is the thread that runs through most of my work, and as part of my research on evals at Composo I have been working on a specific version of it: getting one model to judge another model's output well, at scale, without a person checking every case. I wanted to manage that without fine-tuning, which is expensive to run and to maintain, tends not to generalise far beyond the data you trained it on, and makes it awkward to fold in the customer-specific context that, in practice, decides what "good" even means for a given task. So the question I set out to answer was a narrow one: how far can you push an LLM judge using only drop-in changes at inference time, with no training involved?

The full version is in a paper, [On Cost-Effective LLM-as-a-Judge Improvement Techniques](https://arxiv.org/abs/2604.13717), which I will be presenting at the ICML 2026 Workshop on Statistical Frameworks for Uncertainty in Agentic Systems. This is the shorter, less formal version: what we tried, what surprised me, and where it goes next. All the numbers and detail are in the paper, and the [code is on GitHub](https://github.com/composo-ai/llm-judge-criteria-ensembling).

## The idea

If the core problem is noise, then the interventions worth making are the ones that control noise, rather than the ones that try to make the judge cleverer. A single score is one draw from a distribution, and we read far too much into that one draw. That is the lens behind everything we tested: treat the judge as a noisy estimator, and ask what cheaply tightens it.

## What we tried

We tested four drop-in techniques on RewardBench 2, each coming at that noise idea from a different angle:

- **Ensembling**: ask for several independent scores and average them. More draws, less noise.
- **Task-specific criteria**: add a single sentence to the prompt saying what actually matters for that category, for example correct reasoning for maths, or appropriate refusal for safety. Close to free.
- **Calibration context**: show the judge a previously scored example to anchor its sense of the scale. This was aimed squarely at the well-known anchoring problem.
- **Adaptive escalation**: use a cheap model by default and only call an expensive one when the cheap model looks uncertain.

The headline is that the two cheapest things, criteria and ensembling, did almost all of the work. Together they took accuracy from roughly 72% to 84% on the full-size model, and none of the cleverer methods beat that combination once you account for cost.

## What surprised me

A few of the results caught me out.

The first is that a small model, given a large ensemble and the criteria prompt, scored higher than any of the big models did. A mini model (Haiku 4.5) at eight samples with criteria reached 85.8%, above every full-size configuration we ran, and at roughly a quarter of the cost. The instinct to reach for the biggest model on the hardest judgements turns out to be the wrong one here. Averaging a lot of cheap, noisy opinions beat one expensive, careful one.

The second is the anchoring result. Anchoring, where a judge's score drifts depending on what it has just seen, is something people talk about a great deal, and calibration context was our attempt to address it head on. It helped a little when we were taking a single sample. But once we were ensembling, it added essentially nothing: the ensemble had already removed the noise the calibration trick was targeting. The careful, targeted prompt was beaten by simply sampling a few more times and averaging. I think that is a useful reminder that the obvious-sounding fix is not always the one that pays off.

The third, and the one I keep coming back to, is where the gains actually landed. The biggest jumps were in the hardest categories. Maths, which the base judge was fairly weak at, improved a lot from nothing more than the one-sentence criterion. Precise instruction-following, which is brutally hard in every setup we tried, recovered a long way once criteria and ensembling were combined. The cheap interventions helped most exactly where the judge was weakest, which is close to the opposite of what I would have predicted.

## Where this goes next

The limitation I keep coming back to is this. Ensembling here averages many samples from the same model, which cancels noise but does nothing about bias. Every sample comes from the same model with the same blind spots, so a confident, consistent mistake stays exactly that, and taking more samples will not rescue you. The honest next step on the judge itself is to stop treating the score as a single number and decompose what the judge is doing, so that you can separate systematic bias from noise instead of averaging over both. AISI's [Skewed Score](https://www.aisi.gov.uk/blog/llm-judges-on-trial-a-new-statistical-framework-to-assess-autograders) work points in this direction, modelling autograder bias explicitly rather than reading it off an agreement score.

The version I find more interesting takes the same lens somewhere with higher stakes. Once you start trusting a cheap consistency or variance check as a detector, say to flag a model that is deliberately underperforming on an evaluation, or sandbagging, the bias and variance picture quietly tells you when it will fail. A model that underperforms only when it senses it is being watched can move its tell out of the bias a detector sees within a single context, and into the variance across contexts that the detector cannot, so the cheap check looks clean at exactly the moment it should be alarmed. AISI's [auditing-games work](https://arxiv.org/abs/2512.07810) has already shown black-box detection being defeated by this kind of conditional sandbagger, though qualitatively, without detection rates attached. What I am working on now is pointing the bias and variance decomposition from [The Hot Mess of AI](https://arxiv.org/abs/2601.23045) at exactly this: to work out when this kind of cheap detector gives false reassurance, and whether the blind spot can be closed without already knowing what the model is reacting to. That is the next post.

It is also why I find the judge-reliability work interesting in the first place. When a check is only there to rank chatbot answers, an unreliable check is a product problem. When the same kind of check is meant to catch a model hiding what it can do, that same noise becomes a safety problem, and it is worth knowing exactly when you can and cannot trust the instrument.

---

I will be at **ICML in Seoul this July**. If you will be around and want to talk evaluation, judge reliability, or where this goes next, I would be glad to chat. Reach me on [LinkedIn](https://www.linkedin.com/in/ryanlail/) or [X](https://x.com/ryan__lail).
