6.824 2020 Lecture 8: Zookeeper Case Study

Reading: "ZooKeeper: wait-free coordination for internet-scale systems", Patrick
Hunt, Mahadev Konar, Flavio P. Junqueira, Benjamin Reed.  Proceedings of the 2010
USENIX Annual Technical Conference.

What questions does this paper shed light on?
  * Can we have coordination as a stand-alone general-purpose service?
    What should the API look like?
    How can other distributed applications use it?
  * We paid lots of money for Nx replica servers.
    Can we get Nx performance from them?

First, performance.
  For now, view ZooKeeper as some service replicated with a Raft-like scheme.
  Much like Lab 3.
  [clients, leader/state/log, followers/state/log]

Does this replication arrangement get faster as we add more servers?
  Assume a busy system, lots of active clients.
  Writes probably get slower with more replicas!
    Since leader must send each write to growing # of followers.
  What about reads?

Q: Can replicas serve read-only client requests form their local state?
   Without involving the leader or other replicas?
   Then total read capacity would be O(# servers), not O(1)!

Q: Would reads from followers be linearizable?
   Would reads always yield fresh data?
   No:
     Replica may not be in majority, so may not have seen a completed write.
     Replica may not yet have seen a commit for a completed write.
     Replica may be entirely cut off from the leader (same as above).
   Linearizability forbids stale reads!

Q: What if a client reads from an up-to-date replica, then a lagging replica?
   It may see data values go *backwards* in time! Also forbidden.

Raft and Lab 3 avoid these problems.
  Clients have to send reads to the leader.
  So Lab 3 reads are linearizable.
  But no opportunity to divide the read load over the followers.
  
How does ZooKeeper skin this cat?
  By changing the definition of correctness!
  It allows reads to yield stale data.
  But otherwise preserves order.

Ordering guarantees (Section 2.3)
  * Linearizable writes
    clients send writes to the leader
    the leader chooses an order, numbered by "zxid"
    sends to replicas, which all execute in zxid order
    this is just like the labs
  * FIFO client order
    each client specifies an order for its operations (reads AND writes)
    writes:
      writes appear in the write order in client-specified order
      this is the business about the "ready" file in 2.3
    reads:
      each read executes at a particular point in the write order
      a client's successive reads execute at non-decreasing points in the order
      a client's read executes after all previous writes by that client
        a server may block a client's read to wait for previous write, or sync()

Why does this make sense?
  I.e. why OK for reads to return stale data?
       why OK for client 1 to see new data, then client 2 sees older data?

At a high level:
  not as painful for programmers as it may seem
  very helpful for read performance!

Why is ZooKeeper useful despite loose consistency?
  sync() causes subsequent client reads to see preceding writes.
    useful when a read must see latest data
  Writes are well-behaved, e.g. exclusive test-and-set operations
    writes really do execute in order, on latest data.
  Read order rules ensure "read your own writes".
  Read order rules help reasoning.
    e.g. if read sees "ready" file, subsequent reads see previous writes.
         (Section 2.3)
         Write order:      Read order:
         delete("ready")
         write f1
         write f2
         create("ready")
                           exists("ready")
                           read f1
                           read f2
         even if client switches servers!
    e.g. watch triggered by a write delivered before reads from subsequent writes.
         Write order:      Read order:
                           exists("ready", watch=true)
                           read f1
         delete("ready")
         write f1
         write f2
                           read f2

A few consequences:
  Leader must preserve client write order across leader failure.
  Replicas must enforce "a client's reads never go backwards in zxid order"
    despite replica failure.
  Client must track highest zxid it has read
    to help ensure next read doesn't go backwards
    even if sent to a different replica

Other performance tricks in ZooKeeper:
  Clients can send async writes to leader (async = don't have to wait).
  Leader batches up many requests to reduce net and disk-write overhead.
    Assumes lots of active clients.
  Fuzzy snapshots (and idempotent updates) so snapshot doesn't stop writes.

Is the resulting performance good?
  Table 1
  High read throughput -- and goes up with number of servers!
  Lower write throughput -- and goes down with number of servers!
  21,000 writes/second is pretty good!
    Maybe limited by time to persist log to hard drives.
    But still MUCH higher than 10 milliseconds per disk write -- batching.

The other big ZooKeeper topic: a general-purpose coordination service.
  This is about the API and how it can help distributed s/w coordinate.
  It is not clear what such an API should look like!

What do we mean by coordination as a service?
  Example: VMware-FT's test-and-set server
    If one replica can't talk to the other, grabs t-a-s lock, becomes sole server
    Must be exclusive to avoid two primaries (e.g. if network partition)
    Must be fault-tolerant
  Example: GFS (more speculative)
    Perhaps agreement on which meta-data replica should be master
    Perhaps recording list of chunk servers, which chunks, who is primary
  Other examples: MapReduce, YMB, Crawler, etc.
    Who is the master; lists of workers; division of labor; status of tasks
  A general-purpose service would save much effort!

Could we use a Lab 3 key/value store as a generic coordination service?
  For example, to choose new GFS master if multiple replicas want to take over?
  perhaps
    Put("master", my IP address)
    if Get("master") == my IP address:
      act as master
  problem: a racing Put() may execute after the Get()
    2nd Put() overwrites first, so two masters, oops
    Put() and Get() are not a good API for mutual exclusion!
  problem: what to do if master fails?
    perhaps master repeatedly Put()s a fresh timestamp?
    lots of polling...
  problem: clients need to know when master changes
    periodic Get()s?
    lots of polling...

Zookeeper API overview (Figure 1)
  the state: a file-system-like tree of znodes
  file names, file content, directories, path names
  typical use: configuration info in znodes
    set of machines that participate in the application
    which machine is the primary
  each znode has a version number
  types of znodes:
    regular
    ephemeral
    sequential: name + seqno

Operations on znodes (Section 2.2)
  create(path, data, flags)
    exclusive -- only first create indicates success
  delete(path, version)
    if znode.version = version, then delete
  exists(path, watch)
    watch=true means also send notification if path is later created/deleted
  getData(path, watch)
  setData(path, data, version)
    if znode.version = version, then update
  getChildren(path, watch)
  sync()
    sync then read ensures writes before sync are visible to same client's read
    client could instead submit a write

ZooKeeper API well tuned to synchronization:
  + exclusive file creation; exactly one concurrent create returns success
  + getData()/setData(x, version) supports mini-transactions
  + sessions automate actions when clients fail (e.g. release lock on failure)
  + sequential files create order among multiple clients
  + watches -- avoid polling

Example: add one to a number stored in a ZooKeeper znode
  what if the read returns stale data?
    write will write the wrong value!
  what if another client concurrently updates?
    will one of the increments be lost?
  while true:
    x, v := getData("f")
    if setData(x + 1, version=v):
      break
  this is a "mini-transaction"
    effect is atomic read-modify-write
  lots of variants, e.g. test-and-set for VMware-FT

Example: Simple Locks (Section 2.4)
  acquire():
    while true:
      if create("lf", ephemeral=true), success
      if exists("lf", watch=true)
        wait for notification

  release(): (voluntarily or session timeout)
    delete("lf")

  Q: what if lock released just as loser calls exists()?

Example: Locks without Herd Effect
  (look at pseudo-code in paper, Section 2.4, page 6)
  1. create a "sequential" file
  2. list files
  3. if no lower-numbered, lock is acquired!
  4. if exists(next-lower-numbered, watch=true)
  5.   wait for event...
  6. goto 2

  Q: could a lower-numbered file be created between steps 2 and 3?
  Q: can watch fire before it is the client's turn?
  A: yes
     lock-10 <- current lock holder
     lock-11 <- next one
     lock-12 <- my request

     if client that created lock-11 dies before it gets the lock, the
     watch will fire but it isn't my turn yet.

Using these locks
  Different from single-machine thread locks!
    If lock holder fails, system automatically releases locks.
    So locks are not really enforcing atomicity of other activities.
    To make writes atomic, use "ready" trick or mini-transactions.
  Useful for master/leader election.
    New leader must inspect state and clean up.
  Or soft locks, for performance but not correctness
    e.g. only one worker does each Map or Reduce task (but OK if done twice)
    e.g. a URL crawled by only one worker (but OK if done twice)

ZooKeeper is a successful design.
  see ZooKeeper's Wikipedia page for a list of projects that use it
  Rarely eliminates all the complexity from distribution.
    e.g. GFS master still needs to replicate file meta-data.
    e.g. GFS primary has its own plan for replicating chunks.
  But does bite off a bunch of common cases:
    Master election.
    Persistent master state (if state is small).
    Who is the current master? (name service).
    Worker registration.
    Work queues.
  
Topics not covered:
  persistence
  details of batching and pipelining for performance
  fuzzy snapshots
  idempotent operations
  duplicate client request detection

References:
  https://zookeeper.apache.org/doc/r3.4.8/api/org/apache/zookeeper/ZooKeeper.html
  ZAB: http://dl.acm.org/citation.cfm?id=2056409
  https://zookeeper.apache.org/
  https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf  (wait free, universal
  objects, etc.)

ZooKeeper FAQ

Q: Why are only update requests A-linearizable? Why not reads as well?

A: The authors want high total read throughput, so they want replicas
to be able to satisfy client reads without involving the leader. A
given replica may not know about a commited write (if it's not in the
majority that the leader waited for), or may know about a write but
not yet know if it is committed. Thus a replica's state may lag behind
the leader and other replicas. Thus serving reads from replicas can
yield data that doesn't reflect recent writes -- that is, reads can
return stale results.

Q: How does linearizability differ from serializability?

A: The usual definition of serializability is much like
linearizability, but without the requirement that operations respect
real-time ordering. Have a look at this explanation:
http://www.bailis.org/blog/linearizability-versus-serializability/

Section 2.3 of the ZooKeeper paper uses "serializable" to indicate
that the system behaves as if writes (from all clients combined) were
executed one by one in some order. The "FIFO client order" property
means that reads occur at specific points in the order of writes, and
that a given client's successive reads never move backwards in that
order. One thing that's going on here is that the guarantees for
writes and reads are different.

Q: What is pipelining?

There are two things going on here. First, the ZooKeeper leader
(really the leader's Zab layer) batches together multiple client
operations in order to send them efficiently over the network, and in
order to efficiently write them to disk. For both network and disk,
it's often far more efficient to send a batch of N small items all at
once than it is to send or write them one at a time. This kind of
batching is only effective if the leader sees many client requests at
the same time; so it depends on there being lots of active clients.

The second aspect of pipelining is that ZooKeeper makes it easy for
each client to keep many write requests outstanding at a time, by
supporting asynchronous operations. From the client's point of view,
it can send lots of write requests without having to wait for the
responses (which arrive later, as notifications after the writes
commit). From the leader's point of view, that client behavior gives
the leader lots of requests to accumulate into big efficient batches.

A worry with pipelining is that operations that are in flight might be
re-ordered, which would cause the problem that the authors to talk
about in 2.3. If a the leader has many write operations in flight
followed by write to ready, you don't want those operations to be
re-ordered, because then other clients may observe ready before the
preceding writes have been applied. To ensure that this cannot
happen, Zookeeper guarantees FIFO for client operations; that is the
client operations are applied in the order they have been issued.

Q: What does wait-free mean?

A: The precise definition: A wait-free implementation of a concurrent
data object is one that guarantees that any process can complete any
operation in a finite number of steps, regardless of the execution
speeds of the other processes. This definition was introduced in the
following paper by Herlihy:
https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf

Zookeeper is wait-free because it processes one client's requests
without needing to wait for other clients to take action. This is
partially a consequence of the API: despite being designed to support
client/client coordination and synchronization, no ZooKeeper API call
is defined in a way that would require one client to wait for another.
In contrast, a system that supported a lock acquire operation that
waited for the current lock holder to release the lock would not be
wait-free.

Ultimately, however, ZooKeeper clients often need to wait for each
other, and ZooKeeper does provide a waiting mechanism -- watches. The
main effect of wait-freedom on the API is that watches are factored
out from other operations. The combination of atomic test-and-set
updates (e.g. file creation and writes condition on version) with
watches allows clients to synthesize more complex blocking
abstractions (e.g. Section 2.4's locks and barriers).

Q: How does the leader know the order in which a client wants a bunch
of asynchronous updates to be performed?

A: The paper doesn't say. The answer is likely to involve the client
numbering its asynchronous requests, and the leader tracking for each
client (really session) what number it should next expect. The leader
has to keep state per session anyway (for client session timeouts), so
it might be little extra work to track a per-session request sequence
number. This information would have to be preserved when a leader
fails and another server takes over, so the client sequence numbers
are likely passed along in replicated log entries.

Q: What does a client do if it doesn't get a reply for a request? Does
it re-send, in case the network lost a request or reply, or the leader
crashed before committing? How does ZooKeeper avoid re-sends leading
to duplicate executions?

A: The paper doesn't say how all this works. Probably the leader
tracks what request numbers from each session it has received and
committed, so that it can filter out duplicate requests. Lab 3 has a
similar arrangement.

Q: If a client submits an asynchronous write, and immediately
afterwards does a read, will the read see the effect of the write?

A: The paper doesn't explicitly say, but the implication of the "FIFO
client order" property of Section 2.3 is that the read will see the
write. That seems to imply that a server may block a read until the
server has received (from the leader) all of the client's preceding
writes. ZooKeeper probably manages this by having the client send, in
its read request, the zxid of the latest preceding operation that the
client submitted.

Q: What is the reason for implementing 'fuzzy snapshots'?

A: A precise snapshot would correspond to a specific point in the log:
the snapshot would include every write before that point, and no
writes after that point; and it would be clear exactly where to start
replay of log entries after a reboot to bring the snapshot up to date.
However, creation of a precise snapshot requires a way to prevent any
writes from happening while the snapshot is being created and written
to disk. Blocking writes for the duration of snapshot creation would
decrease performance a lot.

The point of ZooKeeper's fuzzy snapshots is that ZooKeeper creates the
snapshot from its in-memory database while allowing writes to the
database. This means that a snapshot does not correspond to a
particular point in the log -- a snapshot includes a more or less
random subset of the writes that were concurrent with snapshot
creation. After reboot, ZooKeeper constructs a consistent snapshot by
replaying all log entries from the point at which the snapshot
started. Because updates in Zookeeper are idempotent and delivered in
the same order, the application-state will be correct after reboot and
replay---some messages may be applied twice (once to the state before
recovery and once after recovery) but that is ok, because they are
idempotent. The replay fixes the fuzzy snapshot to be a consistent
snapshot of the application state.

The Zookeeper leader turns the operations in the client API into
idempotent transactions. For example, if a client issues a conditional
setData and the version number in the request matches, the Zookeeper
leader creates a setDataTXN that contains the new data, the new
version number, and updated time stamps. This transaction (TXN) is
idempotent: Zookeeper can execute it twice and it will result in the
same state.

Q: How does ZooKeeper choose leaders?

A: Zookeeper uses ZAB, an atomic broadcast system, which has leader
election built in, much like Raft. Here's a paper about Zab:
http://dl.acm.org/citation.cfm?id=2056409

Q: How does Zookeeper's performance compare to other systems
such as Paxos?

A: It has impressive performance (in particular throughput); Zookeeper
would beat the pants of your implementation of Raft. 3 zookeeper
servers process 21,000 writes per second. Your raft with 3 servers
commits on the order of tens of operations per second (assuming a
magnetic disk for storage) and maybe hundreds per second with
SSDs.

Q: How does the ordering guarantee solve the race conditions in
Section 2.3?

If a client issues many write operations to various z-nodes, and then the
write to Ready, then Zookeeper will guarantee that all the writes
will be applied to the z-nodes before the write to Ready. Thus, if
another client observes Ready, then all the preceding writes must
have been applied and thus it is ok for the client to read the info
in the z-nodes.

Q: How big is the ZooKeeper database? It seems like the server must
have a lot of memory.

It depends on the application, and, unfortunately, the paper doesn't
report the authors' experience in this area. Since Zookeeper is
intended for configuration and coordination, and not as a
general-purpose data store, an in-memory database seems reasonable.
For example, you could imagine using Zookeeper for GFS's master and
that amount of data should fit in the memory of a well-equipped
server, as it did for GFS.

Q: What's a universal object?

A: It is a theoretical statement of how good the API of Zookeeper is
based on a theory of concurrent objects that Herlihy introduced:
https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf. We won't
spend any time on this statement and theory, but if you care there is
a gentle introduction on this wikipedia page:
https://en.wikipedia.org/wiki/Non-blocking_algorithm.

The authors appeal to this concurrent-object theory in order to show
that Zookeeper's API is general-purpose: that the API includes enough
features to implement any coordination scheme you'd want.

Q: How does a client know when to leave a barrier (top of page 7)?

A: Leaving the barrier involves each client watching the znodes for
all other clients participating in the barrier. Each client waits for
all of them to be gone. If they are all gone, they leave the barrier
and continue computing.

Q: Is it possible to add more servers into an existing ZooKeeper
without taking the service down for a period of time?

It is -- although when the original paper was published, cluster
membership was static. Nowadays, ZooKeeper supports "dynamic
reconfiguration":

https://zookeeper.apache.org/doc/r3.5.3-beta/zookeeperReconfig.html

... and there is actually a paper describing the mechanism:

https://www.usenix.org/system/files/conference/atc12/atc12-final74.pdf

How do you think this compares to Raft's dynamic configuration change
via overlapping consensus, which appeared two years later?

Q: How are watches implemented in the client library?

It depends on the implementation. In most cases, the client library
probably registers a callback function that will be invoked when the
watch triggers.

For example, a Go client for ZooKeeper implements it by passing a
channel into "GetW()" (get with watch); when the watch triggers, an
"Event" structure is sent through the channel. The application can
check the channel in a select clause.

See https://godoc.org/github.com/samuel/go-zookeeper/zk#Conn.GetW.