---
layout: post
title: "Making LLM judges more reliable without fine-tuning"
date: 2026-05-29
tags: [research, llm-as-a-judge, evaluation, icml]
description: >-
  How far you can push an LLM judge using only frontier models and inference-time
  changes, what that buys for scaling oversight, and where it goes next.
subtitle: "Improving RewardBench2 performance by 13.5%"
---

- 📄 Paper: [arXiv:2604.13717](https://arxiv.org/abs/2604.13717)
- 💻 Code & data: [composo-ai/llm-judge-criteria-ensembling](https://github.com/composo-ai/llm-judge-criteria-ensembling)
- 📍 Venue: [ICML 2026 Workshop on Statistical Frameworks for Uncertainty in Agentic Systems](https://agentic-uncertainty-icml2026.github.io/)

---

At Composo I've been building LLM evaluations for high-stakes domains like medical applications. The question I keep coming back to is how to get evaluations that are good enough to trust, cheap enough to run at volume, and honest about their own uncertainty, without needing a pile of domain-specific data for every new use case.

You can buy a lot of evaluation quality by fine-tuning a reward model, but it is slow and expensive, it has to be redone every time a new model ships, and it usually keeps you off the latest frontier models. So I wanted to know how far you can get using only frontier models and drop-in changes at inference time, with no training. This matters beyond cost: how much oversight you can run over deployed systems, guardrails and the like, is bounded right now by how good and how cheap LLM-based evaluation can be, so better and cheaper evaluation widens that directly.

I had been using a set of these techniques to improve evaluation on real customer data; for the paper we consolidated them and ran standardised experiments on RewardBench 2 (paper and code above). We tested four: ensembling, a one-line task-specific criterion, a calibration example to fight anchoring, and adaptive escalation to a bigger model when the cheap one looked unsure. The two simplest, ensembling and criteria, did almost all the work, taking accuracy from 71.7% to 85.8%, and the more complex methods were dominated once you priced them in. One thing I would change: we ran the sweep starting with the large model, so I only realised late that the simple methods dominated, where doing the cheap end first would have saved the effort I spent on the complex approaches.

The result that changed how I think about this: a small model with a large ensemble beat every large one. Haiku 4.5 at eight samples with criteria reached 85.8%, at about a quarter of the cost of the large-model ensemble. A judge is not really an autograder that emits a score, it is a conditioned distribution, and a smaller, noisier model can carry useful signal in the shape of that distribution that you throw away by taking a single sample.

That reframing is what I am working on next. Ensembling the same model averages away noise but not bias, because every sample shares the model's blind spots. Decomposing the error in the ensemble's outputs should give a better handle on uncertainty, in the spirit of AISI's [Skewed Score](https://www.aisi.gov.uk/blog/llm-judges-on-trial-a-new-statistical-framework-to-assess-autograders), and might rescue the model-routing idea that underperformed here.

---

I will be at **ICML in Seoul this July**. If you are around and want to talk evals, judge reliability, or any of the above, find me on [LinkedIn](https://www.linkedin.com/in/ryanlail/) or [X](https://x.com/ryan__lail).
