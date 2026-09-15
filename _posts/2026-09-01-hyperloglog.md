---
layout: post
title: "Exceptional Data Structures: HyperLogLog"
date: 2026-09-01 09:18:00 -1600
categories: [Exceptional Data Structures]
tags: [c++, data engineering, probabilistic, data structures]
math: true
---

## Introduction

Imagine you receive an unbounded stream of items or just have a very large data set (IP addresses, strings, integers, anything hashable) and you are required to compute its *cardinality* (the number of unique items you have seen). 

You might quickly reach for a `set` or equivalent, where you keep track of what you have already seen in-memory. If you have already seen an item, you can just discard it, otherwise, add it to the set.
```
seen = set()

for item in stream:
	if item not in seen:
		seen.add(item) 
```
Simple! But because of the data's unboundedness or very large size, you might quickly run into an out-of-memory (OOM) error. 

Enter the HyperLogLog. In exchange for some accuracy (more later), we can shrink our memory footprint and achieve $O(1)$ behaviour for space complexity. Effectively meaning that we can roughly track billions of items, while occupying RAM in the kilobyte range.

## Core Mechanism

At its core, the HyperLogLog relies on a clever hashing mechanism. To build intuition, consider a simplified example: 

---
Your stream just sent you the string `"Hello, World!"`. 

1. We'll hash it to produce a hash value.
   
   $$
   hash('Hello,\; World!') \rightarrow (1805101170)_{10}
   $$

    > Our hash function must be deterministic (hashing the same item again should produce the same hash), uniformly distributed (more later) and fixed-width. A good candidate is Murmur3's 128-bit version, taking only the first (or second) 64-bits.
    {: .prompt-warning } 

2. Next, take the binary representation of the hash, and count the number of leading zeros.

   $$
   (78042699)_{10} \rightarrow (00000100101001101101011001001011)_2
   $$

3. In this case, $5$. Keep track of the most leading zeros you've seen so far.

4. Repeat this for every item in your stream or data set, updating the maximum. 

---

That's about all the detail needed to build the intuition. Now, let's turn to the estimation. 

Let's assume that the maximum number of leading zeros observed so far is $10$. This roughly translates to *"we hashed so many items that the most number of leading zeros we saw was $10$"*.

But, can we estimate how many items we must have hashed to produce a value of $10$? Yes! 

Because we assumed our hash function is uniformly distributed (every bit pattern is equally likely), we can apply some basic probability theory to estimate this number. However, before we jump into any math or plugging numbers into formulae, we should build an intuition around bit patterns and probability. 

> In the official algorithm, the position of the first set bit (1-indexed) is counted instead of the number of leading zeros. But it is mathematically equivalent to counting the number of leading zeros. 
{: .prompt-warning }

### Bits & Coin Flips

We can model fairly flipping coins as generating uniformly distributed bits very well.

$$
0 \rightarrow Heads \\ \\
1 \rightarrow Tails
$$

We know the probability of seeing 'heads' on a fairly tossed coin is exactly $0.5$, and so, we can map the probability of a fairly (uniformly) generated bit being '$0$' as, also, $0.5$.

$$
P(C=H) = 0.5 \\ \\
P(B=0) = 0.5 
$$

Now, we can move on to multiple 'heads', and therefore,  multiple bits. The probability of tossing $3$ 'heads' is $1/8$, and so is the probability of generating $3$ '$0s$'.

$$
P(H_3) = 1/2^3 = 1/8 \\ \\ 
P(0_3) = 1/2^3 = 1/8
$$

But what does $1/8$ mean really? Well, if read as $1$-in-$8$, it means that the occurrence of said event is once every eight times ***on average***. So, every $8$ times we toss three coins, we will read $3$ heads once ***on average***. The same way, if we generate $3$ random bits $8$ times, we will observe $3$ '$0s$' once ***on average***. 

This generalizes to the formula $1/{2^{n}}$ where $n$ is the number of sequential but independent events by virtue of the <a href="https://en.wikipedia.org/wiki/Probability#Independent_events" target="_blank" rel="noopener noreferrer">multiplication rule.</a>

$$
P(E_n) = 1/2^n
$$

### Estimation

Hopefully, we've grasped how probability theory applies to bit sequences with the help of the most classic probabilistic experiment.

If we apply this to our earlier example of observing a maximum of $10$ leading zeros, we get a probability of $1/1024$.

$$
P(0_{10}) = 1/2^{10} = 1/1024
$$

If we read this in the fashion we established earlier, we understand that this bit pattern occurs once every $1024$ times, ***on average***. After some rephrasing, we can say that we need to hash about $1024$ items before we observe a random hash with $10$ leading zeros. 

### Accuracy & Noise

*In theory*, our implementation so far is great. *In practice*, our implementation will not be as great. Currently, our basic algorithm has very high **variance**. We can easily get (un)lucky and hash only a few items but still see many leading zeros, or vice versa.

For example, we might hash ~$1024$ items, but only see a maximum of $9$ leading zeros. Our estimate would be $512$, which is off by $2\times$! 

To address this, the original paper implements the HyperLogLog with multiple registers, then computes a harmonic mean over these registers. 

The first few bits of the hash are used to index a collection of registers, and the rest used to count leading zeros within that register. This collection can be a simple array of integers. They represent the maximum number of zeros seen *at that index*. You might think of it as our basic algorithm being applied on a collection of maximums rather than just a single one. Then, during estimation, the harmonic mean is taken over these registers. This pairing reduces noise and the impact of outliers on our overall estimate. No longer will a single outlying register bring the average up. 

Altogether, the HyperLogLog boasts a standard error rate of about $1-2\%$, while using just a few $KBs$ of memory. 

## Design Choices & Tradeoffs

Using a fixed-bit width hash can have some implications we can take advantage of. 

First, let's assume the commonly used $64$-bit hash while using the first few bits, let's say `b` for indexing. So, the absolute maximum number a register can possibly be at is $(64 - b)$. 

This can be well-packed into a `uint8_t` (or a single byte). An optimal value for `b` is $14$, this means we'll have $2^{14} = 16,384$ registers. A byte per register means $16,384$ bytes or $16KB$.

That is a remarkably small memory footprint, to track up to trillions of items! But a small memory footprint like $16KB$ also lets us fit the register array in $L2$/$L3$ cache, meaning hot data stays close to the CPU.

### Going Concurrent

For thread-safety, we can make use of C++'s `std::atomic`, specifically `std::atomic_ref<uint8_t>`. Without going into too much detail, we can use a simple `compare-and-exchange` loop when updating the registers. 

However, this will likely lead to *false sharing*. Because registers are so small, many will end up living on the same cache line. A thread writing to a register might end up invalidating the cache of another thread trying to write on a *different* register, on the *same* cache line. It could be expected that the amount of false sharing decreases over time as register values increase, and therefore, become less likely to be written-to.  

In theory, we might do away with thread-safety within our implementation, and let each thread have its own instance of a HyperLogLog. Merging these instances is possible and trivial by using a `std::max` across all registers. This eliminates false sharing, but, now our memory consumption is bound by the number of threads.
## Implementation
Finally, the nitty gritty. A thread-safe and lock-free implementation can be found [here](https://github.com/shiv42x/HyperLogLog/). It is developed in `C++20`.

WIP: Having made the above design choices, I'm interested in profiling this implementation and comparing performance with the alternative choices. Therefore, I may update this post and implementation in the near future.

## Conclusion

Hopefully, now your unbounded data streams or large data sets have been accounted for, at least in terms of cardinality and by trading some accuracy. The HyperLogLog has shown success where naive approaches will fail. 

It is important to highlight that the HyperLogLog is *exceptional*. In that, it is in *exceptional* situations that you might want to reach for this data structure, since such large amounts of data are rare in the wild. But, in my opinion, it's a good tool to have in your toolbox given its memory footprint and concurrent nature. 
