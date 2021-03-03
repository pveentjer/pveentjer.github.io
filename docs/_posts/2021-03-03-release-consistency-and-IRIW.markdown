---
layout: post
title:  "Release consistency and IRIW"
date:   2021-03-03 18:00:30 +0200
categories: jekyll update
---
The primary goal to make this post was to polish up my understanding why release consistency is failing the IRIW litmus test

So what is release consistency? Let's have a look at the following example:

Thread1:
A=1
B=1

Thread2:
    rb=B
        ra=A
           If (rb==1 and ra==0) print(“violation”)

Can it be that “violation” gets printed with release consistency? If there is release consistency, “violation” can’t be printed. If a thread sees B=1, then it should also see A=1.

Todo: add Graph

Practical example: dinner. So the cook cooks dinner (A=1) and then signals that the food is read (B=1), if I hear the cook saying the dinner is ready (rb==1), then the dinner should be ready (ra==1). 

This sounds very obvious but in a system where there are multiple versions of the data, data could get out of sync. For example a distributed system with multiple replicas of the data or a system with incoherent caches.

Release consistency is a consistency model where loads have acquire-semantics and stores have release-semantics. For more detailed information see the excellent blogpost of Jeff Preshing: https://preshing.com/20120913/acquire-and-release-semantics/
Store release
So we need to prevent that the 2 stores of thread 1 get reordered; this can be done by making B=1 a release store:

A=1
[LoadStore][StoreStore]
B=1

In this case we only need to prevent the 2 stores from being reordered, so we can get rid of the [LoadStore].

In C++ you would use an atomic.store with memory_order_release and in Java you would use a VarHandle.setRelease.
Acquire load
We also need to prevent the 2 loads from being reordered, this can be done by doing making r1=B an acquire load:

r1=B 
[LoadLoad][LoadStore]
r2=A

In this case we can get rid of the [LoadStore] fence since we only need to order 2 loads.

In C++ you would use an atomic.load with memory_order_acquire and in Java you would use a VarHandle.getAcquire.

Note: on the X86 loads are acquire-loads and stores are release-stores, so it just is a restriction on the compiler optimizations.
IRIW Litmus Test
Total Store Order (TSO), the memory model for the X86, is a relaxation of sequential consistency (SC) due to the existence of store buffers which can cause older stores to be reordered with newer loads to a different address. So instead of enforcing a total order over all loads and stores, it only requires a total order over the stores. In simple terms: it should not be possible that one CPUs sees one order of stores and a different CPU sees a different order. 

A litmus test for TSO is the Independent Read, Independent Write (IRIW) test.

CPU1: A=1

CPU2: B=1

CPU3: r1=A
    CPU3: r2=B

CPU4: r3=B
    CPU4: r4=A

So we have 2 completely independent stores; with TSO it should not be possible to see these 2 stores in a different order.

TODO: Add chart that shows the cycle.




TSO is more strict than release consistency because apart from having acquire-loads and release-stores, there should be a total order over all the stores (or you should be able to pretend there is such an order). TSO requires consistent caches. So we can add the cache coherence order to the mix which is a total order over all 


So the question is what the value of r4 can be. If there is a total order over the stores, then r4 should be 1.. Draw diagram. If r4 would be allowed to be 0, then you would get a cycle and the execution valid according to TSO.

Release consistency is a relaxation of TSO whereby such a total order on the stores doesn’t need to be enforced.

Could it be that we end up with r1=1,r2=0 and r3=1 and r4=0? Yes, because for release consistency we just need to make sure that the 2 loads don’t get reordered, but it doesn’t say anything about independent writes being ordered.

How could this happen in practice? If CPU caches would not be coherent. CPU1 and CPU2 could be on one node and CPU3 and CPU4 could be on another node. CPU2 sees the A=1 of CPU1 immediately. And CPU4 sees the B=1 of CPU3 immediately. BUt it takes some time for the A=1 to arrive on the node of CPU3/4 and/or B=1 with to arrive on node of CPU1/2.
TSO and IRIW

Cache coherence has the following 2 properties:
There needs to be a total order over all loads and stores on a single address
A load needs to see the most recent store before it in the memory order.

 
