# Zookeeper

[ZooKeeper: wait-free coordination for internet-scale systems](./zookeeper-wait-free-coordination-for-internet-scale-systems.pdf)
Patrick Hunt, Mahadev Konar, Flavio P. Junqueira, Benjamin Reed.
Proceedings of the 2010 USENIX Annual Technical Conference.

Why are we reading this paper?
  Widely-used replicated state machine service
    Inspired by Chubby (Google's global lock service)
    Originally at Yahoo!, now outside too (Mesos, HBase, etc.)
  Open source
    As Apache project (http://zookeeper.apache.org/)
  Case study of building replicated services, given a Paxos/ZAB/Raft library
    Similar issues show up in lab 3
  API supports a wide-range of use cases
    Application that need a fault-tolerant "master" don't need to roll their own
    Zookeeper is generic enough that they should be able to use Zookeeper
  High performance
    Unlike lab 3's replicate key/value service

Motivation: many applications in datacenter cluster need to coordinate
  Example: GFS
    master has list of chunk servers for each chunk
    master decides which chunk server is primary
    etc.
  Other examples: YMB, Crawler, etc.
    YMB needs master to shard topics
    Crawler needs master that commands page fetching
      (e.g., a bit like the master in mapreduce)
  Applications also need to find each other
    MapReduce needs to know IP:PORT of GFS master
    Load balancer needs to know where web servers are
  Coordination service typically used for this purpose

Motivation: performance -- lab3
  dominated by Raft
  consider a 3-node Raft
  before returning to client, Raft performs
    leader persists log entry
    in parallel, leader send message to followers
      each follower persist log entry
      each follower responds
  -> 2 disk writes and one round trip
    if magnetic disk: 2*10msec = 50 msg/sec
    if SSD: 2*2msec+1msec = 200 msg/sec
  Zookeeper performs 21,000 msg/sec
    asynchronous calls
    allows pipelining

Alternative plan: develop fault-tolerant master for each application
  announce location via DNS
  OK, if writing master isn't complicated
  But, master often needs to be:
    fault tolerant
      every application figures how to use Raft?
    high performance
      every application figures how to make "read" operations fast?
  DNS propagation is slow
    fail-over will take a long time!
  Some application settle for single-point of failure
    E.g., GFS and MapReduce
    Less desirable

Zookeeper: a generic coordination service
  Design challenges:
    What API?
    How to make master fault tolerant?
    How to get good performance?
  Challenges interact
    good performance may influence API
    e.g., asynchronous interface to allow pipelining

Zookeeper API overview
  [diagram: ZooKeeper, client sessions, ZAB layer]
  replicated state machine
    several servers implementing the service
    operations are performed in global order
      with some exceptions, if consistency isn't important
  the replicated objects are: znodes
    hierarchy of znodes
      named by pathnames
    znodes contain *metadata* of application
      configuration information
        machines that participate in the application
        which machine is the primary
      timestamps
      version number
    types of znodes:
      regular
      empheral
      sequential: name + seqno
        If n is the new znode and p is the parent znode, then the sequence
        value of n is never smaller than the value in the name of any other
        sequential znode ever created under p.

  sessions
    clients sign into zookeeper
    session allows a client to fail-over to another Zookeeper service
      client know the term and index of last completed operation (zxid)
      send it on each request
        service performs operation only if caught up with what client has seen
    sessions can timeout
      client must refresh a session continuously
        send a heartbeat to the server (like a lease)
      ZooKeeper considers client "dead" if doesn't hear from a client
      client may keep doing its thing (e.g., network partition)
        but cannot perform other ZooKeeper ops in that session
    no analogue to this in Raft + Lab 3 KV store

Operations on znodes
  create(path, data, flags)
  delete(path, version)
      if znode.version = version, then delete
  exists(path, watch)
  getData(path, watch)
  setData(path, data, version)
    if znode.version = version, then update
  getChildren(path, watch)
  sync()
   above operations are *asynchronous*
   all operations are FIFO-ordered per client
   sync waits until all preceding operations have been "propagated"

Check: can we just do this with lab 3's KV service?
  flawed plan: GFS master on startup does Put("gfs-master", my-ip:port)
    other applications + GFS nodes do Get("gfs-master")
  problem: what if two master candidates' Put()s race?
    later Put() wins
    each presumed master needs to read the key to see if it actually is the master
      when are we assured that no delayed Put() thrashes us?
      every other client must have seen our Put() -- hard to guarantee
  problem: when master fails, who decides to remove/update the KV store entry?
    need some kind of timeout
    so master must store tuple of (my-ip:port, timestamp)
      and continuously Put() to refresh the timestamp
      others poll the entry to see if the timestamp stops changing
  lots of polling + unclear race behavior -- complex
  ZooKeeper API has a better story: watches, sessions, atomic znode creation
    + only one creation can succeed -- no Put() race
    + sessions make timeouts easy -- no need to store and refresh explicit timestamps
    + watches are lazy notifications -- avoids commiting lots of polling reads

Ordering guarantees
  all write operations are totally ordered
    if a write is performed by ZooKeeper, later writes from other clients see it
    e.g., two clients create a znode, ZooKeeper performs them in some total order
  all operations are FIFO-ordered per client
  implications:
    a read observes the result of an earlier write from the same client
    a read observes some prefix of the writes, perhaps not including most recent write
      -> read can return stale data
    if a read observes some prefix of writes, a later read observes that prefix too

Example "ready" znode:
  A failure happens
  A primary sends a stream of writes into Zookeeper
    W1...Wn C(ready)
  The final write updates ready znode
    -> all preceding writes are visible
  The final write causes the watch to go off at backup
    backup issues R(ready) R1...Rn
    however, it will observe all writes because zookeeper will delay read until
      node has seen all txn that watch observed
  Lets say failure happens during R1 .. Rn, say after return Rj to client
    primary deletes ready file -> watch goes off
    watch alert is sent to client
    client knows it must issue new R(ready) R1 ...Rn
  Nice property: high performance
    pipeline writes and reads
    can read from *any* zookeeper node

Example usage 1: slow lock
  acquire lock:
   retry:
     r = create("app/lock", "", empheral)
     if r:
       return
     else:
       getData("app/lock", watch=True)

    watch_event:
       goto retry

  release lock: (voluntarily or session timeout)
    delete("app/lock")

Example usage 2: "ticket" locks
  acquire lock:
     n = create("app/lock/request-", "", empheral|sequential)
   retry:
     requests = getChildren(l, false)
     if n is lowest znode in requests:
       return
     p = "request-%d" % n - 1
     if exists(p, watch = True)
       goto retry

    watch_event:
       goto retry

  Q: can watch_even fire before lock it is the client's turn
  A: yes
     lock/request-10 <- current lock holder
     lock/request-11 <- next one
     lock/request-12 <- my request

     if client associated with request-11 dies before it gets the lock, the
     watch even will fire but it isn't my turn yet.

Using locks
  Not straight forward: a failure may cause your lock to be revoked
    client 1 acquires lock
      starts doing its stuff
      network partitions
      zookeeper declares client 1 dead (but it isn't)
    client 2 acquires lock, but client 1 still believes it has it
      can be avoided by setting timeouts correctly
      need to disconnect client 1 session before ephemeral nodes go away
      requires session heartbeats to be replicated to majority
        N.B.: paper doesn't discuss this
  For some cases, locks are a performance optimization
    for example, client 1 has a lock on crawling some urls
    client will do it 2 now, but that is fine
  For other cases, locks are a building block
    for example, application uses it to build transaction
    the transactions are all-or-nothing
    we will see an example in the Frangipani paper

Zookeeper simplifies building applications but is not an end-to-end solution
  Plenty of hard problems left for application to deal with
  Consider using Zookeeper in GFS
    I.e., replace master with Zookeeper
  Application/GFS still needs all the other parts of GFS
    the primary/backup plan for chunks
    version numbers on chunks
    protocol for handling primary fail over
    etc.
  With Zookeeper, at least master is fault tolerant
    And, won't run into split-brain problem
    Even though it has replicated servers

Implementation overview
  Similar to lab 3 (see last lecture)
  two layers:
    ZooKeeper services  (K/V service)
    ZAB layer (Raft layer)
  Start() to insert ops in bottom layer
  Some time later ops pop out of bottom layer on each replica
    These ops are committed in the order they pop out
    on apply channel in lab 3
    the abdeliver() upcall in ZAB

Challenge: Duplicates client requests
  Scenario
    Primary receives client request, fails
    Client resends client request to new primary
  Lab 3:
    Table to detect duplicates
    Limitation: one outstanding op per client
    Problem problem: cannot pipeline client requests
  Zookeeper:
    Some ops are idempotent period
    Some ops are easy to make idempotent
      test-version-and-then-do-op
      e.g., include timestamp and version in setDataTXN

Challenge: Read operations
  Many operations are read operations
    they don't modify replicated state
  Must they go through ZAB/Raft or not?
  Can any replica execute the read op?
  Performance is slow if read ops go through Raft

Problem: read may return stale data if only master performs it
  The primary may not know that it isn't the primary anymore
    a network partition causes another node to become primary
    that partition may have processed write operations
  If the old primary serves read operations, it won't have seen those write ops
   => read returns stale data

Zookeeper solution: don't promise non-stale data (by default)
  Reads are allowed to return stale data
    Reads can be executed by any replica
    Read throughput increases as number of servers increases
    Read returns the last zxid it has seen
     So that new primary can catch up to zxid before serving the read
     Avoids reading from past
  Only sync-read() guarantees data is not stale

Sync optimization: avoid ZAB layer for sync-read
  must ensure that read observes last committed txn
  leader puts sync in queue between it and replica
    if ops ahead of in the queue commit, then leader must be leader
    otherwise, issue null transaction
  in same spirit read optimization in Raft paper
    see last par section 8 of raft paper

Performance (see table 1)
  Reads inexpensive
    Q: Why more reads as servers increase?
  Writes expensive
    Q: Why slower with increasing number of servers?
  Quick failure recovery (figure 8)
    Decent throughout even while failures happen

References:
  ZAB: http://dl.acm.org/citation.cfm?id=2056409
  https://zookeeper.apache.org/
  https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf  (wait free, universal
  objects, etc.)

### ZooKeeper FAQ

Q: Why are only update requests A-linearizable?

A: Because the authors want reads to scale with the number of servers,
so they want to them to execute a server without requiring any
interaction with other servers. This comes at the cost of
consistency: they are allowed to return stale data.

Q: How does linearizability differ from serializability?

A: Serializability is a correctness condition that is typically used for
systems that provide transactions; that is, systems that support
grouping multiple operations into an atomic operations.

Linearizability is typically used for systems without transactions.
When the Zookeeper paper refers to "serializable" in their definition
of linearizability, they just mean a serial order.

We talk about serializability in subsequent papers. Here is a blog
post that explains the difference, if you are curious:
http://www.bailis.org/blog/linearizability-versus-serializability/

Although the blog post gives precise definitions, designers are not
that precise when using those terms when they describe their
system, so often you have to glean from the context what
correctness condition the designers are shooting for.

Q: What is pipelining?

Zookeeper "pipelines" the operations in the client API (create,
delete, exists, etc). What pipelining means here is that these
operations are executed asynchronously by clients. The client calls
create, delete, sends the operation to Zookeeper and then returns.
At some point later, Zookeeper invokes a callback on the client that
tells the client that the operation has been completed and what the
results are. This asynchronous interface allow a client to pipeline
many operations: after it issues the first, it can immediately issue a
second one, and so on. This allows the client to achieve high
throughput; it doesn't have to wait for each operation to complete
before starting a second one.

A worry with pipelining is that operations that are in flight might be
re-ordered, which would cause the problem that the authors to talk
about in 2.3. If a the leader has many write operations in flight
followed by write to ready, you don't want those operations to be
re-ordered, because then other clients may observe ready before the
preceding writes have been applied. To ensure that this cannot
happen, Zookeeper guarantees FIFO for client operations; that is the
client operations are applied in the order they have been issued.

Q: What about Zookeeper's use case makes wait-free better than locking?

I think you mean "blocking" -- locking (as in, using locks) and blocking (as
in, waiting for a request to return before issuing another one) are very
different concepts.

Many RPC APIs are blocking: consider, for example, clients in Lab 2/3 -- they
only ever issue one request, patiently wait for it to either return or time out,
and only then send the next one. This makes for an easy API to use and reason
about, but doesn't offer great performance. For example, imagine you wanted to
change 1,000 keys in Zookeeper -- by doing it one at a time, you'll spend most
of your time waiting for the network to transmit requests and responses (this is
what labs 2/3 do!). If you could instead have *several* requests in flight at
the same time, you can amortize some of this cost. Zookeeper's wait-free API
makes this possible, and allows for higher performance -- a key goal of the
authors' use case.

Q: What does wait-free mean?

A: The precise definition is as follows: A wait-free implementation of
a concurrent data object is one that guarantees that any process can
complete any operation in a finite number of steps, regardless of the
execution speeds of the other processes. This definition was
introduced in the following paper by Herlihy:
https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf

The implementation of Zookeeper API is wait-free because requests return
to clients without waiting for other, slow clients or servers. Specifically,
when a write is processed, the server handling the client write returns as
soon as it receives the state change (搂4.2). Likewise, clients' watches fire
as a znode is modified, and the server does *not* wait for the clients to
acknowledge that they've received the notification before returning to the
writing client.

Some of the primitives that client can implement with Zookeeper APIs are
wait-free too (e.g., group membership), but others are not (e.g., locks,
barrier).

Q: What is the reason for implementing 'fuzzy snapshots'? How can
state changes be idempotent?

A: If the authors had to decided to go for consistent snapshots,
Zookeeper would have to stop all writes will making a snapshot for
the in-memory database. You might remember that GFS went for
this plan, but for large database, this could hurt the performance of
Zookeeper seriously. Instead the authors go for a fuzzy snapshot
scheme that doesn't require blocking all writes while the snapshot is
made. After reboot, they construct a consistent snapshot by
replaying the messages that were sent after the checkpoint started.
Because all updates in Zookeeper are idempotent and delivered in
the same order, the application-state will be correct after reboot and
replay---some messages may be applied twice (once to the state
before recovery and once after recovery) but that is ok, because they
are idempotent. The replay fixes the fuzzy snapshot to be a consistent
snapshot of the application state.

Zookeeper turns the operations in the client API into something that
it calls a transaction in a way that the transaction is idempotent. For
example, if a client issues a conditional setData and the version
number in the request matches, Zookeeper creates a setDataTXN
that contains the new data, the new version number, and updated
time stamps. This transaction (TXN) is idempotent: Zookeeper can
execute it twice and it will result in the same state.

Q: How does ZooKeeper choose leaders?

A: Zookeeper uses ZAB, an atomic broadcast system, which has leader
election build in, much like Raft. If you are curious about the
details, you can find a paper about ZAB here:
http://dl.acm.org/citation.cfm?id=2056409

Q: How does Zookeeper's performance compare to other systems
such as Paxos?

A: It has impressive performance (in particular throughput); Zookeeper
would beat the pants of your implementation of Raft. 3 zookeeper
server process 21,000 writes per second. Your raft with 3 servers
commits on the order of tens of operations per second (assuming a
magnetic disk for storage) and maybe hundreds per second with
SSDs.

Q: How does the ordering guarantee solve the race conditions in
Section 2.3?

If a client issues many write operations to a z-node, and then the
write to Ready, then Zookeeper will guarantee that all the writes
will be applied to the z-node before the write to Ready. Thus, if
another client observes Ready, then the all preceding writes must
have been applied and thus it is ok for other clients to read the info
in the z-node.

Q: How big is the ZooKeeper database? It seems like the server must
have a lot of memory.

It depends on the application, and, unfortunately, the paper doesn't
report on it for the different application they have used Zookeeper
with. Since Zookeeper is intended for configuration services/master
services (and not for a general-purpose data store), however, an
in-memory database seems reasonable. For example, you could
imagine using Zookeeper for GFS's master and that amount of data
should fit in the memory of a well-equipped server, as it did for GFS.

Q: What's a universal object?

A: It is a theoretical statement of how good the API of Zookeeper is
based on a theory of concurrent objects that Herlihy introduced:
https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf. We won't
spend any time on this statement and theory, but if you care there is
a gentle introduction on this wikipedia page:
https://en.wikipedia.org/wiki/Non-blocking_algorithm.

The reason that authors appeal to this concurrent-object theory is
that they claim that Zookeeper provides a good coordination kernel
that enables new primitives without changing the service. By
pointing out that Zookeeper is an universal object in this
concurrent-object theory, they support this claim.

Q: How does a client know when to leave a barrier (top of page 7)?

A: Leaving the barrier involves each client watching the znodes for
all other clients participating in the barrier. Each client waits for
all of them to be gone. If they are all gone, they leave the barrier
and continue computing.

Q: Is it possible to add more servers into an existing ZooKeeper without taking the service down for a period of time?

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
