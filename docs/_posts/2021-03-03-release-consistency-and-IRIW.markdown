---
layout: post
title:  "Release consistency and IRIW"
date:   2021-03-03 18:00:30 +0200
categories: jekyll update
---
This post/blog is work in progress.

The primary goal of this post is to polish up my understanding why release consistency is failing the IRIW litmus test

To answer this question, lets first have a look what release consistency is. Imagine a cook is preparing food and on completion the cook notifies the customers the food is ready. As soon as the customers hear the cook, they go to the table and expect to food to be ready.

Lets cast the example to code: A,B are global variables initialized as 0, and ra, rb are local variables.

{% highlight java %}
Thread1:
    A=1
    B=1

Thread2:
    rb=B
    ra=A
    if (rb==1 and ra==0) print('violation')
{% endhighlight %}

To guarantee this code will behave as expected, we can use release consistency: a consistency model where loads are acquire-loads and stores are release-stores. For more information see the excellent post of Jeff Preshing: https://preshing.com/20120913/acquire-and-release-semantics/.

In practice you do not want to convert all loads to acquire-loads and stores to release-stores. Even though on the X86 loads are acquire-loads and stores are release-stores, it will prevent a lot of compilizer optimizations. So you just want do convert them at points where data is being synchronized which in this case is on B. 

In the below digraph a happens-before edge is established between the write of A=1 and the read of A=1 (remember that the happens before relation is transisitive) and therefor our example should work as expected.

![Release Consistency](/Images/release_consistency.png "Release Consistency")

# Implementing a release-store

To implement the release-store of B, we can insert the following fences:
{% highlight java %}
A=1
[LoadStore]
[StoreStore]
B=1
{% endhighlight %}

Since we only need to prevent the 2 stores from being reordered, [LoadStore] isn't strictly needed.

In C++ you would use an atomic.store with memory_order_release and in Java a VarHandle.setRelease. Release-stores on the X86 are cheap because there is no [StoreLoad]; unlike with volatile write in Java or atomic store with memory_order_seq_cst.

# Acquire load
To implement the acquire-load, we need to insert the following fences:

{% highlight java %}
r1=B 
[LoadLoad]
[LoadStore]
r2=A
{% endhighlight %}

In this case we can get rid of the [LoadStore] fence since we only need to order 2 loads.

In C++ you would use an atomic.load with memory_order_acquire and in Java you would use a VarHandle.getAcquire.

## Weak consistency

Release-consistency is a weak form of consistency and it can be used on systems where there are mulpliple versions of the data, but they could have gotten out of sync. E.g. in a distributed system where the replicas have diverged or in a processor that would allow for caches to get incoherent.

Release consistency is weaker than Total Store Order (TSO), the memory model for the X86. And TSO is a relaxation of sequential consistency (SC) due to the existence of store buffers which can cause older stores to be reordered with newer loads to a different address. SC requires a total order over all loads and stores, TSO only requires a total order over the stores. In simple terms: it should not be possible that one CPUs sees one order of stores and a different CPU sees a different order. 

Unlike release consistency, TSO requires caches to be coherent: there should be a total order over all memory accesses on a single address, and a load needs to see the last write before it.

## IRIW Litmus Test

A litmus test for TSO is the Independent Read, Independent Write (IRIW) test.

{% highlight java %}
CPU1: 
    A=1

CPU2: 
    B=1

CPU3: 
    r1=A
    r2=B

CPU4: 
    r3=B
    r4=A
{% endhighlight %}
    
So we have 2 completely independent stores; with TSO it should not be possible to see these 2 stores in a different order. This is depected in the below digraph.

![IRIW TSO](/Images/iriw_TSO.png "IRIW TSO")

Execution has cycle: cycle bad.

cc is the cache coherence order. The [LoadLoad] comes from TSO; which prevents loads from being reordered. Ans as can be seen, 



So the question is what the value of r4 can be. If there is a total order over the stores, then r4 should be 1.. Draw diagram. If r4 would be allowed to be 0, then you would get a cycle and the execution valid according to TSO.

Release consistency is a relaxation of TSO whereby such a total order on the stores doesn’t need to be enforced.

![IRIW RC](/Images/iriw_RC.png "IRIW RC")

Could it be that we end up with r1=1,r2=0 and r3=1 and r4=0? Yes, because for release consistency we just need to make sure that the 2 loads don’t get reordered, but it doesn’t say anything about independent writes being ordered.

How could this happen in practice? If CPU caches would not be coherent. CPU1 and CPU2 could be on one node and CPU3 and CPU4 could be on another node. CPU2 sees the A=1 of CPU1 immediately. And CPU4 sees the B=1 of CPU3 immediately. BUt it takes some time for the A=1 to arrive on the node of CPU3/4 and/or B=1 with to arrive on node of CPU1/2.
TSO and IRIW


 

 
