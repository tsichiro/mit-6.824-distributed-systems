# [The design of a Practical System for Fault-Tolerant Virtual Machines](the-design-of-a-practical-system-for-fault-tolerant-virtual-machines.pdf)

Scales, Nelson, and Venkitachalam, SIGOPS OSR Vol 44, No 4, Dec 2010

<!-- TOC -->

- [[The design of a Practical System for Fault-Tolerant Virtual Machines](the-design-of-a-practical-system-for-fault-tolerant-virtual-machines.pdf)](#the-design-of-a-practical-system-for-fault-tolerant-virtual-machinesthe-design-of-a-practical-system-for-fault-tolerant-virtual-machinespdf)
    - [Very ambitious system:](#very-ambitious-system)
    - [Overview](#overview)
        - [Why does this idea work?](#why-does-this-idea-work)
        - [What sources of divergence must we guard against?](#what-sources-of-divergence-must-we-guard-against)
        - [Examples of divergence?](#examples-of-divergence)
        - [Example: timer interrupts](#example-timer-interrupts)
        - [Example: disk/network input](#example-disknetwork-input)
        - [Why the bounce buffer?](#why-the-bounce-buffer)
        - [Note that the backup must lag by one event (one log entry)](#note-that-the-backup-must-lag-by-one-event-one-log-entry)
        - [Example: non-functional instructions](#example-non-functional-instructions)
        - [What about disk/network output?](#what-about-disknetwork-output)
        - [Why the Output Rule?](#why-the-output-rule)
        - [The Output Rule is a big deal](#the-output-rule-is-a-big-deal)
    - [Performance (table 1)](#performance-table-1)
    - [When might FT be attractive?](#when-might-ft-be-attractive)
    - [What about replication for high-throughput services?](#what-about-replication-for-high-throughput-services)
    - [Summary:](#summary)
    - [VMware FT FAQ](#vmware-ft-faq)

<!-- /TOC -->

## Very ambitious system:

- Goal: fault-tolerance for existing server software
- Goal: clients should not notice a failure
- Goal: no changes required to client or server software
- Very ambitious!

## Overview

- [diagram: app, O/S, VM-FT underneath, shared disk, network, clients]
- words:
  - hypervisor == monitor == VMM (virtual machine monitor)
  - app and O/S are "guest" running inside a virtual machine
- two machines, primary and backup
- shared disk for persistent storage
  - shared so that bringing up a new backup is faster
- primary sends all inputs to backup over logging channel

### Why does this idea work?

- It's a replicated state machine
- Primary and backup boot with same initial state (memory, disk files)
- Same instructions, same inputs -> same execution
  - All else being equal, primary and backup will remain identical

### What sources of divergence must we guard against?

- Many instructions are guaranteed to execute exactly the same on primary and backup.
  - As long as memory+registers are identical, which we're assuming by induction.
- When might execution on primary differ from backup?
  - Inputs from external world (the network).
  - Data read from storage server.
  - Timing of interrupts.
  - Instructions that aren't pure functions of state, such as cycle counter.
  - Races.

### Examples of divergence?

- They all sound like "if primary fails, clients will see inconsistent story from backup."
- Lock server grants lock to client C1, rejects later request from C2.
  - Primary and backup had better agree on input order!
  - Otherwise, primary fails, backup now tells clients that C2 holds the lock.
- Lock server revokes lock after one minute.
  - Suppose C1 holds the lock, and the minute is almost exactly up.
  - C2 requests the lock.
  - Primary might see C2's request just before timer interrupt, reject.
  - Backup might see C2's request just after timer interrupt, grant.
- So: backup must see same events, in same order, at same point in instruction stream.

### Example: timer interrupts

- Goal: primary and backup should see interrupt at exactly the same point in execution
  - i.e. between the same pair of executed instructions
- Primary:
  - FT fields the timer interrupt
  - FT reads instruction number from CPU
  - FT sends "timer interrupt at instruction X" on logging channel
  - FT delivers interrupt to primary, and resumes it
  - (this relies on special support from CPU to count instructions, interrupt after X)
- Backup:
  - ignores its own timer hardware
  - FT sees log entry *before* backup gets to instruction X
  - FT tells CPU to interrupt at instruction X
  - FT mimics a timer interrupt, resumes backup

### Example: disk/network input

- Primary and backup *both* ask h/w to read
  - FT intercepts, ignores on backup, gives to real h/w on primary
- Primary:
  - FT tells the h/w to DMA data into FT's private "bounce buffer"
  - At some point h/w does DMA, then interrupts
  - FT gets the interrupt
  - FT pauses the primary
  - FT copies the bounce buffer into the primary's memory
  - FT simulates an interrupt to primary, resumes it
  - FT sends the data and the instruction # to the backup
- Backup:
  - FT gets data and instruction # from log stream
  - FT tells CPU to interrupt at instruction X
  - FT copies the data during interrupt

### Why the bounce buffer?

- I.e. why wait until primary/backup aren't executing before copying the data?
- We want the data to appear in memory at exactly the same point in
  - execution of the primary and backup.
- Otherwise they may diverge.

### Note that the backup must lag by one event (one log entry)

- Suppose primary gets an interrupt, or input, after instruction X
- If backup has already executed past X, it cannot handle the input correctly
- So backup FT can't start executing at all until it sees the first log entry
  - Then it executes just to the instruction # in that log entry
  - And waits for the next log entry before restarting backup

### Example: non-functional instructions

- even if primary and backup have same memory/registers,
  - some instructions still execute differently
- e.g. reading the current time or cycle count or processor serial #
- Primary:
  - FT sets up the CPU to interrupt if primary executes such an instruction
  - FT executes the instruction and records the result
  - sends result and instruction # to backup
- Backup:
  - backup also interrupts when it tries to execute that instruction
  - FT supplies value that the primary got

### What about disk/network output?

- Primary and backup both execute instructions for output
- Primary's FT actually does the output
- Backup's FT discards the output

But: the paper's Output Rule (Section 2.2) says primary primary must tell backup when it produces output, and delay the output until the backup says it has received the log entry.

### Why the Output Rule?

- Suppose there was no Output Rule.
- The primary emits output immediately.
- Suppose the primary has seen inputs I1 I2 I3, then emits output.
- The backup has received I1 and I2 on the log.
- The primary crashes and the packet for I3 is lost by the network.
- Now the backup will go live without having processed I3.
  - But some client has seen output reflecting the primary having executed I3.
  - So that client may see anomalous state if it talks to the service again.
- So: the primary doesn't emit output until it knows that the backup
  - has seen all inputs up to that output.

### The Output Rule is a big deal

- Occurs in some form in all replication systems
- A serious constraint on performance
- An area for application-specific cleverness
  - Eg. maybe no need for primary to wait before replying to read-only operation
- FT has no application-level knowledge, must be conservative

Q: What if the primary crashes just after getting ACK from backup,

- but before the primary emits the output?
- Does this mean that the output won't ever be generated?

A: Here's what happens when the primary fails and the backup takes over.

- The backup got some log entries from the primary.
- The backup continues executing those log entries WITH OUTPUT SUPPRESSED.
- After the last log entry, the backup starts emitting output
- In our example, the last log entry is I3
- So after input I3, the client will start emitting outputs
- And thus it will emit the output that the primary failed to emit

Q: But what if the primary crashed *after* emitting the output?

- Will the backup emit the output a *second* time?

A: Yes.

- OK for TCP, since receivers ignore duplicate sequence numbers.
- OK for writes to shared disk, since backup will write same data to same block #.

Duplicate output at cut-over is pretty common in replication systems

- Not always possible for clients &c to ignore duplicates
- For example, if output is vending money from an ATM machine

Q: Does FT cope with network partition -- could it suffer from split brain?

- E.g. if primary and backup both think the other is down.
- Will they both "go live"?

A: The shared disk breaks the tie.

- Shared disk server supports atomic test-and-set.
- Only one of primary/backup can successfully test-and-set.
- If only one is alive, it will win test-and-set and go live.
- If both try, one will lose, and halt.

Shared storage is single point of failure

- If shared storage is down, service is down
- Maybe they have in mind a replicated storage system

Q: Why don't they support multi-core?

## Performance (table 1)

- FT/Non-FT: impressive!
  - little slow down
- Logging bandwidth
  - Directly reflects disk read rate + network input rate
  - 18 Mbit/s for my-sql
- These numbers seem low to me
  - Applications can read a disk at at least 400 megabits/second
  - So their applications aren't very disk-intensive

## When might FT be attractive?

- Critical but low-intensity services, e.g. name server.
- Services whose software is not convenient to modify.

## What about replication for high-throughput services?

- People use application-level replicated state machines for e.g. databases.
  - The state is just the DB, not all of memory+disk.
  - The events are DB commands (put or get), not packets and interrupts.
- Result: less fine-grained synchronization, less overhead.
- GFS use application-level replication, as do Lab 2 &c

## Summary:

- Primary-backup replication
  - VM-FT: clean example
- How to cope with partition without single point of failure?
  - Next lecture
- How to get better performance?
  - Application-level replicated state machines

## VMware FT FAQ

Q: The introduction says that it is more difficult to ensure
deterministic execution on physical servers than on VMs. Why is this
the case?

A: Ensuring determinism is easier on a VM because the hypervisor
emulates and controls many aspects of the hardware that might differ
between primary and backup executions, for example the precise timing
of interrupt delivery.

Q: What is a hypervisor?

A: A hypervisor is part of a Virtual Machine system; it's the same as
the Virtual Machine Monitor (VMM). The hypervisor emulates a computer,
and a guest operating system (and applications) execute inside the
emulated computer. The emulation in which the guest runs is often
called the virtual machine. In this paper, the primary and backup are
guests running inside virtual machines, and FT is part of the
hypervisor implementing each virtual machine.

Q: Both GFS and VMware FT provide fault tolerance. How should we
think about when one or the other is better?

A: FT replicates computation; you can use it to transparently add
fault-tolerance to any existing network server. FT provides fairly
strict consistency and is transparent to server and client. You might
use FT to make an existing mail server fault-tolerant, for example.
GFS, in contrast, provides fault-tolerance just for storage. Because
GFS is specialized to a specific simple service (storage), its
replication is more efficient than FT. For example, GFS does not need
to cause interrupts to happen at exactly the same instruction on all
replicas. GFS is usually only one piece of a larger system to
implement complete fault-tolerant services. For example, VMware FT
itself relies on a fault-tolerant storage service shared by primary
and backup (the Shared Disk in Figure 1), which you could use
something like GFS to implement (though at a detailed level GFS
wouldn't be quite the right thing for FT).

Q: How do Section 3.4's bounce buffers help avoid races?

A: The problem arises when a network packet or requested disk block
arrives at the primary and needs to be copied into the primary's memory.
Without FT, the relevant hardware copies the data into memory while
software is executing. Guest instructions could read that memory
during the DMA; depending on exact timing, the guest might see or not
see the DMA'd data (this is the race). It would be bad if the primary
and backup both did this, but due to slight timing differences one
read just after the DMA and the other just before. In that case they
would diverge.

FT avoids this problem by not copying into guest memory while the
primary or backup is executing. FT first copies the network packet or
disk block into a private "bounce buffer" that the primary cannot
access. When this first copy completes, the FT hypervisor interrupts
the primary so that it is not executing. FT records the point at which
it interrupted the primary (as with any interrupt). Then FT copies the
bounce buffer into the primary's memory, and after that allows the
primary to continue executing. FT sends the data to the backup on the
log channel. The backup's FT interrupts the backup at the same
instruction as the primary was interrupted, copies the data into the
backup's memory while the backup is into executing, and then resumes
the backup.

The effect is that the network packet or disk block appears at exactly
the same time in the primary and backup, so that no matter when they
read the memory, both see the same data.

Q: What is "an atomic test-and-set operation on the shared storage"?

A: The system uses a network disk server, shared by both primary and backup
(the "shared disk" in Figure 1). That network disk server has a
"test-and-set service". The test-and-set service maintains a flag that
is initially set to false. If the primary or backup thinks the other
server is dead, and thus that it should take over by itself, it first
sends a test-and-set operation to the disk server. The server executes
roughly this code:

``` pseudocode
  test-and-set() {
    acquire_lock()
    if flag == true:
      release_lock()
      return false
    else:
      flag = true
      release_lock()
      return true
```

The primary (or backup) only takes over ("goes live") if test-and-set
returns true.

The higher-level view is that, if the primary and backup lose network
contact with each other, we want only one of them to go live. The danger
is that, if both are up and the network has failed, both may go live and
develop split brain. If only one of the primary or backup can talk to
the disk server, then that server alone will go live. But what if both
can talk to the disk server? Then the network disk server acts as a
tie-breaker; test-and-set returns true only to the first call.

Q: How much performance is lost by following the Output Rule?

A: Table 2 provides some insight. By following the output rule, the
transmit rate is reduced, but not hugely.

Q: What if the application calls a random number generator? Won't that
yield different results on primary and backup and cause the executions
to diverge?

A: The primary and backup will get the same number from their random
number generators. All the sources of randomness are controlled by the
hypervisor. For example, the application may use the current time, or
a hardware cycle counter, or precise interrupt times as sources of
randomness. In all three cases the hypervisor intercepts the the
relevant instructions on both primary and backup and ensures they
produce the same values.

Q: How were the creators certain that they captured all possible forms
of non-determinism?

A: My guess is as follows. The authors work at a company where many
people understand VM hypervisors, microprocessors, and internals of guest
OSes well, and will be aware of many of the pitfalls. For VM-FT
specifically, the authors leverage the log and replay support from a
previous a project (deterministic replay), which must have already
dealt with sources of non-determinism. I assume the designers of
deterministic replay did extensive testing and gained much experience
with sources of non-determinism that authors of VM-FT leverage.

Q: What happens if the primary fails just after it sends output to the
external world?

A: The backup will likely repeat the output after taking over, so that
it's generated twice. This duplication is not a problem for network
and disk I/O. If the output is a network packet, then the receiving
client's TCP software will discard the duplicate automatically. If the
output event is a disk I/O, disk I/Os are idempotent (both write the
same data to the same location, and there are no intervening I/Os).

Q: Section 3.4 talks about disk I/Os that are outstanding on the primary when a failure happens; it says "Instead, we re-issue the pending I/Os during the go-live process of the backup VM." Where are the pending I/Os located/stored, and how far back does the re-issuing
need to go?

A: The paper is talking about disk I/Os for which there is a log entry
indicating the I/O was started but no entry indicating completion.
These are the I/O operations that must be re-started on the backup.
When an I/O completes, the I/O device generates an I/O completion
interrupt. So, if the I/O completion interrupt is missing in the log,
then the backup restarts the I/O. If there is an I/O completion
interrupt in the log, then there is no need to restart the I/O.

Q: How secure is this system?

A: The authors assume that the primary and backup follow the protocol
and are not malicious (e.g., an attacker didn't compromise the
hypervisors). The system cannot handle compromised hypervisors. On the
other hand, the hypervisor can probably defend itself against
malicious or buggy guest operating systems and applications.

Q: Is it reasonable to address only the fail-stop failures? What are other type of failures?

A: It is reasonable, since many real-world failures are essentially fail-stop, for example many network and power failures. Doing better
than this requires coping with computers that appear to be operating
correctly but actually compute incorrect results; in the worst case,
perhaps the failure is the result of a malicious attacker. This larger
class of non-fail-stop failures is often called "Byzantine". There are
ways to deal with Byzantine failures, which we'll touch on at the end
of the course, but most of 6.824 is about fail-stop failures.
