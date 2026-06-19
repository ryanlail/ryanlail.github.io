---
layout: post
title: "Making LLM judges more reliable without fine-tuning"
date: 2026-05-29
tags: [research, llm-as-a-judge, evaluation, icml]
description: >-
  Improving RewardBench 2 performance by 13.5pp; accepted at the ICML 2026
  Workshop "Statistical Frameworks for Uncertainty in Agentic Systems"
subtitle: 'Improving RewardBench 2 performance by 13.5pp; accepted at the ICML 2026 Workshop "Statistical Frameworks for Uncertainty in Agentic Systems"'
---

- 📄 Paper: [arXiv:2604.13717](https://arxiv.org/abs/2604.13717)
- 💻 Code & data: [composo-ai/llm-judge-criteria-ensembling](https://github.com/composo-ai/llm-judge-criteria-ensembling)
- 📍 Presenting at: ICML 2026 Workshop "[Statistical Frameworks for Uncertainty in Agentic Systems](https://agentic-uncertainty-icml2026.github.io/)"; also accepted at ICML 2026 Workshop "[Combining Theory and Benchmarks: Towards A Virtuous Cycle to Understand and Guarantee Foundation Model Performance](https://sites.google.com/view/icml-ctb/home)"

---

When evaluating AI systems, we have to make a trade-off between quality and scalability. We can either:

1. Ask human experts to analyse the AI's actions and outputs to ensure they comply with the AI controller's definition of quality and safety. Naturally, this doesn't scale very well.
2. Use an automated system, including classic NLP methods (e.g. n-gram overlap metrics like BLEU/ROUGE, or named entity recognition) or modern LLM-based methods, "LLM as a Judge", to rate the quality of an AI's actions and outputs. This scales much better than human experts; however, agreement between human experts and automated systems is well-documented as an issue.

We have been working on improving automated quality checks using LLM as a Judge. Other work has shown that fine-tuning dedicated reward models can prove very effective at performing better on reward model benchmarks (e.g. [Skywork Reward Model](https://github.com/SkyworkAI/Skywork-Reward-V2) dominated the [RewardBench leaderboard](https://huggingface.co/spaces/allenai/reward-bench)). However, the problems I'm working on at Composo AI often leave us with customers asking for state-of-the-art automated evaluation quality, without requiring their data to be fine-tuned on (which they often don't want to give to a third party, and would require a large amount of specific data they don't have). Also, fine-tuning is quickly becoming a thing of the past in the frontier lab offerings (OpenAI [deprecated their fine-tuning API](https://developers.openai.com/api/docs/deprecations)), meaning we are often stuck on open-weight models, which have a performance impact too.

With this in mind, we have been exploring how we can improve the quality of automated LLM as a Judge evaluations, using only techniques that can be applied to standard model inference APIs. I had been using a set of these techniques to improve evaluation on real customer data; for the paper we consolidated them and ran standardised experiments on [RewardBench 2](https://arxiv.org/abs/2506.01937) (paper and code above). We tested four: ensembling, a one-line task-specific criterion, a calibration example, and adaptive escalation to a bigger model when the cheap one looked unsure. The two simplest, ensembling and criteria, dominated the cost-accuracy Pareto frontier, taking accuracy from 72.3% to 85.8%. The most promising technique, ensembling, controlled the variance term of the bias-variance error decomposition of LLM Judge errors, where the improvement was greatest in the smallest models. From an AI Safety perspective, safety guardrails are bounded by how good, fast and cheap automated evaluations are, so this improvement can also enable more reliable safety guardrails.

Ensembling the same model averages away variance but not bias, because every sample shares the model's blind spots. Decomposing the error in the ensemble's outputs should give a better understanding of uncertainty, and studying the bias error is what I'm thinking about next.

---

I will be at **ICML in Seoul this July**. If you are around and want to talk evals, judge reliability, or any of the above, find me on [LinkedIn](https://www.linkedin.com/in/ryanlail/) or [X](https://x.com/ryan__lail).
