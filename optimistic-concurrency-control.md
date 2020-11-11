6.824 2020 Lecture 14: FaRM, Optimistic Concurrency Control

why are we reading about FaRM?
  another take on transactions+replication+sharding
    this is still an open research area!
  motivated by huge performance potential of RDMA NICs

how does FaRM differ from Spanner?
  both replicate and use two-phase commit (2pc) for transactions
  Spanner:
    a deployed system
    focuses on geographic replication
      e.g. copies on East and West coasts, in case data centers fail
    is most innovative for read-only transactions -- TrueTime
    performance: r/w xaction takes 10 to 100 ms (Tables 3 and 6)
  FaRM
    a research prototype, to explore potential of RDMA
    all replicas are in same data center (wouldn't make sense otherwise)
    RDMA restricts design options: thus Optimistic Concurrency Control (OCC)
    performance: 58 microseconds for simple transactions (6.3, Figure 7)
      i.e. 100 times faster than Spanner
    performance: throughput of 100 million/second on 90 machines (Figure 7)
      extremely impressive, particularly for transactions+replication
  They target different bottlenecks:
    Spanner: speed of light and network delays
    FaRM: CPU time on servers

the overall setup
  all in one data center
  configuration manager, using ZooKeeper, chooses primaries/backups
  sharded w/ primary/backup replication
    P1 B1
    P2 B2
    ...
    can recover as long as at least one replica of each shard
    i.e. f+1 replicas tolerate f failures
  transaction clients (which they run in the servers)
  transaction code acts as two-phase-commit Transaction Coordinator (TC)

how do they get high performance?
  sharding over many servers (90 in the evaluation)
  data must fit in total RAM (so no disk reads)
  non-volatile RAM (so no disk writes)
  one-sided RDMA (fast cross-network access to RAM)
  fast user-level access to NIC
  transaction+replication protocol that exploits one-sided RDMA

NVRAM (non-volatile RAM)
  FaRM writes go to RAM, not disk -- eliminates a huge bottleneck
  RAM write takes 200 ns, hard drive write takes 10 ms, SSD write 100 us
    ns = nanosecond, ms = millisecond, us = microsecond
  but RAM loses content in power failure! not persistent by itself.
  why not just write to RAM of f+1 machines, to tolerate f failures?
    might be enough if failures were always independent
    but power failure is not independent -- may strike 100% of machines!
  so:
    batteries in every rack, can run machines for a few minutes
    power h/w notifies s/w when main power fails
    s/w halts all transaction processing
    s/w writes FaRM's RAM to SSD; may take a few minutes
    then machine shuts down
    on re-start, FaRM reads saved memory image from SSD
    "non-volatile RAM"
  what if crash prevents s/w from writing SSD?
    e.g bug in FaRM or kernel, or cpu/memory/hardware error
    FaRM copes with single-machine crashes by copying data
      from RAM of machines' replicas to other machines
      to ensure always f+1 copies
    crashes (other than power failure) must be independent!
  summary:
    NVRAM eliminates persistence write bottleneck
    leaving network and CPU as remaining bottlenecks

why is the network often a performance bottleneck?
  the usual setup for RPC over TCP over LAN:
    app                       app
    ---                       ---
    socket buffers            buffers
    TCP                       TCP
    NIC driver                driver
    NIC  -------------------- NIC
  lots of expensive CPU operations:
    system calls
    copy messages
    interrupts
  slow:
    hard to build RPC than can deliver more than a few 100,000 / second
    wire b/w (e.g. 10 gigabits/second) is rarely the limit for short RPC
    per-packet CPU costs are the limiting factor for small messages

FaRM uses two networking ideas:
  Kernel bypass
  RDMA

Kernel bypass
  [diagram: FaRM user program, CPU cores, DMA queues, NIC]
  application directly interacts with NIC -- no system calls, no kernel
  NIC DMAs into/out of user RAM
  FaRM s/w polls DMA areas to check for new messages
  
RDMA (remote direct memory access)
  [src host, NIC, switch, NIC, target memory, target CPU]
  remote NIC directly reads/writes memory
    Sender provides memory address
    Remote CPU is not involved!
    This is "one-sided RDMA"
    Reads an entire cache line, atomically
    (Not sure about writes)
  RDMA NICs use reliable protocol, with ACKs
  one server's throughput: 10+ million/second (Figure 2)
  latency: 5 microseconds (from their NSDI 2014 paper)

Performance would be amazing if clients could directly access 
  DB records on servers via RDMA!

Q: Can transactions just directly read/write with one-sided RDMA?

How to combine RDMA with replication and transactions?
  The protocols we've seen so far require active server participation.
  e.g. is that record locked?
       which is the latest version?
       is that write committed yet?
  Not immediately compatible with one-sided RDMA.

two classes of concurrency control for transactions:
  pessimistic:
    wait for lock on first use of object; hold until commit/abort
    called two-phase locking
    conflicts cause delays
  optimistic:
    read objects without locking
    don't install writes until commit
    commit "validates" to see if other xactions conflicted
    valid: commit the writes
    invalid: abort
    called Optimistic Concurrency Control (OCC)

FaRM uses OCC
  the reason:
    OCC lets FaRM read using one-sided RDMA reads
    server needn't actively participate (no lock, due to OCC)
  how does FaRM validate? we'll look at Figure 4 in a minute.

FaRM transaction API (simplified):
  txCreate()
  o = txRead(oid)  -- RDMA
  o.f += 1
  txWrite(oid, o)  -- purely local
  ok = txCommit()  -- Figure 4

what's an oid?
  <region #, address>
  region # indexes a mapping to [ primary, backup1, ... ]
  target RDMA NIC uses address directly to read or write RAM

server memory layout
  regions, each an array of objects
  object layout
    header with version #, and lock flag in high bit of version #
  for each other server
    (written by RDMA, read by polling)
    incoming log
    incoming message queue
  all this in non-volatile RAM (i.e. written to SSD on power failure)

Figure 4: transaction execution / commit protocol
  let's consider steps in Figure 4 one by one
  focus on concurrency control (not fault tolerance)

Execute phase
  TC (the client) reads the objects it needs from servers
    including records that it will write
    using one-sided RDMA reads
    without locking
    this is the optimism in Optimistic Concurrency Control
  TC remembers the version numbers
  TC buffers writes

LOCK (first message in commit protocol)
  TC sends to primary of each written object
  TC uses RDMA to append to its log at each primary
  LOCK record contains oid, version # xaction read, new value
  LOCK is now logged in primary's NVRAM, in case power fails

what does primary do on receipt of LOCK?
  it polls incoming logs in RAM, sees our LOCK
  if object locked, or version != what xaction read, reply "no"
  otherwise set the lock flag and return "yes"
  lock check, version check, and lock set are atomic
    using atomic compare-and-swap instructuion
    "locked" flag is high-order bit in version number
    in case other CPU also processing a LOCK, or a client is reading w/ RDMA
  if object already locked, does not block, just replies "no"
    which will cause the TC to abort the xaction

TC waits for all LOCK reply messages
  if any "no", abort
    append ABORT to primaries' logs so they can release locks
    returns "no" from txCommit()

let's ignore VALIDATE and COMMIT BACKUP for now

at this point primaries need to know TC's decision

TC appends COMMIT-PRIMARY to primaries' logs
  TC only waits for RDMA hardware acknowledgement (ack)
    does not wait for primary to process log entry
    hardware ack means safe in primary's NVRAM
  TC returns "yes" from txCommit()

when primary processes COMMIT-PRIMARY in its log:
  copy new value over object's memory
  increment object's version #
  clear object's lock flag

example:
  T1 and T2 both want to increment x
    x = x + 1
  what results does serializability allow?
    i.e. what outcomes are possible if run one at a time?
    x = 2, both clients told "success"
    x = 1, one client told "success", other "aborted"
    x = 0, both clients told "aborted"

what if T1 and T2 are exactly in step?
  T1: Rx0  Lx  Cx
  T2: Rx0  Lx  Cx
  what will happen?

or
  T1:    Rx0 Lx Cx
  T2: Rx0          Lx  Cx

or
  T1: Rx0  Lx  Cx
  T2:             Rx0  Lx  Cx

intuition for why FaRM's OCC provides serializability:
  i.e. checks "was execution same as one at a time?"
  if there was no conflicting transaction:
    the versions won't have changed
  if there was a conflicting transaction:
    one or the other will see a lock or changed version #

what about VALIDATE in Figure 4?
  it is an optimization for objects that are just read by a transaction
  VALIDATE = one-sided RDMA read to re-fetch object's version # and lock flag
  if lock set, or version # changed since read, TC aborts
  does not set the lock, thus faster than LOCK+COMMIT

VALIDATE example:
x and y initially zero
T1:
  if x == 0:
    y = 1
T2:
  if y == 0:
    x = 1
(this is a classic test example for strong consistency)
T1,T2 yields y=1,x=0
T2,T1 yields x=1,y=0
aborts could leave x=0,y=0
but serializability forbids x=1,y=1

suppose simultaneous:
  T1:  Rx  Ly  Vx  Cy
  T2:  Ry  Lx  Vy  Cx
  what will happen?
  the LOCKs will both succeed!
  the VALIDATEs will both fail, since lock bits are both set
  so both will abort -- which is OK

how about:
  T1:  Rx  Ly  Vx      Cy
  T2:  Ry          Lx  Vy  Cx
  T1 commits
  T2 aborts since T2's Vy sees T1's lock or higher version
but we can't have *both* V's before the other L's
so VALIDATE seems correct in this example
  and fast: one-sided VALIDATE read rather than LOCK+COMMIT writes

a purely read-only FaRM transaction uses only one-sided RDMA reads
  no writes, no log records
  very fast!

what about fault tolerance?
  some computers crash and don't reboot
  most interesting if TC and some primaries crash
  but we assume one backup from each shard survives

the critical issue:
  if a transaction was interrupted by a failure,
    and a client could have been told a transaction committed,
    or a committed value could have been read by another xaction,
  then the transaction must be preserved and completed during recovery.

look at Figure 4.
a committed write might be revealed as soon the
  first COMMIT-PRIMARY is sent (since primary writes and unlocks).
so by then, all of the transaction's writes must be on all
  f+1 replicas of all relevant shards.
the good news: LOCK and COMMIT-BACKUP achieve this.
  LOCK tells all primaries the new value(s).
  COMMIT-BACKUP tells all backups the new value(s).
  TC doesn't send COMMIT-PRIMARY until all LOCKs and COMMIT-BACKUPS complete.
  backups may not have processed COMMIT-BACKUPs, but in NVRAM logs.

similarly, TC doesn't return to client until at least one
  COMMIT-PRIMARY is safe in primary log.
  this means' TC's decision will survive f failures of any shard.
  since there's one shard with a full set of COMMIT-BACKUP or COMMIT-PRIMARY.
  any of which is evidence that the primary decided to commit.

FaRM is very impressive; does it fall short of perfection?
  * works best if few conflicts, due to OCC.
  * data must fit in total RAM.
  * replication only within a datacenter (no geographic distribution).
  * the data model is low-level; would need e.g. SQL library.
  * details driven by specific NIC features; what if NIC had test-and-set?
  * requires somewhat unusual RDMA and NVRAM hardware.

summary
  super high speed distributed transactions
  hardware is exotic (NVRAM and RDMA) but may be common soon
  use of OCC for speed and to allow fast one-sided RDMA reads

FAQ FaRM

Q: What are some systems that currently uses FaRM?

A: Just FaRM. It's not in production use; it's a recent research
prototype. I suspect it will influence future designs, and perhaps
itself be developed into a production system.

Q: Why did Microsoft Research release this? Same goes for the research
branches of Google, Facebook, Yahoo, etc. Why do they always reveal
the design of these new systems? It's definitely better for the
advancement of tech, but it seems like (at least in the short term) it
is in their best interest to keep these designs secret.

A: These companies only publish papers about a tiny fraction of the
software they write. One reason they publish is that these systems are
partially developed by people with an academic background (i.e. who
have PhDs), who feel that part of their mission in life is to help the
world understand the new ideas they invent. They are proud of their
work and want people to appreciate it. Another reason is that such
papers may help the companies attract top talent, because the papers
show that intellectually interesting work is going on there.

Q: Does FaRM really signal the end of necessary compromises in
consistency/availability in distributed systems?

A: I suspect not. For example, if you are willing to do
non-transactional one-sided RDMA reads and writes, you can do them
about 5x as fast as FaRM can do full transactions. Perhaps few people
want that 5x more performance today, but they may someday. Along
another dimension, there's a good deal of interest in transactions for
geographically distributed data, for which FaRM doesn't seem very
relevant.

Q: This paper does not really focus on the negatives of FaRM. What are
some of the biggest limitations FaRM?

A: Here are some guesses. The data has to fit in RAM. OCC will produce
lots of aborts if transactions conflict a lot. The transaction API
(described in their NSDI 2014 paper) looks awkward to use because
replies return in callbacks. Application code has to tightly
interleave executing application transactions and polling RDMA NIC
queues and logs for messages from other computers. Application code
can see inconsistencies while executing transactions that will
eventually abort. For example, if the transaction reads a big object
at the same time that a commiting transaction is overwriting the
object. The risk is that the application may crash if it isn't
defensively written. Applications may not be able to use their own
threads very easily because FaRM pins threads to cores, and uses all
cores. Of course, FaRM is a research prototype intended to explore new
ideas. It is not a finished product intended for general use. If
people continue this line of work, we might eventually see descendants
of FaRM with fewer rough edges.

Q: What's a NIC?

A: A Network Interface Card -- the hardware that connects a computer
to the network.

Q: What is RDMA? One-sided RDMA?

A: RDMA is a special feature implemented in some modern NICs (network
interface cards). The NIC looks for special command packets that
arrive over the network, and executes the commands itself (and does
not give the packets to the CPU). The commands specify memory
operations such as write a value to an address or read from an address
and send the value back over the network. In addition, RDMA NICs allow
application code to directly talk to the NIC hardware to send the
special RDMA command packets, and to be notified when the "hardware
ACK" packet arrives indicating that the receiving NIC has executed the
command.

"One-sided" refers to a situation where application code in one
computer uses these RDMA NICs to directly read or write memory in
another computer without involving the other computer's CPU. FaRM's
"Validate" phase in Section 4 / Figure 4 uses only a one-sided read.

FaRM sometimes uses RDMA as a fast way to implement an RPC-like scheme
to talk to software running on the receiving computer. The sender uses
RDMA to write the request message to an area of memory that the
receiver's FaRM software is polling (checking periodically); the
receiver sends its reply in the same way. The FaRM "Lock" phase uses
RDMA in this way.

The benefit of RDMA is speed. A one-sided RDMA read or write takes as
little as 1/18 of a microsecond (Figure 2), while a traditional RPC
might take 10 microseconds. Even FaRM's use of RDMA for messaging is a
lot faster than traditional RPC: user-space code in the receiver
frequently polls the incoming NIC queues in order to see new messages
quickly, rather than involving interrupts and user/kernel transitions.

Q: Why is FaRM's RDMA-based RPC faster than traditional RPC?

A: Traditional RPC requires the application to make a system call to
the local kernel, which asks the local NIC to send a packet. At the
receiving computer, the NIC writes the packet to a queue in memory and
interrrupts the receving computer's kernel. The kernel copies the
packet to user space and causes the application to see it. The
receving application does the reverse to send the reply (system call
to kernel, kernel talks to NIC, NIC on the other side interrupts its
kernel, &c). This point is that a huge amount of code is executed for
each RPC, and it's not very fast.

Q: Much of FaRM's performance comes from the hardware. In what ways
does the software design contribute to performance?

A: It's true that one reason FaRM is fast is that the hardware is
fast. But the hardware has been around for many years now, yet no-one
has figured out how to put all the parts together in a way that really
exploits the hardware's potential. One reason FaRM does so well is
that they simultaneously put a lot of effort into optimizing the
network, the persistent storage, and the use of CPU; many previous
systems have optimized one but not all. A specific design point is the
way FaRM uses fast one-sided RDMA (rather than slower full RPC) for
many of the interactions.

Q: Do other systems use UPS (uninterruptable power supplies, with
batteries) to implement fast but persistent storage?

A: The idea is old; for example the Harp replicated file service used
it in the early 1990s. Many storage systems use batteries in related
ways (e.g. in RAID controllers) to avoid having to wait for disk
writes. However, the kind of battery setup that FaRM uses isn't
particularly common, so software that has to be general purpose can't
rely on it. If you configure your own hardware to have batteries, then
it would make sense to modify your Raft (or k/v server) to exploit
your batteries.

Q: Would the FaRM design still make sense without the battery-backed RAM?

A: I'm not sure FaRM would work without non-volatile RAM, because then
the one-sided log writes (e.g. COMMIT-BACKUP in Figure 4) would not
persist across power failures. You could modify FaRM so that all log
updates were written to SSD before returning, but then it would have
much lower performance. An SSD write takes about 100 microseconds,
while FaRM's one-sided RDMA writes to non-volatile RAM take only a few
microseconds.

Q: A FaRM server copies RAM to SSD if the power is about to fail.
Could they use mechanical hard drives instead of SSDs?

A: They use SSDs because they are fast. They could have used hard
drives without changing the design. However, it would then have taken
longer to write the data to disk during a power outage, and that would
require bigger batteries. Maybe the money they'd save by using hard
drives would be outweighed by the increased cost of batteries.

Q: What is the distinction between primaries, backups, and
configuration managers in FaRM? Why are there three roles?

A: The data is sharded among many primary/backup sets. The point of
the backups is to store a copy of the shard's data and logs in case
the primary fails. The primary performs all reads and writes to data
in the shard, while the backups perform only the writes (in order to
keep their copy of the data identical to the primary's copy). There's
just one configuration manager. It keeps track of which primaries and
backups are alive, and keeps track of how the data is sharded among
them. At a high level this arrangement is similar to GFS, which also
sharded data among many primary/backup sets, and also had a master
that kept track of where data is stored.

Q: Would FaRM make sense at small scale?

A: I think FaRM is only interesting if you need to support a huge
number of transactions per second. If you only need a few thousand
transactions per second, you can use off-the-shelf mature technology
like MySQL. You could probably set up a considerably smaller FaRM
system than the authors' 90-machine system. But FaRM doesn't make
sense unless you are sharding and replicating data, which means you
need at least four data servers (two shards, two servers per shard)
plus a few machines for ZooKeeper (though probably you could run
ZooKeeper on the four machines). Then maybe you have a system that
costs on the order of $10,000 dollars and can execute a few million
simple transactions per second, which is pretty good.

Q: Section 3 seems to say that a single transaction's reads may see
inconsistent data. That doesn't seem like it would be serializable!

A: Farm only guarantees serializability for transactions that commit.
If a transaction sees the kind of inconsistency Section 3 is talking
about, FaRM will abort the transaction. Applications must handle
inconsistency in the sense that they should not crash, so that they
can get as far as asking to commit, so that FaRM can abort them.

Q: How does FaRM ensure that a transaction's reads are consistent?
What happens if a transaction reads an object that is being modified
by a different transaction?

A: There are two dangers here. First, for a big object, the reader may
read the first half of the object before a concurrent transaction has
written it, and the second half after the concurrent transaction has
written it, and this might cause the reading program to crash. Second,
the reading transaction can't be allowed to commit if it might not be
serializable with a concurrent writing transaction.

Based on my reading of the authors' previous NSDI 2014 paper, the
solution to the first problem is that every cache line of every object
has a version number, and single-cache-line RDMA reads and writes are
atomic. The reading transaction's FaRM library fetches all of the
object's cache lines, and then checks whether they all have the same
version number. If yes, the library gives the copy of the object to
the application; if no, the library reads it again over RDMA. The
second problem is solved by FaRM's validation scheme described in
Section 4. In the VALIDATE step, if another transaction has written an
object read by our transaction since our transaction started, our
transaction will be aborted.

Q: How does log truncation work? When can a log entry be removed?
If one entry is removed by a truncate call, are all previous entries
also removed?

A: The TC tells the primaries and backups to delete the log entries
for a transaction after the TC sees that all of them have a
COMMIT-PRIMARY or COMMIT-BACKUP in their log. In order that recovery
will know that a transaction is done despite truncation, page 62
mentions that primaries remember completed transaction IDs even after
truncation. Truncation implies that all log entries before the
truncation point are deleted; this works because each primary/backup
has a separate log per TC.

Q: Is it possible for an abort to occur during COMMIT-BACKUP, perhaps
due to hardware failure?

A: I believe so. If one of the backups doesn't respond, and the TC
crashes, then there's a possibility that the transaction might be
aborted during recovery.

Q: Since this is an optimistic protocol, does it suffer when many
transactions need to modify the same object?

A: When multiple transactions modify the same object at the same time,
some of them will see during Figure 4's LOCK phase that the lock is
already held. Each such transaction will abort (they do not wait until
the lock is released), and will restart from the beginning. If that
happens a lot, performance will indeed suffer. But, for the
applications the authors measure, FaRM gets fantastic performance.
Very likely one reason is that their applications have relatively few
conflicting transactions, and thus not many aborts.

Q: Figure 7 shows significant increase in latency when the number of
operations exceeds 120 per microsecond. Why is that?

A: I suspect the limit is that the servers can only process about 140
million operations per second in total. If clients send operations
faster than that, some of them will have to wait; this waiting causes
increased latency.  