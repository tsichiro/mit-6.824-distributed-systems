# Lecture 4: Primary/Backup Replication

<!-- TOC -->

- [Lecture 4: Primary/Backup Replication](#lecture-4-primarybackup-replication)
  - [Fault tolerance](#fault-tolerance)
    - [What failures will we try to cope with?](#what-failures-will-we-try-to-cope-with)
    - [But not:](#but-not)
    - [Behaviors](#behaviors)
  - [Core idea: replication](#core-idea-replication)
    - [Example: fault-tolerant MapReduce master](#example-fault-tolerant-mapreduce-master)
    - [Big Questions:](#big-questions)
  - [Two main approaches:](#two-main-approaches)
    - [State transfer is simpler](#state-transfer-is-simpler)
    - [Replicated state machine can be more efficient](#replicated-state-machine-can-be-more-efficient)
    - [At what level to define a replicated state machine?](#at-what-level-to-define-a-replicated-state-machine)
  - [The design of a Practical System for Fault-Tolerant Virtual Machines](#the-design-of-a-practical-system-for-fault-tolerant-virtual-machines)

<!-- /TOC -->

- Primary/Backup Replication for Fault Tolerance
- Case study of VMware FT, an extreme version of the idea

## Fault tolerance

- we'd like a service that continues despite failures
- some ideal properties:
  - available: still useable despite [some class of] failures
  - strongly consistent: looks just like a single server to clients
  - transparent to clients
  - transparent to server software
  - efficient

### What failures will we try to cope with?

- Fail-stop failures
- Independent failures
- Network drops some/all packets
- Network partition

### But not:

- Incorrect execution
- Correlated failures
- Configuration errors
- Malice

### Behaviors

- Available (e.g. if one server halts)
- Wait (e.g. if network totally fails)
- Stop forever (e.g. if multiple servers crash)
- Malfunction (e.g. if h/w computes incorrectly, or software has a bug)

## Core idea: replication

- *Two* servers (or more)
- Each replica keeps state needed for the service
- If one replica fails, others can continue

### Example: fault-tolerant MapReduce master

- lab 1 workers are already fault-tolerant, but not master
  - master is a "single point of failure"
- can we have two masters, in case one fails?
- [diagram: M1, M2, workers]
- state:
  - worker list
  - which jobs done
  - which workers idle
  - TCP connection state
  - program memory and stack
  - CPU registers

### Big Questions:

- What state to replicate?
- Does primary have to wait for backup?
- When to cut over to backup?
- Are anomalies visible at cut-over?
- How to bring a replacement up to speed?

## Two main approaches:

- State transfer
  - "Primary" replica executes the service
  - Primary sends [new] state to backups
- Replicated state machine
  - All replicas execute all operations
  - If same start state,
    - same operations,
    - same order,
    - deterministic,
    - then same end state

### State transfer is simpler

- But state may be large, slow to transfer
- VM-FT uses replicated state machine

### Replicated state machine can be more efficient

- If operations are small compared to data
- But complex to get right
- Labs 2/3/4 use replicated state machines

### At what level to define a replicated state machine?

- K/V put and get?
  - "application-level" RSM
  - usually requires server and client modifications
  - can be efficient; primary only sends high-level operations to backup
- x86 instructions?
  - might allow us to replicate any existing server w/o modification!
  - but requires much more detailed primary/backup synchronization
  - and we have to deal with interrupts, DMA, weird x86 instructions

## [The design of a Practical System for Fault-Tolerant Virtual Machines](the-design-of-a-practical-system-for-fault-tolerant-virtual-machines.md)
