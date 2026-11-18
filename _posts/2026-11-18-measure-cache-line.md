---
layout: post
title: "Measuring the Size of a Cache Line"
date: 2026-11-18
---

<div class="note" markdown="1">
If you do not know what a cache line is and would like to know more about CPU internals,
consider reading my (much lengthier) [previous blog post](../riscv-benchmark) first.
You can think of it as a unit of cache, the smallest amount of data transferred at once. 

</div>

Recently I came across a piece of code in which `cacheline` was a hardcoded constant.
In that context it was fine, but it got me thinking, would I be able to measure it?

<details markdown="1">
<summary>Looking up cache line size</summary>
The point of this experiment is the challenge itself and the knowledge learned along the way.
If you actually need to write cross-platform code, there are plenty of ways to look up the size.
One such way on Linux is to use `lscpu`.

```
$ lscpu -C
NAME ONE-SIZE ALL-SIZE WAYS TYPE        LEVEL SETS PHY-LINE COHERENCY-SIZE
L1d       32K     128K    8 Data            1   64        1             64
L1i       64K     256K    8 Instruction     1  128        1             64
L2         2M       2M   16 Unified         2 2048        1             64
L3         6M       6M   12 Unified         3 8192        1             64
```
My Intel N100 CPU uses 64 byte cache lines for all of its caches.

In C++17 and later you can use `std::hardware_constructive_interference_size`.
If the goal is to avoid the false sharing phenomenon described later in this blog post
you should use `std::hardware_destructive_interference_size` instead,
though it should be noted that these values can be inaccurate.

```
// Print values from C++ implementation
std::println("Constructive: {} bytes", std::hardware_constructive_interference_size);
std::println("Destructive:  {} bytes", std::hardware_destructive_interference_size);
// Constructive: 64 bytes
// Destructive:  64 bytes
```
</details>

What immediately came to mind was to exploit a common performance mistake in multithreaded
programming called false sharing. We should be able to measure the cache line size of at least the
L1 data cache this way. Technically it's possible that the different caches have different cache line
sizes, though it would be unusual.

## False Sharing

False sharing happens when two cores modify different data that happens to sit inside the same cache line.

On x86, each core typically has its own L1 data cache.
This means that each time a core modifies data, if that data also lives in the caches of
any of the other cores, the cache lines in the other cores have to be invalidated.
If the two cores are modifying the same data, for example with atomics, this is a necessary cost.
But if they are modifying different data in the same cache line,
the performance drop can be avoided by moving the two different pieces of data inside different cache lines.

## Sweeping Alignment

The idea is simple: Put two pieces of data next to each other and modify them in threads
pinned to different cores. Then increase the padding between the pieces of data and repeat.
We should see a sharp drop in runtime once the padding is large enough to place
the two pieces of data in different cache lines.

```
template <size_t N>
struct alignas(256) shared_data {
    uint8_t data1{0};
    alignas(N) uint8_t data2{0};
};
```

We can modify each byte by simply incrementing it.
```
for (size_t i{0}; i < 1'000'000'000; ++i) {
    ++data; // Each time this is executed,
            // if data1 and data2 are on the same cache line,
            // the cache line is invalidated on the other core
            // forcing a refetch and a "ping-pong" effect
}
```

Now we just have to do an exponential sweep over `N` and modify `data1` from one thread while modifying `data2` from another thread.

```
Second byte aligned to   1, runtime (µs):    2607175
Second byte aligned to   2, runtime (µs):    2604494
Second byte aligned to   4, runtime (µs):    2608744
Second byte aligned to   8, runtime (µs):    2607885
Second byte aligned to  16, runtime (µs):    2616199
Second byte aligned to  32, runtime (µs):    2611368
Second byte aligned to  64, runtime (µs):     794811
Second byte aligned to 128, runtime (µs):     785105
```

{% include figure.html src="measure-cache-line/alignment-plot" height="759" alt="" caption="" %}

As expected, we see a large drop in runtime when data2 is aligned to 64 bytes (padding of 63).

## Closing Thoughts

We had to make a lot of assumptions throughout the process (multicore, private L1d, write-invalidate as cache coherence protocol etc.).
It would be an interesting thought experiment to try to find a way
to do this for a mystery CPU without any knowledge about its architecture.

A quick Google search reveals that [others have tried this without using multiple cores](https://lemire.me/blog/2023/12/12/measuring-the-size-of-the-cache-line-empirically/),
though the results aren't as clean.

## Source Code
If you'd like to try it out on your own computer, you can access [the source here](https://github.com/iburaky2/measure-cache-line).
Keep in mind that you may have to modify some of the hardcoded constants.