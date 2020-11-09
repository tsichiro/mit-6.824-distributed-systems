# Lecture 1: Introduction

<!-- TOC -->

- [Lecture 1: Introduction](#lecture-1-introduction)
  - [What is a distributed system](#what-is-a-distributed-system)
  - [Why distributed](#why-distributed)
  - [But/Challenges](#butchallenges)
  - [Why take this course](#why-take-this-course)
  - [MAIN TOPICS](#main-topics)
    - [Topic: implementation](#topic-implementation)
    - [Topic: performance](#topic-performance)
    - [Topic: fault tolerance](#topic-fault-tolerance)
    - [Topic: consistency](#topic-consistency)
  - [CASE STUDY](#case-study)

<!-- /TOC -->

## What is a distributed system

- multiple cooperating computers
- storage for big web sites, MapReduce, peer-to-peer sharing, &c
- lots of critical infrastructure is distributed

## Why distributed

- to organize **physically** separate entities
- to achieve **security** via **isolation**
- to **tolerate faults** via replication
- to scale up throughput via **parallel** CPUs/mem/disk/net

## But/Challenges

- complex: many **concurrent** parts
- must cope with **partial failure**
- tricky to realize **performance** potential

## Why take this course

- interesting -- hard problems, powerful solutions
- used by real systems -- driven by the rise of big Web sites
- active research area -- lots of progress + big unsolved problems
- hands-on -- you'll build serious systems in the labs

## MAIN TOPICS

- This is a course about **infrastructure**, to be used by applications.
- About **abstractions** that hide distribution from applications.
- Three big kinds of abstraction:
  - **Storage**.
  - **Communication**.
  - **Computation**.
- [diagram: users, application servers, storage servers]

### Topic: implementation

- RPC, threads, concurrency control.

### Topic: performance

- The dream: **scalable** throughput.
  - Nx servers -> Nx total throughput via parallel CPU, disk, net.
  - So handling more load only requires buying more computers.
- Scaling gets harder as N grows:
  - Load im-balance, stragglers.
  - Non-parallelizable code: initialization, interaction.
  - Bottlenecks from shared resources, e.g. network.
- Note that some performance problems aren't easily attacked by scaling
  - e.g. decreasing response time for a single user request
  - might require programmer effort rather than just more computers

### Topic: fault tolerance

- 1000s of servers, complex net -> always something broken
- We'd like to hide these failures from the application.
- We often want:
  - **Availability** -- app can make progress despite failures
  - **Durability/Recoverability** -- app will come back to life when failures are repaired
- Big idea: replicated servers.
  - If one server crashes, client can proceed using the other(s).

### Topic: consistency

- General-purpose infrastructure needs well-defined behavior.
  - E.g. "Get(k) yields the value from the most recent Put(k,v)."
- Achieving good behavior is hard!
  - "Replica" servers are hard to keep identical.
  - Clients may crash midway through multi-step update.
  - Servers crash at awkward moments, e.g. after executing but before replying.
  - Network may make live servers look dead; risk of "split brain".
- Consistency and performance are enemies.
  - Consistency requires communication, e.g. to get latest Put().
  - "Strong consistency" often leads to slow systems.
  - High performance often imposes "weak consistency" on applications.
- People have pursued many design points in this spectrum.

## CASE STUDY

- [MapReduce](mapreduce-simplified-data-processing-on-large-clusters.md)
