---
layout: post
title: "Scalable Timer Management with Hierarchical Time Wheels"
date: 2026-01-26
tags:
  - Hierarchical Time Wheels
  - Scheduler
  - Scalability
  - system-design
math: true
---

Timer management(AKA Scheduler) is a fundamental service in large-scale systems. Timeouts, delayed retries, background maintenance, and scheduling all rely on the ability to execute actions at a specified future time. While the problem appears trivial at small scale, it becomes non-trivial when a system must manage hundreds of thousands or millions of concurrent timers with low and predictable latency.

The core requirement of a timer subsystem is to efficiently support timer insertion and expiration while minimizing CPU overhead and avoiding unpredictable performance degradation as the number of active timers grows.

## Limitations of Binary Heap–Based Scheduling
A common implementation strategy uses a binary min-heap ordered by timer expiration timestamps. This approach ensures that the earliest timer can be identified in constant time, while insertion and removal require logarithmic time.

```
insert(timer):
    heap.push(timer)

run():
    while true:
        t = heap.peek()
        if t.expire_at <= now():
            heap.pop()
            execute(t)
        else:
            sleep(t.expire_at - now())
```

Although asymptotically efficient, this approach exhibits poor scalability in practice. Each timer insertion incurs O(log N) comparisons and pointer traversals, which translate into cache misses and branch mispredictions on modern hardware. More importantly, timers with far-future expiration times participate fully in heap ordering, imposing maintenance costs even when they are irrelevant to near-term execution.

As timer volume increases, heap maintenance dominates CPU usage, making this design unsuitable for timer-heavy systems.

## Time Bucketing as an Alternative
The key observation is that exact ordering of all timers is unnecessary. At any moment, the system only needs to identify timers that are about to expire. This leads to the idea of discretizing time into fixed intervals and grouping timers into buckets based on their expiration window rather than their exact timestamp.

A single-level time wheel implements this idea using a circular array of buckets, where each bucket represents a fixed time slice.

```
add_timer(delay):
    slot = (current_tick + delay) mod WHEEL_SIZE
    wheel[slot].append(timer)

tick():
    for timer in wheel[current_tick]:
        execute(timer)
    clear(wheel[current_tick])
    current_tick = next(current_tick)
```

This design enables constant-time insertion and expiration checks. However, its applicability is limited by the fixed size and resolution of the wheel, which restrict the maximum representable delay.

### Hierarchical Time Wheel Design

Hierarchical time wheels extend time bucketing across multiple levels of granularity. Each level represents a larger time unit, forming a hierarchy analogous to seconds, minutes, and hours on a clock. Timers are initially placed in the coarsest level capable of representing their delay and are progressively moved to finer-grained levels as time advances.

```
add_timer(delay):
    level = select_level(delay)
    slot  = compute_slot(level, delay)
    wheel[level][slot].append(timer)
```

Time advancement proceeds from the lowest level upward. When a wheel completes a full rotation, timers in the next higher level are cascaded downward and redistributed.

```
advance(level):
    slot = current_slot[level]
    for timer in wheel[level][slot]:
        if level == 0:
            execute(timer)
        else:
            reinsert(timer, level - 1)
    clear(wheel[level][slot])
    current_slot[level] = next(slot)
```

This hierarchical structure preserves constant-time operations while supporting arbitrarily large delays.

> 💡 **Note:** Hierarchical time wheels avoid global ordering and isolate far-future timers from near-term scheduling decisions. Their array-based structure improves cache locality and bounds the amount of work performed per tick, resulting in stable latency characteristics even under heavy timer loads.

### Kafka Delay Wheel vs Linux Kernel Timer Wheels

There are two widely cited and production-proven implementations of hierarchical time wheels, each shaped by fundamentally different system constraints: the delay wheel used in Apache Kafka and the timer wheel subsystem of the Linux kernel. Although both share the same conceptual foundation, their designs diverge significantly at the core to serve different operational goals.

**Kafka** employs a hierarchical time wheel as an optimization for managing delayed operations such as request timeouts, retries, and quota enforcement. At its core, Kafka’s wheel is designed for **amortized efficiency under high concurrency** rather than strict timing precision.

Timers are bucketed based on coarse-grained expiration windows, allowing constant-time insertion and bounded per-tick processing. However, Kafka does not rely solely on the wheel. Timers that exceed the wheel’s resolution or fall outside its representable range are redirected to a fallback priority queue. This hybrid design ensures correctness for edge cases while keeping the common path free from heap overhead. In effect, Kafka treats the time wheel as a fast-path mechanism, using the heap only when precision cannot be safely approximated.

In contrast, the **Linux kernel** implements hierarchical timer wheels as a complete and self-sufficient scheduling mechanism. Kernel timers operate under strictly bounded delay ranges and predefined resolutions, enabling the design to avoid heap-based structures altogether. Each level of the wheel corresponds to a fixed time granularity, and timers are deterministically cascaded toward finer levels as time advances. This design prioritizes predictability and bounded execution cost, properties that are essential in kernel space where unbounded latency or dynamic memory behavior is unacceptable. By constraining the problem domain, the kernel achieves both efficiency and determinism without auxiliary data structures.

A generic hierarchical time wheel, as described in this work, occupies a conceptual middle ground between these two systems. It inherits the structural efficiency of kernel-style wheels while retaining the flexibility required in user-space environments. Depending on workload characteristics and precision requirements, such a design may function as a standalone scheduler or be augmented with auxiliary structures, following Kafka’s hybrid approach. This adaptability makes hierarchical time wheels a broadly applicable abstraction rather than a single fixed implementation.