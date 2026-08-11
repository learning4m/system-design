# Project Reactor Threading, `publishOn`, `subscribeOn`, and Backpressure

A practical mental model for understanding threading in **plain Project Reactor** and **Spring WebFlux/Reactor Netty**.

> **Important correction:** `publishOn(Schedulers.parallel())` does **not** make individual `onNext` calls run concurrently across multiple `parallel-*` threads for a single subscription. A `publishOn` worker is selected for that subscription and signals downstream are serialized on that worker. Likewise, Reactor follows the Reactive Streams rule that signals to a subscriber are serialized. `flatMap` can run inner work concurrently, but it merges/serializes signals before passing them downstream.

---

## Table of Contents

1. [Start with the right mental model](#1-start-with-the-right-mental-model)
2. [Plain Reactor vs WebFlux event loops](#2-plain-reactor-vs-webflux-event-loops)
3. [`subscribeOn`](#3-subscribeon)
4. [`publishOn`](#4-publishon)
5. [`subscribeOn` vs `publishOn`](#5-subscribeon-vs-publishon)
6. [Blocking work and `boundedElastic`](#6-blocking-work-and-boundedelastic)
7. [How completion is communicated](#7-how-completion-is-communicated)
8. [`publishOn` queues, prefetch, and backpressure](#8-publishon-queues-prefetch-and-backpressure)
9. [`flatMap` concurrency](#9-flatmap-concurrency)
10. [`publishOn(parallel)` vs concurrent `flatMap`](#10-publishonparallel-vs-concurrent-flatmap)
11. [Is downstream `onNext` concurrent?](#11-is-downstream-onnext-concurrent)
12. [`flatMapSequential` and `concatMap`](#12-flatmapsequential-and-concatmap)
13. [Why console examples use `blockLast()`](#13-why-console-examples-use-blocklast)
14. [Spring WebFlux + Netty](#14-spring-webflux--netty)
15. [Common mistakes](#15-common-mistakes)
16. [Interview-ready summary](#16-interview-ready-summary)

---

# 1. Start with the right mental model

A Reactor pipeline is primarily a **chain of Publishers, operators, and Subscribers**.

```text
Publisher
   |
   v
 operator
   |
   v
 operator
   |
   v
Subscriber
```

Operators do **not automatically create threads**.

For example:

```java
Flux.range(1, 3)
    .map(i -> i * 10)
    .filter(i -> i > 10)
    .subscribe(System.out::println);
```

With no asynchronous boundary, subscription and signal processing normally happen synchronously on the calling thread:

```text
main
 |
 +--> subscribe()
      |
      +--> range
           |
           +--> map
                |
                +--> filter
                     |
                     +--> onNext
```

So plain Reactor should **not** be thought of as:

> “Every Reactor application has an event-loop thread.”

That is not true.

---

# 2. Plain Reactor vs WebFlux event loops

## Plain Reactor Core

In standalone Reactor Core, there is no mandatory Netty-style I/O event loop.

Schedulers provide execution resources when requested:

```java
Schedulers.parallel()
Schedulers.single()
Schedulers.boundedElastic()
```

For example:

```java
Flux.range(1, 5)
    .publishOn(Schedulers.parallel())
    .map(this::calculate)
```

Reactor creates a scheduling boundary and downstream work is executed by a worker obtained from the `parallel` scheduler.

```text
main thread

Flux.range
    |
    v
 publishOn
    |
    | queue / scheduling boundary
    v
parallel-* worker
    |
    v
 map
    |
    v
 subscriber
```

The `parallel-*` thread is a scheduler worker. It is **not a Java NIO Selector event loop**.

## Spring WebFlux with Reactor Netty

With WebFlux on Reactor Netty, there are actual network event-loop threads:

```text
reactor-http-nio-1
reactor-http-nio-2
...
```

Conceptually:

```text
Network socket
     |
     v
OS readiness mechanism
(epoll / kqueue / Java NIO)
     |
     v
Netty EventLoop
     |
     v
WebFlux / Reactor pipeline
```

Those event-loop threads must not be blocked.

---

# 3. `subscribeOn`

`subscribeOn` controls **where subscription to the source is initiated**.

Example:

```java
Mono.fromCallable(() -> blockingDatabaseCall())
    .subscribeOn(Schedulers.boundedElastic())
    .map(this::transform)
    .subscribe();
```

Conceptually:

```text
main
 |
 | subscribe()
 v
subscribeOn
 |
 | schedules subscription
 v
boundedElastic-1
 |
 +--> fromCallable()
 |
 +--> map()
 |
 +--> subscriber
```

Unless another asynchronous boundary appears, signals continue on the thread producing them.

## Why it is useful

A common use case is wrapping a blocking API:

```java
Mono.fromCallable(() -> jdbcRepository.findById(id))
    .subscribeOn(Schedulers.boundedElastic());
```

This keeps the blocking call away from a Netty event-loop or another thread that should remain non-blocking.

### Important nuance

It is often said:

> "`subscribeOn` moves everything upstream."

That is a useful shortcut, but the more precise model is:

> `subscribeOn` schedules the **subscription process** on the selected scheduler, which usually causes a synchronous source and its signal chain to execute there until another asynchronous boundary changes execution context.

---

# 4. `publishOn`

`publishOn` changes the **execution context for downstream signals**.

```java
Flux.range(1, 3)
    .map(i -> {
        log("before", i);
        return i;
    })
    .publishOn(Schedulers.parallel())
    .map(i -> {
        log("after", i);
        return i;
    })
    .blockLast();
```

Typical thread behavior:

```text
main
 |
 +--> range
 |
 +--> map(before)
 |
 +--> publishOn
        |
        | queue
        v
     parallel-1
        |
        +--> map(after)
        |
        +--> downstream
```

A typical output resembles:

```text
[main]       before 1
[main]       before 2
[main]       before 3

[parallel-1] after 1
[parallel-1] after 2
[parallel-1] after 3
```

## Critical point

This:

```java
.publishOn(Schedulers.parallel())
```

does **not** mean:

```text
item 1 -> parallel-1
item 2 -> parallel-2
item 3 -> parallel-3
```

for one normal subscription.

Instead, `publishOn` obtains a scheduler worker and drains its queue serially:

```text
                  +----------------+
upstream -------->| publishOn queue |
                  +-------+--------+
                          |
                          v
                     parallel-1
                          |
                     onNext(1)
                     onNext(2)
                     onNext(3)
```

Therefore, use `publishOn(parallel())` primarily for **thread/context switching**, not for per-element parallelization.

---

# 5. `subscribeOn` vs `publishOn`

Consider:

```java
Mono.fromCallable(this::load)
    .subscribeOn(Schedulers.boundedElastic())
    .publishOn(Schedulers.parallel())
    .map(this::calculate)
    .block();
```

Thread flow:

```text
main
 |
 | subscribe
 v
subscribeOn(boundedElastic)
 |
 v
boundedElastic-1
 |
 +--> load()
 |
 +--> emits result
 |
 v
publishOn(parallel)
 |
 | enqueue signal
 v
parallel-1
 |
 +--> calculate()
 |
 +--> downstream
```

| Operator | Main purpose |
|---|---|
| `subscribeOn` | Choose scheduler used to subscribe to/source work |
| `publishOn` | Shift downstream signal processing |
| `boundedElastic` | Isolate blocking work |
| `parallel` | Fixed-size scheduler intended for CPU-bound/non-blocking work |

---

# 6. Blocking work and `boundedElastic`

Suppose a blocking call takes two seconds:

```java
Mono.fromCallable(() -> {
    Thread.sleep(2000);
    return repository.load();
})
.subscribeOn(Schedulers.boundedElastic());
```

The important distinction is:

```text
Logical pipeline waiting != thread blocking
```

The pipeline cannot produce the result until the operation completes.

But a Netty event-loop thread does not have to sit inside:

```java
Thread.sleep(...)
```

The blocking occurs on:

```text
boundedElastic-1
```

instead.

Conceptually:

```text
Netty/event-loop thread              boundedElastic-1
-----------------------              ----------------
receive request
      |
subscribe to blocking Mono --------> execute blocking call
return to event loop                 |
                                     | blocked here
handle other network events          |
handle other connections             |
                                     v
                               blocking call completes
                                     |
                                     v
                              emit Reactor signal
```

So `boundedElastic` does not make blocking code non-blocking.

It **isolates the blocking** onto threads intended to tolerate it.

Also, `boundedElastic` is bounded: excessive blocking work can still exhaust its capacity or build a large pending-task queue. Concurrency should still be controlled.

---

# 7. How completion is communicated

There are two cases worth separating.

## Network I/O

Reactor Netty uses Netty's event loop and operating-system/JDK networking mechanisms to detect socket readiness.

```text
Socket
  |
  v
Selector / epoll / kqueue
  |
  v
Netty EventLoop
  |
  v
channelRead(...)
  |
  v
Reactor signal
```

A Java NIO `SelectableChannel` is therefore relevant to **network readiness**.

## Blocking work on `boundedElastic`

A blocking Java method is different:

```java
Mono.fromCallable(this::blockingCall)
    .subscribeOn(Schedulers.boundedElastic());
```

There is no `SelectableChannel` representing completion of the arbitrary Java method.

The worker executes the callable and, when it returns, emits the normal Reactive Streams signal:

```text
boundedElastic thread
       |
       | blockingCall()
       |
       v
 result available
       |
       v
   onNext(result)
       |
       v
 downstream
```

If a later `publishOn` exists:

```text
boundedElastic-1
      |
   result
      |
      v
publishOn queue
      |
      v
parallel-1
      |
 downstream
```

Without `publishOn`, downstream can simply continue on the bounded-elastic thread.

There is **not automatically a special Reactor event loop to which every result must return**.

---

# 8. `publishOn` queues, prefetch, and backpressure

A thread boundary requires asynchronous handoff.

```java
Flux.range(1, 1_000)
    .publishOn(Schedulers.parallel())
    .map(this::slowProcessing)
```

Conceptually:

```text
UPSTREAM                         DOWNSTREAM

main                             parallel-1
 |                                   ^
 | onNext                            |
 v                                   |
+---------------------------------------+
|          publishOn queue              |
+---------------------------------------+
```

`publishOn` requests a bounded amount of data ahead of downstream demand. Its default prefetch is commonly `Queues.SMALL_BUFFER_SIZE` (typically 256 in Reactor versions commonly used today), but code should not depend on that implementation default if the exact value matters.

You can configure it:

```java
.publishOn(Schedulers.parallel(), 32)
```

Conceptually:

```text
prefetch = 32

upstream
   |
   | requested items
   v
+----------------+
| publishOn queue|
| up to ~32      |
+----------------+
   |
   v
slow downstream
```

## Replenishment

Reactor does not normally wait for the queue to become completely empty before requesting more.

It replenishes demand as queued elements are consumed.

Simplified:

```text
Initial:
request(prefetch)

Queue fills
[1][2][3]...[N]

Downstream consumes elements
      |
      v
Reactor requests another batch
before starvation
```

This keeps throughput high while maintaining bounded demand.

## Important correction: backpressure is not thread blocking

When downstream cannot accept more data, a compliant upstream is not told:

> “Block your thread until queue space appears.”

Instead, demand controls production:

```text
Subscriber
    |
    | request(n)
    v
Publisher
```

When outstanding demand reaches zero, a backpressure-aware publisher should stop producing further `onNext` signals until more demand arrives.

That is **protocol-level flow control**, not necessarily `Thread.wait()` or `Thread.sleep()`.

---

# 9. `flatMap` concurrency

`flatMap` serves a very different purpose from `publishOn`.

```java
Flux.range(1, 5)
    .flatMap(id ->
        Mono.fromCallable(() -> blockingCall(id))
            .subscribeOn(Schedulers.boundedElastic()),
        3
    )
    .blockLast();
```

Here:

```text
maxConcurrency = 3
```

means up to three inner publishers can be active concurrently.

```text
Outer Flux
   |
   +---- item 1 ---> Inner Mono 1 ---> boundedElastic-1
   |
   +---- item 2 ---> Inner Mono 2 ---> boundedElastic-2
   |
   +---- item 3 ---> Inner Mono 3 ---> boundedElastic-3
   |
   +---- item 4 ---> waits for available concurrency
   |
   +---- item 5 ---> waits for available concurrency
```

When one completes:

```text
Mono 2 completes
      |
      v
flatMap merges result
      |
      +---- allows another inner subscription
```

Results may arrive out of source order:

```text
source order:
1 2 3 4 5

completion order:
3 1 2 5 4
```

`flatMap` is therefore useful for **concurrent asynchronous work**.

---

# 10. `publishOn(parallel)` vs concurrent `flatMap`

This distinction is crucial.

## `publishOn(Schedulers.parallel())`

```java
Flux.range(1, 4)
    .publishOn(Schedulers.parallel())
    .map(this::cpuWork)
```

Conceptually:

```text
1
2 ----> publishOn queue ----> parallel-1 ----> map()
3                              serial drain
4
```

For one subscription, this does **not** make four `map()` invocations execute simultaneously.

## `flatMap` with scheduled inner publishers

```java
Flux.range(1, 4)
    .flatMap(i ->
        Mono.fromCallable(() -> blockingCall(i))
            .subscribeOn(Schedulers.boundedElastic()),
        4
    );
```

Conceptually:

```text
                  +--> boundedElastic-1 --> task(1)
                  |
source --> flatMap+--> boundedElastic-2 --> task(2)
                  |
                  +--> boundedElastic-3 --> task(3)
                  |
                  +--> boundedElastic-4 --> task(4)
```

The **inner work really can execute concurrently**.

Then `flatMap` merges the results:

```text
task 1 ----\
task 2 -----\
task 3 ------> flatMap merge ---> downstream onNext
task 4 -----/
```

### Comparison

| Question | `publishOn(parallel())` | `flatMap` + async inner publishers |
|---|---|---|
| Main purpose | Thread boundary | Concurrent composition |
| Per-element concurrent work | No, not by itself | Yes |
| Creates inner publishers | No | Yes |
| Ordering | Preserves signal order through boundary | Completion order may differ |
| Concurrency control | Not a per-item concurrency setting | `maxConcurrency` |
| Typical use | Shift downstream CPU/non-blocking work | Concurrent I/O/task execution |

If you want true Reactor CPU parallelization, another pattern is:

```java
flux.parallel()
    .runOn(Schedulers.parallel())
    .map(this::cpuWork)
    .sequential();
```

That creates multiple rails; a plain `publishOn(parallel())` does not.

---

# 11. Is downstream `onNext` concurrent?

This is a particularly important Reactive Streams rule.

For a well-behaved Reactor sequence, signals delivered to a downstream Subscriber are **serialized**.

Even if `flatMap` has several inner publishers completing concurrently:

```text
boundedElastic-1 ----\
boundedElastic-2 -----\
boundedElastic-3 ------> flatMap merge ---> downstream
boundedElastic-4 -----/
```

`flatMap` coordinates them so the downstream does not receive overlapping `onNext` invocations from the same subscription.

Think of it as:

```text
Concurrent inner work
       |
       v
+------------------+
| flatMap arbiter  |
| / merge logic    |
+------------------+
       |
       v
onNext(A)
onNext(B)
onNext(C)

serialized downstream
```

Therefore this earlier mental model is wrong:

```text
parallel-1 ---> subscriber.onNext(1)
parallel-2 ---> subscriber.onNext(2)   simultaneously
parallel-3 ---> subscriber.onNext(3)
```

for a normal single Reactor subscription.

## Does that mean shared mutable state is always safe?

No.

For example:

```java
List<Integer> shared = new ArrayList<>();
```

may still be accessed by:

- multiple independent subscriptions,
- other application threads,
- manually created publishers that violate serialization,
- callbacks outside the Reactor chain.

But within one correctly implemented Reactor subscription, downstream signal serialization is a fundamental guarantee.

Avoid relying on mutable shared state when a functional operator can express the operation:

```java
.reduce(...)
.collectList()
.scan(...)
```

---

# 12. `flatMapSequential` and `concatMap`

These operators differ in **inner execution and emission ordering**.

## `flatMap`

```text
Execute concurrently: YES
Preserve source order: NO

1 ----slow--------> result 1
2 --fast--> result 2
3 ----medium----> result 3

output:
2, 3, 1
```

## `flatMapSequential`

Inner publishers can be subscribed to eagerly/concurrently, but results are emitted in source order.

```text
1 ----slow--------> result 1
2 --fast--> result 2
3 ----medium----> result 3

results may finish:
2, 3, 1

downstream sees:
1, 2, 3
```

This may require buffering completed results while waiting for an earlier publisher.

## `concatMap`

Only one inner publisher is active at a time:

```text
item 1 --> inner 1 --> complete
                         |
item 2 ------------------+
           |
           v
        inner 2 --> complete
                     |
item 3 ---------------+
```

Therefore:

```text
Concurrency: 1
Ordering: preserved
```

---

# 13. Why console examples use `blockLast()`

Consider:

```java
Flux.range(1, 5)
    .publishOn(Schedulers.parallel())
    .subscribe(System.out::println);
```

Once asynchronous scheduling occurs, `subscribe()` can return before processing completes.

In a small standalone program, you usually want:

```java
Flux.range(1, 5)
    .publishOn(Schedulers.parallel())
    .doOnNext(System.out::println)
    .blockLast();
```

`blockLast()` blocks the **calling/main thread** until completion.

```text
main
 |
 | subscribe
 v
async Reactor processing
 |
 | ...
 |
 v
onComplete
 |
 v
blockLast returns
 |
 v
main exits
```

A demo sometimes uses:

```java
Thread.sleep(2000);
```

but that is only an arbitrary JVM-lifetime trick.

Prefer `block()` / `blockLast()` for standalone demos where intentionally waiting at the outermost imperative boundary is appropriate.

Do **not** put `block()` inside a WebFlux request processing chain.

---

# 14. Spring WebFlux + Netty

Now combine Reactor with Reactor Netty.

Suppose:

```java
@GetMapping("/customer/{id}")
public Mono<Customer> customer(@PathVariable long id) {
    return Mono.fromCallable(() -> jdbcCall(id))
        .subscribeOn(Schedulers.boundedElastic());
}
```

Conceptual flow:

```text
                 NETWORK
                    |
                    v
          reactor-http-nio-1
          Netty event loop
                    |
                    | invokes controller
                    v
             subscribe to Mono
                    |
                    | scheduling
                    v
           boundedElastic-1
                    |
                    | JDBC blocks here
                    |
                    v
               Customer
                    |
                    v
            Reactor propagation
                    |
                    v
          Netty response handling
                    |
                    v
                 socket
```

While JDBC is blocked:

```text
boundedElastic-1
     |
     X  blocked waiting for JDBC
```

the Netty event-loop thread is available to process other socket events:

```text
reactor-http-nio-1

request A -> dispatch
request B -> read
request C -> write
request D -> parse
...
```

This is why blocking work must not run directly on a Netty event loop.

---

# 15. Common mistakes

## Mistake 1: "`Schedulers.parallel()` is the Reactor event loop."

Better:

> `Schedulers.parallel()` is Reactor's shared fixed-size scheduler for parallel CPU-oriented work. In WebFlux with Reactor Netty, the `reactor-http-nio-*` threads are the network event loops.

---

## Mistake 2: "`publishOn(parallel())` parallelizes every element."

Incorrect.

```java
.publishOn(Schedulers.parallel())
.map(this::work)
```

creates a scheduling boundary, but the downstream of a single subscription is normally drained serially by one worker.

For per-element concurrency use an appropriate pattern such as:

```java
.flatMap(...)
```

or CPU rails:

```java
.parallel()
.runOn(Schedulers.parallel())
.sequential()
```

---

## Mistake 3: "`flatMap` allows simultaneous calls to my final subscriber."

The inner work can be concurrent.

The merged downstream Reactive Streams signals are serialized.

```text
Concurrent work                 Serialized signals

inner-1 -----\
inner-2 ------> flatMap ------> onNext
inner-3 -----/                  onNext
                               onNext
```

---

## Mistake 4: "`boundedElastic` makes blocking non-blocking."

No.

```text
Thread.sleep(5 seconds)
```

still blocks a thread for five seconds.

`boundedElastic` merely places that blocking on a more appropriate bounded pool.

---

## Mistake 5: "Backpressure blocks the producer thread."

Not necessarily.

Reactive backpressure is based on demand:

```text
request(n)
```

A compliant publisher emits only according to available demand. That is different from physically blocking a Java thread.

---

## Mistake 6: "`publishOn` always has exactly a 256-element queue."

Treat 256 as a common Reactor default, not a universal semantic guarantee.

Use an explicit prefetch when it matters:

```java
.publishOn(Schedulers.parallel(), 32)
```

---

# 16. Interview-ready summary

```text
                    PROJECT REACTOR
                          |
          +---------------+---------------+
          |                               |
     subscribeOn                      publishOn
          |                               |
controls where                    shifts downstream
subscription/source               signal processing
is scheduled                      to another scheduler
          |                               |
          v                               v
 boundedElastic                  parallel / single
 for blocking work               scheduling boundary
```

For concurrent work:

```text
                    flatMap
                       |
            +----------+----------+
            |          |          |
          inner1     inner2     inner3
            |          |          |
            +----------+----------+
                       |
                       v
                 merge/serialize
                       |
                       v
                   downstream
```

For WebFlux:

```text
Client
  |
  v
Network
  |
  v
Netty EventLoop
reactor-http-nio-*
  |
  +---- non-blocking work ----> stay on event loop
  |
  +---- blocking API ---------> boundedElastic
                                   |
                                   v
                               completion
                                   |
                                   v
                             Reactor signals
                                   |
                                   v
                              HTTP response
```

## Final rules

1. **Reactor does not automatically create a thread per pipeline.**
2. **Plain Reactor has no mandatory Netty-style event loop.**
3. **`subscribeOn` influences subscription/source execution.**
4. **`publishOn` shifts downstream execution across a scheduler boundary.**
5. **`publishOn(parallel())` does not by itself parallelize elements.**
6. **`flatMap` can execute inner publishers concurrently.**
7. **Downstream Reactive Streams signals are serialized for a normal subscription.**
8. **`boundedElastic` isolates blocking; it does not eliminate blocking.**
9. **Network readiness in Reactor Netty is handled by Netty/OS I/O mechanisms; arbitrary bounded-elastic task completion is normal Reactor signal propagation.**
10. **Prefetch and backpressure bound in-flight demand, but poorly designed buffering or excessive concurrency can still cause memory pressure.**
11. **Use `block()`/`blockLast()` only at deliberate imperative boundaries, such as standalone demos—not inside reactive WebFlux request chains.**
