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

At Composo we spend a lot of time on evaluation: working out whether a model's output is actually any good, at scale, without a person checking every case. The obvious way to improve an evaluator is to fine-tune one, and we were keen to avoid that if we could. Fine-tuning is expensive to run and to maintain, the resulting judge tends not to generalise far beyond the data you trained it on, and it is awkward to fold in the customer-specific context that, in practice, decides what "good" even means for a given task. So the question we started with was a simple one: how far can you push an LLM judge using only drop-in changes at inference time, with no training involved?

The full version of what we found is in a paper, [On Cost-Effective LLM-as-a-Judge Improvement Techniques](https://arxiv.org/abs/2604.13717), which we'll be presenting at the ICML 2026 Workshop on Statistical Frameworks for Uncertainty in Agentic Systems. This is the shorter, less formal version: what we tried, what surprised me, and where I think it goes next. All the numbers and detail are in the paper, and the [code is on GitHub](https://github.com/composo-ai/llm-judge-criteria-ensembling).

## The idea

The starting point was something that is slightly obvious and slightly not. When an LLM generates text, it is sampling from a distribution, and everyone is comfortable with that. When an LLM judges something, say giving a response a score out of 10, it is also sampling from a distribution, and that feels much less front of mind. Ask the same judge the same thing twice and you will often get two different scores.

Once you take that seriously, a lot of what looks like "the judge is unreliable" starts to look more like "the judge is noisy". Any single score is one draw from a distribution, and we tend to read far too much into that one draw. That reframing is really what motivated the techniques we tested. If the core problem is noise, then the fixes should be about controlling noise rather than trying to make the judge cleverer.

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

The thing I am most interested in is a limitation of what we did. Ensembling here averages many samples from the same model. That cancels noise, but it does not cancel bias. Every sample comes from the same model with the same blind spots, so a confident, consistent mistake stays exactly that, and averaging more of them does not help.

The direction I find interesting is combining estimators that are chosen to have different biases, so their errors are less correlated and the averaging cancels something other than just variance. That points towards decomposing what a judge is doing into parts that fail in different ways, rather than treating the score as a single opaque number. There is some nice work on decomposing LLM and judge errors along these lines ([AISI work on decomposing LLM errors](https://www.aisi.gov.uk/blog/llm-judges-on-trial-a-new-statistical-framework-to-assess-autograders), and [the "hot mess of AI" piece](https://alignment.anthropic.com/2026/hot-mess-of-ai/)) that I would like to build on.

It is also the part that feels like it matters beyond our own eval problem. As we lean more heavily on models to check the outputs of other models, the reliability of the judge becomes the reliability of the whole oversight process. Getting trustworthy signal out of imperfect judges is worth caring about for reasons well past a benchmark score.

---

I'll be at **ICML in Seoul this July**. If you'll be around and want to talk evaluation, judge reliability, or where this goes next, I'd love to chat. Reach me on [LinkedIn](https://www.linkedin.com/in/ryanlail/) or [X](https://x.com/ryan__lail).
