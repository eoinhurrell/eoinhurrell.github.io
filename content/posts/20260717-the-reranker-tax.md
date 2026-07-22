+++
title = "The Reranker Tax: When a Smart Layer Can't Save a Weak Foundation"
author = ["Eoin H"]
publishDate = 2026-07-16T00:00:00+00:00
tags = ["post"]
draft = false
+++

Every time you choose to add complexity rather than measure and improve the system you already have, you gamble with your time and your focus. The new component arrives with a tax — latency, cost, more moving parts to operate — and it pays off only when the thing underneath it was already doing its job. Without a solid foundation and good measurement, the upgrade is motion, not progress.

This is a post about one instance of that gamble — adding a reranker to a search system — and how the same shape repeats somewhere less obvious: the AI agent you point at a search tool you built. In both cases the temptation is to say "the smart layer will sort it out." It won't. A smart layer can only refine what the layer beneath it hands it.

Modern search systems have a common architecture; first, retrieve a large candidate set of items, then rerank them to order by relevance for the final search results. The reranker is the smart layer; retrieval is the foundation.

This two-stage architecture implies a reranker tax, which is paid in three ways — latency (more work, slower response), cost (added architecture, added GPU), and complexity (another service to run, monitor, and debug). When building the smart reranker layer it's often overlooked that the tax is only worth paying when your retrieval gets lots of good items but isn't ranking them well — high-stakes, low-QPS retrieval, hard queries where the first-stage lexical or dense retriever genuinely can't separate near-duplicates, and a listwise reranker earns its keep on subtle ordering.

## The smart layer needs support - A reranker can only reorder what it's given

A reranker reorders what the first stage hands it, so reranked nDCG@10 is hard-capped by first-stage Recall@k — you cannot promote a document the retriever never returned. This is the load-bearing fact. No matter how expensive the model, it cannot rescue quality if the documents that would have helped were never retrieved in the first place.

Too often the decision is made to build the reranker as a fix for feelings of poor quality, to add the smart layer without context. Diagnose your first stage before you add a reranker. Measure the recall@k of your retrieval system alone, to establish the potential quality of the items your reranker would rerank. If your retrieval recall is low, no reranker — however expensive — can help, because the relevant documents aren't in the candidate pool to be promoted. Spend on recall first.

## Measure your systems to quantify the potential of additions

It should be easy to do a small experiment to test. I'm using [retrieve-bench](https://github.com/eoinhurrell/retrieve-bench) a project I'm building to benchmark retrieval systems, to test the impact of reranking. For this experiment I'm interested in quality (nDCG@10) and speed (the p50/p95 latency delta), not just accuracy alone, to show the tradeoff curve.

I take [BM25](https://www.geeksforgeeks.org/nlp/what-is-bm25-best-matching-25-algorithm/) over BEIM's SciFact dataset — 5,183 scientific abstracts, 300 queries — as the first stage, deliberately a lexical retriever and not a hybrid, so the reranker's contribution isn't diluted by a strong dense stage sitting in front of it. Onto that we bolt a single reranker: a pointwise cross-encoder ([cross-encoder/ms-marco-MiniLM-L6-v2](https://huggingface.co/cross-encoder/ms-marco-MiniLM-L6-v2)) that rescores the top-k candidates the first stage returns. Quality is nDCG@10 with retrieve-bench's bootstrapped 95% confidence interval (seed=42, 1000 resamples) — every number carries a ±; cost is per-query p50/p95 latency in milliseconds (mean and upper bounds of how long a query takes), measured steady-state after a warmup so model load isn't charged to the bill. The one knob we turn is the first-stage depth k, the size of the candidate pool the reranker gets to reorder.

### Figure A — the tradeoff curve

{{< figure src="/images/20260717-reranker-tax-curve.png" >}}

Figure A plots the result. On the x-axis is p50 latency (log scale); on the y-axis is nDCG@10 with its confidence interval. BM25 sits in the far-left corner — essentially free, 0.11 milliseconds per query, nDCG@10 of 0.662. The cross-encoder enters at three depths, and the picture is brutal: at k=5 and k=10 it is dominated by free BM25, costing 47 ms and 100 ms per query respectively to match or undershoot BM25's quality, because at those depths the first stage simply hasn't retrieved enough relevant documents for the reranker to reorder anything useful. Only at k=20 does the cross-encoder clear BM25, lifting nDCG@10 by +0.009 (0.662 → 0.671) — a real gain, but a tiny one that does not escape the confidence interval, and it costs roughly 2,000× the latency (0.11 ms → 225 ms). So, your smart layer, be it reranker or AI agent, might only lead to marginal improvements, and perform significantly slower.

### Figure B — the recall ceiling

{{< figure src="/images/20260717-reranker-tax-recall-ceiling.png" >}}

Figure B explains why the shallow-k points in Figure A don't benefit from the reranker, and it is the more important of the two figures. As we widen k from 5 to 10 to 20, Recall@k climbs from 0.72 to 0.77 to 0.82, and only at the top of that climb does the cross-encoder's lift appear: at k=5 and k=10 the reranker moves the needle by essentially zero, because the relevant documents aren't in the candidate pool to be promoted; at k=20 it finally finds enough good candidates to reorder into the +0.009 gain. That is the lesson - you need good candidates in your pool before a reranker pays off, put another way; the tools your smart layer has access to need to work well.

SciFact is a corpus where BM25 is unusually strong — lexical matching on technical terms does the work, and there is no dense stage to dilute the credit. This is the _easy_ case for the baseline, which makes it a _lower bound_ on the reranker's value, not a typical case. And even here, in the case most favourable to the existing system, the unmeasured upgrade doesn't clear the baseline within the confidence interval — at roughly two thousand times the latency. That is not an argument against rerankers. It is the strongest possible argument for measuring first. The only honest way to know whether the upgrade is worth its tax, for your corpus and your queries, is to measure.

## Agents repeat the pattern - the smart layer needs a solid foundation

Here is where the shape repeats. You build a search tool — an MCP server, a retrieval API, a knowledge base an agent can call — and point an AI agent at it: Cursor, Claude Code, whatever wraps the model. The temptation is identical to the reranker's: _the agent will sort it out._ It won't; an agent curates its context window from what its tools hand it, like a reranker orders the documents its given. Both are curators working over a set the foundation produced — and a curator can arrange what it was given, never summon what was never retrieved. If the search tool is incapable of producing good results no smart layer on top if it will change that. Agents can reformulate their approach and try again in the hope of adequate results, but this takes time and tokens, latency and cost, the tax compounds.

Every rephrased query still runs against the same index, the same embeddings, the same blind spots; reformulation lets the agent ask differently, not make the retriever return a document it has no path to. The recall cap is still there; it's just hidden under the agent's looping, which looks like productive work.

Measuring and understanding what your retrieval is actually missing. A smart layer can only refine what the layer beneath hands it, so before you bolt one on, measure what that layer is actually handing you. That goes for the reranker, and for the agent you point at it. The tax is worth paying — but only once you know what it's buying.
