# [Raft (extended) (2014)](in-search-of-an-understandable-consensus-algorithm-extended-version.pdf)

## why are we reading this paper?

- distributed consensus is a hard problem that people have worked on for decades

## this lecture

- today: Raft elections and log handling (Lab 2A, 2B)
- next: Raft persistence, client behavior, snapshots (Lab 2C, Lab 3)

## overall topic: fault-tolerant services using replicated state machines (RSM)

- [clients, replica servers]
- example: configuration server, like MapReduce or GFS master
- example: key/value storage server, put()/get() (lab3)
- goal: same client-visible behavior as single non-replicated server
  - but available despite some number of failed servers
- strategy:
  - each replica server executes same commands in same order
  - so they remain replicas (i.e., identical) as they execute
  - so if one fails, others can continue
  - i.e. on failure, client switches to another server
- both GFS and VMware FT have this flavor

## a critical question: how to avoid split brain?

- suppose client can contact replica A, but not replica B
- can client proceed with just replica A?
- if B has really crashed, client *must* proceed without B,
  - otherwise the service can't tolerate faults!
- if B is up but network prevents client from contacting it,
  - maybe client should *not* proceed without it,
  - since it might be alive and serving other clients -- risking split brain

### example of why split brain cannot be allowed:

- fault-tolerant key/value database
- C1 and C2 are in different network partitions and talk to different servers
- C1: put("k1", "v1")
- C2: put("k1", "v2")
- C1: get("k1") -> ???
- correct answer is "v2", since that's what a non-replicated server would yield
- but if two servers are independently serving C1 and C2 due to partition,
  - C1's get could yield "v1"

### problem: computers cannot distinguish crashed machines vs a partitioned network

- both manifest as being unable to communicate with one or more machines

## We want a state-machine replication scheme that meets three goals:

- 1. remains available despite any one (fail-stop) failure
- 2. handles partition w/o split brain
- 3. if too many failures: waits for repair, then resumes

### The big insight for coping w/ partition: majority vote

- 2f+1 servers to tolerate f failures, e.g. 3 servers can tolerate 1 failure
- must get majority (f+1) of servers ti agree to make progress
  - failure of f servers leaves a majority of f+1, which can proceed
- why does majority help avoid split brain?
  - at most one partition can have a majority
- note: majority is out of all 2f+1 servers, not just out of live ones
- the really useful thing about majorities is that any two must intersect
  - servers in the intersection will only vote one way or the other
  - and the intersection can convey information about previous decisions

### Two partition-tolerant replication schemes were invented around 1990

- Paxos and View-Stamped Replication
- in the last 10 years this technology has seen a lot of real-world use
- the Raft paper is a good introduction to modern techniques

## *** topic: Raft overview

state machine replication with Raft -- Lab 3 as example:
- [diagram: clients, 3 replicas, k/v layer, raft layer, logs]
- server's Raft layers elect a leader
- clients send RPCs to k/v layer in leader
  - Put, Get, Append
- k/v layer forwards request to Raft layer, doesn't respond to client yet
- leader's Raft layer sends each client command to all replicas
  - via AppendEntries RPCs
  - each follower appends to its local log (but doesn't commit yet)
  - and responds to the leader to acknowledge
- entry becomes "committed" at the leader if a majority put it in their logs
  - guaranteed it won't be forgotten
  - majority -> will be seen by the next leader's vote requests for sure
- servers apply operation to k/v state machine once leader says it's committed
  - they find out about this via the next AppendEntries RPC (via commitIndex)
- leader responds to k/v layer after it has committed
  - k/v layer applies Put to DB, or fetches Get result
- then leader replies to client w/ execution result

why the logs?
- the service keeps the state machine state, e.g. key/value DB
  - why isn't that enough?
- it's important to number the commands
  - to help replicas agree on a single execution order
  - to help the leader ensure followers have identical logs
- replicas also use the log to store commands
  - until the leader commits them
  - so the leader can re-send if a follower misses some
  - for persistence and replay after a reboot (next time)

there are two main parts to Raft's design:
- electing a new leader
- ensuring identical logs despite failures

*** topic: leader election (Lab 2A)

why a leader?
- ensures all replicas execute the same commands, in the same order

Raft numbers the sequence of leaders using "terms"
- new leader -> new term
- a term has at most one leader; might have no leader
- each election is also associated with one particular term
  - and there can only be one successful election per term
- the term numbering helps servers follow latest leader, not superseded leader

when does Raft start a leader election?
- AppendEntries are implied heartbeats; plus leader sends them periodically
- if other server(s) don't hear from current leader for an "election timeout"
  - they assume the leader is down and start an election
- [state transition diagram, Figure 4: follower, candidate, leader]
- followers increment local currentTerm, become candidates, start election
- note: this can lead to un-needed elections; that's slow but safe
- note: old leader may still be alive and think it is the leader

what happens when a server becomes candidate?
- three possibilities:
  - 1) gets majority, converts to leader
    - locally observes and counts votes
    - note: not resilient to byzantine faults!
  - 2) fails to get a majority, hears from another leader who did
    - via incoming AppendEntries RPC
    - defers to the new leader's authority, and becomes follower
  - 3) fails to get a majority, but doesn't hear from a new leader
    - e.g., if in minority network partition
    - times out and starts another election (remains candidate)
- note: in case (3), it's possible to keep incrementing the term
  - but cannot add log entries, since in minority and not the leader
  - once partition heals, an election ensues because of the higher term
  - but: either logs in majority partition are longer (so high-term
    - candidate gets rejected) or they are same length if nothing happened
    - in the majority partition (so high-term candidate can win, but no damage)

how to ensure at most one leader in a term?
- (Figure 2 RequestVote RPC and Rules for Servers)
- leader must get "yes" votes from a majority of servers
- each server can cast only one vote per term
- at most one server can get majority of votes for a given term
  - -> at most one leader even if network partition
  - -> election can succeed even if some servers have failed

how does a new leader establish itself?
- winner gets yes votes from majority
- immediately sends AppendEntries RPC (heart-beats) to everybody
  - the new leader's heart-beats suppress any new election

an election may not succeed for two reasons:
- * less than a majority of servers are reachable
- * simultaneous candidates split the vote, none gets majority

what happens if an election doesn't succeed?
- another timeout (no heartbeat), another election
- higher term takes precedence, candidates for older terms quit

how to set the election timeout?
- each server picks a random election timeout
  - helps avoid split votes
- randomness breaks symmetry among the servers
  - one will choose lowest random delay
  - avoids everybody starting elections at the same time, voting for themselves
- hopefully enough time to elect before next timeout expires
- others will see new leader's AppendEntries heartbeats and
  - not become candidates
- what value?
  - at least a few heartbeat intervals (network can drop or delay a heartbeat)
  - random part long enough to let one candidate succeed before next starts
  - short enough to allow a few re-tries before tester gets upset
    - tester requires election to complete in 5 seconds or less


*** topic: the Raft log (Lab 2B)

we talked about how the leader replicates log entries
- important distinction: replicated vs. committed entries
  - committed entries are guaranteed to never go away
  - replicated, but uncommitted entries may be overwritten!
- helps to think of an explicit "commit frontier" at each participant

will the servers' logs always be exact replicas of each other?
- no: some replicas may lag
- no: we'll see that they can temporarily have different entries
- the good news:
  - they'll eventually converge
  - the commit mechanism ensures servers only execute stable entries

extra criterion: leader cannot simply replicate and commit old term's entries
- [Figure 8 example]
- S1 fails to replicate term 2 entry to majority, then fails
- S5 becomes leader in term 3, adds entry, but fails to replicate it
- S1 comes back, becomes leader again
  - works on replicating old entry from term 2 to force followers to adopt its log
  - are we allowed to commit once the term 2 entry is on a majority of servers?
- Turns out the answer is no! Consider what would happen if we did:
  - term 2 entry replicated to S3
  - S1 commits it, since it's on a majority of servers
  - S1 then fails again
  - S5 gets elected for term 4, since it has a term 3 entry at the end of the log
  - everybody with a term 2 entry at the end of the log votes for S5
- S5 becomes leader, and now forces *its* log (with the term 3 entry) on others
  - term 2 entry at index 2 will get overwritten by term 3 entry
  - but it's supposed to be committed!
  - therefore, contradicts Leader Completeness property
- solution: wait until S1 has also replicated and committed a term 4 entry
  - ensures that S5 can no longer be elected if S1 fails
  - thus it's now okay to commit term 2 entry as well

when is it legal for Raft to overwrite log entries? (cf. Figure 7 question)
- they must be uncommitted
- may truncate and overwrite a much longer log
  - Figure 7 (f) is a case in point
- e.g., a leader adds many entries to its log, but fails to replicate them
  - perhaps it's in a network partition
- other leaders in later terms add entries at the same indices (Fig 7 (a)-(e))
  - and commit at least some of them
  - now cannot change this log index any more
- outdated server receives AppendEntries, overwrites uncommitted log entries
  - even if the log is much longer than the current leader's!
- this is okay, because the leader only responds to clients after entries commit
  - so leader who produced the overwritten entries in (f) cannot have done so

Raft FAQ

Q: Does Raft sacrifice anything for simplicity?

A: Raft gives up some performance in return for clarity; for example:

* Every operation must be written to disk for persistence; performance
  probably requires batching many operations into each disk write.

* There can only usefully be a single AppendEntries in flight from the
  leader to each follower: followers reject out-of-order
  AppendEntries, and the sender's nextIndex[] mechanism requires
  one-at-a-time. A provision for pipelining many AppendEntries would
  be better.

* The snapshotting design is only practical for relatively small
  states, since it writes the entire state to disk. If the state is
  big (e.g. if it's a big database), you'd want a way to write just
  parts of the state that have changed recently.

* Similarly, bringing recovering replicas up to date by sending them a
  complete snapshot will be slow, needlessly so if the replica already
  has an old snapshot.

* Servers may not be able to take much advantage of multi-core because
  operations must be executed one at a time (in log order).

These could be fixed by modifying Raft, but the result might have less
value as a tutorial introduction.

Q: Is raft used in real-world software, or do companies generally roll
their own flavor of Paxos (or use a different consensus protocol)?

A: Raft is pretty new, so there hasn't been much time for people to design
new systems based on it. Most of the state-machine replication systems I
hear about are based on the older Multi-Paxos and Viewstamped
Replication protocols.

Q: What is Paxos? In what sense is Raft simpler?

A: There is a protocol called Paxos that allows a set of servers to
agree on a single value. While Paxos requires some thought to
understand, it is far simpler than Raft. Here's an easy-to-read paper
about Paxos:

  http://css.csail.mit.edu/6.824/2014/papers/paxos-simple.pdf

However, Paxos solves a much smaller problem than Raft. To build a
real-world replicated service, the replicas need to agree on an
indefinite sequence of values (the client commands), and they need
ways to efficiently recover when servers crash and restart or miss
messages. People have built such systems with Paxos as the starting
point; look up Google's Chubby and Paxos Made Live papers, and
ZooKeeper/ZAB. There is also a protocol called Viewstamped
Replication; it's a good design, and similar to Raft, but the paper
about it is hard to understand.

These real-world protocols are complex, and (before Raft) there was
not a good introductory paper describing how they work. The Raft
paper, in contrast, is relatively easy to read and fairly detailed.
That's a big contribution.

Whether the Raft protocol is inherently easier to understand than
something else is not clear. The issue is clouded by a lack of good
descriptions of other real-world protocols. In addition, Raft
sacrifices performance for clarity in a number of ways; that's fine
for a tutorial but not always desirable in a real-world protocol.

Q: How long had Paxos existed before the authors created Raft? How
widespread is Raft's usage in production now?

A: Paxos was invented in the late 1980s. Raft was developed around
2012.

Raft closely resembles a protocol called Viewstamped Replication,
originally published in 1988. There were replicated fault-tolerant file
servers built on top of Viewstamped Replication in the early 1990s,
though not in production use.

A bunch of real-world systems are derived from Paxos: Chubby, Spanner,
Megastore, and Zookeeper/ZAB. Starting in the early 2000s big web
sites and cloud providers needed fault-tolerant services, and Paxos
was more or less re-discovered at that time and put into production
use.

There are several real-world users of Raft: Docker
(https://docs.docker.com/engine/swnarm/raft/) and etcd
(https://etcd.io). Other systems said to be using Raft include
CockroachDB, RethinkDB, and TiKV. Maybe you can find more starting at
http://raft.github.io/

Q: How does Raft's performance compare to Paxos in real-world applications?

A: The fastest Paxos-derived protocols are probably much faster than
Raft; have a look at ZAB/ZooKeeper and Paxos Made Live. But I suspect
one could modify Raft to have very good performance. etcd3
claims to have achieved better performance than zookeeper and many
Paxos-based implementations
(https://www.youtube.com/watch?v=hQigKX0MxPw).

Q: Why are we learning/implementing Raft instead of Paxos?

A: We're using Raft in 6.824 because there is a paper that clearly
describes how to build a complete replicated service using Raft. I
know of no satisfactory paper that describes how to build a complete
replicated server system based on Paxos.

Q: Are there systems like Raft that can survive and continue to
operate when only a minority of the cluster is active?

A: Not with Raft's properties. But you can do it with different
assumptions, or different client-visible semantics. The basic problem
is split-brain -- the possibility of more than one server acting as
leader. There are two approaches that I know of.

If somehow clients and servers can learn exactly which servers are live
and which are dead (as opposed to live but partitioned by network
failure), then one can build a system that can function as long as one
is alive, picking (say) the lowest-numbered server known to be alive.
However, it's hard for one computer to decide if another computer is
dead, as opposed to the network losing the messages between them. One
way to do it is to have a human decide -- the human can inspect each
server and decide which are alive and dead.

The other approach is to allow split-brain operation, and to have a way
for servers to reconcile the resulting diverging state after partitions
are healed. This can be made to work for some kinds of services, but has
complex client-visible semantics (usually called "eventual
consistency"). Have a look at the Bayou and Dynamo papers which are
assigned later in the course.

Q: In Raft, the service which is being replicated is not available to
the clients during an election process. In practice how much of a
problem does this cause?

A: The client-visible pause seems likely to be on the order of a tenth of a
second. The authors expect failures (and thus elections) to be rare,
since they only happen if machines or the network fails. Many servers
and networks stay up continuously for months or even years at a time, so
this doesn't seem like a huge problem for many applications.

Q: Are there other consensus systems that don't have leader-election
pauses?

A: There are versions of Paxos-based replication that do not have a leader
or elections, and thus don't suffer from pauses during elections.
Instead, any server can effectively act as leader at any time. The cost
of not having a leader is that more messages are required for each
agreement.

Q: How are Raft and VMware FT related?

A: VM-FT can replicate any virtual machine guest, and thus any
server-style software. Raft can only replicate software designed
specifically for replication for Raft. For such software, Raft would
likely be much more efficient than VM-FT.

Q: Why can't a malicious person take over a Raft server, or forge
incorrect Raft messages?

A: Raft doesn't have any defense against attacks like this. It assumes
that all participants are following the protocol, and that only the
correct set of servers is participating.

A real deployment would have to do better than this. The most
straightforward option is to place the servers behind a firewall to
filter out packets from random people on the Internet, and to ensure
that all computers and people inside the firewall are trustworthy.

There may be situations where Raft has to operate on the same network as
potential attackers. In that case a good plan would be to authenticate
the Raft packets with some cryptographic scheme. For example, give each
legitimate Raft server a public/private key pair, have it sign all the
packets it sends, give each server a list of the public keys of
legitimate Raft servers, and have the servers ignore packets that aren't
signed by a key on that list.

Q: The paper mentions that Raft works under all non-Byzantine
conditions. What are Byzantine conditions and why could they make Raft
fail?

A: "Non-Byzantine conditions" means that the servers are fail-stop:
they either follow the Raft protocol correctly, or they halt. For
example, most power failures are non-Byzantine because they cause
computers to simply stop executing instructions; if a power failure
occurs, Raft may stop operating, but it won't send incorrect results
to clients.

Byzantine failure refers to situations in which some computers execute
incorrectly, because of bugs or because someone malicious is
controlling the computers. If a failure like this occurs, Raft may
send incorrect results to clients.

Most of 6.824 is about tolerating non-Byzantine faults. Correct
operation despite Byzantine faults is more difficult; we'll touch on
this topic at the end of the term.

Q: In Figure 1, what does the interface between client and server look
like?

A: Typically an RPC interface to the server. For a key/value storage
server such as you'll build in Lab 3, it's Put(key,value) and
Get(value) RPCs. The RPCs are handled by a key/value module in the
server, which calls Raft.Start() to ask Raft to put a client RPC in
the log, and reads the applyCh to learn of newly committed log
entries.

Q: What if a client sends a request to a leader, the the leader
crashes before sending the client request to all followers, and the
new leader doesn't have the request in its log? Won't that cause the
client request to be lost?

A: Yes, the request may be lost. If a log entry isn't committed, Raft
may not preserve it across a leader change.

That's OK because the client could not have received a reply to its
request if Raft didn't commit the request. The client will know (by
seeing a timeout or leader change) that its request wasn't served, and
will re-send it.

The fact that clients can re-send requests means that the system has
to be on its guard against duplicate requests; you'll deal with this
in Lab 3.

Q: If there's a network partition, can Raft end up with two leaders
and split brain?

A: No. There can be at most one active leader.

A new leader can only be elected if it can contact a majority of servers
(including itself) with RequestVote RPCs. So if there's a partition, and
one of the partitions contains a majority of the servers, that one
partition can elect a new leader. Other partitions must have only a
minority, so they cannot elect a leader. If there is no majority
partition, there will be no leader (until someone repairs the network
partition).

Q: Suppose a new leader is elected while the network is partitioned,
but the old leader is in a different partition. How will the old
leader know to stop committing new entries?

A: The old leader will either not be able to get a majority of
successful responses to its AppendEntries RPCs (if it's in a minority
partition), or if it can talk to a majority, that majority must
overlap with the new leader's majority, and the servers in the overlap
will tell the old leader that there's a higher term. That will cause
the old leader to switch to follower.

Q: When some servers have failed, does "majority" refer to a majority
of the live servers, or a majority of all servers (even the dead
ones)?

A: Always a majority of all servers. So if there are 5 Raft peers in
total, but two have failed, a candidate must still get 3 votes
(including itself) in order to elected leader.

There are many reasons for this. It could be that the two "failed"
servers are actually up and running in a different partition. From
their point of view, there are three failed servers. If they were
allowed to elect a leader using just two votes (from just the two
live-looking servers), we would get split brain. Another reason is
that we need the majorities of any two leader to overlap at at least
one server, to guarantee that a new leader sees the previous term
number and any log entries committed in previous terms; this requires
a majority out of all servers, dead and alive.

Q: What if the election timeout is too short? Will that cause Raft to
malfunction?

A: A bad choice of election timeout does not affect safety, it only
affects liveness.

If the election timeout is too small, then followers may repeatedly
time out before the leader has a chance to send out any AppendEntries.
In that case Raft may spend all its time electing new leaders, and no
time processing client requests. If the election timeout is too large,
then there will be a needlessly large pause after a leader failure
before a new leader is elected.

Q: Why randomize election timeouts?

A: To reduce the chance that two peers simultaneously become
candidates and split the votes evenly between them, preventing anyone
from being elected.

Q: Can a candidate declare itself the leader as soon as it receives
votes from a majority, and not bother waiting for further RequestVote
replies?

A: Yes -- a majority is sufficient. It would be a mistake to wait
longer, because some peers might have failed and thus not ever reply.

Q: Can a leader in Raft ever stop being a leader except by crashing?

A: Yes. If a leader's CPU is slow, or its network connection breaks,
or loses too many packets, or delivers packets too slowly, the other
servers won't see its AppendEntries RPCs, and will start an election.

Q: When are followers' log entries sent to their state machines?

A: Only after the leader says that an entry is committed, using the
leaderCommit field of the AppendEntries RPC. At that point the
follower can execute (or apply) the log entry, which for us means send
it on the applyCh.

Q: Should the leader wait for replies to AppendEntries RPCs?

A: The leader should send the AppendEntries RPCs concurrently, without
waiting. As replies come back, the leader should count them, and mark
the log entry as committed only when it has replies from a majority of
servers (including itself).

One way to do this in Go is for the leader to send each AppendEntries
RPC in a separate goroutine, so that the leader sends the RPCs
concurrently. Something like this:

  for each server {
    go func() {
      send the AppendEntries RPC and wait for the reply
      if reply.success == true {
        increment count
        if count == nservers/2 + 1 {
          this entry is committed
        }
      }
    } ()
  }

Q: What happens if more than half of the servers die? 

A: The service can't make any progress; it will keep trying to elect a
leader over and over. If/when enough servers come back to life with
persistent Raft state intact, they will be able to elect a leader and
continue.

Q: Why is the Raft log 1-indexed?

A: You should view it as zero-indexed, but starting out with one entry
(at index=0) that has term 0. That allows the very first AppendEntries
RPC to contain 0 as PrevLogIndex, and be a valid index into the log.

Q: When network partition happens, wouldn't client requests in
minority partitions be lost?

A: Yes, only the partition with a majority of servers can commit and
execute client operations. The servers in the minority partition(s)
won't be able to commit client operations, so they won't reply to
client requests. Clients will keep re-sending the requests until they
can contact a majority Raft partition.

Q: Is the argument in 5.4.3 a complete proof?

A: 5.4.3 is not a complete proof. Here are some places to look:

http://ramcloud.stanford.edu/~ongaro/thesis.pdf
http://verdi.uwplse.org/raft-proof.pdf

> What are some uses of Raft besides GFS master replication?

You could (and will) build a fault-tolerant key/value database using Raft.

You could make the MapReduce master fault-tolerant with Raft.

You could build a fault-tolerant locking service.


> When raft receives a read request does it still commit a no-op?

Section 8 mentions two different approaches. The leader could send out a
heartbeat first; or there could be a convention that the leader can't
change for a known period of time after each heartbeat (the lease).
Real systems usually use a lease, since it requires less
communication.

Section 8 says the leader sends out a no-op only at the very beginning
of its term.

> The paper states that no log writes are required on a read, but then
> immediately goes on to introduce committing a no-op as a technique to
> get the committed. Is this a contradiction or are no-ops not
> considered log 'writes'?

The no-op only happens at the start of the term, not for each read.


> I find the line about the leader needing to commit a no-op entry in
> order to know which entries are committed pretty confusing. Why does
> it need to do this?

The problem situation is shown in Figure 8, where if S1 becomes leader
after (b), it cannot know if its last log entry (2) is committed or not.
The situation in which the last log entry will turn out not to be
committed is if S1 immediately fails, and S5 is the next leader; in that
case S5 will force all peers (including S1) to have logs identical to
S5's log, which does not include entry 2.

But suppose S1 manages to commit a new entry during its term (term 4).
If S5 sees the new entry, S5 will erase 3 from its log and accept 2 in
its place. If S5 does not see the new entry, S5 cannot be the next
leader if S1 fails, because it will fail the Election Restriction.
Either way, once S1 has committed a new entry for its term, it can
correctly conclude that every preceding entry in its log is committed.

The no-op text at the end of Section 8 is talking about an optimization
in which the leader executes and answers read-only commands (e.g.
get("k1")) without committing those commands in the log. For example,
for get("k1"), the leader just looks up "k1" in its key/value table and
sends the result back to the client. If the leader has just started, it
may have at the end of its log a put("k1", "v99"). Should the leader
send "v99" back to the client, or the value in the leader's key/value
table? At first, the leader doesn't know whether that v99 log entry is
committed (and must be returned to the client) or not committed (and
must not be sent back). So (if you are using this optimization) a new
Raft leader first tries to commit a no-op to the log; if the commit
succeeds (i.e. the leader doesn't crash), then the leader knows
everything before that point is committed.


> How does using the heartbeat mechanism to provide leases (for
> read-only) operations work, and why does this require timing for
> safety (e.g. bounded clock skew)?

I don't know exactly what the authors had in mind. Perhaps every
AppendEntries RPC the leader sends out says or implies that the no other
leader is allowed to be elected for the next 100 milliseconds. If the
leader gets positive responses from a majority, then the leader can
serve read-only requests for the next 100 milliseconds without further
communication with the followers.

This requires the servers to have the same definition of what 100
milliseconds means, i.e. they must have clocks that tick at the close to
the same rate.


> What exactly do the C_old and C_new variables in section 6 (and figure
> 11) represent? Are they the leader in each configuration?

They are the set of servers in the old/new configuration.

The paper doesn't provide details. I believe it's the identities
(network names or addresses) of the servers.

> When transitioning from cluster C_old to cluster C_new, how can we
> create a hybrid cluster C_{old,new}? I don't really understand what
> that means. Isn't it following either the network configuration of
> C_old or of C_new? What if the two networks disagreed on a connection?

During the period of joint consensus (while Cold,new is active), the
leader is required to get a majority from both the servers in Cold and
the servers in Cnew.

There can't really be disagreement, because after Cold,new is committed
into the logs of both Cold and Cnew (i.e. after the period of joint
consensus has started), any new leader in either Cold or Cnew is
guaranteed to see the log entry for Cold,Cnew.


> I'm confused about Figure 11 in the paper. I'm unsure about how
> exactly the transition from 'C_old' to 'C_old,new' to 'C_new' goes.
> Why is there the issue of the cluster leader not being a part of the
> new configuration, where the leader steps down once it has committed
> the 'C_new' log entry? (The second issue mentioned in Section 6)


Suppose C_old={S1,S2,S3} and C_new={S4,S5,S6}, and that S1 is the leader
at the start of the configuration change. At the end of the
configuration change, after S1 has committed C_new, S1 should not be
participating any more, since S1 isn't in C_new. One of S4, S5, or S6
should take over as leader.


> About cluster configuration: During the configuration change time, if
> we have to stop receiving requests from the clients, then what's the
> point of having this automated configuration step? Doesn't it suffice
> to just 1) stop receiving requests 2) change the configurations 3)
> restart the system and continue?


The challenge here is ensuring that the system is correct even if there
are failures during this process, and even if not all servers get the
"stop receiving requests" and "change the configuration" commands at the
same time. Any scheme has to cope with the possibility of a mix of
servers that have and have not seen or completed the configuration
change -- this is true even of a non-automated system. The paper's
protocol is one way to solve this problem.


> The last two paragraphs of section 6 discuss removed servers
> interfering with the cluster by trying to get elected even though
> theyâ€™ve been removed from the configuration.
> Wouldnâ€™t a simpler solution be to require servers to be
> shut down when they leave the configuration? It seems that leaving the
> cluster implies that a server canâ€™t send or receive RPCs to
> the rest of the cluster anymore, but the paper doesnâ€™t
> assume that. Why not? Why canâ€™t you assume that the servers
> will shut down right away?

I think the immediate problem is that the Section 6 protocol doesn't
commit Cnew to the old servers, it only commits Cnew to the servers in
Cnew. So the servers that are not in Cnew never learn when Cnew takes
over from Cold,new.

The paper does say this:

  When the new configuration has been committed under the rules of Cnew,
  the old configuration is irrelevant and servers not in the new
  configuration can be shut down.

So perhaps the problem only exists during the period of time between the
configuration change and when an administrator shuts down the old
servers. I don't know why they don't have a more automated scheme.


> How common is it to get a majority from both the old and new
> configurations when doing things like elections and entry commitment,
> if it's uncommon, how badly would this affect performance?


I imagine that in most cases there is no failure, and the leader gets
both majorities right away. Configuration change probably only takes a
few round trip times, i.e. a few dozen milliseconds, so the requirement
to get both majorities will slow the system down for only a small amount
of time. Configuration change is likely to be uncommon (perhaps every
few months); a few milliseconds of delay every few months doesn't seem
like a high price.

> and how important is the decision to have both majorities?

The requirement for both majorities is required for correctness, to
cover the possibility that the leader fails during the configuration
change.


> Just to be clear, the process of having new members join as non-voting
> entities isn't to speed up the process of replicating the log, but
> rather to influence the election process? How does this increase
> availability? These servers that need to catch up are not going to be
> available regardless, right?


The purpose of non-voting servers is to allow those servers to get a
complete copy of the leader's log without holding up new commits. The
point is to allow a subsequent configuration change to be quick. If the
new servers didn't already have nearly-complete logs, then the leader
wouldn't be able to commit Cold,new until they caught up; and no new
client commands can be executed between the time Cold,new is first sent
out and the time at which it is committed.


> If the cluster leader does not have the new configuration, why doesn't
> it just remove itself from majority while committing C_new, and then
> when done return to being leader? Is there a need for a new election
> process?


Is this about the "second issue" in Section 6? The situation they
describe is one in which the leader isn't in the new configuration at
all. So after Cnew is committed, the leader shouldn't be participating
in Raft at all.


> How does the non-voting membership status work in the configuration change
> portion of Raft. Does that server state only last during the changeover
> (i.e. while c_new not committed) or do servers only get full voting privileges
> after being fully "caught up"? If so, at what point are they considered
> "caught up"?

The paper doesn't have much detail here. I imagine that the leader won't
start the configuration change until the servers to be added (the
non-voting members) are close to completely caught up. When the leader
sends out the Cold,new log entry in AppendEntries RPCs to those new
servers, the leader will bring them fully up to date (using the Figure 2
machinery). The leader won't be able to commit the Cold,new message
until a majority of those new servers are fully caught up. Once the
Cold,new message is committed, those new servers can vote.


> A question on "In Search of an Understandable Consensus Algorithm" (the RAFT paper)
> (section 6-end)
>
>  - I don't disagree that having servers deny RequestVotes that are less than
>    the minimum election timeout from the last heartbeat is a good idea (it
>    helps prevent unnecessary elections in general), but why did they choose
>    that method specifically to prevent servers not in a configuration running
>    for election? It seems like it would make more sense to check if a given
>    server is in the current configuration. E.g., in the lab code we are using,
>    each server has the RPC addresses of all the servers (in the current
>    configuration?), and so should be able to check if a requestVote RPC came
>    from a valid (in-configuration) server, no?


I agree that the paper's design seems a little awkward, and I don't know
why they designed it that way. Your idea seems like a reasonable
starting point. One complication is that there may be situations in
which a server in Cnew is leader during the joint consensus phase, but
at that time some servers in Cold may not know about the joint consensus
phase (i.e. they only know about Cold, not Cold,new); we would not want
the latter servers to ignore the legitimate leader.


> When exactly does joint consensus begin, and when does it end? Does joint
> consensus begin at commit time of "C_{o,n}"?


Joint consensus is in progress when the current leader is aware of
Cold,new. If the leader doesn't manage to commit Cold,new, and crashes,
and the new leader doesn't have Cold,new in its log, then joint
consensus ends early. If a leader manages to commit Cold,new, then joint
consensus has not just started but will eventually complete, when a
leader commits Cnew.

> Can the configuration log entry be overwritten by a subsequent leader
> (assuming that the log entry has not been committed)?


Yes, that is possible, if the original leader trying to send out Cold,new
crashes before it commits the Cold,new.


> How can the "C_{o,n}" log entry ever be committed? It seems like it must be
> replicated to a majority of "old" servers (as well as the "new" servers), but
> the append of "C_{o,n}" immediately transitions the old server to new, right?


The commit does not change the set of servers in Cold or Cnew. For example,
perhaps the original configuration contains servers S1, S2, S3; then Cold
is {S1,S2,S3}. Perhaps the desired configuration is S4, S5, S6; then
Cnew is {S4,S5,S6}. Once Cnew is committed to the log, the configuration
is Cnew={S4,S5,S6}; S1,S2, and S3 are no longer part of the configuration.


> When snapshots are created, is the data and state used the one for the
> client application? If it's the client's data then is this something
> that the client itself would need to support in addition to the
> modifications mentioned in the raft paper?


Example: if you are building a key/value server that uses Raft for
replication, then there will be a key/value module in the server that
stores a table of keys and values. It is that table of keys and values
that is saved in the snapshot.


> The paper says that "if the follower receives a snapshot that
> describves a prefix of its log, then log entries covered by the
> snapshot are deleted but entries following the snapshot are retained".
> This means that we could potentially be deleting operations on the
> state machinne.

I don't think information will be lost. If the snapshot covers a prefix
of the log, that means the snapshot includes the effects of all the
operations in that prefix. So it's OK to discard that prefix.


> It seems that snapshots are useful when they are a lot smaller than
> applying the sequence of updates (e.g., frequent updates to a few
> keys). What happens when a snapshot is as big as the sum of its
> updates (e.g., each update inserts a new unique key)? Are there any
> cost savings from doing snapshots at all in this case?


If the snapshot is about as big as the log, then there may not be a lot
of value in having snapshots. On the other hand, perhaps the snapshot
organizes the data in a way that's easier to access than a log, e.g. in
a sorted table. Then it might be faster to re-start the service after a
crash+reboot from a snapshotted table than from the log (which you would
have to sort).

It's much more typical, however, for the log to be much bigger than the
state.

> Also wouldn't a InstallSnapshot incur heavy bandwidth costs?

Yes, if the state is large (as it would be for e.g. a database).
However, this is not an easy problem to solve. You'd probably want the
leader to keep enough of its log to cover all common cases of followers
lagging or being temporarily offline. You might also want a way to
transfer just the differences in server state, e.g. just the parts of
the database that have changed recently.


> Is there a concern that writing the snapshot can take longer than the
> eleciton timeout because of the amount of data that needs to be
> appended to the log?

You're right that it's a potential problem for a large server. For
example if you're replicating a database with a gigabyte of data, and
your disk can only write at 100 megabytes per second, writing the
snapshot will take ten seconds. One possibility is to write the snapshot
in the background (i.e. arrange to not wait for the write, perhaps by
doing the write from a child process), and to make sure that snapshots
are created less often than once per ten seconds.


> Under what circumstances would a follower receive a snapshot that is a
> prefix of its own log?

The network can deliver messages out of order, and the RPC handling
system can execute them out of order. So for example if the leader sends
a snapshot for log index 100, and then one for log index 110, but the
network delivers the second one first.

>  Additionally, if the follower receives a snapshot that is a
> prefix of its log, and then replaces the entries in its log up to that
> point, the entries after that point are ones that the leader is not
> aware of, right?

The follower might have log entries that are not in a received snapshot
if the network delays delivery of the snapshot, or if the leader has
sent out log entires but not yet committed them.

> Will those entries ever get committed?

They could get committed.


> How does the processing of InstallSnapshot RPC handle reordering, when the
> check at step 6 references log entries that have been compacted? Specifically,
> shouldn't Figure 13 include: 1.5: If lastIncludedIndex < commitIndex, return
> immediately.  or alternatively 1.5: If there is already a snapshot and
> lastIncludedIndex < currentSnapshot.lastIncludedIndex, return immediately.

I agree -- for Lab 3B the InstallSnapshot RPC handler must reject stale
snapshots. I don't know why Figure 13 doesn't include this test; perhaps
the authors' RPC system is better about order than ours. Or perhaps the
authors intend that we generalize step 6 in Figure 13 to cover this case.


> What happens when the leader sends me an InstallSnapshot command that
> is for a prefix of my log, but I've already undergone log compaction
> and my snapshot is ahead? Is it safe to assume that my snapshot that
> is further forward subsumes the smaller snapshot?


Yes, it is correct for the recipient to ignore an InstallSnapshot if the
recipient is already ahead of that snapshot. This case can arise in Lab
3, for example if the RPC system delivers RPCs out of order.


> How do leaders decide which servers are lagging and need to be sent a
> snapshot to install?

If a follower rejects an AppendEntries RPC for log index i1 due to rule
#2 or #3 (under AppendEntries RPC in Figure 2), and the leader has
discarded its log before i1, then the leader will send an
InstallSnapshot rather than backing up nextIndex[].


> Is there a concern that writing the snapshot can take longer than the
> eleciton timeout because of the amount of data that needs to be
> appended to the log?

You're right that it's a potential problem for a large server. For
example if you're replicating a database with a gigabyte of data, and
your disk can only write at 100 megabytes per second, writing the
snapshot will take ten seconds. One possibility is to write the snapshot
in the background (i.e. arrange to not wait for the write, perhaps by
doing the write from a child process), and to make sure that snapshots
are created less often than once per ten seconds.


> In actual practical use of raft, how often are snapshots sent?

I have not seen an analysis of real-life Raft use. I imagine people
using Raft would tune it so that snapshots were rarely needed (e.g. by
having leaders keep lots of log entries with which to update lagging
followers).


> Is InstallSnapshot atomic? If a server crashes after partially
> installing a snapshot, and the leader re-sends the InstallSnapshot
> RPC, is this idempotent like RequestVote and AppendEntries RPCs?

The implementation of InstallSnapshot must be atomic.

It's harmless for the leader to re-send a snapshot.


> Why is an offset needed to index into the data[] of an InstallSNapshot
> RPC, is there data not related to the snapshot? Or does it overlap
> previous/future chunks of the same snapshot? Thanks!

The complete snapshot may be sent in multiple RPCs, each containing a
different part ("chunk") of the complete snapshot. The offset field
indicates where this RPC's data should go in the complete snapshot.

> How does copy-on-write help with the performance issue of creating snapshots?


The basic idea is for th server to fork(), giving the child a complete
copy of the in-memory state. If fork() really copied all the memory, and
the state was large, this would be slow. But most operating systems
don't copy all the memory in fork(); instead they mark the pages as
"copy-on-write", and make them read-only in both parent and child. Then
the operating system will see a page fault if either tries to write a
page, and the operating system will only copy the page at that point.
The net effect is usually that the child sees a copy of its parent
process' memory at the time of the fork(), but with relatively little
copying.


> What data compression schemes, such as VIZ, ZIP, Huffman encoding,
> etc. are most efficient for Raft snapshotting?

It depends on what data the service stores. If it stores images, for
example, then maybe you'd want to compress them with JPEG.

If you are thinking that each snapshot probably shares a lot of content
with previous snapshots, then perhaps you'd want to use some kind of
tree structure which can share nodes across versions.



> Quick clarification: Does adding an entry to the log count as an
> executed operation?

No. A server should only execute an operation in a log entry after the
leader has indicated that the log entry is committed. "Execute" means
handing the operation to the service that's using Raft. In Lab 3,
"execute" means that Raft gives the committed log entry to your
key/value software, which applies the Put(key,value) or Get(key) to its
table of key/value pairs.


> According to the paper, a server disregards RequestVoteRPCs when they
> think a current leader exists, but then the moment they think a
> current leader doesn't exist, I thought they try to start their own
> election. So in what case would they actually cast a vote for another
> server?

> For the second question, I'm still confused: what does the paper mean
> when it says a server should disregard a RequestVoteRPC when it thinks
> a current leader exists at the end of Section 6? In what case would a
> server think a current leader doesn't exist but hasn't started its own
> election? Is it if the server thinks it hasn't yet gotten a heartbeat
> from the server but before its election timeout?


Each server waits for a randomly chosen election timeout; if it hasn't heard
from the leader for that whole period, and no other server has started an
election, then the server starts an election. Whichever server's election timer
expires first is likely to get votes from most or all of the servers before any
other server's timer expires, and thus is likely to win the election.

Suppose the heartbeat interval is 10 milliseconds (ms). The leader sends
out heartbeats at times 10, 20, and 30.

Suppose server S1 doesn't hear the heartbeat at time 30. S1's election timer
goes off at time 35, and S1 sends out RequestVote RPCs.

Suppose server S2 does hear the heartbeat at time 30, so it knows the
server was alive at that time. S2 will set its election timer to go off
no sooner than time 40, since only a missing heartbeat indicates a
possibly dead server, and the next heartbeat won't come until time 40.
When S2 hears S1'a RequestVote at time 35, S2 can ignore the
RequestVote, because S2 knows that it heard a heartbeat less than one
heartbeat interval ago.


> I'm a little confused by the "how to roll back quickly" part of the
> Lecture 6 notes (and the corresponding part of the paper):
>
>   paper outlines a scheme towards end of Section 5.3:
>   if follower rejects, includes this in reply:
>     the term of the conflicting entry
>     the index of the first entry for conflicting term
>   if leader knows about the conflicting term:
>     move nextIndex[i] back to its last entry for the conflicting term
>   else:
>     move nextIndex[i] back to follower's first index
>
> I think according to the paper, the leader should move nextIndex[i] to
> the index of the first entry for conflicting term. What does the
> situation "if leader knows about the conflicting term" mean?

The paper's description of the algorithm is not complete, so we have to
invent the details for ourselves. The notes have the version I invented;
I don't know if it's what the authors had in mind.

The specific problem with the paper's "index of the first entry for the
conflicting term" is that the leader might no have entries at all for
the conflicting term. Thus my notes cover two cases -- if the server
knows about the conflicting term, and if it doesn't.


> What are the tradeoffs in network/ performance in decreasing nextIndex
> by a factor of 2 each time at each mismatch? i.e. first by 1,2,4, 8
> and so on

The leader will overshoot by up to a factor of two, and thus have to
send more entries than needed. Of course the Figure 2 approach is also
wasteful if one has to back up a lot. Best might be to implement
something more precise, for example the optimization outlined towards
the end of section 5.3.


> Unrelatedly - How does your experience teaching Raft and Paxos
> correspond to section 9.1 of the paper? Do your experiences support
> their findings?


I was pretty happy with the Paxos labs from two years ago. I'm pretty
happy with the current Raft labs too. The Raft labs are more ambitious:
unlike the Paxos labs, the Raft labs have a leader, persistence, and
snapshots. I don't think we have any light to shed on the findings in
Section 9.1; we didn't perform a side-by-side experiment on the
students, and our Raft labs are noticeably more ambitious.


> - What has been the impact of Raft, from the perspective of academic
> - researchers in the field? Is it considered significant, inspiring,
> - non-incremental work? Or is it more of "okay, this seems like a
> - natural progression, and is a bit easier to teach, so let's teach
> - this?"


The Raft paper does a better job than any paper I know of in explaining
modern replicated state machine techniques. I think it has inspired lots
of people to build their own replication implementations.


> The paper states that there are a fair amount of implementations of Raft out
> in the wild. Have there been any improvement suggestions that would make sense
> to include in a revised version of the algorithm?

Here's an example:

https://www.cl.cam.ac.uk/~ms705/pub/papers/2015-osr-raft.pdf