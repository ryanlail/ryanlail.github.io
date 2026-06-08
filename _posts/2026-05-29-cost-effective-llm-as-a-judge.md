---
layout: post
title: "Making LLM judges more reliable without fine-tuning"
date: 2026-05-29
tags: [research, llm-as-a-judge, evaluation, icml]
description: >-
  Most of what we call LLM-judge unreliability is just noise. What cheaply
  averages it away, what does not, and why the same question gets sharper when
  the judge is a safety monitor.
---

- 📄 Paper: [arXiv:2604.13717](https://arxiv.org/abs/2604.13717)
- 💻 Code & data: [composo-ai/llm-judge-criteria-ensembling](https://github.com/composo-ai/llm-judge-criteria-ensembling)

---

An LLM writing prose is sampling from a distribution, and nobody finds that strange. An LLM scoring a response is doing the same thing, which somehow feels stranger: ask the same judge the same question twice and you will often get two different scores. Once you take that seriously, a lot of what we call judge unreliability is just noise, and the question is how much of it you can average away.

Reliable evaluation is the thread through most of my work, and at Composo I wanted to make our judges better without fine-tuning, which is costly, generalises poorly, and struggles to use the customer-specific context that decides what "good" even means. So the question I set out to answer was narrow: how far can you push an LLM judge with drop-in changes at inference time and no training? The obvious reason to care is the cost, but it is not mine. The same cheap checks that tighten a judge are the ones you would reach for to catch a model underperforming on purpose, and a check doing that job has to be trusted in a way a product metric never does. The full version is the paper linked above, which I am presenting at the ICML 2026 Workshop on Statistical Frameworks for Uncertainty in Agentic Systems. This is the short one.

We tried four drop-in techniques on RewardBench 2: ensembling, a one-line task-specific criterion, a calibration example to fight anchoring, and adaptive escalation to a bigger model when the cheap one looked unsure. Two did almost all the work: criteria and ensembling took the full-size model from roughly 72% to 84%, and nothing cleverer beat that once you priced it in. What helped was reducing the judge's noise, not making it smarter.

What did not work is the more telling half. Calibration was aimed straight at anchoring and helped at a single sample, but once we ensembled it added nothing, since the ensemble had already removed the noise it targeted: a careful prompt beaten by sampling a few more times and averaging. The same logic turns up elsewhere. A small model with a large ensemble beat every big one, Haiku 4.5 at eight samples with criteria reaching 85.8% at about a quarter of the cost, and the cheap fixes helped most where the judge was weakest, with the largest jumps in maths and precise instruction-following.

The limitation I keep circling is that ensembling the same model cancels noise but not bias. Every sample shares the model's blind spots, so a confident, consistent mistake survives any amount of averaging. The honest next step on judges is to decompose the score and separate systematic bias from noise rather than average over both, roughly what AISI's [Skewed Score](https://www.aisi.gov.uk/blog/llm-judges-on-trial-a-new-statistical-framework-to-assess-autograders) does.

The version I find more interesting raises the stakes. The moment you trust a cheap consistency or variance check as a detector, say to catch a model deliberately underperforming on an eval, or sandbagging, the bias and variance picture tells you when it breaks. A model that sandbags only when it senses an eval can push its tell out of the within-context bias a detector sees and into the across-context variance it cannot, so the check looks clean exactly when it should be alarmed. AISI's [auditing-games work](https://arxiv.org/abs/2512.07810) already shows black-box detection defeated by conditional sandbaggers, but only qualitatively. I am now pointing the bias and variance decomposition from [The Hot Mess of AI](https://arxiv.org/abs/2601.23045) at this, to work out when these detectors give false reassurance and whether the blind spot can be closed without already knowing the trigger. That is the next post.

A check that ranks chatbot answers is a product problem; the same check pointed at a model hiding what it can do is a safety problem. Nothing about the instrument changes, only how much it costs you to be wrong about it.

---

I will be at **ICML in Seoul this July**. If you are around and want to talk evals, judge reliability, or any of the above, find me on [LinkedIn](https://www.linkedin.com/in/ryanlail/) or [X](https://x.com/ryan__lail).
