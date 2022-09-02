---
layout: post
title: "What is the difference between sequential consistency and coherence"
categories: memory-models
---

# Introduction

The goal of this post is to explain the difference between sequential consistency and coherence. TLDR: coherence is sequential consistency per location.

## Sequential consistency

For an execution to be sequential consistent, it needs to have the same outcome as a different execution that has the following properties:
- total order of loads and stores over **all** locations
- this order needs to be consistent with the program order of each of the cores.
- a read needs to see the most recent write before it in this order.

So the actual ordering of loads and stores isn't relevant, as long as the outcome can be explained by a different execution that has the above properties. This gives a lot of flexibility to the compiler and the CPU/memory-subsystem.

## Coherence

For an execution to be coherent, it needs to have the same outcome as a different execution that has the following properties:
- total order of loads and stores over a **single** location
- this order needs to be consistent with the program order of each of the cores.
- a read needs to see the most recent write before it in this order.

So the only difference between sequential consistency and coherence is that sequential consistency requires a total order over loads/stores over **all** locations while coherence only over a **single** location. So you could see coherence as sequential consistency per location.

Therefor any execution that is sequential consistent, is coherent. But not every execution that is coherent, is sequential consistent. This is demonstrated in the next section.

This also brings us to the difference between consistency and coherence: consistency orders loads/stores between different locations, but coherence only orders loads/stores the the same location.

It is important to realize, that coherence (and sequential consistency) provides no real time guarantees. So reads and writes can be skewed as long as the above properties are preserved. 

In reality the real time order is likely to be preserved because modern caches are linearizable; the linearization point of a read/write to a cache-line is between issueing the request and getting the response. And since linearizability is composable, the whole cache is linearizable. 

## A coherent but non sequential consistent example

An example of an execution that is coherent but not sequential consistent, is the Independent Reads of Independent Writes (IRIW) test:

```
int A=0,B=0

CPU-1:
   A=1
   
CPU-2:
   B=1
   
CPU-3:
   r1=A
   [LoadLoad]
   r2=B
   
CPU-4:
   r3=B
   [LoadLoad]
   r4=A
```

Can it be that r1=1, r2=0, r3=1, r4=0? So can it be that the stores are seen in different orders? If a system is sequential consistent, then this can't happen since there needs to be a total order over all loads/stores; so either A=1 was first, or B=1 was first. But with a system that is coherent, the above is allowed since coherence doesn't say anything about loads/stores to different addresses. 

For an excellent read on this topic and a much more in depth explanation of the above, please check out [A Primer on Memory Consistency and Cache Coherence, Second Edition](https://www.morganclaypool.com/doi/10.2200/S00962ED2V01Y201910CAC049) 
 
