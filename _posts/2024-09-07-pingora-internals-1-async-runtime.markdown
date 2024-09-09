---
title: "Pingora async runtime and threading model"
layout: post
date: 2024-09-07 08:00
image: https://cf-assets.www.cloudflare.com/slt3lc6tev37/f24aeLVQUjlbXRxDzdWm9/4f560a3a3e7da5d10093728548029ed7/pingora-open-source.png
headerImage: false
tag:
- pingora
- nginx
- rust
- data structures
- http proxy
- performance
- multithreading
- tokio
- thread-per-core
- async
category: blog
author: Ricardo Pallas
projects: true
description: Dive deep into Pingora async runtime and threading model.
---

# Pingora async runtime and threading model

## Introduction

Cloudflare [open-sourced](https://blog.cloudflare.com/pingora-open-source/) their **Rust** framework for building programmable **network services** called **[Pingora](https://github.com/cloudflare/pingora)**. They developed it to replace NGINX to overcome its limitations, as explained in [their blog](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/):

> Today we are excited to talk about Pingora, a new HTTP proxy we’ve built in-house using Rust that serves over 1 trillion requests a day, boosts our performance, and enables many new features for Cloudflare customers, all while requiring only a third of the CPU and memory resources of our previous proxy infrastructure.
>
>...
>
> Over the years, our usage of NGINX has run up against limitations. For some limitations, we optimized or worked around them. But others were much harder to overcome.

I am starting a **series of posts where I will dive deep into the internals of Pingora**. Each post will cover a different aspect of Pingora's architecture. In this post, we will discuss its **runtime** for running async Rust and its **threading model**.


## Pingora's async runtime

Pingora's async runtime is based on [Tokio](https://tokio.rs/). Tokio is the *de facto* runtime for write asynchronous, non-blocking code in Rust:

> Tokio is scalable, built on top of the async/await language feature, which itself is scalable. When dealing with networking, there's a limit to how fast you can handle a connection due to latency, so the only way to scale is to handle many connections at once. With the async/await language feature, increasing the number of concurrent operations becomes incredibly cheap, allowing you to scale to a large number of concurrent tasks.

Pingora offers **two multi-threaded runtimes** (with and without work stealing) that we will discuss later. But first, let's explain how Tokio's runtime works, as it forms the basis of Pingora.

### Tokio's runtime

The Tokio [runtime](https://docs.rs/tokio/1.40.0/tokio/runtime/index.html) is composed of three main components:
1. An **I/O event loop**, called the driver, which drives I/O resources and dispatches I/O events to tasks that depend on them.
1. A **scheduler** to execute tasks that use these I/O resources.
1. A **timer** for scheduling work to run after a set period of time.

The `tokio::main` macro provides a default-configured runtime:

```rust
use futures::future::join_all;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("Executing main");

    let mut handles = Vec::new();

    for i in 0..5 {
        let handle = tokio::spawn(async move {
            println!("Executing task {}", i);
        });
        handles.push(handle);
    }

    join_all(handles).await;
    Ok(())
}
```

In the code above, the `main` async function is scheduled on the Tokio runtime. The function also schedules five additional async tasks on the same runtime by calling `tokio::spawn`.

Tokio's runtime provides two main task scheduling strategies:
1. Multi-Thread Scheduler
1. Current-Thread Scheduler

#### Multi-Thread Scheduler

The multi-thread scheduler executes futures on a thread pool, using a work-stealing strategy. By default, it will start a worker thread for each CPU core available on the system.

At its most basic level, a runtime has a collection of tasks that need to be scheduled. It will repeatedly remove a task from that collection and schedule it (by calling `poll`). When the collection is empty, the thread will go to sleep until a task is added to the collection.

The multi-thread runtime maintains one global queue, and a local queue for each worker thread. The runtime will prefer to choose the next task to schedule from the local queue, and will only pick a task from the global queue if the local queue is empty.

If both the local queue and global queue are empty, the worker thread **will attempt to steal tasks from the local queue of another worker thread**. Stealing is done by moving half of the tasks in one local queue to another local queue.

```
 Multi-Thread Scheduler with 2 threads                        
┌────────────────────────────────────────────────────────────┐
│                                                            │
│   Tokio runtime                                            │
│                                                            │
│   ┌────────────────────────────────────────────────────┐   │
│   │ Global queue                                       │   │
│   │  (empty)                                           │   │
│   └──────────┬─────────────────────────────┬───────────┘   │
│              │                             │               │
│              │                             │               │
│              ▼                             ▼               │
│   ┌──────────────────────┐      ┌──────────────────────┐   │
│   │ Thread 1             │      │ Thread 2             │   │
│   │                      │      │                      │   │
│   │ ┌──────────────────┐ │      │ ┌──────────────────┐ │   │
│   │ │ Local queue      │ │      │ │ Local queue      │ │   │
│   │ │ Task 1           │ │      │ │ (empty)          │ │   │
│   │ │ Task 2           │ │      │ │                  │ │   │
│   │ │ Task 3           │ │      │ │                  │ │   │
│   │ │ Task 4           │ │      │ │                  │ │   │
│   │ └──────────────────┘ │      │ └──────────────────┘ │   │
│   │                      │      │                      │   │
│   │  Executing Task 1    │      │  Idle                │   │
│   │                      │      │                      │   │
│   └──────────────────────┘      └──────────────────────┘   │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

In the diagram above, the Tokio runtime has two threads. Thread 1 has four tasks in its local queue, while Thread 2 has no tasks. Since the global queue is also empty, Thread 2 will steal half of the tasks from Thread 1. This ensures that tasks are balanced across CPU cores, maximizing the use of available hardware resources and increasing throughput.

```
 Multi-Thread Scheduler with 2 threads                        
┌────────────────────────────────────────────────────────────┐
│                                                            │
│   Tokio runtime                                            │
│                                                            │
│   ┌────────────────────────────────────────────────────┐   │
│   │ Global queue                                       │   │
│   │  (empty)                                           │   │
│   └──────────┬─────────────────────────────┬───────────┘   │
│              │                             │               │
│              │                             │               │
│              ▼                             ▼               │
│   ┌──────────────────────┐      ┌──────────────────────┐   │
│   │ Thread 1             │      │ Thread 2             │   │
│   │                      │      │                      │   │
│   │ ┌──────────────────┐ │      │ ┌──────────────────┐ │   │
│   │ │ Local queue      │ │      │ │ Local queue      │ │   │
│   │ │ Task 1           │ │      │ │ Task 3           │ │   │
│   │ │ Task 2           │ │      │ │ Task 4           │ │   │
│   │ └──────────────────┘ │      │ └──────────────────┘ │   │
│   │                      │      │                      │   │
│   │  Executing Task 1    │      │  Executing Task 3    │   │
│   │                      │      │                      │   │
│   └──────────────────────┘      └──────────────────────┘   │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

#### Single-Thread Scheudler

The single-thread scheduler in Tokio operates with a single thread and uses a global and a local queues to manage tasks. This scheduler is designed to handle all asynchronous tasks on a single core. In this model, there is no need for task migration or work stealing, as all tasks are handled by the single thread:

```
 Single-Thread Tokio Scheduler                        
┌──────────────────────────────┐
│                              │
│   Tokio Runtime              │
│                              │
│   ┌──────────────────────┐   │
│   │ Thread               │   │
│   │                      │   │
│   │ ┌──────────────────┐ │   │
│   │ │ Global Queue     │ │   │
│   │ │ (empty)          │ │   │
│   │ └──────────────────┘ │   │
│   │                      │   │
│   │ ┌──────────────────┐ │   │
│   │ │ Local Queue      │ │   │
│   │ │ Task 1           │ │   │
│   │ │ Task 2           │ │   │
│   │ │ Task 3           │ │   │
│   │ │ Task 4           │ │   │
│   │ └──────────────────┘ │   │
│   │                      │   │
│   │  Executing Task 1    │   │
│   │                      │   │
│   └──────────────────────┘   │
│                              │
└──────────────────────────────┘
```

## Pingora' threading model

As already mentioned before, Pingora’s threading model is based on Tokio’s asynchronous runtime, and it provides two main flavors of runtimes: a stealing (multi-threaded with work stealing) and a no-steal (multi-threaded without work stealing) approach.

The stealing flavour is **just the standard Tokio multi-thread runtime without any customizations** that we already discussed.

On the other hand, the no-steal flavour is a **set of OS threads** each with **its own single-threaded Tokio runtime**.

## Pingora's multi-thread runtime without work stealing

The no-steal runtime model in Pingora is an alternative where each thread runs its own independent Tokio runtime with no task migration between threads. This means each thread operates as a single-threaded runtime, but the overall system can still utilize multiple cores by spawning multiple such runtimes:

```
 Multi-Threaded No Stealing Pingora Runtime with 2 Threads                            
┌────────────────────────────────────────────────────────────┐
│                                                            │
│   ┌────────────────────────────────────────────────────┐   │
│   │   Single-thread Tokio Runtime (Thread 1)           │   │
│   └────────────────────────────────────────────────────┘   │
│                                                            │
│   ┌────────────────────────────────────────────────────┐   │
│   │   Single-thread Tokio Runtime (Thread 2)           │   │
│   └────────────────────────────────────────────────────┘   │
│                                                            │
│   ┌──────────────────────┐     ┌──────────────────────┐    │
│   │ Thread 1             │     │ Thread 2             │    │
│   │                      │     │                      │    │
│   │ ┌──────────────────┐ │     │ ┌──────────────────┐ │    │
│   │ │ Global Queue     │ │     │ │ Global Queue     │ │    │
│   │ │ (Tasks 1, 2, 3)  │ │     │ │ (Tasks 4, 5, 6)  │ │    │
│   │ └──────────────────┘ │     │ └──────────────────┘ │    │
│   │                      │     │                      │    │
│   │ ┌──────────────────┐ │     │ ┌──────────────────┐ │    │
│   │ │ Local Queue      │ │     │ │ Local Queue      │ │    │
│   │ │ (empty)          │ │     │ │ (empty)          │ │    │
│   │ └──────────────────┘ │     │ └──────────────────┘ │    │
│   │                      │     │                      │    │
│   │  Executing Task 1    │     │  Executing Task 5    │    │
│   │                      │     │                      │    │
│   └──────────────────────┘     └──────────────────────┘    │
│                                                            │
│   ┌────────────────────────────────────────────────────┐   │
│   │               Multi-Core Utilization               │   │
│   └────────────────────────────────────────────────────┘   │
│                                                            │
└────────────────────────────────────────────────────────────┘

```

As we can see in the diagram above, each thread has its own Tokio runtime and task queues, and tasks scheduled on one thread remain on that thread throughout their lifetime. This means there is no work stealing, each thread is responsible for its own work, and no tasks are stolen or migrated across threads. Pingora ensures that any new task spawned in this runtime is randomly assigned to one of the threads.

This runtime flavour allows a thread-per-core thread model: it spawns multiple OS threads, with one thread typically mapped to each available CPU core.

#### Alternative thread-per-core runtimes

One improvement that Pingora could add to its non-stealing runtime is the use of `LocalSet` ([see docs](https://docs.rs/tokio/1.40.0/tokio/runtime/struct.Builder.html#method.new_current_thread)). Since it is composed of Tokio single-thread runtimes the futures shouldn't be required to be `Send` and `Sync` as they are always run in the same thread. `LocalSet`  provides a way to spawn and manage non-Send tasks within the context of a single-threaded runtime.

There are also other existing alternative runtimes that lend themselves to thread-per-core architectures: [glommio](https://crates.io/crates/glommio) from DataDog and [monoio](https://crates.io/crates/monoio) from ByteDance.

I recommend reading [this post](https://emschwartz.me/async-rust-can-be-a-pleasure-to-work-with-without-send-sync-static/) about async Rust without `Send + Sync + 'static`.

### Which Pingora runtime should I use?

Let's discuss the pros and cons of each alternative:

#### Work stealing

**Pros:**
- It dynamically balances the load across threads. If one thread is overwhelmed with tasks, other threads can steal work from the busy thread’s queue.
- By distributing tasks, it helps ensure that all CPU cores are utilized efficiently.

**Cons:**
- Some overhead is introduced due to the need for coordination and synchronization when threads steal tasks from each other. This can cause additional latency in certain contexts.


#### Thread-Per-Core

**Pros:**
- Since threads do not need to steal work from one another, there is less synchronization overhead.
- Tasks are bound to specific threads, which can be beneficial for tasks that are stateful or have significant initialization costs.


**Cons:**
- If some tasks require more time to execute than others, this could lead to some CPU cores being busy while others remain idle.

In my opinion, in the particular case of an HTTP proxy like Pingora, if the incoming traffic is predictable and each request requires a similar amount of time to be processed, a thread-per-core model might provide consistent performance with reduced thread contention. I opened a discussion [here](https://github.com/cloudflare/pingora/discussions/376). 


## Conclusion

In summary, Pingora offers two async runtimes: one with work stealing and one without.

- The work-stealing runtime is good for handling varying loads by balancing tasks across threads, but it introduces some overhead.
- The thread-per-core model, where each thread runs its own Tokio runtime, can provide more consistent performance with less contention, especially if workloads are predictable.

In my opinion, Pingora should consider improving its non-stealing policy by allowing the use of non-Send and non-Sync futures with a `LocalSet`. On top of that, they could even explore supporting other runtimes like [glommio](https://crates.io/crates/glommio) and [monoio](https://crates.io/crates/monoio).

