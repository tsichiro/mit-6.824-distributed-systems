## LEC 11: Optimistic Concurrency Control

6.824 2017 Lecture 10: FaRM

course note: final project proposals due this Friday!
  can do final project or lab 4
  form group of 2-3 students
  please talk to us/email us about project ideas!

why are we reading this paper?
  many people want distributed transactions
  but they are thought to be slow
  this paper suggests that needn't be true -- very surprising performance!

big performance picture
  90 million *replicated* *persistent* *transactions* per second (Figure 7)
    1 million transactions/second per machine
    each with a few messages, for replication and commit
    very impressive
  a few other systems get 1 million ops/second per machine, e.g. memcached
    but not transactions + replicated + persistent (often not any of these!)
  perspective on 90 million:
    10,000 Tweets per second
    2,000,000 e-mails per second

how do they get high performance?
  data must fit in total RAM (so no disk reads)
  non-volatile RAM (so no disk writes)
  one-sided RDMA (fast cross-network access to RAM)
  fast user-level access to NIC
  transaction+replication protocol that exploits one-sided RDMA

NVRAM
  FaRM writes go to RAM, not disk -- eliminates a huge bottleneck
  can write RAM in 200 ns, but takes 10 ms to write hard drive, 100 us for SSD
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

why is the network often a performance bottleneck?
  the usual setup:
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
    and all twice if RPC
  slow:
    hard to build RPC than can deliver more than a few 100,000 / second
    wire b/w (e.g. 10 gigabits/second) is rarely the limit for short RPC
    these per-packet CPU costs are the limiting factor for small messages

Kernel bypass
  application access to NIC h/w is streamlined
  application directly interacts with NIC -- no system calls, no kernel
  shared memory mapping between app and NIC
  sender gives NIC an RDMA command
  for RPC, receiver s/w polls memory which RDMA writes

FaRM's network setup
  [hosts, 56 gbit NICs, expensive switch]
  NIC does "one-sided RDMA": memory read/write, not packet delivery
  sender says "write this data at this address", or "read this address"
    NIC *hardware* executes at the far end
    returns a "hardware acknowledgement"
  no interrupt, kernel, copy, read(), &c at the far end
  one server's throughput: 10+ million/second (Figure 2)
  latency: 5 microseconds (from their NSDI 2014 paper)
  FaRM uses RDMA in three ways:
    one-sided read of objects during transaction execution (also VALIDATE)
    RPC composed of one-sided writes to primary's logs or message queues
    one-sided write into backup's log

big challenge:
  how to use one-sided read/write for transactions and replication?
  protocols we've seen require receiver CPU to actively process messages
    e.g. Raft and two-phase-commit

let's review distributed transactions

remember this example:
  x and y are bank balances, maybe on different servers
  T1:             T2:
    add(x, 1)       tmp1 = get(x)
    add(y, -1)      tmp2 = get(y)
                    print tmp1, tmp2
  x and y start at $10
  we want serializability:
    results should be as if transactions ran one at a time in some order
  only two orders are possible
    T1 then T2 yields 11, 9
    T2 then T1 yields 10, 10
    serializability allows no other result

what if T1 runs entirely between T2's two get()s?
  would print 10,9 if the transaction protocol allowed it
  but it's not allowed!
what if T2 runs entirely between T1's two adds()s?
  would print 11,10 if the transaction protocol allowed it
  but it's not allowed!

two classes of concurrency control for transactions:
  pessimistic:
    wait for lock on first use of object; hold until commit/abort
    called two-phase locking
    conflicts cause delays
  optimistic:
    access object without locking; commit "validates" to see if OK
      valid: do the writes
      invalid: abort
    called Optimistic Concurrency Control (OCC)

FaRM uses OCC
  the reason: OCC lets FaRM read using one-sided RDMA reads
    server storing the object does not need to set a lock, due to OCC
  how does FaRM validate? we'll look at Figure 4 in a minute.

every FaRM server runs application transactions and stores objects
  an application transaction is its own transaction coordinator (TC)

FaRM transaction API (simplified):
  txCreate()
  o = txRead(oid)  -- RDMA
  o.f += 1
  txWrite(oid, o)  -- purely local
  ok = txCommit()  -- Figure 4

txRead
  one-sided RDMA to fetch object direct from primary's memory -- fast!
  also fetches object's version number, to detect concurrent writes

txWrite
  must be preceded by txRead
  just writes local copy; no communication

what's in an oid?
  <region #, address>
  region # indexes a mapping to [ primary, backup1, ... ]
  target NIC can use address directly to read or write RAM
    so target CPU doesn't have to be involved

server memory layout
  regions, each an array of objects
  object layout
    header with version # and lock
  for each other server
    (written by RDMA, read by polling)
    incoming log
    incoming message queue
  all this in non-volatile RAM (i.e. written to SSD on power failure)

every region replicated on one primary, f backups -- f+1 replicas
  [diagram of a few regions, primary/backup]
  only the primary serves reads; all f+1 see commits+writes
  replication yields availability if <= f failures
    i.e. available as long as one replica stays alive; better than Raft

transaction execution / commit protocol w/o failure -- Figure 4
  let's consider steps in Figure 4 one by one
  thinking about concurrency control for now (not replication)

LOCK (first message in commit protocol)
  TC sends to primary of each written object
  TC uses RDMA to append to its log at each primary
  LOCK record contains oid, version # xaction read, new value
  primary s/w polls log, sees LOCK, validates, sends "yes" or "no" reply message
  note LOCK is both logged in primary's NVRAM *and* an RPC exchange

what does primary CPU do on receipt of LOCK?
  (for each object)
  if object locked, or version != what xaction read, reply "no"
    implemented with atomic compare-and-swap
    "locked" flag is high-order bit in version number
  otherwise set the lock flag and return "yes"
  note: does *not* block if object is already locked

TC waits for all LOCK reply messages
  if any "no", abort
    send ABORT to primaries so they can release locks
    returns "no" from txCommit()

let's ignore VALIDATE and COMMIT BACKUP for now

TC sends COMMIT-PRIMARY to primary of each written object
  uses RDMA to append to primary's log
  TC only waits for hardware ack -- does not wait for primary to process log entry
  TC returns "yes" from txCommit()

what does primary do when it processes the COMMIT-PRIMARY in its log?
  copy new value over object's memory
  increment object's version #
  clear object's lock flag

example:
  T1 and T2 both want to increment x
  both say
    tmp = txRead(x)
    tmp += 1
    txWrite(x)
    ok = txCommit()
  x should end up with 0, 1, or 2, consistent with how many successfully committed

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

intuition for why validation checks serializability:
  i.e. checks "was execution one at a time?"
  if no conflict, versions don't change, and commit is allowed
  if conflict, one will see lock or changed version #

what about VALIDATE in Figure 4?
  it is an optimization for objects that are just read by a transaction
  VALIDATE = one-sided RDMA read to fetch object's version # and lock flag
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
(this is a classic test example for consistency)
T1,T2 yields y=1,x=0
T2,T1 yields x=1,y=0
aborts could leave x=0,y=0
but serializability forbids x=1,y=1

suppose simultaneous:
  T1:  Rx  Ly  Vx  Cy
  T2:  Ry  Lx  Vy  Cx
  the LOCKs will both succeed
  the VALIDATEs will both fail, since lock bits are both set
  so both will abort -- which is OK

how about:
  T1:  Rx  Ly  Vx      Cy
  T2:  Ry          Lx  Vy  Cx
  then T1 commits, T2 still aborts since T2's Vy sees T1's lock or higher version
but we can't have *both* V's before the other L's
so VALIDATE seems correct in this example
  and fast: faster than LOCK, no COMMIT required

what about fault tolerance?
  defense against losing data?
    durable? available?
  integrity of underway transactions despite crashes?
  partitions?

high-level replication diagram
  o o region 1
  o o region 2
  o CM
  o o o ZK

f+1 copies of each region to tolerate <= f failures in each region
  TCs send all writes to all copies (TC's COMMIT-BACKUP)
  not immediately available if a server crashes
    transaction reads and commits will wait
  but CM will soon notice, make a new copy, recover transactions

reconfiguration
  one ZooKeeper cluster (a handful of replicas)
    stores just configuration #, set of servers in this config, and CM
    breaks ties if multiple servers try to become CM
    chooses the active partition if partitioned (majority partition)
  a Configuration Manager (CM) (not replicated)
    monitors liveness of all servers via rapid ping
    manages reconfiguration
      renews leases
        only activates if it gets a response from majority of machines
      checks that at least one copy of each region exists
      assigns regions to primary/backup sets
      tells servers to make new copies
      manages completion of interrupted transactions

let's look back the at the Figure 4 commit protocol to see how
  any xaction that might have committed will be visible despite failed servers.
  "might have committed":
    TC might have replied "yes" to client
    primary might have revealed update to a subsequent read

after TC sees "yes" from all LOCKs and VALIDATEs,
  TC appends COMMIT-BACKUP to each backup's log
  after all ack, appends COMMIT-PRIMARY to each primary's log
  after one ack, reports "commit" to application

note TC replicates to backups; primaries don't replicate
  COMMIT-BACKUP contains written value, enough to update backup's state

why TC sends COMMIT-PRIMARY only after acks for *all* COMMIT-BACKUPs?
  a primary may execute as soon as it sees COMMIT-PRIMARY
    and show the update to other transactions
  so by that point each object's new value must in f+1 logs (per region)
    so f can fail without losing the new value
  if there's even one backup that doesn't have the COMMIT-BACKUP
    that object's writes are in only f logs
    all f could fail along with TC
    then we'd have exposed commit but maybe permanently lost one write!

why TC waits for an ack from a COMMIT-PRIMARY?
  so that there is a complete f+1 region that's aware of the commit
  before then, only f backups per region knew (from COMMIT-BACKUPs)
  but we're assuming up to f per region to fail

the basic line of reasoning for why recovery is possible:
  if TC could have reported "commit", or a primary could have exposed value,
  then all f+1 in each region have LOCK or COMMIT-BACKUP in log,
  so f can fail from any/every region without losing writes.
  if recovery sees one or more COMMIT-*, and a COMMIT-* or LOCK
    from each region, it commits; otherwise aborts.
    i.e. evidence TC decided commit, plus each object's writes.
    (Section 5.3, Step 7)

FaRM is very impressive; does it fall short of perfection?
  * works best if few conflicts, due to OCC.
  * data must fit in total RAM.
  * the data model is low-level; would need e.g. SQL library.
  * details driven by specific NIC features; what if NIC had test-and-set?

summary
  distributed transactions have been viewed as too slow for serious use
  maybe FaRM demonstrates that needn't be true

FAQ FaRM
Q: What are some systems that currently uses FaRM?

A: Just FaRM. It's not in production use; it's a very recent research prototype. I suspect it will influence future designs, and perhaps itself be developed into a production system.

Q: Why did Microsoft Research release this? Same goes for the research branches of Google, Facebook, Yahoo, etc. Why do they always reveal the design of these new systems? It's definitely better for the advancement of tech, but it seems like (at least in the short term) it is in their best interest to keep these designs secret.

A: These companies only publish papers about a tiny fraction of the software they write. One reason they publish is that these systems are partially developed by people with an academic background (i.e. who have PhDs), who feel that part of their mission in life is to help the world understand the new ideas they invent. They are proud of their work and want people to appreciate it. Another reason is that such papers may help the companies attract top talent, because the papers show that intellectually interesting work is going on there.

Q: How does FaRM compare to Thor in terms of performance? It seems easier to understand.

A: FaRM is faster than Thor by many orders of magnitude.

Q: Does FaRM really signal the end of necessary compromises in consistency/availability in distributed systems?

A: I suspect not. For example, if you are willing to do non-transactional one-sided RDMA reads and writes, you can do them about 5x as fast as FaRM can do full transactions. Perhaps few people want that 5x more performance today, but they may someday. Along another dimension, there's a good deal of interest in transactions (or some good semantics) for geographically distributed data, for which FaRM doesn't seem very relevant.

Q: This paper does not really focus on the negatives of FaRM. What are some of the biggest cons of using FaRM?

A: Here are some guesses.
The data has to fit in RAM.
Use of OCC isn't great if there are many conflicting transactions.
The transaction API (described in their NSDI 2014 paper) looks awkward to use because replies return in callbacks.
Application code has to tightly interleave executing application transactions and polling RDMA NIC queues and logs for messages from other computers.
Application code can see inconsistencies while executing transactions that will eventually abort. For example, if the transaction reads a big object at the same time that a commiting transaction is overwriting the object. The risk is that the application may crash if it isn't defensively written.
Applications may not be able to use their own threads very easily because FaRM pins threads to cores, and uses all cores. Of course, FaRM is a research prototype intended just to explore some new ideas. It is not a finished product intended for general use. If people continue this line of work, we might eventually see descendants of FaRM with fewer rough edges.

Q: How does RDMA differ from RPC? What is one-sided RDMA?

A: RDMA is a special feature implemented in some modern NICs (network interface cards). The NIC looks for special command packets that arrive over the network, and executes the commands itself (and does not give the packets to the CPU). The commands specify memory operations such as write a value to an address or read from an address and send the value back over the network. In addition, RDMA NICs allow application code to directly talk to the NIC hardware to send the special RDMA command packets, and to be notified when the "hardware ACK" packet arrives indicating that the receiving NIC has executed the command.

"One-sided" refers to a situation where application code in one computer uses these RDMA NICs to directly read or write memory in another computer without involving the other computer's CPU. FaRM's "Validate" phase in Section 4 / Figure 4 uses only a one-sided read. FaRM sometimes uses RDMA as a fast way to implement an RPC-like scheme to talk to software running on the receiving computer. The sender uses RDMA to write the request message to an area of memory that the receiver's FaRM software is polling (checking periodically); the receiver sends its reply in the same way. The FaRM "Lock" phase uses RDMA in this way.

Traditional RPC requires the application to make a system call to the local kernel, which asks the local NIC to send a packet. At the receiving computer, the NIC writes the packet to a queue in memory and interrrupts the receving computer's kernel. The kernel copies the packet to user space and causes the application to see it. The receving application does the reverse to send the reply (system call to kernel, kernel talks to NIC, NIC on the other side interrupts its kernel, &c). This point is that a huge amount of code is executed for each RPC, and it's not very fast.

The benefit of RDMA is speed. A one-sided RDMA read or write takes as little as 1/18 of a microsecond (Figure 2), while a traditional RPC might take 10 microseconds. Even FaRM's use of RDMA for messaging is a lot faster than traditional RPC because, while both side's CPUs are involved, neither side's kernel is involved.

Q: It seems like the performance of FaRM mainly comes from the hardware. What is the remarking points of the design of FaRM other than harware parts?

A: It's true that one reason FaRM is fast is that the hardware is fast. But the hardware has been around for many years now, yet no-one has figured out how to put all the parts together in a way that really exploits the hardware's potential. One reason FaRM does so well is that they simultaneously put a lot of effort into optimizing the network, the persistent storage, and the use of CPU; many previous systems have optimized one but not all. A specific design point is the way FaRM uses fast one-sided RDMA (rather than slower full RPC) for many of the interactions.

Q: The paper states that FaRM exploits recent hardware trends, in particular the prolification of UPS to make DRAM memory non-volatile. Have any of the systems we have read about exploited this trend since they were created? For example, do modern implementations of Raft no longer have to worry about persisting non-volatile state because they can be equipped with a UPS?

A: The idea is old; for example the Harp replicated file service used it in the early 1990s. Many storage systems have used batteries in other ways (e.g. in RAID controllers) to avoid having to wait for disk writes. However, the kind of battery setup that FaRM uses isn't particularly common, so software that has to be general purpose can't rely on it. If you configure your own hardware to have batteries, then it would make sense to modify your Raft (or k/v server) to exploit your batteries.

Q: The paper mentions the usage of hardware, such as DRAM and Li-ion batteries for better recovery times and storage. Does the usage of FaRM without the hardware optimizations still provide any benefit over past transactional systems?

A: I'm not sure FaRM would work without non-volatile RAM, because then the one-sided log writes (e.g. COMMIT-BACKUP in Figure 4) would not persist across power failures. You could modify FaRM so that all log updates were written to SSD before returning, but then it would have much lower performance. An SSD write takes about 100 microseconds, while FaRM's one-sided RDMA writes to non-volatile RAM take only a few microseconds.

Q: I noticed that the non-volatility of this system relies on the fact that DRAM is being copied to SSD in the case of power outages. Why did they specify SSD and not allow copying to a normal hard disk? Were they just trying to use the fastest persistent storage device, and therefore did not talk about hard disks for simplicity鈥檚 sake?

A: Yes, they use SSDs because they are fast. They could have used hard drives without changing the design. However, it would then have taken longer to write the data to disk during a power outage, and that would require bigger batteries. Maybe the money they'd save by using hard drives would be outweighed by the increased cost of batteries.

Q: If FaRM is supposed to be for low-latency high-throughput storage, why bother with in-memory solutions instead of regular persistent storage (because the main trade-offs are exactly that -- presistent storage is low-latency high-throughput)?

A: RAM has dramatically higher throughput and lower latency than hard drives or SSDs. So if you want very high throughput or low latency, you need to manipulate data stored in RAM, and you have to avoid any use of disk or SSD in the common case. That's how FaRM works -- ordinarily it uses RAM for all data, and only uses SSDs to save RAM content if the power fails and restore it when power recovers.

Q: I'm confused about how RMDA in FaRM paper works or how this is crucial to its design. Is it correct that what the paper gives is a design optimized for CPU usage (since they mentioned their design is mostly CPU-bound)? So this would work even if we use SSD instead of in-memory access?

A: Previous systems have generally been limited by network or disk (or SSD). FaRM uses RDMA to eliminate (or drastically relax) the network bottleneck, and it uses battery-backed RAM to avoid having to use the disk and thus eliminate the disk/SSD as a bottleneck. But there's always *some* resource that prevents performance from being infinite. In FaRM's case it is CPU speed. If most FaRM accesses had to use SSD instead of memory, FaRM would run a lot slower, and would be SSD-limited.

Q: What is the distinction between primaries, backups, and configuration managers in FaRM? Why are there three roles?

A: The data is sharded among many primary/backup sets. The point of the backups is to store a copy of the shard in case the primary fails. The primary performs all reads and writes to data in the shard, while the backups perform only the writes (in order to keep their copy of the data identical to the primary's copy). There's just one configuration manager. It keeps track of which primaries and backups are alive, and keeps track of how the data is sharded among them. At a high level this arrangement is similar to GFS, which also sharded data among many primary/backup sets, and also had a master that kept track of where data is stored.

Q: Is FaRM useful or efficient over a range of scales? The authors describe a system that runs $12+/GB for a PETABYTE of storage, not to mention the overhead of 2000 systems and network infrastructure to support all that DRAM+SSD+UPS. This just seems silly-huge and expensive for what they modestly describe as "sufficient to hold the data sets of many interesting applications". How do you even test and verify something like that?

A: I think FaRM is only interesting if you need to support a huge number of transactions per second. If you only need a few thousand transactions per second, you can use off-the-shelf mature technology like MySQL. You could probably set up a considerably smaller FaRM system than the authors' 90-machine system. But FaRM doesn't make sense unless you are sharding and replicating data, which means you need at least four data servers (two shards, two servers per shard) plus a few machines for ZooKeeper (though probably you could run ZooKeeper on the four machines). Then maybe you have a system that costs on the order of $10,000 dollars and can execute a few million simple transactions per second, which is pretty good.

Q:I'm a little confused about the difference between the strict serializability and the fact that Farm does not ensure atomicity across reads ( even if committed transactions are serializable ).

A: Farm only guarantees serializability for transactions that commit. If a transaction sees the kind of inconsistency they are talking about, FaRM will abort the transaction. Applications must handle inconsistency in the sense that they should not crash, so that they can get as far as asking to commit, so that FaRM can abort them.

Q: By bypassing the Kernel, how does FaRM ensure that the read done by RDMA is consistent? What happens if read is done in the middle of a transaction?

A; There are two dangers here. First, for a big object, the reader may read the first half of the object before a concurrent transaction has written it, and the second half after the concurrent transaction has written it, and this might cause the reading program to crash. Second, the reading transaction can't be allowed to commit if it might not be serializable with a concurrent writeable transaction.

Based on my reading of the author's previous NSDI 2014 paper, the solution to the first problem is that every cache line of every object has a version number, and single-cache-line RDMA reads and writes are atomic. The reading transaction's FaRM library fetches all of the object's cache lines, and then checks whether they all have the same version number. If yes, the library gives the copy of the object to the application; if no, the library reads it again over RDMA. The second problem is solved by FaRM's validation scheme described in Section 4. In the VALIDATE step, if another transaction has written an object read by our transaction since our transaction started, our transaction will be aborted.

Q: How do truncations work? When can an a log entry be removed? If one entry is removed by a truncate call, are all previous entries also removed?

A: The TC tells the primaries and backups to delete the log entries for a transaction after the TC sees that all of them have a COMMIT-PRIMARY or COMMIT-BACKUP in their log. In order that recovery will know that a transaction is done despite truncation, page 62 mentions that primaries remember completed transaction IDs even after truncation. I don't think in general that truncation of one record implies truncation of all previous records. It may be the case, though, that each TC truncates its own transactions in order. Since each primary/backup has a separate log per TC, the result may be that for each log truncation occurs in log order.

Q: I'm confused when is it possible to abort in the COMMIT-BACKUP stage. Is this only due to hardware failures?

A: I believe so. If one of the backups doesn't respond, and the TC crashes, then there's a possibility that the transaction might be aborted during recovery.

Q: Since this is an optimistic protocol, does it suffer when demand for a small number of resources is high? It seems like this scheme would perform poorly if it had to effectively backtrack every time a transaction had to be aborted.

A: Yes, if there are many conflicting transactions, FaRM seems likely to perform badly due to aborts.

Q: Aren't locks a major hamstring to performance in this system?

A: If there were many conflicting transactions, then the Figure 4 commit protocol would probably cause lots of wasteful aborts. Specifically because transactions would encounter locks. On the other hand, for the applications the authors measure, FaRM gets fantastic performance despite the locks. Very likely one reason is that their applications have relatively few conflicting transactions, and thus not much contention for locks.

Q: Figure 7 shows significant decrease in performance when the number of operations crosses 120M . Is it because of the optimistic concurrency protocol, so after some threshold too many transactions get aborted?

A: I suspect the limit is that the servers can only process about 150 million operations per second. If clients send operations faster than that, some of them will have to wait; this waiting causes increased latency.

Q: If yes, then what are some ways a) to make the performance graph flatter? b) to avoid being in 120M+ range?

A: Is it up to the user to make sure that the server is not overloaded before sending a request? Usually there's a self-throttling effect with these systems -- if the system is slow (has high latency), the clients will generate requests slower. Either because each client generates a sequence of requests, so that if the first is delayed, that delays the client's submission of the next request. Or because there are ultimately users causing the requests to be generated, and the users get bored waiting for a slow system and go away. But nevertheless whoever deploys the system must be careful to ensure that the storage system (and all other parts of the system) are fast enough to execute likely workloads with reasonably low latency.
