---
layout: post
title: CacheBlend: Fast Large Language Model Serving for RAG with Cached
description: "Paper review series 2025 - 2"
img:
date: 2025-01-30  +1045
---

Presented this paper at my team's reading group. An interesting paper which proposes KV cache related optimizations for RAG.

In RAG, a given document or book or set of files are broken into sentence chunks and fed into a vector embedding database. When a user query arrives, the query is matched against the sentence embeddings and relevant chunks are retrieved and prepended to the query. 

This increases the latency of the inference requests since the entire prefix (consisting of multiple retrieved chunks) needs to be prefilled.

There are 4 possible ways of doing this, varying in the amount of computation needed (latency of inference) and therefore accuracy of the output:
1. Baseline (Full KV recompute): High latency. Intended output. Has complete causal cross-attention between all the retrieved chunks.
2. Prefix caching: Cache the self-attention KV of each of the chunks. If prefix matches then we can reuse the cached KV of that first chunk alone (only self-attention within chunk 1). Others are only self-attention and don't have cross-attention with chunk 1 and so on and so forth. So cannot be used. This gives small improvement with no accuracy loss whatsoever.
3. Full KV Reuse: Cache the self-attention KV of each of the chunks. Perform no cross attention at all. This results in bad output accuracy. They show this both through an input example (Fig 3) and a graph (Fig 2). Problem is cross-attention cannot be computed a priori since order of retrieval is not known beforehand.
4. Selective KV Recomputation: Their solution. Need trade-off between full KV Reuse and full KV Recompute

Their solution is to get the best of the full KV reuse (low latency) and the full KV Recompute (high accuracy) solutions. They perform recomputation only for a small fraction of the tokens (around 15%). Further, they optimize by pipelining (perform recomputation of the 15% in parallel with the loading of the KV Cache of the remaining 85%). This hides the inference latency cost while giving good accuracy.

Their solution has 2 parts:
1. How to do the selective re-computation?
2. Optimization by pipelining the recomputation and the loading phases


Part 1: Selective recomputation

To select the tokens which need to be recomputed, the paper defines HDKV (High-Deviation KV tokens) as those whose staleness causes high deviation in the attention. Attention sparsity and experiments also speak of this effect. The problem is that these tokens can only be identified after re-computing their KV, which is what needs to be avoided in the first place. The authors empirically show that the HDKV are the same across layers. Using this insight, they then use a gradual filtering method where they recompute all tokens in the first layer, followed by r+d% tokens in the second layer (where r is the recompute ratio they want to achieve), followed by gradually lesser numbers until they hit r% recompute.

Part 2: Pipelining

In their system, they have 2 estimators: a recompute delay estimator, which estimates the time it takes to recompute r% of the tokens for a particular model etc. and a loading delay estimator, which estimates time to load n tokens' KV cache from storage. Using both these estimators, they can (i) given a recompute ratio, identify the storage location (GPU, CPU RAM, SSD) which will hide the recomputation latency the best (no gains from storing closer, since recomputation is the bottleneck). Or (ii) given a storage location, identify the highest recomputation ratio to hide loading latency (no gains from recomputing lesser, since loading is the bottleneck). 

Their evaluation shows that their approach gives them a reduction of upto 40-50% latency for the prefill time on various datasets without much loss in accuracy.

## Key Takeaways
1. I am partial to their key idea, of optimizing RAG by reusing cache, as I too identified the recomputation in RAG as having potential for optimization by caching. My personal brilliance however stopped short at Full KV Reuse, which I did not deduce had such critical accuracy issues. So, the KV Reuse (no cross-attention approach) is a key idea but only upon implementation and testing can one truly know whether it is also a feasible one at that. 
2. Selective recomputation is done and dusted by a variety of KV cache optimization papers. Attention sparsity is widely identified and used.
3. I do like their systems optimization of pipelining the recomputation latency by hiding it with the loading latency, giving them a larger cache storage.
