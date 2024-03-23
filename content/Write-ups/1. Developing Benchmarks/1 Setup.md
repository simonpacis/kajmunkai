---
title: Setup
draft: "true"
---
In defining benchmarks, we need to specify what it is we are looking to improve by way of fine-tuning. We need different benchmarks for different stages of the project. For phase 1, that is [[1 Planning|Fine-tuning LLaMa 2 to be conversational in Danish]], we need one set of benchmarks, and for phase 2 (which will probably be further divided at a later point) we need a different set of benchmarks.

# Phase 1 Benchmarks
What we are looking for in terms of improvement here:
- Model to become conversationally fluent in Danish.

We need some metrics for comparison. I do not assume that there will be any spelling errors, as LLaMa 2 is already trained on some Danish data and has an inherent understanding of the language. 



The best type of benchmarks for us would then be:
1. Language proficiency tests, focusing on Danish conversational skills.
	1. With quantitative metrics on context relevance.
2. Comparison against baseline performance metrics before and after fine-tuning with Danish datasets.
3. Human evaluation of conversational fluency and accuracy in Danish, incorporating diverse and contextually rich dialogue scenarios.
4. Quantitative metrics on response accuracy, coherence, and context relevance in Danish conversations.
	- <small>It is unlikely that we will see any improvement in terms of cultural awareness, since the dataset will be translated from English and therefore not include specifically Danish cultural references. It will, however, be relevant to test for anyway, if for no other reason than for us to be aware of the models limitations.</small>

We can rule some of these out, as the problems have already been solved previously in the process.

Point 1

These can all be grouped together into the following two categories:

- Language proficiency and fluency
	- Benchmark types 1-2.
- Evaluation and metrics
	- Benchmark types 3-5.

## 



We are not trying to solve general LLM problems, such as [common sense reasoning](https://arxiv.org/abs/1905.07830), we are instead interested in the following: