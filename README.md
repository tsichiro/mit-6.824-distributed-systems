# MIT_6.824_DistributedSystems

- Lecture 1: [Introduction](./introduction.md)
- Lecture 2: [Infrastructure: RPC and threads](./go-threads-crawler-rpc.md)
- Lecture 3: [GFS](the-google-file-system.md)
- Lecture 4: [Primary/Backup Replication](./primary-backup-replication.md)
- Lecture 5: [Fault Tolerance: Raft](./in-search-of-an-understandable-consensus-algorithm-extended-version.md)

## Lecture 6: Fault Tolerance: Raft

6.824 2018 Lecture 6: Raft (2)

Recall the big picture:
  key/value service as the example, as in Lab 3
  goal: same client-visible behavior as single non-replicated server
  goal: available despite minority of failed/disconnected servers
  watch out for network partition and split brain!
  [diagram: clients, k/v layer, k/v table, raft layer, raft log]
  [client RPC -> Start() -> majority commit protocol -> applyCh]
  "state machine", application, service

a few reminders:
  leader commits/executes after a majority replies to AppendEntries
  leader tells commit to followers, which execute (== send on applyCh)
  why wait for just a majority? why not wait for all peers?
    availability requires progress even if minority are crashed
  why is a majority sufficient?
    any two majorities overlap
    so successive leaders' majorities overlap at at least one peer
    so next leader is guaranteed to see any log entry committed by previous leader
  it's majority of all peers (dead and alive), not just majority of live peers

*** topic: the Raft log (Lab 2B)

as long as the leader stays up:
  clients only interact with the leader
  clients aren't affected by follower actions

things only get interesting when changing leaders
  e.g. after the old leader fails
  how to change leaders without clients seeing anomalies?
    stale reads, repeated operations, missing operations, different order, &c

what do we want to ensure?
  if any server executes a given command in a log entry,
    then no server executes something else for that log entry
  (Figure 3's State Machine Safety)
  why? if the servers disagree on the operations, then a
    change of leader might change the client-visible state,
    which violates our goal of mimicing a single server.
  example:
    S1: put(k1,v1) | put(k1,v2) | ...
    S2: put(k1,v1) | put(k2,x)  | ...
    can't allow both to execute their 2nd log entries!

how can logs disagree after a crash?
  a leader crashes before sending last AppendEntries to all
    S1: 3
    S2: 3 3
    S3: 3 3
  worse: logs might have different commands in same entry!
    after a series of leader crashes, e.g.
        10 11 12 13  <- log entry #
    S1:  3
    S2:  3  3  4
    S3:  3  3  5

Raft forces agreement by having followers adopt new leader's log
  example:
  S3 is chosen as new leader for term 6
  S3 sends an AppendEntries with entry 13
     prevLogIndex=12
     prevLogTerm=5
  S2 replies false (AppendEntries step 2)
  S3 decrements nextIndex[S2] to 12
  S3 sends AppendEntries w/ entries 12+13, prevLogIndex=11, prevLogTerm=3
  S2 deletes its entry 12 (AppendEntries step 3)
  similar story for S1, but have to go back one farther

the result of roll-back:
  each live follower deletes tail of log that differs from leader
  then each live follower accepts leader's entries after that point
  now followers' logs are identical to leader's log

Q: why was it OK to forget about S2's index=12 term=4 entry?

could new leader roll back *committed* entries from end of previous term?
  i.e. could a committed entry be missing from the new leader's log?
  this would be a disaster -- old leader might have already said "yes" to a client
  so: Raft needs to ensure elected leader has all committed log entries

why not elect the server with the longest log as leader?
  example:
    S1: 5 6 7
    S2: 5 8
    S3: 5 8
  first, could this scenario happen? how?
    S1 leader in term 6; crash+reboot; leader in term 7; crash and stay down
      both times it crashed after only appending to its own log
    next term will be 8, since at least one of S2/S3 learned of 7 while voting
    S2 leader in term 8, only S2+S3 alive, then crash
  all peers reboot
  who should be next leader?
    S1 has longest log, but entry 8 could have committed !!!
    so new leader can only be one of S2 or S3
    i.e. the rule cannot be simply "longest log"

end of 5.4.1 explains the "election restriction"
  RequestVote handler only votes for candidate who is "at least as up to date":
    candidate has higher term in last log entry, or
    candidate has same last term and same length or longer log
  so:
    S2 and S3 won't vote for S1
    S2 and S3 will vote for each other
  so only S2 or S3 can be leader, will force S1 to discard 6,7
    ok since 6,7 not on majority -> not committed -> no reply sent to clients
    -> clients will resend commands in 6,7

the point:
  "at least as up to date" rule ensures new leader's log contains
    all potentially committed entries
  so new leader won't roll back any committed operation

The Question (from last lecture)
  figure 7, top server is dead; which can be elected?

depending on who is elected leader in Figure 7, different entries
  will end up committed or discarded
  c's 6 and d's 7,7 may be discarded OR committed
  some will always remain committed: 111445566

how to roll back quickly
  the Figure 2 design backs up one entry per RPC -- slow!
  lab tester may require faster roll-back
  paper outlines a scheme towards end of Section 5.3
    no details; here's my guess; better schemes are possible
  S1: 4 5 5      4 4 4      4
  S2: 4 6 6  or  4 6 6  or  4 6 6
  S3: 4 6 6      4 6 6      4 6 6
  S3 is leader for term 6, S1 comes back to life
  if follower rejects AppendEntries, it includes this in reply:
    the follower's term in the conflicting entry
    the index of follower's first entry with that term
  if leader has log entries with the follower's conflicting term:
    move nextIndex[i] back to leader's last entry for the conflicting term
  else:
    move nextIndex[i] back to follower's first index for the conflicting term

*** topic: persistence (Lab 2C)

what would we like to happen after a server crashes?
  Raft can continue with one missing server
    but we must repair soon to avoid dipping below a majority
  two strategies:
  * replace with a fresh (empty) server
    requires transfer of entire log (or snapshot) to new server (slow)
    we *must* support this, in case failure is permanent
  * or reboot crashed server, re-join with state intact, catch up
    requires state that persists across crashes
    we *must* support this, for simultaneous power failure
  let's talk about the second strategy -- persistence

if a server crashes and restarts, what must Raft remember?
  Figure 2 lists "persistent state":
    log[], currentTerm, votedFor
  a Raft server can only re-join after restart if these are intact
  thus it must save them to non-volatile storage
    non-volatile = disk, SSD, battery-backed RAM, &c
    save after each change
    before sending any RPC or RPC reply
  why log[]?
    if a server was in leader's majority for committing an entry,
      must remember entry despite reboot, so any future leader is
      guaranteed to see the committed log entry
  why votedFor?
    to prevent a client from voting for one candidate, then reboot,
      then vote for a different candidate in the same (or older!) term
    could lead to two leaders for the same term
  why currentTerm?
    to ensure that term numbers only increase
    to detect RPCs from stale leaders and candidates

some Raft state is volatile
  commitIndex, lastApplied, next/matchIndex[]
  Raft's algorithms reconstruct them from initial values

persistence is often the bottleneck for performance
  a hard disk write takes 10 ms, SSD write takes 0.1 ms
  so persistence limits us to 100 to 10,000 ops/second
  (the other potential bottleneck is RPC, which takes << 1 ms on a LAN)
  lots of tricks to cope with slowness of persistence:
    batch many new log entries per disk write
    persist to battery-backed RAM, not disk

how does the service (e.g. k/v server) recover its state after a crash+reboot?
  easy approach: start with empty state, re-play Raft's entire persisted log
    lastApplied is volatile and starts at zero, so you may need no extra code!
  but re-play will be too slow for a long-lived system
  faster: use Raft snapshot and replay just the tail of the log

*** topic: log compaction and Snapshots (Lab 3B)

problem:
  log will get to be huge -- much larger than state-machine state!
  will take a long time to re-play on reboot or send to a new server

luckily:
  a server doesn't need *both* the complete log *and* the service state
    the executed part of the log is captured in the state
    clients only see the state, not the log
  service state usually much smaller, so let's keep just that

what constrains how a server discards log entries?
  can't forget un-committed entries -- might be part of leader's majority
  can't forget un-executed entries -- not yet reflected in the state
  executed entries might be needed to bring other servers up to date

solution: service periodically creates persistent "snapshot"
  [diagram: service with state, snapshot on disk, raft log, raft persistent]
  copy of entire state-machine state as of execution of a specific log entry
    e.g. k/v table
  service writes snapshot to persistent storage (disk)
  service tells Raft it is snapshotted through some log index
  Raft discards log before that index
  a server can create a snapshot and discard prefix of log at any time
    e.g. when log grows too long

relation of snapshot and log
  snapshot reflects only executed log entries
    and thus only committed entries
  so server will only discard committed prefix of log
    anything not known to be committed will remain in log

so a server's on-disk state consists of:
  service's snapshot up to a certain log entry
  Raft's persisted log w/ following log entries
  the combination is equivalent to the full log

what happens on crash+restart?
  service reads snapshot from disk
  Raft reads persisted log from disk
    sends service entries that are committed but not in snapshot

what if a follower lags and leader has discarded past end of follower's log?
  nextIndex[i] will back up to start of leader's log
  so leader can't repair that follower with AppendEntries RPCs
  thus the InstallSnapshot RPC
  (Q: why not have leader discard only entries that *all* servers have?)

what's in an InstallSnapshot RPC? Figures 12, 13
  term
  lastIncludedIndex
  lastIncludedTerm
  snapshot data

what does a follower do w/ InstallSnapshot?
  reject if term is old (not the current leader)
  reject (ignore) if follower already has last included index/term
    it's an old/delayed RPC
  empty the log, replace with fake "prev" entry
  set lastApplied to lastIncludedIndex
  replace service state (e.g. k/v table) with snapshot contents

note that the state and the operation history are roughly equivalent
  designer can choose which to send
  e.g. last few operations (log entries) for lagging replica,
    but entire state (snapshot) for a replica that has lost its disk.
  still, replica repair can be very expensive, and warrants attention

The Question:
  Could a received InstallSnapshot RPC cause the state machine to go
  backwards in time? That is, could step 8 in Figure 13 cause the state
  machine to be reset so that it reflects fewer executed operations? If
  yes, explain how this could happen. If no, explain why it can't
  happen.

*** topic: configuration change (not needed for the labs)

configuration change (Section 6)
  configuration = set of servers
  sometimes you need to
    move to a new set of servers, or
    increase/decrease the number of servers
  human initiates configuration change, Raft manages it
  we'd like Raft to cope correctly with failure during configuration change
    i.e. clients should not notice (except maybe dip in performance)

why doesn't a straightforward approach work?
  suppose each server has the list of servers in the current config
  change configuration by telling each server the new list
    using some mechanism outside of Raft
  problem: they will learn new configuration at different times
  example: want to replace S3 with S4
    we get as far as telling S1 and S4 that the new config is 1,2,4
    S1: 1,2,3  1,2,4
    S2: 1,2,3  1,2,3
    S3: 1,2,3  1,2,3
    S4:        1,2,4
  OOPS! now *two* leaders could be elected!
    S2 and S3 could elect S2
    S1 and S4 could elect S1

Raft configuration change
  idea: "joint consensus" stage that includes *both* old and new configuration
    avoids any time when both old and new can choose leader independently
  system starts with Cold
  system administrator asks the leader to switch to Cnew
  Raft has special configuration log entries (sets of server addresses)
  each server uses the last configuration in its own log
  1. leader commits Cold,new to a majority of both Cold and Cnew
  2. after Cold,new commits, leader commits Cnew to servers in Cnew

what if leader crashes at various points in this process?
  can we have two leaders for the next term?
  if that could happen, each leader must be one of these:
    A. in Cold, but does not have Cold,new in log
    B. in Cold or Cnew, has Cold,new in log
    C. in Cnew, has Cnew in log
  we know we can't have A+A or C+C by the usual rules of leader election
  A+B? no, since B needs majority from Cold as well as Cnew
  A+C? no, since can't proceed to Cnew until Cold,new committed to Cold
  B+B? no, since B needs majority from both Cold and Cnew
  B+C? no, since B needs majority from Cnew as well as Cold

good! Raft can switch to a new set of servers w/o risk of two active leaders

*** topic: performance

Note: many situations don't require high performance.
  key/value store might.
  but GFS or MapReduce master might not.

Most replication systems have similar common-case performance:
  One RPC exchange and one disk write per agreement.
  So Raft is pretty typical for message complexity.

Raft makes a few design choices that sacrifice performance for simplicity:
  Follower rejects out-of-order AppendEntries RPCs.
    Rather than saving for use after hole is filled.
    Might be important if network re-orders packets a lot.
  No provision for batching or pipelining AppendEntries.
  Snapshotting is wasteful for big states.
  A slow leader may hurt Raft, e.g. in geo-replication.

These have a big effect on performance:
  Disk writes for persistence.
  Message/packet/RPC overhead.
  Need to execute logged commands sequentially.
  Fast path for read-only operations.

Papers with more attention to performance:
  Zookeeper/ZAB; Paxos Made Live; Harp

### FAQ

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
> they鈥檝e been removed from the configuration.
> Wouldn鈥檛 a simpler solution be to require servers to be
> shut down when they leave the configuration? It seems that leaving the
> cluster implies that a server can鈥檛 send or receive RPCs to
> the rest of the cluster anymore, but the paper doesn鈥檛
> assume that. Why not? Why can鈥檛 you assume that the servers
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

## LEC 7: Spinnaker

2018 Lecture 7: Raft (3) -- Snapshots, Linearizability, Duplicate Detection

this lecture:
  Raft snapshots
  linearizability
  duplicate RPCs
  faster gets

*** Raft log compaction and snapshots (Lab 3B)

problem:
  log will get to be huge -- much larger than state-machine state!
  will take a long time to re-play on reboot or send to a new server

luckily:
  a server doesn't need *both* the complete log *and* the service state
    the executed part of the log is captured in the state
    clients only see the state, not the log
  service state usually much smaller, so let's keep just that

what entries *can't* a server discard?
  un-executed entries -- not yet reflected in the state
  un-committed entries -- might be part of leader's majority

solution: service periodically creates persistent "snapshot"
  [diagram: service state, snapshot on disk, raft log, raft persistent]
  copy of service state as of execution of a specific log entry
    e.g. k/v table
  service writes snapshot to persistent storage (disk)
  service tells Raft it is snapshotted through some log index
  Raft discards log before that index
  a server can create a snapshot and discard prefix of log at any time
    e.g. when log grows too long

what happens on crash+restart?
  service reads snapshot from disk
  Raft reads persisted log from disk
  Raft log may start before the snapshot (but definitely not after)
    Raft will (re-)send committed entries on the applyCh
      since applyIndex starts at zero after reboot
    service will see repeats, must detect repeated index, ignore

what if follower's log ends before leader's log starts?
  nextIndex[i] will back up to start of leader's log
  so leader can't repair that follower with AppendEntries RPCs
  thus the InstallSnapshot RPC

what's in an InstallSnapshot RPC? Figures 12, 13
  term
  lastIncludedIndex
  lastIncludedTerm
  snapshot data

what does a follower do w/ InstallSnapshot?
  ignore if term is old (not the current leader)
  ignore if follower already has last included index/term
    it's an old/delayed RPC
  if not ignored:
    empty the log, replace with fake "prev" entry
    set lastApplied to lastIncludedIndex
    replace service state (e.g. k/v table) with snapshot contents

philosophical note:
  state is often equivalent to operation history
  you can often choose which one to store or communicate
  we'll see examples of this duality later in the course

practical notes:
  Raft layer and service layer cooperate to save/restore snapshots
  Raft's snapshot scheme is reasonable if the state is small
  for a big DB, e.g. if replicating gigabytes of data, not so good
    slow to create and write to disk
  perhaps service data should live on disk in a B-Tree
    no need to explicitly persist, since on disk already
  dealing with lagging replicas is hard, though
    leader should save the log for a while
    or save identifiers of updated records
  and fresh servers need the whole state

*** linearizability

we need a definition of "correct" for Lab 3 &c
  how should clients expect Put and Get to behave?
  often called a consistency contract
  helps us reason about how to handle complex situations correctly
    e.g. concurrency, replicas, failures, RPC retransmission,
         leader changes, optimizations
  we'll see many consistency definitions in 6.824
    e.g. Spinnaker's timeline consistency

"linearizability" is the most common and intuitive definition
  formalizes behavior expected of a single server

linearizability definition:
  an execution history is linearizable if
    one can find a total order of all operations,
    that matches real-time (for non-overlapping ops), and
    in which each read sees the value from the
    write preceding it in the order.

a history is a record of client operations, each with
  arguments, return value, time of start, time completed

example 1:
  |-Wx1-| |-Wx2-|
    |---Rx2---|
      |-Rx1-|
"Wx1" means "write value 1 to record x"
"Rx1" means "a read of record x yielded value 1"
order: Wx1 Rx1 Wx2 Rx2
  the order obeys value constraints (W -> R)
  the order obeys real-time constraints (Wx1 -> Wx2)
  so the history is linearizable

example 2:
  |-Wx1-| |-Wx2-|
    |--Rx2--|
              |-Rx1-|
Wx2 then Rx2 (value), Rx2 then Rx1 (time), Rx1 then Wx2 (value). but
that's a cycle -- so it cannot be turned into a linear order. so this
is not linearizable.

example 3:
|--Wx0--|  |--Wx1--|
            |--Wx2--|
        |-Rx2-| |-Rx1-|
order: Wx0 Wx2 Rx2 Wx1 Rx1
so it's linearizable.
note the service can pick the order for concurrent writes.
  e.g. Raft placing concurrent ops in the log.

example 4:
|--Wx0--|  |--Wx1--|
            |--Wx2--|
C1:     |-Rx2-| |-Rx1-|
C2:     |-Rx1-| |-Rx2-|
we have to be able to fit all operations into a single order
  maybe: Wx2 C1:Rx2 Wx1 C1:Rx1 C2:Rx1
    but where to put C2:Rx2?
      must come after C2:Rx1 in time
      but then it should have read value 1
  no order will work:
    C1's reads require Wx2 before Wx1
    C2's reads require Wx1 before Wx2
    that's a cycle, so there's no order
  not linearizable!
so: all clients must see concurrent writes in the same order

example 5:
ignoring recent writes is not linearizable
Wx1      Rx1
    Wx2
this rules out split brain, and forgetting committed writes

You may find this page useful:
https://www.anishathalye.com/2017/06/04/testing-distributed-systems-for-linearizability/

*** duplicate RPC detection (Lab 3)

What should a client do if a Put or Get RPC times out?
  i.e. Call() returns false
  if server is dead, or request dropped: re-send
  if server executed, but request lost: re-send is dangerous

problem:
  these two cases look the same to the client (no reply)
  if already executed, client still needs the result

idea: duplicate RPC detection
  let's have the k/v service detect duplicate client requests
  client picks an ID for each request, sends in RPC
    same ID in re-sends of same RPC
  k/v service maintains table indexed by ID
  makes an entry for each RPC
    record value after executing
  if 2nd RPC arrives with the same ID, it's a duplicate
    generate reply from the value in the table

design puzzles:
  when (if ever) can we delete table entries?
  if new leader takes over, how does it get the duplicate table?
  if server crashes, how does it restore its table?

idea to keep the duplicate table small
  one table entry per client, rather than one per RPC
  each client has only one RPC outstanding at a time
  each client numbers RPCs sequentially
  when server receives client RPC #10,
    it can forget about client's lower entries
    since this means client won't ever re-send older RPCs

some details:
  each client needs a unique client ID -- perhaps a 64-bit random number
  client sends client ID and seq # in every RPC
    repeats seq # if it re-sends
  duplicate table in k/v service indexed by client ID
    contains just seq #, and value if already executed
  RPC handler first checks table, only Start()s if seq # > table entry
  each log entry must include client ID, seq #
  when operation appears on applyCh
    update the seq # and value in the client's table entry
    wake up the waiting RPC handler (if any)

what if a duplicate request arrives before the original executes?
  could just call Start() (again)
  it will probably appear twice in the log (same client ID, same seq #)
  when cmd appears on applyCh, don't execute if table says already seen

how does a new leader get the duplicate table?
  all replicas should update their duplicate tables as they execute
  so the information is already there if they become leader

if server crashes how does it restore its table?
  if no snapshots, replay of log will populate the table
  if snapshots, snapshot must contain a copy of the table

but wait!
  the k/v server is now returning old values from the duplicate table
  what if the reply value in the table is stale?
  is that OK?

example:
  C1           C2
  --           --
  put(x,10)
               first send of get(x), 10 reply dropped
  put(x,20)
               re-sends get(x), gets 10 from table, not 20

what does linearizabilty say?
C1: |-Wx10-|          |-Wx20-|
C2:          |-Rx10-------------|
order: Wx10 Rx10 Wx20
so: returning the remembered value 10 is correct

*** read-only operations (end of Section 8)

Q: does the Raft leader have to commit read-only operations in
   the log before replying? e.g. Get(key)?

that is, could the leader respond immediately to a Get() using
  the current content of its key/value table?

A: no, not with the scheme in Figure 2 or in the labs.
   suppose S1 thinks it is the leader, and receives a Get(k).
   it might have recently lost an election, but not realize,
   due to lost network packets.
   the new leader, say S2, might have processed Put()s for the key,
   so that the value in S1's key/value table is stale.
   serving stale data is not linearizable; it's split-brain.
   
so: Figure 2 requires Get()s to be committed into the log.
    if the leader is able to commit a Get(), then (at that point
    in the log) it is still the leader. in the case of S1
    above, which unknowingly lost leadership, it won't be
    able to get the majority of positive AppendEntries replies
    required to commit the Get(), so it won't reply to the client.

but: many applications are read-heavy. committing Get()s
  takes time. is there any way to avoid commit
  for read-only operations? this is a huge consideration in
  practical systems.

idea: leases
  modify the Raft protocol as follows
  define a lease period, e.g. 5 seconds
  after each time the leader gets an AppendEntries majority,
    it is entitled to respond to read-only requests for
    a lease period without commiting read-only requests
    to the log, i.e. without sending AppendEntries.
  a new leader cannot execute Put()s until previous lease period
    has expired
  so followers keep track of the last time they responded
    to an AppendEntries, and tell the new leader (in the
    RequestVote reply).
  result: faster read-only operations, still linearizable.

note: for the Labs, you should commit Get()s into the log;
      don't implement leases.

Spinnaker optionally makes reads even faster, sacrificing linearizability
  Spinnaker's "timeline reads" are not required to reflect recent writes
    they are allowed to return an old (though committed) value
  Spinnaker uses this freedom to speed up reads:
    any replica can reply to a read, allowing read load to be parallelized
  but timeline reads are not linearizable:
    replica may not have heard recent writes
    replica may be partitioned from the leader!

in practice, people are often (but not always) willing to live with stale
  data in return for higher performance


### FAQ

Spinnaker FAQ

Q: What is timeline consistency?

A: It's a form of relaxed consistency that can provide faster reads at
the expense of visibly anomalous behavior. All replicas apply writes
in the same order (clients send writes to the leader, and the leader
picks an order and forwards the writes to the replicas). Clients are
allowed to send reads to any replica, and that replica replies with
whatever data it current has. The replica may not yet have received
recent writes from the leader, so the client may see stale data. If a
client sends a read to one replica, and then another, the client may
get older data from the second read. Unlike strong consistency,
timeline consistency makes clients aware of the fact that there are
replicas, and that the replicas may differ.

Q: When there is only 1 node up in the cohort, the paper says it鈥檚
still timeline consistency; how is that possible?

A: Timeline consistency allows the system to return stale values: a
get can return a value that was written in the past, but is not the
most recent one. Thus, if a server is partitioned off from the
majority, and perhaps has missed some Puts, it can still execute a
timeline-consistent Get.

Q: What are the trade-offs of the different consistency levels?

A: Strongly-consistent systems are easier to program because they
behave like a non-replicated systems. Building applications on top of
eventually-consistent systems is generally more difficult because the
programmer must consider scenarios that couldn't come up in a
non-replicated system. On the other hand, one can often obtain higher
performance from weaker consistency models.

Q: How does Spinnaker implement timeline reads, as opposed to
consistent reads?

A: Section 5 says that clients are allowed to send timeline read
requests to any replica, not just the leader. Replicas serve timeline
read requests from their latest committed version of the data, which
may be out of date (because the replica hasn't seen recent writes).

Q: Why are Spinnaker's timeline reads faster than its consistent
reads, in Figure 8?

A: It's not clear why timeline reads improve read performance; perhaps
the read throughput is higher by being split over multiple replicas,
perhaps the leader can serve reads without talking to the followers,
or perhaps all replicas can execute reads without waiting for
processing from prior writes to complete. On the other hand, the paper
says the read workload is uniform random (ruling out the first
explanation, since the Figure 2 arrangement evenly spreads the work
regardless); the paper implies that the leader doesn't send read
requests to the followers (ruling out the second explanation), and
it's not obvious why either kind of read would ever have to wait for
concurrent writes to complete.

Q: Why do Spinnaker's consistent reads have much higher performance
than Cassandra's quorum reads, in Figure 8?

A: The paper says that Spinnaker's consistent reads are processed only
by the leader, while Cassandra's reads involve messages to two
servers. It's not clear why it's legal for consistent reads to consult
only the leader -- suppose leadership has changed, but the old leader
isn't yet aware; won't that cause the old leader to serve up stale
data to a consistent read? We'd expect that either the leader wait for
the read to be committed to the log, or that the leader have a lease,
but the paper doesn't mention anything about leases.

Q: What is the CAP theorem about?

A: The key point is that if there is a network partition, you have two
choices: both partitions can continue to operate independently,
sacrificing consistency (since they won't see each others' writes); or
at most one partition can continue to operate, preserving consistency
but sacrificing forward progress in the other partitions.

Q: Where does Spinnaker sit in the CAP scheme?

A: Spinnaker's consistent reads allow operation in at most one
partition: reads in the majority partition see the latest write, but
minority partitions are not allowed to operate. Raft and Lab 3 are
similar. Spinnaker's timeline reads allow replicas in minority
partitions to process reads, but they are not consistent, since they may
not reflect recent writes.

Q: Could Spinnaker use Raft as the replication protocol rather than Paxos?

A: Although the details are slightly different, my thinking is that
they are interchangeable to first order. The main reason that we
reading this paper is as a case study of how to build a complete
system using a replicated log. It is similar to lab 3 and lab 4,
except you will be using your Raft library.

Q: The paper mentions Paxos hadn't previously been used for database
replication. What was it used for?

A: Paxos was not often used at all until recently. By 2011 most uses
were for configuration services (e.g., Google's Chubby or Zookeeper),
but not to directly replicate data.

Q: What is the reason for logical truncation?

A: The logical truncation exists because Spinnaker merges logs of 3
cohorts into a single physical log for performance on magnetic disks.
This complicates log truncation, because when one cohort wants to
truncate the log there maybe log entries from other cohorts, which it
cannot truncate. So, instead of truncating the log, it remembers its
entries that are truncated in a skip list. When the cohort recovers,
and starts replaying log entries from the beginning of the log, it
skips the entries in the skip list, because they are already present
in the last checkpoint of memtables.

Q: What exactly is the key range (i.e. 'r') defined in the paper for
leader election?

A: One of the key ranges of a shard (e.g., [0,199] in figure 2).

Q: Is there a more detailed description of Spinnaker's replication
protocol somewhere?

A: http://www.vldb.org/2011/files/slides/research15/rSession15-1.ppt

Q: How does Spinnaker's leader election ensure there is at most one leader?

A: The new leader is the candidate with the max n.lst in the Zookeeper
under /r/candidates, using Zookeeper sequence numbers to break ties.

Q: Does Spinnaker have something corresponding to Raft's terms?

Yes, it has epoch numbers (see appendix B).

Q: Section 9.1 says that a Spinnaker leader replies to a consistent
read without consulting the followers. How does the leader ensure that
it is still the leader, so that it doesn't reply to a consistent read
with stale data?

A: Unclear. Maybe the leader learns from Zookeeper that it isn't the
leader anymore, because some lease/watch times out.

Q: Would it be possible to replace the use of Zookeeper with a
Raft-like leader election protocol?

A: Yes, that would be a very reasonable thing to do. My guess is that
they needed Zookeeper for shard assignment and then decided to also
use it for leader election.

Q: What is the difference between a forced and a non-forced log write?

A: After a forced write returns, then it is guaranteed that the data
is on persistent storage. After a non-forced write returns, the write
to persistent storage has been issued, but may not yet be on
persistent storage.

Q: Step 6 of Figure 7 seems to say that the candidate with the longest
long gets to be the next leader. But in Raft we saw that this rule
doesn't work, and that Raft has to use the more elaborate Election
Restriction. Why can Spinnaker safely use longest log?

A: Spinnaker actually seems to use a rule similar to Raft's. Figure 7
compares LSNs (Log Sequence Numbers), and Appendix B says that an LSN
has the "epoch number" in the high bits. The epoch number is
equivalent to Raft's term. So step 6's "max n.lst" actuall boils down
to "highest epoch wins; if epochs are equal, longest log wins."


## LEC 8: Zookeeper

6.824 2017 Lecture 8: Zookeeper Case Study

Reading: "ZooKeeper: wait-free coordination for internet-scale systems", Patrick
Hunt, Mahadev Konar, Flavio P. Junqueira, Benjamin Reed.  Proceedings of the 2010
USENIX Annual Technical Conference.

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

## LEC 9: Guest lecturer on Go

Q: Can I stop these complaints about my unused variable/import?

A: There's a good explanation at https://golang.org/doc/faq#unused_variables_and_imports.

Q: Is the defer keyword in other languages?

A: Defer was new in Go. We originally added it to provide a way to
recover from panics (see "recover" in the spec), but it turned out
to be very useful for idioms like "defer mu.Unlock()" as well.
Later, Swift added a defer statement too. It seems clearly inspired
by Go but I'm not sure how close the details are.

Q: Why is the type after the variable declaration, unlike C languages?

A: There's a good explanation at https://blog.golang.org/gos-declaration-syntax.

Q: Why not adopt classes and OOP like in C++ and Java?

A: We believe that Go's approach to object-oriented programming,
which is closer to Smalltalk than to Java/C++/Simula, is more
lightweight and makes it easier to adapt large programs. I talked
about this at Google I/O in 2010. See
https://github.com/golang/go/wiki/GoTalks#go-programming for links
to the video and slides.

Q: Why does struct require a trailing comma on a multiline definition?

A: Originally it didn't, but all statements were terminated by
semicolons. We made semicolons optional shortly after the public
release of Go. When we did that, we tried to avoid Javascript's
mistake of making the semicolon rules very complex and error-prone.
Instead we have a simple rule: every line ends in an implicit
semicolon unless the final token is something that cannot possibly
end a statement (for example, a plus sign, or a comma). One effect
of this is that if you don't put the trailing comma on the line,
it gets an implicit semicolon, which doesn't parse well. It's
unfortunate, and it wasn't that way before the semicolon rules, but
we're so happy about not typing semicolons all the time that we'll
live with it. The original proposal for semicolon insertion is at
https://groups.google.com/d/msg/golang-nuts/XuMrWI0Q8uk/kXcBb4W3rH8J.
See the next answer also.

Q: Why are list definitions inconsistent, where some need commas and some do not?

A: The ones that don't need commas need semicolons, but those
semicolons are being inserted automatically (see previous answer).
The rule is that statements are separated by semicolons and smaller
pieces of syntax by commas:

	import "x";
	import "y";
	
	var x = []int{
		1,
		2,
		3,
	}

When you factor out a group of imports, you still have semicolons:

	import (
		"x";
		"y";
	)
	
	var x = []int{
		1,
		2,
		3,
	}

But then when we made semicolons optional, the semicolons disappeared
from the statement blocks leaving the commas behind:

	import (
		"x"
		"y"
	)
	
	var x = []int{
		1,
		2,
		3,
	}

Now the distinction is between nothing and something, instead of
two different characters, and it's more pronounced. If we had known
from the start that semicolons would be optional I think we might
have used them in more syntactic forms, or maybe made some forms
accept either commas or semicolons. At this point that seems
unlikely, though.

Q: Why does Go name its while loops "for"?

A: C has both while(cond) {} and for(;cond;) {}. It didn't seem
like Go needed two keywords for the same thing.

Q: There seem to be a lot of new languages emerging these days,
including Rust, D, Swift and Nim, among probably others.  Are there
any lessons you've learned from these other languages and their
communities that you wish you'd been able to incorporate into Go?

A: I do watch those languages for developments. I think they've
learned things from Go and I hope we've also learned things from
them. Some day I'd like the Go compiler to do a better job of
inferring ownership rules, or maybe even having lightweight ownership
expressions in the type system. Javari, Midori, Pony, and Rust are
inspirations here. I wrote a bit more about this at
https://research.swtch.com/go2017.

Q: Why the focus on concurrency and goroutines?

A: We knew from past experience that good concurrency support using
channels and lightweight processes would make writing the kinds of
systems we built at Google a lot easier, as I hope the lecture
showed. There's a bit more explanation at https://golang.org/doc/faq#csp,
and some background about our earlier experiences at
https://swtch.com/~rsc/thread/.

## LEC 10: Distributed Transactions

6.824 2018 Lecture 10: Distributed Transactions

Topics:
  distributed transactions = concurrency control + atomic commit

what's the problem?
  lots of data records, sharded on multiple servers, lots of clients
  [diagram: clients, servers, data sharded by key]
  client application actions often involve multiple reads and writes
    bank transfer: debit and credit
    vote on an article: check if already voted, record vote, increment count
    install bi-directional links in a social graph
  we'd like to hide interleaving and failure from application writers
  this is a traditional database concern
    today's material originated with [distributed] databases
    but the ideas are used in many distributed systems

example situation
  x and y are bank balances -- records in database tables
  x and y are on different servers (maybe at different banks)
  x and y start out as $10
  client C1 is doing a transfer of $1 from x to y
  client C2 is doing an audit, to check that no money has been lost
  C1:             C2:
  add(x, 1)       tmp1 = get(x)
  add(y, -1)      tmp2 = get(y)
                  print tmp1, tmp2

what do we hope for?
  x=11
  y=9
  C2 prints 10,10 or 11,9

what can go wrong?
  unhappy interleaving of C1 and C2's operations
    e.g. C2 executes entirely between C1's two operations, printing 11,10
  server or network failure
  account x or y doesn't exist

the traditional plan: transactions
  client tells the transaction system the start and end of each transaction
  system arranges that each transaction is:
    atomic: all writes occur, or none, even if failures
    serializable: results are as if transactions executed one by one
    durable: committed writes survive crash and restart
  these are the "ACID" properties
  applications rely on these properties!
  we are interested in *distributed* transactions
    data sharded over multiple servers

the application code for our example might look like this:
  T1:
    begin_transaction()
    add(x, 1)
    add(y, -1)
    end_transaction()
  T2:
    begin_transaction()
    tmp1 = get(x)
    tmp2 = get(y)
    print tmp1, tmp2
    end_transaction()

a transaction can "abort" if something goes wrong
  an abort un-does any record modifications
  the transaction might voluntarily abort, e.g. if the account doesn't exist
  the system may force an abort, e.g. to break a locking deadlock
  some servers failures result in abort
  the application might (or might not) try the transaction again

distributed transactions have two big components:
  concurrency control
  atomic commit

first, concurrency control
  correct execution of concurrent transactions

The traditional transaction correctness definition is "serializability"
  you execute some concurrent transactions, which yield results
    "results" means new record values, and output
  the results are serializable if:
    there exists a serial execution order of the transactions
    that yields the same results as the actual execution
  (serial means one at a time -- no parallel execution)
  (this definition should remind you of linearizability)

You can test whether an execution's result is serializable by
  looking for an order that yields the same results.
  for our example, the possible serial orders are
    T1; T2
    T2; T1
  so the correct (serializable) results are:
    T1; T2 : x=11 y=9 "11,9"
    T2; T1 : x=11 y=9 "10,10"
  the results for the two differ; either is OK
  no other result is OK
  the implementation might have executed T1 and T2 in parallel
    but it must still yield results as if in a serial order

what if T1's operations run entirely between T2's two get()s?
  would the result be serializable?
  T2 would print 10,9
  but 10,9 is not one of the two serializable results!
what if T2 runs entirely between T1's two adds()s?
  T2 would print 11,10
  but 11,10 is not one of the two serializable results!

Serializability is good for programmers
  It lets them ignore concurrency

two classes of concurrency control for transactions:
  pessimistic:
    lock records before use
    conflicts cause delays (waiting for locks)
  optimistic:
    use records without locking
    commit checks if reads/writes were serializable
    conflict causes abort+retry, but faster than locking if no conflicts
    called Optimistic Concurrency Control (OCC)

today: pessimistic concurrency control
next week: optimistic concurrency control

"Two-phase locking" is one way to implement serializability
  2PL definition:
    a transaction must acquire a record's lock before using it
    a transaction must hold its locks until *after* commit or abort 

2PL for our example
  suppose T1 and T2 start at the same time
  the transaction system automatically acquires locks as needed
  so first of T1/T2 to use x will get the lock
  the other waits
  this prohibits the non-serializable interleavings

details:
  each database record has a lock
  if distributed, the lock is typically stored at the record's server
    [diagram: clients, servers, records, locks]
    (but two-phase locking isn't affected much by distribution)
  an executing transaction acquires locks as needed, at the first use
    add() and get() implicitly acquires record's lock
    end_transaction() releases all locks
  all locks are exclusive (for this discussion, no reader/writer locks)
  the full name is "strong strict two-phase locking"
  related to thread locking (e.g. Go's Mutex), but easier:
    explicit begin/end_transaction
    DB understands what's being locked (records)
    possibility of abort (e.g. to cure deadlock)

Why hold locks until after commit/abort?
  why not release as soon as done with the record?
  example of a resulting problem:
    suppose T2 releases x's lock after get(x)
    T1 could then execute between T2's get()s
    T2 would print 10,9
    oops: that is not a serializable execution: neither T1;T2 nor T2;T1
  example of a resulting problem:
    suppose T1 writes x, then releases x's lock
    T2 reads x and prints
    T1 then aborts
    oops: T2 used a value that never really existed
    we should have aborted T2, which would be a "cascading abort"; awkward

Could 2PL ever forbid a correct (serializable) execution?
  yes; example:
    T1        T2
    get(x)  
              get(x)
              put(x,2)
    put(x,1) 
  locking would forbid this interleaving
  but the result (x=1) is serializable (same as T2;T1)

Locking can produce deadlock, e.g.
  T1      T2
  get(x)  get(y)
  get(y)  get(x)
The system must detect (cycles? lock timeout?) and abort one of the transactions

The Question: describe a situation where Two-Phase Locking yields
higher performance than Simple Locking. Simple locking: lock *every*
record before *any* use; release after abort/commit. 

Next topic: distributed transactions versus failures

how can distributed transactions cope with failures?
  suppose, for our example, x and y are on different "worker" servers
  suppose x's server adds 1, but y's crashes before subtracting?
  or x's server adds 1, but y's realizes the account doesn't exist?
  or x and y both do their part, but aren't sure if the other did?

We want "atomic commit":
  A bunch of computers are cooperating on some task
  Each computer has a different role
  Want to ensure atomicity: all execute, or none execute
  Challenges: failures, performance

We're going to develop a protocol called "two-phase commit"
  Used by distributed databases for multi-server transactions
  We'll assume the database is *also* locking

Two-phase commit without failures:
  the transaction is driven from the Transaction Coordinator
  [time diagram: TC, A, B]
  TC sends put(), get(), &c RPCs to A, B
    The modifications are tentative, only to be installed if commit.
  TC sees transaction_end()
  TC sends PREPARE messages to A and B.
  If A (or B) is willing to commit,
    respond YES.
    then A/B in "prepared" state.
  otherwise, respond NO.
  If both say YES, TC sends COMMIT messages.
  If either says NO, TC sends ABORT messages.
  A/B commit if they get a COMMIT message.
    I.e. they write tentative records to the real DB.
    And release the transaction's locks on their records.

Why is this correct so far?
  Neither A or B can commit unless they both agreed.

What if B crashes and restarts?
  If B sent YES before crash, B must remember!
  Because A might have received a COMMIT and committed.
  So B must be able to commit (or not) even after a reboot.

Thus subordinates must write persistent (on-disk) state:
  B must remember on disk before saying YES, including modified data.
  If B reboots, disk says YES but no COMMIT, B must ask TC, or wait for TC to re-send.
  And meanwhile, B must continue to hold the transaction's locks.
  If TC says COMMIT, B copies modified data to real data.

What if TC crashes and restarts?
  If TC might have sent COMMIT before crash, TC must remember!
    Since one worker may already have committed.
  And repeat that if anyone asks (i.e. if A/B didn't get msg).
  Thus TC must write COMMIT to disk before sending COMMIT msgs.

What if TC never gets a YES/NO from B?
  Perhaps B crashed and didn't recover; perhaps network is broken.
  TC can time out, and abort (since has not sent any COMMIT msgs).
  Good: allows servers to release locks.

What if B times out or crashes while waiting for PREPARE from TC?
  B has not yet responded to PREPARE, so TC can't have decided commit
  so B can unilaterally abort, and release locks
  respond NO to future PREPARE

What if B replied YES to PREPARE, but doesn't receive COMMIT or ABORT?
  Can B unilaterally decide to abort?
    No! TC might have gotten YES from both,
    and sent out COMMIT to A, but crashed before sending to B.
    So then A would commit and B would abort: incorrect.
  B can't unilaterally commit, either:
    A might have voted NO.

So: if B voted YES, it must "block": wait for TC decision.
  
Two-phase commit perspective
  Used in sharded DBs when a transaction uses data on multiple shards
  But it has a bad reputation:
    slow: multiple rounds of messages
    slow: disk writes
    locks are held over the prepare/commit exchanges; blocks other xactions
    TC crash can cause indefinite blocking, with locks held
  Thus usually used only in a single small domain
    E.g. not between banks, not between airlines, not over wide area
  Faster distributed transactions are an active research area:
    Lower message and persistence cost
    Special cases that can be handled with less work
    Wide-area transactions
    Less consistency, more burden on applications

Raft and two-phase commit solve different problems!
  Use Raft to get high availability by replicating
    i.e. to be able to operate when some servers are crashed
    the servers all do the *same* thing
  Use 2PC when each subordinate does something different
    And *all* of them must do their part
  2PC does not help availability
    since all servers must be up to get anything done
  Raft does not ensure that all servers do something
    since only a majority have to be alive

What if you want high availability *and* atomic commit?
  Here's one plan.
  [diagram]
  Each "server" should be a Raft-replicated service
  And the TC should be Raft-replicated
  Run two-phase commit among the replicated services
  Then you can tolerate failures and still make progress
  You'll build something like this to transfer shards in Lab 4
  Next meeting's FaRM has a different approach

Distributed Transactions FAQ

Q: How does this material fit into 6.824?

A: When people build distributed systems that spread data and
execution over many computers, they need ways to specify behavior,
both to reason about internal correctness and to define
application-visible semantics. There are many possible ways to define
these semantics. Databases often provide relatively strong semantics
involving transactions and serializability, and implement them with
two-phase locking and (for distributed databases) two-phase commit.
Today's reading from the 6.033 textbook explains those ideas. Later
we'll look at systems that provide similarly strong semantics, as well
as systems that relax consistency in search of higher performance.

Q: Why is it so important for transactions to be atomic?

A: What "transaction" means is that the steps inside the transaction occur
atomically with respect to failures and other transactions. Atomic here
means "all or none". Transactions are a feature provided by some storage
systems to make programming easier. An example of the kind of
situation where transactions are helpful is bank transfers. If the bank
wants to transfer $100 from Alice's account to Bob's account, it would
be very awkward if a crash midway through this left Alice debited by
$100 but Bob *not* credited by $100. So (if your storage system supports
transactions) the programmer can write something like

BEGIN TRANSACTION
  decrease Alice's balance by 100;
  increase Bob's balance by 100;
END TRANSACTION

and the transaction system will make sure the transaction is atomic;
either both happen, or neither, even if there's a failure somewhere.

Q: Could one use Raft instead of two-phase commit?

A: Two-phase commit and Raft solve different problems.

Two-phase commit causes different computers to do *different* things
(e.g. Alice's bank debits Alice, Bob's bank credits Bob), and we want
them *all* to do their thing, or none of them. Two-phase commit
systems are typically not available (cannot make progress) in the face
of failures, since they require all participating computers to perform
their part of the transaction.

Raft causes the peers to all do the *same* thing (so they remain
replicas). It's OK to wait only for a majority, since the peers are
replicas, and therefor we can make the system available in the face of
failures.

Q: In two-phase commit, why would a worker send an abort message,
rather than a PREPARED message?

A: The reason we care most about is if the subordinate crashed and
rebooted after it did some of its work for the transaction but before
it received the prepare message; during the crash it will have lost
the record of tentative updates it made and locks it acquired, so it
cannot complete the transaction. Another possibility (depending on how
the DB works) is if the worker detected a violated constraint on
the data (e.g. the transaction tried to write a record with a
duplicate key in a table that requires unique keys). Another
possibility is that the worker is involved in a deadlock, and
must abort to break the deadlock.

Q: Can two-phase locking generate deadlock?

A: Yes. If two transactions both use records R1 and R2, but in
opposite orders, they will each acquire one of the locks, and then
deadlock trying to get the other lock. Databases are able to detect
these deadlocks and break them. A database can detect deadlock by
timing out lock acquisition, or by finding cycles in the waits-for
graph among transactions. Deadlocks can be broken by aborting one of
the participating transactions.

Q: Why does it matter whether locks are held until after a transaction
commits or aborts?

A: If transactions release locks before they commit, it can be hard to
avoid certain non-serializable executions due to aborts or crashes. In
this example, suppose T1 releases the lock on x after it updates x,
but before it commits:

  T1:           T2:
  x = x + 1
                y = x
                commit

  commit
  
It can't be legal for y to end up greater than x. Yet if T1 releases
its lock on x, then T2 acquires the lock, writes y, and commits, but
then T1 aborts or the system crashes and cannot complete T1, we will
end up with y greater than x.

It's to avoid having to cope with the above that people use the
"strong strict" variant of 2PL, which only releases locks after a
commit or abort.

Q: What is the point of the two-phase locking rule that says a
transaction isn't allowed to acquire any locks after the first time
that it releases a lock?

A: The previous question/answer outlines one answer.

Another answer is that, even without failure or abort, acquiring after
releasing can lead to non-serializable executions.

  T1:         T2:
  x = x + 1
              z = x + y
  y = y + 1

Suppose x and y start out as zero, and both transactions execute, and
successfully commit. The only final values of z that are allowed by
serializability are zero and 2 (corresponding to the orders T2;T1 and
T1;T2). But if T1 releases its lock on x before acquiring the lock on
y and modifying y, T2 could completely execute and commit while T1 is
between its two statements, giving z a value of 1, which is not legal.
If T1 keeps its lock on x while using y, as two-phase locking demands,
this problem is avoided.

Q: Does two-phase commit solve the dilemma of the two generals
described in the reading's Section 9.6.4?

A: If there are no failures, and no lost messages, and all messages are
delivered quickly, two-phase commit can solve the dilemma. If the
Transaction Coordinator (TC) says "commit", both generals attack at
the appointed time; if the TC says "abort", neither attacks.

In the real world, messages can be lost and delayed, and the TC could
crash and not restart for a while. Then we could be in a situation
where general G1 heard "commit" from the TC, and general G2 heard
nothing. They might be in this state when the time appointed for
attack arrives. What should G1 and G2 do at this point? I can't think
of a set of rules for the generals to follow that leads to an
acceptable outcome across a range of situations.

This set of rules doesn't work, since it leads to only one general
attacking:

  * if you heard "commit" from the TC, do attack.
  * if you heard "abort" from the TC, don't attack.
  * if you heard nothing from the TC, don't attack.

We can't have this either, since then if G1 heard "abort" and
G2 heard nothing, we'd again have only one general attacking:

  * if you heard "commit" from the TC, do attack.
  * if you heard "abort" from the TC, don't attack.
  * if you heard nothing from the TC, do attack.     [note the "do" here]

This is safe, but leads to the generals never attacking no matter what:

  * if you heard "commit" from the TC, don't attack.
  * if you heard "abort" from the TC, don't attack.
  * if you heard nothing from the TC, don't attack.

The real difficulty in the dilemma is that there's a hard deadline at
which both generals (or neither) must simultaneously attack. If there
is no deadline, and it's OK for the participants (workers) to commit
at different times, then two-phase commit is useful.

Q: Are the locks exclusive, or can they allow multiple readers to have
simultaneous access?

A: By default, "lock" in 6.824 refers to an exclusive lock. But there
are databases that can grant locking access to a record to either
multiple readers, or a single writer. Some care has to be taken when a
transaction reads a record and then writes it, since the lock will
initially be a read lock and then must be upgraded to a write lock.
There's also increased opportunity for deadlock in some situations; if
two transactions simultaneously want to increment the same record,
they might deadlock when upgrading a read lock to a write lock on that
record, whereas if locks are always exclusive, they won't deadlock.

Q: How should one decide between pessimistic and optimistic
concurrency control?

A: If your transactions conflict a lot (use the same records, and one
or more transactions writes), then locking is better. Locking causes
transactions to wait, whereas when there are conflicts, most OCC
systems abort one of the transactions; aborts (really the consequent
retries) are expensive.

If your transactions rarely conflict, then OCC is preferable to
locking. OCC doesn't spend CPU time acquiring/releasing locks and (if
conflicts are rare) OCC rarely aborts. The "validation" phase of OCC
systems often uses locks, but they are usually held for shorter
periods of time than the locks in pessimistic designs.

Q: What should two-phase commit workers do if the transaction
coordinator crashes?

A: If a worker has told the coordinator that it is ready to commit,
then the worker cannot later change its mind. The reason is that the
coordinator may (before it crashed) have told other workers to commit.
So the worker has to wait (with locks held) for the coordinator to
reboot and re-send its decision.

Waiting indefinitely with locks held is a real problem, since the
locks can force a growing set of other transactions to block as well.
So people tend to avoid two-phase commit, or they try to make
coodinators reliable. For example, Google's Spanner replicates
coordinators (and all other servers) using Paxos.

Q: Why don't people use three-phase commit, which allows workers to
commit or abort even if the coordinator crashes?

A: Three-phase commit only works if the network is reliable, or if
workers can reliably distinguish between the coordinator being dead
and the network not delivering packets. For example, three-phase
commit won't work correctly if there's a network partition. In most
practical networks, partition is possible.

[Read 6.033 Chapter 9, just 9.1.5, 9.1.6, 9.5.2, 9.5.3, 9.6.3](https://ocw.mit.edu/resources/res-6-004-principles-of-computer-system-design-an-introduction-spring-2009/online-textbook/)

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

## LEC 12: Big Data: Spark

6.824 2018 Lecture 12: Spark Case Study

Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing
Zaharia, Chowdhury, Das, Dave, Ma, McCauley, Franklin, Shenker, Stoica
NSDI 2012

Today: more distributed computations
  Case study: Spark
  Why are we reading Spark?
    Widely-used for datacenter computations
    popular open-source project, hot startup (Databricks)
    Support iterative applications better than MapReduce
    Interesting fault tolerance story
    ACM doctoral thesis award

MapReduce make life of programmers easy
  It handles:
    Communication between nodes
    Distribute code
    Schedule work
    Handle failures
  But restricted programming model
    Some apps don't fit well with MapReduce

Many algorithms are iterative
  Page-rank is the classic example
    compute a rank for each document, based on how many docs point to it
    the rank is used to determine the place of the doc in the search results
  on each iteration, each document:
    sends a contribution of r/n to its neighbors
       where r is its rank and n is its number of neighbors.
    update rank to alpha/N + (1 - alpha)*Sum(c_i),
       where the sum is over the contributions it received
       N is the total number of documents.
  big computation:
    runs over all web pages in the world
    even with many machines it takes long time

MapReduce and iterative algorithms
  MapReduce would be good at one iteration of the algorithms
    Each map does part of the documents
    Reduce for update the rank of a particular doc
  But what to do for the next iteration?
    Write results to storage
    Start a new MapReduce job for the next iteration
    Expensive
    But fault tolerant

Challenges
  Better programming model for iterative computations
  Good fault tolerance story

One solution: use DSM
  Good for iterative programming
    ranks can be in shared memory
    workers can update and read
  Bad for fault tolerance
    typical plan: checkpoint state of memory
      make a checkpoint every hour of memory
    expensive in two ways:
      write shared memory to storage during computation
      redo all work since last checkpoint after failure
  Spark more MapReduce flavor
    Restricted programming model, but more powerful than MapReduce
    Good fault tolerance plan

Better solution: keep data in memory
  Pregel, Dryad, Spark, etc.
  In Spark
    Data is stored in data sets (RDDs)
    "Persist" the RDD in memory
    Next iteration can refer to the RDD

Other opportunities
  Interactive data exploration
    Run queries over the persisted RDDs
  Like to have something SQL-like
    A join operator over RDDs

Core idea in Spark: RDDs
  RDDs are immutable --- you cannot update them
  RDDs support transformations and actions
  Transformations:  compute a new RDD from existing RDDs
    map, reduceByKey, filter, join, ..
    transformations are lazy: don't compute result immediately
    just a description of the computation
  Actions: for when results are needed
    counts result, collect results, get a specific value

Example use:
  lines = spark.textFile("hdfs://...")
  errors = lines.filter(_.startsWith("ERROR"))    // lazy!
  errors.persist()    // no work yet
  errors.count()      // an action that computes a result
  // now errors is materialized in memory
  // partitioned across many nodes
  // Spark, will try to keep in RAM (will spill to disk when RAM is full)

Reuse of an RDD
  errors.filter(_.contains("MySQL")).count()
  // this will be fast because reuses results computed by previous fragment
  // Spark will schedule jobs across machines that hold partition of errors

Another reuse of RDD
  errors.filter(_.contains("HDFS")).map(_.split('\t')(3)).collect()

RDD lineage
  Spark creates a lineage graph on an action
  Graphs describe the computation using transformations
    lines -> filter w ERROR -> errors -> filter w. HDFS -> map -> timed fields
  Spark uses the lineage to schedule job
    Transformation on the same partition form a stage
      Joins, for example, are a stage boundary
      Need to reshuffle data
    A job runs a single stage
      pipeline transformation within a stage
    Schedule job where the RDD partition is

Lineage and fault tolerance
  Great opportunity for *efficient* fault tolerance
    Let's say one machine fails
    Want to recompute *only* its state
    The lineage tells us what to recompute
      Follow the lineage to identify all partitions needed
      Recompute them
  For last example, identify partitions of lines missing
    Trace back from child to parents in lineage
    All dependencies are "narrow"
      each partition is dependent on one parent partition
    Need to read the missing partition of lines
      recompute the transformations

RDD implementation
  list of partitions
  list of (parent RDD, wide/narrow dependency)
    narrow: depends on one parent partition  (e.g., map)
    wide: depends on several parent partitions (e.g., join)
  function to compute (e.g., map, join)
  partitioning scheme (e.g., for file by block)
  computation placement hint

Each transformation takes (one or more) RDDs, and outputs the transformed RDD.

Q: Why does an RDD carry metadata on its partitioning?  A: so transformations
  that depend on multiple RDDs know whether they need to shuffle data (wide
  dependency) or not (narrow). Allows users control over locality and reduces
  shuffles.

Q: Why the distinction between narrow and wide dependencies?  A: In case of
  failure.  Narrow dependency only depends on a few partitions that need to be
  recomputed.  Wide dependency might require an entire RDD

Example: PageRank (from paper):
  // Load graph as an RDD of (URL, outlinks) pairs
  val links = spark.textFile(...).map(...).persist() // (URL, outlinks)
  var ranks = // RDD of (URL, rank) pairs
  for (i <- 1 to ITERATIONS) {
    // Build an RDD of (targetURL, float) pairs
    // with the contributions sent by each page
    val contribs = links.join(ranks).flatMap {
      (url, (links, rank)) => links.map(dest => (dest, rank/links.size))
    }
    // Sum contributions by URL and get new ranks
    ranks = contribs.reduceByKey((x,y) => x+y)
     .mapValues(sum => a/N + (1-a)*sum)
  }

Lineage for PageRank
  See figure 3
  Each iteration creates two new RDDs:
    ranks0, ranks1, etc.
    contribs0, contribs1, etc.
  Long lineage graph!
    Risky for fault tolerance.
    One node fails, much recomputation
  Solution: user can replicate RDD
    Programmer pass "reliable" flag to persist()
     e.g., call ranks.persist(RELIABLE) every N iterations
    Replicates RDD in memory
    With REPLICATE flag, will write to stable storage (HDFS)
  Impact on performance
   if user frequently perist w/REPLICATE, fast recovery, but slower execution
   if infrequently, fast execution but slow recovery

Q: Is persist a transformation or an action?  A: neither. It doesn't create a
 new RDD, and doesn't cause materialization. It's an instruction to the
 scheduler.

Q: By calling persist without flags, is it guaranteed that in case of fault that
  RDD wouldn't have to be recomputed?  A: No. There is no replication, so a node
  holding a partition could fail.  Replication (either in RAM or in stable
  storage) is necessary

Currently only manual checkpointing via calls to persist.  Q: Why implement
  checkpointing? (it's expensive) A: Long lineage could cause large recovery
  time. Or when there are wide dependencies a single failure might require many
  partition re-computations.

Q: Can Spark handle network partitions? A: Nodes that cannot communicate with
  scheduler will appear dead. The part of the network that can be reached from
  scheduler can continue computation, as long as it has enough data to start the
  lineage from (if all replicas of a required partition cannot be reached,
  cluster cannot make progress)

What happens when there isn't enough memory?
  LRU (Least Recently Used) on partitions
     first on non-persisted
     then persisted (but they will be available on disk. makes sure user cannot overbook RAM)
  User can have control on order of eviction via "persistence priority"
  No reason not to discard non-persisted partitions (if they've already been used)

Performance
  Degrades to "almost" MapReduce behavior
  In figure 7, logistic regression on 100 Hadoop nodes takes 76-80 seconds
  In figure 12, logistic regression on 25 Spark nodes (with no partitions allowed in memory)
    takes 68.8
  Difference could be because MapReduce uses replicated storage after reduce, but Spark by
  default only spills to local disk
    no network latency and I/O load on replicas.
  no architectural reason why MR would be slower than Spark for non-iterative work
    or for iterative work that needs to go to disk
  no architectural reason why Spark would ever be slower than MR

Discussion
  Spark targets batch, iterative applications
  Spark can express other models
    MapReduce, Pregel
  Cannot incorporate new data as it comes in
    But see Streaming Spark
  Spark not good for building key/value store
    Like MapReduce, and others
    RDDs are immutable

References
  http://spark.apache.org/
  http://www.cs.princeton.edu/~chazelle/courses/BIB/pagerank.htm

Q: Is Spark currently in use in any major applications?

A: Yes: https://databricks.com/customers

Q: How common is it for PhD students to create something on the scale
of Spark?

A: Unusual! It is quite an accomplishment. Matei Zaharia won the ACM
doctoral thesis award for this work.

Q: Should we view Spark as being similar to MapReduce?

A: There are similarities, but Spark can express computations that are
difficult to express in a high-performance way in MapReduce, for
example iterative algorithms. You can think of Spark as MapReduce and
more. The authors argue that Spark is powerful enough to express a
range of prior computation frameworks, including MapReduce (see
section 7.1).

There are systems that are better than Spark in incorporating new data
that is streaming in, instead of doing batch processing. For example,
Naiad (https://dl.acm.org/citation.cfm?id=2522738). Spark also has
streaming support, although it implements it in terms of computations
on "micro-batches" (see Spark Streaming, SOSP 2013).

There are also systems that allow more fine-grained sharing between
different machines (e.g., DSM systems) or a system such as Picolo
(http://piccolo.news.cs.nyu.edu/), which targets similar applications
as Spark.

Q: Why are RDDs called immutable if they allow for transformations?

A: A transformation produces a new RDD, but it may not be materialized
explicitly, because transformations are computed lazily. In the
PageRank example, each loop iteration produces a new distrib and rank
RDD, as shown in the corresponding lineage graph.

Q: Do distributed systems designers worry about energy efficiency?

A: Energy is a big concern in CS in general! But most of the focus in
energy-efficiency in cluster computing goes to the design of data
centers and the design of the computers and cooling inside the data
center.

Chip designers pay lots of attention to energy; for example, your
processor dynamically changes the clock rate to avoid making the
processor too hot.

There is less focus on energy in the design of distributed systems,
mostly, I think, because that is not where the big wins are. But there
is some work, for example see http://www.cs.cmu.edu/~fawnproj/.

Q: How do applications figure out the location of an RDD?

A: The application names RDDs with variable names in Scala. Each RDD
has location information associated with it, in its metadata (see
Table 3). The scheduler uses all this information to colocate
computations with the data. An RDD may be generated by multiple nodes
but it is for different partitions of the RDD.

Q: How does Spark achieve fault tolerance?

A: When persisting an RDD, the programmer can specify that it must be
replicated on a few machines. Spark doesn't need complicated protocols
like Raft, however, because RDDs are immutable and can always be
recomputed using the lineage graph.

Q: Why is Spark developed using Scala? What's special about the language?

A: In part because Scala was new and hip when the project started.

One good reason is that Scala provides the ability to serialize and ship
user-defined code ("closures") as discussed in 搂5.2.

This is something that is fairly straightforward in JVM-based languages
(such as Java and Scala), but tricky to do in C, C++ or Go, partly
because of shared memory (pointers, mutexes, etc.) and partly because the
closure needs to capture all variables referred to inside it (which is
difficult unless a language runtime can help with it).

Q: Does anybody still use MapReduce rather than Spark, since Spark seems
to be strictly superior? If so, why do people still use MR?

If the computation one needs fits the MapReduce paradigm well, there is
no advantage to using Spark over using MapReduce.

For example if the computation does a single scan of a large data set
(map phase) followed by an aggregation (reduce phase), the computation
will be dominated by I/O and Spark's in-memory RDD caching will offer
no benefit since no RDD is ever re-used.

Spark does very well on iterative computations with a lot of internal
reuse, but has no architectural edge over MapReduce or Hadoop for simple
jobs that scan a large data set and aggregate (i.e., just map() and
reduceByKey() transformations in Spark speak). On the other hand,
there's also no reason that Spark would be slower for these computations,
so you could really use either here.

Q: Is the RDD concept implemented in any systems other than Spark?

Spark and the specific RDD interface are pretty intimately tied to each
other. However, two key ideas behind RDDs -- deterministic, lineage-based
re-execution and the collections-oriented API -- are certainly widely-used
in many systems. For example, DryadLINQ, FlumeJava, and Cloud Dataflow
offer similar collection-oriented APIs; and the Dryad and Ciel systems
referenced by the paper also keep track of how pieces of data are
computed, and re-execute that computation on failure, similar to
lineage-based fault tolerance.

As a matter of fact, RDDs themselves in Spark are now somewhat
deprecated: Spark has recently moved towards something called "DataFrames"
(https://spark.apache.org/docs/latest/sql-programming-guide.html#datasets-and-dataframes),
which implements a more column-oriented representation while maintaining
the good ideas from RDDs.




## LEC 13: Big Data: Naiad

6.824 2018 Lecture 13: Naiad Case Study

Naiad: A Timely Dataflow System
Murray, McSherry, Isaacs, Isard, Barham, Abadi
SOSP 2013

Today: streaming and incremental computation
  Case study: Naiad
  Why are we reading Naiad?
    elegant design
    impressive performance
    open-source and still being developed

recap: Spark improved performance for iterative computations
  i.e., ones where the same data is used again and again
  but what if the input data changes?
    may happen, e.g., because crawler updates pages (new links)
    or because an application appends new record to a log file
  Spark (as in last lecture's paper) will actually have to start over!
    run all PageRank iterations again, even if just one link changed
    (but: Spark Streaming -- similar to Naiad, but atop Spark)

Incremental processing
  input vertices
    could be a file (as in Spark/MR)
    or a stream of events, e.g. web requests, tweets, sensor readings...
    inject (key, value, epoch) records into graph
  fixed data-flow
    records flow into vertices, which may emit zero or more records in response
  stateful vertices (long-lived)
    a bit like cached RDDs
    but mutable: state changes in response to new records arriving at the vertex
    ex: GroupBy with sum() stores one record for every group
      new record updates current aggregation value

Iteration with data-flow cycles
  contrast: Spark has no cycles (DAG), iterates by adding new RDDs
    no RDD can depend on its own child
  good for algorithms with loops
    ex: PageRank sends rank updates back to start of computation
  "root" streaming context
    special, as introduces inputs with epochs (specified by program)
  loop contexts
    can nest arbitrarily
    cannot partially overlap
  [Figure 3 example w/ nested loop]

Problem: ordering required!
  [example: delayed PR rank update, next iteration reads old rank]
  intuition: avoid time-travelling updates
    vertex sees update from past iteration when it already moved on
    vertex emits update from a future iteration it's not at yet
  key ingredient: hierarchical timestamps
    timestamp: (epoch, [c1, ..., ck]) where ci is the ith-level loop context
  ingress, egress, feedback vertices modify timestamps
    Ingress: append new loop counter at end (start into loop)
    Egress: drop loop counter from end (break out of loop)
    Feedback: increment loop counter at end (iterate)
  [Figure 3 timestamp propagation: lecture question]
    epoch 1:
    (a, 2): goes around the loop three times (value 4 -> 8 -> 16)
      A: (1, []), I: (1, [0]), ... F: (1, [1]), ... F: (1, [2]), ... E: (1, [])
    (b, 6): goes around the loop only once (value 6 -> 12)
      A: (1, []), I: (1, [0]), ... F: (1, [1]), ... E: (1, [])
    epoch 2:
    (a, 5): dropped at A due to DISTINCT.
  timestamps form a partial order
    (1, [5, 7]) < (1, [5, 8]) < (1, [6, 2]) < (2, [1, 1])
    but also: (1, [5, 7]) < (1, [5]) -- lexicographic ordering on loop counters
    and, for two top-level loops: (1, lA [5]) doesn't compare to (1, lB [2])
  allows vertices to order updates
    and to decide when it's okay to release them
    so downstream vertex gets a consistent outputs in order

Programmer API
  good news: end users mostly don't worry about the timestamps
    timely dataflow is low-level infrastructure
    allow special-purpose implementations by experts for specific uses
  [Figure 2: runtime, timely dataflow, differential dataflow, other libs]
  high-level APIs provided by libraries built on the timely dataflow abstraction
    e.g.: LINQ (SQL-like), BSP (Pregel-ish), Datalog
    similar trend towards special-purpose libraries in Spark ecosystem
      SparkSQL, GraphX, MLLib, Streaming, ..

Low-level vertex API
  need to explicitly deal with timestamps
  notifications
    "you'll no longer receive records with this timestamp or any earlier one"
    vertex may e.g., decide to compute final value and release
  two callbacks invoked by the runtime on vertex:
    OnRecv(edge, msg, timestamp) -- here are some messages
    OnNotify(timestamp)          -- no more messages <= timestamp coming
  two API calls available to vertex code:
    SendBy(edge, msg, timestamp) -- send a message at current or future time
    NotifyAt(timestamp)          -- call me back when at timestamp
  allows different strategies
    incrementally release records for a time, finish up with notification
    buffer all records for a time, then release on notification
  [code example: Distinct Count vertex]

Progress tracking
  protocol to figure out when to deliver notifications to vertices
  intuition
    ok to deliver notification once it's *impossible* for predecessors to
    generate records with earlier timestamp
  single-threaded example
    "pointstamps": just a timestamp + location (edge or vertex)
    [lattice diagram of vertices/edges on x-axis, times on y-axis]
    arrows indicate could-result-in
      follow partial order on timestamps
      ex: (1, [2]) at B in example C-R-I (1, [3]) but also (1, [])
      => not ok to finish (1, []) until everything has left the loop
    remove active pointstamp once no events for it are left
      i.e., occurrency count (OC) = 0
    frontier: pointstamp without incoming arrows
      keeps moving down the time axis, but speed differs for locations
  distributed version
    strawman 1: send all events to a single coordinator
      slow, as need to wait for this coordinator
      this is the "global frontier" -- coordinator has all information
    strawman 2: process events locally, then inform all other workers
      broadcast!
      workers now maintain "local frontier", which approximates global one
      workers can only be *behind*: progress update may still be in the network
        local frontier can never advance past global frontier
        hence safe, as will only deliver notifications late
      problem: sends enormous amounts of progress chatter!
        ~10 GB for WCC on 60 computers ("None", Figure 6(c))
    solution: aggregate events locally before broadcasting
      each event changes OC by +1 or -1 => can combine
      means we may wait a little longer for an update (always safe)
        only sends updates when set of active pointstamps changes
      global aggregator merges updates from different workers
        like global coordinator in strawman, but far fewer messages

Fault tolerance
  perhaps the weakest part of the paper
  option 1: write globally synchronous, coordinated checkpoints
    recovery loads last checkpoint (incl. timestamps, progress info)
    then starts processing from there, possibly repeating inputs
    induces pause times while making checkpoints (cf. tail latency, Fig 7c)
  option 2: log all messages to disk before sending
    no need to checkpoint
    recovery can resume from any point
    but high common-case overhead (i.e., pay price even when there's no failure)
  Q: why not use a recomputation strategy like Spark's?
  A: difficult to do with fine-grained updates. Up to what point do we recompute?
  cleverer scheme developed after this paper
    some vertices checkpoint, some log messages
    can still recover the joint graph
    see Falkirk Wheel paper -- https://arxiv.org/abs/1503.08877

Performance results
  optimized system from ground up, as illustrated by microbenchmarks (Fig 6)
  impressive PageRank performance (<1s per iteration on twitter_rv graph)
    very good in 2013, still pretty good now!
  Naiad matches or beats specialized systems in several different domains
    iterative batch: PR, SCC, WCC &c vs. DB, DryadLINQ -- >10x improvement
    graph processing: PR on twitter vs. PowerGraph -- ~10x improvement
    iterative ML: logistic regression vs. Vowpal Wabbit -- ~40% improvement

References:
  Differential Dataflow: http://cidrdb.org/cidr2013/Papers/CIDR13_Paper111.pdf
  Rust re-implementations:
    * https://github.com/frankmcsherry/timely-dataflow
    * https://github.com/frankmcsherry/differential-dataflow

FAQ Naiad

Q: Is Naiad/timely dataflow in use in production environments anywhere? If not, and what would be barriers to adoption?

A: The original Naiad research prototype isn't used in production, as far as I know. However, one of the authors has implemented a Naiad-like timely dataflow system in Rust:

https://github.com/frankmcsherry/timely-dataflow
https://github.com/frankmcsherry/differential-dataflow

There's no widespread industry use of it yet, perhaps because it's not backed by a company that offers commercial support, and because it doesn't support interaction with other widely-used systems (e.g., reading/writing HDFS data).

One example of real-world adoption is this proposal to use of differential dataflow (which sits on top of Naiad-style timely dataflow) to speed up program analysis in the Rust compiler (https://internals.rust-lang.org/t/lets-push-non-lexical-lifetimes-nll-over-the-finish-line/7115/8). That computation only involves parallel processing on a single machine, not multiple machines, however.


Q: The paper says that Naiad performs better than Spark. In what cases would one prefer which one over the other?

Naiad performs better than Spark for incremental and streaming computations. Naiad's performance should be equal to, or better than, Spark's for other computations such as classic batch and iterative processing.

There is no architectural reason why Naiad would ever be slower than Spark. However, Spark perhaps has a better story on interactive queries. In Spark, they can use existing, cached in-memory RDDs, while Naiad would require the user to recompile the program and run it again. Spark is also quite well-integrated with the Hadoop ecosystem and other systems that people already use (e.g., Python statistics libraries, cluster managers like YARN and Mesos), while Naiad is largely a standalone system.


Q: What was the reasoning for implmenting Naiad in C#? It seems like this language choice introduced lots of pain, and performance would be better in C/C++.

A: I believe it was because C#, as a memory-safe language, made development easier. Outside the data processing path (e.g., in coordination code), raw performance doesn't matter that much, and C/C++ would introduce debugging woes for little gain there.

However, as you note, the authors actually had to do a fair amount of optimization work to address performance problems with C# on the critical data processing path (e.g., garbage collection, 搂3.5). One of the Naiad authors has recently re-implemented the system in Rust, which is a low-level compiled language, and there is some evidence that this implementation performs better than the original Naiad.


Q: I'm confused by the OnRecv/OnNotify/SendBy/NotifyAt API. Does the user need to tell the system when to notify another vertex?

A: OnRecv and OnNotify are *callbacks* that get invoked on a vertex by the Naiad runtime system, not the user. The data-flow vertex has to supply an implementation of how to deal with these callbacks when invoked.

SendBy and NotifyAt are API calls available to a data-flow vertex implementation to emit output records (SendBy) and to tell the runtime system that the vertex would like to receive a callback at a certain time (NotifyAt). It's the runtime system's job to figure, using the progress tracking protocol, when it is appropriate to invoke NotifyAt.

This API allows for significant flexibility: for example, SendAt() can emit a record with a timestamp in the future, rather than buffering the record internally, which makes out-of-order processing much easier. NotifyAt allows the vertex to control how often it receives notifications: maybe it only wishes to be notified every 1,000 timestamps in order to batch its processing work.

Here's an example vertex implementation that both forwards records immediately and also requests a notification for a later time, at which it computes some aggregate statistics and emits them:

class MyFancyVertex {
  function OnRecv(t: Timestamp, m: message) {
    // ... code to process message
    out_m = compute(m);

    // log this message in local state for aggregate statistics
    this.state.append(m);

    // vertex decides to send something without waiting for
    // a notification that all messages prior to t have been received
    this.SendBy(some_edge, out_m, t);

    // now it also waits for a notification do to something when
    // nothing prior to t + 10 can arrive any more.
    // N.B.: simplified, as in reality the timestamps aren't integers
    if (t % 10 == 0) {
      this.NotifyAt(t + 10);
    }
  }

  function OnNotify(t': Timestamp) {
    // no more messages <= t' any more!

    // collect some aggregate statistics over the last 10 times
    stats = collect(this.state);

    this.SendBy(some_other_edge, stats, t');
  }
}


Q: What do the authors mean when they state systems like Spark "requires centralized modifications to the dataflow graph, which introduce substantial overhead"? Why does Spark perform worse than Naiad for iterative computations?

A: Think of how Spark runs an iterative algorithm like PageRank: for every iteration, it creates new RDDs and appends them to the lineage graph (which is Spark's "data-flow graph"). This is because Spark and other systems are built upon data-flow in directed acyclic graphs (DAGs), so to add another iteration, they have to add more operators.

The argument the authors make here is that Naiad's support for cycles is superior because it allows data to flow in cycles without changing the data-flow graph at all. This has advantages: changing the data-flow graph requires talking to the master/driver over the network and waiting for a response for *every* iteration of a loop, and will take at least a few milliseconds. This is the "centralized" part of that statement you quote.

With Naiad, by contrast, updates can flow in cycles the master being involved.

## LEC 14: Distributed Machine Learning: Parameter Server

6.824 2018 Lecture 14: Parameter Server Case Study

Scaling Distributed Machine Learning with the Parameter Server
Li, Andersen, Park, Smola, Ahmed, Josifovski, Long, Shekita, Su
OSDI 2014

Today: distributed machine learning
  Case study: Parameter Server
  Why are we reading this paper?
    influential design
    relaxed consistency
    different type of computation

Machine learning primer
  models are function approximators
    true function is unknown, so we learn an approximation from data
    ex: - f(user profile) -> likelihood of ad click
        - f(picture) -> likelihood picture contains a cat
        - f(words in document) -> topics/search terms for document
    function is typically very high-dimensional
  two phases: training and inference
    during training, expose model to many examples of data
      supervised: correct answer is known, check model response against it
      unsupervised: correct answer not known, measure model quality differently
    during inference, apply trained model to get predictions for unseen data
    parameter server is about making the training phase efficient

Features & parameters
  features: properties of input data that may provide signal to algorithm
    paper does not discuss how they are chosen; whole ML subfield of its own
  ex: computer vision algorithm can detect presence of whisker-like shapes
    if present, provides strong signal (= important param) that picture is a cat
    blue sky, by contrast, provides no signal (= unimportant parameter)
    we'd like the system to learn that whiskers are more important than blue sky
      i.e., "whisker presence" parameter should converge to high weight
  parameters are compact: a single floating point or integer number (weight)
    but there are many of them (millions/billions)
    ex: terms in ad => parameters for likelihood of click
      many unique combinations, some common ("used car") some not ("OSDI 2014")
    form a giant vector of numbers
  training iterates thousands of times to incrementally tune the parameter values
    popular algorithm: gradient descent, which generates "gradient updates"
    train, compute changes, apply update, train again
    iterations are very short (sub-second or a few seconds, esp. with GPUs/TPUs)

Distributed architecture for training
  need many workers
    because training data are too large for one machine
    for parallel speedup
    parameters may not fit on a single machine either
  all workers need access to parameters
  plan: distribute parameters + training data over multiple machines
    partition training data, run in parallel on partitions
    only interaction is via parameter updates
  each worker can access *any* parameter, but typically needs only small subset
    cf. Figure 3
    determined by the training data partitioning
  how do we update the parameters?

Strawman 1: broadcast parameter changes between workers
  at end of training iteration, workers exchange their parameter changes
  then apply them once all have received all changes
  similar to MapReduce-style shuffle
  Q: what are possible problems?
  A: 1) all-to-all broadcast exchange => ridiculous network traffic
     2) need to wait for all other workers before proceeding => idle time

Strawman 2: single coordinator collects and distributes updates
  at end of training iteration, workers send their changes to coordinator
  coordinator collects, aggregates, and sends aggregated updates to workers
    workers modify their local parameters
  Q: what are possible problems?
  A: 1) single coordinator gets congested with updates
     2) single coordinator is single point of failure
     3) what if a worker needs to read parameters it doesn't have?

Strawman 3: use Spark
  parameters for each iteration are an RDD (params_i)
  run worker-side computation in .map() transformation
    then do a reduceByKey() to shuffle parameter updates and apply
  Q: what are possible problems?
  A: 1) very inefficient! need to wait for *every* worker to finish iteration
     2) what if a worker need to read parameters it doesn't have? normal
        straight partitioning with narrow dependency doesn't apply

Parameter Server operation
  [diagram: parameter servers, workers; RM, task scheduler, training data]
  start off: initialize parameters at PS, push to workers
  on each iteration: assign training tasks to worker
    worker computes parameter updates
    worker potentially completes more training tasks for the same iteration
    worker pushes parameter updates to the responsible parameter servers
    parameter servers update parameters via user-defined function
      possibly aggregating parameter changes from multiple workers
    parameter servers replicate changes
      then ack to worker
    once done, worker pulls new parameter values
  key-value interface
    parameters often abstracted as big vector w[0, ..., z] for z parameters
      may actually store differently (e.g., hash table)
    in PS, each logical vector position stores (key, value)
      can index by key
      ex: (feature ID, weight)
    but can still treat the (key, value) parameters as a vector
      e.g., do to vector addition
  what are the bottleneck resources?
    worker: CPU (training compute > parameter update compute)
    parameter server: network (talks to many workers)

Range optimizations
  PS applies operations (push/pull/update) on key ranges, not single parameters
  why? big efficiency win due to batching!
    amortizes overheads for small updates
    e.g., syscalls, packet headers, interrupts
    think what would happen if we sent updates for individual parameters
      (feature ID, weight) parameter has 16 bytes
      IP headers: at least 20 bytes, TCP headers: dito
      40 bytes of header for 16 bytes of data -- 2.5x overhead
      syscall: ~2us, packet transmission: ~5ns at 10 Gbit/s -- 400x overhead
    so send whole ranges at a time to improve throughput
  further improvements:
    skip unchanged parameters
    skip keys with value zero in data for range
      can also use threshold to drop unimportant updates
    combine keys with same value

API
  not super clear in the paper!
  push(range, dest) / pull(range, dest) to send updates and pull parameters
  programmer implements WorkerIterate and ServerIterate methods (cf. Alg. 1)
    can push/pull multiple times and in response to control flow
  each WorkerIterate invocation is a "task"
    runs asynchronously -- doesn't block when pull()/push() invoked
    instead, run another task and come back to it when the RPC returns
  programmer can declare explicit dependencies between tasks
    details unclear

Consistent hashing
  need no single lookup directory to find parameter locations
    unlike earlier PS iteration, which used memcached
  parameter location determined by hashing key: H(k) = point on circle
  ranges on circle assigned to servers by hashing server identifier, i.e., H'(S)
    domains of H and H' must be the same (but hash function must not)
    each server owns keys between its hash point and the next
  well-known technique (will see again in Chord, Dynamo)

Fault tolerance
  what if a worker crashes?
    restart on another machine; load training data, pull parameters, continue
    or just drop it -- will only lose a small part of training data set
      usually doesn't affect outcome much, or training may take a little longer
  what if a parameter server crashes?
    lose all parameters stored there
  replicate parameters on multiple servers
    use consistent hashing ring to decide where
    each server stores neighboring counter-clockwise key ranges
    on update, first update replicas, then reply to worker
      no performance issue because workers are async, meanwhile do other tasks
  on failure, neighboring backup takes over key range
    has already replicated the parameters
    workers now talk to the new master

Relaxed consistency
  many ML algorithms tolerate somewhat stale parameters in training
    intuition: if parameters only change a little, not too bad to use old ones
      won't go drastically wrong (e.g., cat likelihood 85% instead of 91%)
    still converges to a decent model
    though may take longer (more training iterations due to higher error)
  trade-off: wait (& sit idle) vs. continue working with stale parameters
    up to the user to specify (via task dependencies)

Vector clocks
  need a mechanism to synchronize
    when strong consistency is required
    even with relaxed consistency: some workers may by very slow
      want to avoid others running ahead and some parameters getting very stale
  i.e., workers need to be aware of how far along others and the servers are
  vector clocks for each key
    each "clock" indicates where *each* other machine is at for that key
  [example diagram: vector with time entries for N nodes]
  only really works because PS uses ranges!
    vector clock is O(N) size, N = 1,000+
    overhead would be ridiculous if PS stored a vector clock for every key
    but range allows amortizing it
    PS always updates vector clock for a whole range at a time

Performance
  scale: 10s of billions of parameters, 10-100k cores (Fig. 1)
  very little idle time (Fig. 10)
  network traffic optimizations help, particularly at servers (Fig. 11)
  relaxed consistency helps up to a point (Fig. 13)

Real-world use
  TensorFlow, other learning frameworks
    high performance: two PS easily saturate 5-10 GBit/s
  influential design, though APIs vary
  discussion
    ML is an example of an application where inconsistency is often okay
    allows for different and potentially more efficient designs

References
 * PS implementation in TensorFlow: https://www.tensorflow.org/deploy/distributed

Q: Is the parameter server model useful for deep neural networks? It looks like this paper was written in 2014, which was before deep learning suddenly became very popular.

A: Yes! Neural networks tend to have many (millions) of parameters, so they're a good fit for a parameter server model. Indeed, the parameter server model has been quite influential in distributed deep learning frameworks. For example, TensorFlow's distributed execution uses parameter servers: https://www.tensorflow.org/deploy/distributed (the "ps" hosts are parameter servers).

Even though neural networks do not always have the kind of sparse parameter spaces that the examples in the paper use, parameter servers are still beneficial for them. Each PS aggregate gradient changes from multiple workers; if the workers had to do all-to-all exchange of gradient changes, this would require much higher network bandwidth. See this example:

https://stackoverflow.com/questions/39559183/what-is-the-reason-to-use-parameter-server-in-distributed-tensorflow-learning


## LEC 15: Cache Consistency: Frangipani

6.824 2018 Lecture 15: Frangipani

Frangipani: A Scalable Distributed File System
Thekkath, Mann, Lee
SOSP 1997

why are we reading this paper?
  performance via caching
  cache coherence
  decentralized design

what's the overall design?
  a network file system
    works transparently with existing apps (text editors &c)
    much like Athena's AFS
  users; workstations + Frangipani; network; petal
  Petal: block storage service; replicated; striped+sharded for performance
  What's in Petal?
    directories, i-nodes, file content blocks, free bitmaps
    just like an ordinary hard disk file system
  Frangipani: decentralized file service; cache for performance

what's the intended use?
  environment: single lab with collaborating engineers
    == the authors' research lab
    programming, text processing, e-mail, &c
  workstations in offices
  most file access is to user's own files
  need to potentially share any file among any workstations
    user/user collaboration
    one user logging into multiple workstations
  so:
    common case is exclusive access; want that to be fast
    but files sometimes need to be shared; want that to be correct
  this was a common scenario when the paper was written

why is Frangipani's design good for the intended use?
  it caches aggressively in each workstation, for speed
  cache is write-back
    allows updates to cached files/directories without network traffic
  all operations entirely local to workstation -- fast
    including e.g. creating files, creating directories, rename, &c
    updates proceed without any RPCs if everything already cached
    so file system code must reside in the workstation, not server
    "decentralized"
  cache also helps for scalability (many workstations)
    servers were a serious bottleneck in previous systems

what's in the Frangipani workstation cache?
  what if WS1 wants to create and write /u/rtm/grades?
  read /u/rtm information from Petal into WS1's cache
  add entry for "grades" just in the cache
  don't immediately write back to Petal!
    in case WS1 wants to do more modifications

challenges
  WS2 runs "ls /u/rtm" or "cat /u/rtm/grades"
    will WS2 see WS1's write?
    write-back cache, so WS'1 writes aren' in Petal
    caches make stale reads a serious threat
    "coherence"
  WS1 and WS2 concurrently try to create tmp/a and tmp/b
    will they overwrite each others' changes?
    there's no central file server to sort this out!
    "atomicity"
  WS1 crashes while renaming
    but other workstations are still operating
    how to ensure no-one sees the mess? how to clean up?
    "crash recovery"

"cache coherence" solves the "read sees write" problem
  the goal is linearizability AND caching
  there are lots of "coherence protocols"
  a common pattern: file servers, distributed shared memory, multi-core

Frangipani's coherence protocol (simplified):
  lock server (LS), with one lock per file/directory
    owner(lock) = WS, or nil
  workstation (WS) Frangipani cache:
    cached files and directories: present, or not present
    cached locks: locked-busy, locked-idle, unlocked
  workstation rules:
    acquire, then read from Petal
    write to Petal, then release
    don't cache unless you hold the lock
  coherence protocol messages:
    request  (WS -> LS)
    grant (LS -> WS)
    revoke (LS -> WS)
    release (WS -> LS)

the locks are named by files/directories (really i-numbers),
though the lock server doesn't actually understand anything
about file systems or Petal.

example: WS1 changes directory /u/rtm, then WS2 reads it

WS1                      LS            WS2
read /u/rtm
  --request(/u/rtm)-->
                         owner(/u/rtm)=WS1
  <--grant(/u/rtm)---
(read+cache /u/rtm data from Petal)
(create /u/rtm/grades locally)
(when done, cached lock in locked-idle state)
                                       read /u/rtm
                          <--request(/u/rtm)--
   <--revoke(/u/rtm)--
(write modified /u/rtm to Petal)
   --release(/u/rtm)-->
                         owner(/u/rtm)=WS2
                           --grant(/u/rtm)-->
                                       (read /u/rtm from Petal)

the point:
  locks and rules force reads to see last write
  locks ensure that "last write" is well-defined

coherence optimizations
  the "locked-idle" state is already an optimization
  Frangipani has shared read locks, as well as exclusive write locks
  you could imagine WS-to-WS communication, rather than via LS and Petal

next challenge: atomicity
  what if two workstations try to create the same file at the same time?
  are partially complete multi-write operations visible?
    e.g. file create initializes i-node, adds directory entry
    e.g. rename (both names visible? neither?)

Frangipani has transactions:
  WS acquires locks on all file system data that it will modify
  performs modifications with all locks held
  only releases when finished
  thus no other WS can see partially-completed operations
    and no other WS can race to perform updates (e.g. file creation)

note Frangipani's locks are doing two different things:
  cache coherence
  atomic transactions

next challenge: crash recovery

What if a Frangipani workstation dies while holding locks?
  other workstations will want to continue operating...
  can we just revoke dead WS's locks?
  what if dead WS had modified data in its cache?
  what if dead WS had started to write back modified data to Petal?
    e.g. WS wrote new directory entry to Petal, but not initialized i-node
    this is the troubling case

Is it OK to just wait until a crashed workstation reboots?

Frangipani uses write-ahead logging for crash recovery
  So if a crashed workstation has done some Petal writes for an operation,
    but not all, the writes can be completed from the log
  Very traditional -- but...
  1) Frangipani has a separate log for each workstation
     rather than the traditional log per shard of the data
     this avoids a logging bottleneck, eases decentralization
     but scatters updates to a given file over many logs
  2) Frangipani's logs are in shared Petal storage
     WS2 may read WS1's log to recover from WS1 crashing
  Separate logs is an interesting and unusual arrangement

What's in the log?
  log entry:
    (this is a bit of guess-work, paper isn't explicit)
    log sequence number
    array of updates:
      block #, new version #, offset, new bytes
    just contains meta-data updates, not file content updates
  example -- create file d/f produces a log entry:
    a two-entry update array:
      add an "f" entry to d's content block, with new i-number
      initialize the i-node for f
  initially the log entry is in WS local memory (not yet Petal)

When WS gets lock revocation on modified directory from LS:
  1) force its entire log to Petal, then
  2) send the cached updated blocks to Petal, then
  3) release the locks to the LS

Why must WS write log to Petal before updating
  i-node and directory &c in Petal?

Why delay writing the log until LS revokes locks?

What happens when WS1 crashes while holding locks?
  Not much, until WS2 requests a lock that WS1 holds
    LS sends grant to WS1, gets no response
    LS times out, tells WS2 to recover WS1 from its log in Petal
  What does WS2 do to recover from WS1's log?
    Read WS1's log from Petal
    Perform Petal writes described by logged operation
    Tell LS it is done, so LS can release WS1's locks

Note it's crucal that each WS log is in Petal so that it can
  be read by any WS for recovery.

What if WS1 crashes before it even writes recent operations to the log?
  WS1's recent operations may be totally lost if WS1 crashes.
  But the file system will be internally consistent.

Why is it safe to replay just one log, despite interleaved
  operations on same files by other workstations?
Example:
  WS1: delete(d/f)               crash
  WS2:               create(d/f)
  WS3 is recovering WS1's log -- but it doesn't look at WS2's log
  Will recovery re-play the delete? 
    This is The Question
    No -- prevented by "version number" mechanism
    Version number in each meta-data block (i-node) in Petal
    Version number(s) in each logged op is block's version plus one
    Recovery replays only if op's version > block version
      i.e. only if the block hasn't yet been updated by this op
  Does WS3 need to aquire the d or d/f lock?
    No: if version number same as before operation, WS1 couldn't
        have released the lock, so safe to update in Petal

Why is it OK that the log doesn't hold file *content*?
  If a WS crashes before writing content to Petal, it will be lost.
  Frangipani recovery defends the file system's own data structures.
  Applications can use fsync() to do their own recoverably content writes.
  It would be too expensive for Frangipani to log content writes.
  Most disk file systems (e.g. Linux) are similar, so applications
    already know how to cope with loss of writes before crash.

What if:
  WS1 holds a lock
  Network partition
  WS2 decides WS1 is dead, recovers, releases WS1's locks
  But WS1 is alive and subsequently writes data covered by the lock
  Locks have leases!
    Lock owner can't use a lock past its lease period
    LS doesn't start recovery until after lease expires

Is Paxos (== Raft) hidden somewhere here?
  Yes -- choice of lock server, choice of Petal primary/backup
  ensures a single lock server, despite partition
  ensures a single primary for each Petal shard

Performance?
  hard to judge numbers from 1997
  do they hit hardware limits? disk b/w, net b/w
  do they scale well with more hardware?
  what scenarios might we care about?
    read/write lots of little files (e.g. reading my e-mail)
    read/write huge files

Small file performance -- Figure 5
  X axis is number of active workstations
    each workstation runs a file-intensive benchmark
    workstations use different files and directories
  Y axis is completion time for a single workstation
  flat implies good scaling == no significant shared bottleneck
  presumably each workstation is just using its own cache
  possibly Petal's many disks also yield parallel performance

Big file performance
  each disk: 6 MB / sec
    Petal stripes to get more than that
  7 Petal servers, 9 disks per Petal server
    336 MB/s raw disk b/w, but only 100 MB/s via Petal
  a single Frangipani workstation, Table 3
    write: 15 MB/s -- limited by network link
    read: 10 MB/s -- limited by weak pre-fetch (?), could be 15
  lots of Frangipani workstations
    Figure 6 -- reads scale well with more machines
    Figure 7 -- writes hit hardware limits of Petal (2x for replication)

For what workloads is Frangipani likely to have poor performance?
  files bigger than cache?
  lots of read/write sharing?
  caching requires a reasonable working set size
  whole-file locking a poor fit for e.g. distributed DB
  coherence is too good for e.g. web site back-end

Petal details
  Petal provides Frangipani w/ fault-tolerant storage
    so it's worth discussing
  block read/write interface
    compatible with existing file systems
  looks like single huge disk, but many servers and many many disks
    big, high performance
    striped, 64-KB blocks
  virtual: 64-bit sparse address space, allocate on write
    address translation map
  primary/backup (one backup server)
    primary sends each write to the backup
  uses Paxos to agree on primary for each virt addr range
  what about recovery after crash?
    suppose pair is S1+S2
    S1 fails, S2 is now sole server
    S1 restarts, but has missed lots of updates
    S2 remembers a list of every block it wrote!
    so S1 only has to read those blocks, not entire disk
  logging
    virt->phys map and missed-write info

Limitations
  Most useful for e.g. programmer workstations, not so much otherwise
  Frangipani enforces permissions, so workstations must be trusted
    so Athena couldn't run Frangipani on Athena workstations
  Frangipani/Petal split is a little awkward
    both layers log
    Petal may accept updates from "down" Frangipani workstations
    more RPC messages than a simple file server
  A file system is not a great API for many applications, e.g. web site

Ideas to remember
  client-side caching for performance
  cache coherence protocols
  decentralized complex service on simple shared storage layer
  per-client log for decentralized recovery

FAQ for Frangipani, Thekkath, Mann, Lee, SOSP 1997

Q: Why are we reading this paper?

A: Primarily as an illustration of cache coherence.

But there are other interesting aspects. The idea of each client
having its own log, stored in a public place so that anyone can
recover from it, is clever. Further, the logs are intertwined in an
unusual way: the updates to a given object may be spread over many
logs. This makes replaying a single log tricky (hence Frangipani's
version numbers). Building a system out of simple shared storage
(Petal) and smart but decentralized participants is interesting,
particularly recovery from the crash of one participant.

Q: How does Frangipani differ from GFS?

A: A big architectural difference is that GFS has most of the file
system logic in the servers, while Frangipani distributes the logic
over the workstations that run Frangipani. That is, Frangipani doesn't
really have a notion of file server in the way that GFS does, only
file clients. Frangipani's design makes sense when most activity is
client workstations reading and writing a single user's files, which
can occur entirely from the workstation Frangipani cache; that's a
situation in which Frangipani delivers good performance. Frangipani
has a lot of mechanism to ensure that workstation caches stay
coherent, both so that a write on one workstation is immediately
visible to a read on another workstation, and so that complex
operations (like creating a file) are atomic even if other
workstations are trying to look at the file or directory involved.
This last situation is tricky for Frangipani because there's no
designated file server that executes all operations on a given file or
directory.

In contrast, GFS doesn't have caches at all, since its focus is
sequential read and write of giant files that are too big to fit in
any cache. It gets high performance for reads of giant files by
striping each file over many GFS servers. Because GFS has no caches,
GFS does not have a cache coherence protocol.

Frangipani appears as a real file system that you can use with any
existing workstation program. GFS doesn't present as a file system in
that sense; applications have to be written explicitly to use GFS via
library calls.

Q: Why do the Petal servers in the Frangipani system have a block
interface? Why not have file servers (like AFS), that know about
things like directories and files?

One reason is that the authors developed Petal first. Petal already
solved many fault tolerance and scaling problems, so using it
simplified some aspects of the Frangipani design. And this arrangement
moves work from centralized servers to client workstations, which
helps it scale well as more workstations are added.

However, the Petal/Frangipani split makes enforcing invariants on
file-system-level structures harder, since no one entity is in charge.
Frangipani builds its own transaction system (using the lock service
and Frangipani's logs) in order to be able to make complex atomic
updates to the file system stored in Petal.

Q: Can a workstation running Frangipani break security?

A: Yes. Since the file system logic is in the client workstations, the
design places trust in the clients. A user could modify the local
Frangipani software and read/write other users' data in Petal. This
makes Frangipani unattractive if users are not trusted. It might still
make sense in a small organization, or if Frangipani ran on separate
dedicated servers (not on workstations) and talked to user
workstations with a protocol like NFS.

Q: What is Digital Equipment?

A: It's the company that the authors worked at. Digital sold computers
and systems software. Unix was originally developed on Digital
computers (though not at Digital).

Q: What does the comment "File creation takes longer..." in Section
9.2 mean?

A: Because of logging, all changes have to be written to Petal twice:
once to the log, and once to the file system. An operation must be
written to the log first (thus "write-ahead log"). Then the on-disk
file system structures can be written. Only after that can the
operation be deleted from the log. That is, a portion of the log can
only be freed after all the on-disk file system updates from the
operations in that portion have been written to Petal.

So when Frangipani's log fills, Frangipani has to stop processing new
operations, send modified blocks from its cache to Petal for the
operations in the portion of the log it wants to re-use, and only then
free that portion and resume processing new operations.

You might wonder why increasing the log size improved performance.
After all, Frangipani has to perform the updates to the file system in
Petal no matter how long the log is, and those updates seem to be the
limiting performance factor. The paper doesn't say why increasing the
log size helps. Here's a guess. It's probably the case that the
benchmark involves updating the same directory many times, to add new
files to that directory. That directory's blocks will have to be
written to Petal every time part of the log is freed (assuming every
creation modifies the one directory). So letting the log grow longer
reduces the total number of times that directory has to be written to
Petal during the benchmark, which decreases benchmark running time.

Q: Frangipani is over 20 years old now; what's the state-of-the-art in
distributed file systems?

A: While people today do use distributed file systems for workstations
within an organization, such file systems have not recently been a
very active area of research and development. Lots of people use old
protocols such as SMB, NFS, and AFS. Some more recent systems are
xtreemfs, Ceph, and Lustre. A huge amount of network storage is sold
by companies like NetAPP and EMC, but I don't know to what extent
people use the storage at the disk level (via e.g. iSCSI) or as file
servers (e.g. talking to a NetAPP server with NFS).

Q: The paper says Frangipani only does crash recovery for its own file
system meta-data (i-nodes, directories, free bitmaps), but not for
users' file content. What does that mean, and why is it OK?

A: If a user on a workstation writes some data to a Frangipani file
system, and then the workstation immediately crashes, the recently
written data may be lost. That is, if the user logs into another
workstation and looks at the file, the recently written data may be
missing.

Ordinary Unix file systems (e.g. Linux on a laptop) have the same
property: file content written just before a crash may be lost.

Programs that care about crash recovery (e.g. text editors and
databases) can ask Unix to be more careful, at some expense in
performance. In particular, an application can call fsync(), which
tells Unix to force the data to disk immediately so that it will
survive a crash. The paper mentions fsync() in Section 2.1.

The rationale here is that the file system carefully defends its own
internal invariants (on its own meta-data), since otherwise the file
system might not be usable at all after a crash. But the file system
leaves maintaining of invariants in file content up to applications,
since only the application knows which data must be carefully (and
expensively) written to disk right away.

Q: What's the difference between the log stored in the Frangipani
workstation and the log stored on Petal?

The Petal paper hardly mentions Petal's log at all. My guess is that
Petal logs changes to the mapping from logical block number to current
disk location for that block, and logs changes to the "busy" bits that
indicate which block updates the other copy of the block may be missing.
That is, Petal logs information about low-level block operations.

Frangipani logs information about file system operations, which often
involve updating multiple pieces of file system state in Petal. For
example, a Frangipani log entry for deleting a file may say that the
delete operation modified some block and i-node free bitmap bits, and
that the operation erased a particular directory entry.

Both of the logs are stored on Petal's disks. From Petal's point of
view, Frangipani's logs are just data stored in Petal blocks; Petal does
not know anything special about Frangipani's logs.

Q: How does Petal take efficient snapshots of the large virtual disk that
it represents?

A: Petal maintains a mapping from virtual to physical block numbers. The
mapping is actually indexed by a pair: the virtual block number and an
epoch number. There is also a notion of the current epoch number. When
Petal performs a write to a virtual block, it looks at the epoch number
of the current mapping; if the epoch number is less than the current
epoch number, Petal creates a new mapping (and allocates a new physical
block) with the current epoch. A read for a virtual block uses the
mapping with the highest epoch.

Creating a snapshot then merely requires incrementing the current epoch
number. Reading from a snapshot requires the epoch number of the
snapshot to be specified; then each read uses the mapping for the
requested virtual block number that has the highest epoch number <= the
snapshot epoch.

Have a look at Section 2.2 of the Petal paper:

http://www.scs.stanford.edu/nyu/02fa/sched/petal.pdf

Q: The paper says that Frangipani doesn't immediately send new log
entries to Petal. What happens if a workstation crashes after
a system call completes, but before it send the corresponding
log entry to Petal?

A: Suppose an application running on a Frangipani workstation creates
a file. For a while, the information about the newly created file
exists only in the cache (in RAM) of that workstation. If the workstation
crashes before it writes the information to Petal, the new file is
completely lost. It won't be recovered.

So application that looks like

  create file x;
  print "I created file x";

might print "I created file x", but (if it then crashes) file x might
nevertheless not exist.

This may seem unfortunate, but it's fairly common even for local disk
file systems such as Linux. An application that needs to be sure its
data is really permanently saved can call fsync().

Q: What does it mean to stripe a file? Is this similar to sharding? 

A: It's similar to sharding. More specifically it means to distribute
the blocks of a single file over multiple servers (Petal server, for
this paper), so that (for example) the file's block 0 goes on server
0, block 1 goes on server 1, block 2 goes on server 0, block 3 goes on
server 1, &c. One reason to do this is to get high performance for
reads of a single big file, since different parts of the same file can
be read from different servers/disks. Another reason is to ensure that
load is balanced evenly over the servers.

Q: What is the "false sharing" problem mentioned in the paper?

A: The system reads and writes Petal at the granularity of a 512-byte
block. If it stored unrelated items X and Y on the same 512-byte
block, and one workstation needed to modify X while another needed to
modify Y, they would have to bounce the single copy of the block back
and forth between them. Because there's not a way to ask Petal to
write less than 512 bytes at a time. It's sharing because both
workstations are using the same block; it's false because they don't
fundamentally need to.
> Second, I don鈥檛 understand the section 4 bit about how you
> can enforce the stronger condition of never replaying a completed
> update. How is that stronger and why do we care?

Here's an example of the problem. Suppose workstation WS1 creates file
xxx, then deletes it. After that, a different workstation WS2 creates
file xxx. Their logs will look like this:

WS1:  create(xxx)  delete(xxx)
WS2:                           create(xxx)

At this point, correctness requires that xxx exist (since WS2's create
came after WS1's delete).

Now suppose WS1 crashes, and Frangipani recovery replays WS1's log.
Recovery will see the delete(xxx) operation in the log. We know that it
would be incorrect to actually do the delete, because we know that WS2's
create(xxx) came after WS1's delete(xxx). But how does Frangipani's
recovery software conclude that it should ignore the delete(xxx) when
replaying WS1's log?

Section 4 is saying that Frangipani's recovery software knows to ignore
the delete(xxx) because it "never replays a log record desccribing an
update that has already been completed." It achieves this property by
keeping a version number in every block of meta-data, and in every log
entry; if the log entry's version number is <= the meta-data block's
version number, Frangipani recovery knows that the meta-data block has
already been updated by the log entry, and thus that the log entry
should be ignored.

## LEC 16: Cache Consistency: Memcached at Facebook

6.824 2015 Lecture 16: Scaling Memcache at Facebook

Scaling Memcache at Facebook, by Nishtala et al, NSDI 2013

why are we reading this paper?
  it's an experience paper, not about new ideas/techniques
  three ways to read it:
    cautionary tale of problems from not taking consistency seriously
    impressive story of super high capacity from mostly-off-the-shelf s/w
    fundamental struggle between performance and consistency
  we can argue with their design, but not their success

how do web sites scale up with growing load?
  a typical story of evolution over time:
  1. one machine, web server, application, DB
     DB stores on disk, crash recovery, transactions, SQL
     application queries DB, formats, HTML, &c
     but the load grows, your PHP application takes too much CPU time
  2. many web FEs, one shared DB
     an easy change, since web server + app already separate from storage
     FEs are stateless, all sharing (and concurrency control) via DB
     but the load grows; add more FEs; soon single DB server is bottleneck
  3. many web FEs, data sharded over cluster of DBs
     partition data by key over the DBs
       app looks at key (e.g. user), chooses the right DB
     good DB parallelism if no data is super-popular
     painful -- cross-shard transactions and queries probably don't work
       hard to partition too finely
     but DBs are slow, even for reads, why not cache read requests?
  4. many web FEs, many caches for reads, many DBs for writes
     cost-effective b/c read-heavy and memcached 10x faster than a DB
       memcached just an in-memory hash table, very simple
     complex b/c DB and memcacheds can get out of sync
     (next bottleneck will be DB writes -- hard to solve)

the big facebook infrastructure picture
  lots of users, friend lists, status, posts, likes, photos
    fresh/consistent data apparently not critical
    because humans are tolerant?
  high load: billions of operations per second
    that's 10,000x the throughput of one DB server
  multiple data centers (at least west and east coast)
  each data center -- "region":
    "real" data sharded over MySQL DBs
    memcached layer (mc)
    web servers (clients of memcached)
  each data center's DBs contain full replica
  west coast is master, others are slaves via MySQL async log replication

how do FB apps use mc?
  read:
    v = get(k) (computes hash(k) to choose mc server)
    if v is nil {
      v = fetch from DB
      set(k, v)
    }
  write:
    v = new value
    send k,v to DB
    delete(k)
  application determines relationship of mc to DB
    mc doesn't know anything about DB
  FB uses mc as a "look-aside" cache
    real data is in the DB
    cached value (if any) should be same as DB

what does FB store in mc?
  paper does not say
  maybe userID -> name; userID -> friend list; postID -> text; URL -> likes
  basically copies of data from DB

paper lessons:
  look-aside is much trickier than it looks -- consistency
    paper is trying to integrate mutually-oblivious storage layers
  cache is critical:
    not really about reducing user-visible delay
    mostly about surviving huge load!
    cache misses and failures can create intolerable DB load
  they can tolerate modest staleness: no freshness guarantee
  stale data nevertheless a big headache
    want to avoid unbounded staleness (e.g. missing a delete() entirely)
    want read-your-own-writes
    each performance fix brings a new source of staleness
  huge "fan-out" => parallel fetch, in-cast congestion

let's talk about performance first
  majority of paper is about avoiding stale data
  but staleness only arose from performance design

performance comes from parallel get()s by many mc servers
  driven by parallel processing of HTTP requests by many web servers
  two basic parallel strategies for storage: partition vs replication

will partition or replication yield most mc throughput?
  partition: server i, key k -> mc server hash(k)
  replicate: server i, key k -> mc server hash(i)
  partition is more memory efficient (one copy of each k/v)
  partition works well if no key is very popular
  partition forces each web server to talk to many mc servers (overhead)
  replication works better if a few keys are very popular

performance and regions (Section 5)

Q: what is the point of regions -- multiple complete replicas?
   lower RTT to users (east coast, west coast)
   parallel reads of popular data due to replication
   (note DB replicas help only read performance, no write performance)
   maybe hot replica for main site failure?

Q: why not partition users over regions?
   i.e. why not east-coast users' data in east-coast region, &c
   social net -> not much locality
   very different from e.g. e-mail

Q: why OK performance despite all writes forced to go to the master region?
   writes would need to be sent to all regions anyway -- replicas
   users probably wait for round-trip to update DB in master region
     only 100ms, not so bad
   users do not wait for all effects of writes to finish
     i.e. for all stale cached values to be deleted
   
performance within a region (Section 4)

multiple mc clusters *within* each region
  cluster == complete set of mc cache servers
    i.e. a replica, at least of cached data

why multiple clusters per region?
  why not add more and more mc servers to a single cluster?
  1. adding mc servers to cluster doesn't help single popular keys
     replicating (one copy per cluster) does help
  2. more mcs in cluster -> each client req talks to more servers
     and more in-cast congestion at requesting web servers
     client requests fetch 20 to 500 keys! over many mc servers
     MUST request them in parallel (otherwise total latency too large)
     so all replies come back at the same time
     network switches, NIC run out of buffers
  3. hard to build network for single big cluster
     uniform client/server access
     so cross-section b/w must be large -- expensive
     two clusters -> 1/2 the cross-section b/w

but -- replicating is a waste of RAM for less-popular items
  "regional pool" shared by all clusters
  unpopular objects (no need for many copies)
  decided by *type* of object
  frees RAM to replicate more popular objects

bringing up new mc cluster was a serious performance problem
  new cluster has 0% hit rate
  if clients use it, will generate big spike in DB load
    if ordinarily 1% miss rate, and (let's say) 2 clusters,
      adding "cold" third cluster will causes misses for 33% of ops.
    i.e. 30x spike in DB load!
  thus the clients of new cluster first get() from existing cluster (4.3)
    and set() into new cluster
    basically lazy copy of existing cluster to new cluster
  better 2x load on existing cluster than 30x load on DB

important practical networking problems:
  n^2 TCP connections is too much state
    thus UDP for client get()s
  UDP is not reliable or ordered
    thus TCP for client set()s
    and mcrouter to reduce n in n^2
  small request per packet is not efficient (for TCP or UDP)
    per-packet overhead (interrupt &c) is too high
    thus mcrouter batches many requests into each packet
    
mc server failure?
  can't have DB servers handle the misses -- too much load
  can't shift load to one other mc server -- too much
  can't re-partition all data -- time consuming
  Gutter -- pool of idle servers, clients only use after mc server fails

The Question:
  why don't clients send invalidates to Gutter servers?
  my guess: would double delete() traffic
    and send too many delete()s to small gutter pool
    since any key might be in the gutter pool

thundering herd
  one client updates DB and delete()s a key
  lots of clients get() but miss
    they all fetch from DB
    they all set()
  not good: needless DB load
  mc gives just the first missing client a "lease"
    lease = permission to refresh from DB
    mc tells others "try get() again in a few milliseconds"
  effect: only one client reads the DB and does set()
    others re-try get() later and hopefully hit

let's talk about consistency now

the big truth
  hard to get both consistency (== freshness) and performance
  performance for reads = many copies
  many copies = hard to keep them equal

what is their consistency goal?
  *not* read sees latest write
    since not guaranteed across clusters
  more like "not more than a few seconds stale"
    i.e. eventual
  *and* writers see their own writes
    read-your-own-writes is a big driving force

first, how are DB replicas kept consistent across regions?
  one region is master
  master DBs distribute log of updates to DBs in slave regions
  slave DBs apply
  slave DBs are complete replicas (not caches)
  DB replication delay can be considerable (many seconds)

how do we feel about the consistency of the DB replication scheme?
  good: eventual consistency, b/c single ordered write stream
  bad: longish replication delay -> stale reads

how do they keep mc content consistent w/ DB content?
  1. DBs send invalidates (delete()s) to all mc servers that might cache
  2. writing client also invalidates mc in local cluster
     for read-your-writes

why did they have consistency problems in mc?
  client code to copy DB to mc wasn't atomic:
    1. writes: DB update ... mc delete()
    2. read miss: DB read ... mc set()
  so *concurrent* clients had races

what were the races and fixes?

Race 1:
  k not in cache
  C1 get(k), misses
  C1 v = read k from DB
    C2 updates k in DB
    C2 and DB delete(k) -- does nothing
  C1 set(k, v)
  now mc has stale data, delete(k) has already happened
  will stay stale indefinitely, until key is next written
  solved with leases -- C1 gets a lease, but C2's delete() invalidates lease,
    so mc ignores C1's set
    key still missing, so next reader will refresh it from DB

Race 2:
  during cold cluster warm-up
  remember clients try get() in warm cluster, copy to cold cluster
  k starts with value v1
  C1 updates k to v2 in DB
  C1 delete(k) -- in cold cluster
  C2 get(k), miss -- in cold cluster
  C2 v1 = get(k) from warm cluster, hits
  C2 set(k, v1) into cold cluster
  now mc has stale v1, but delete() has already happened
    will stay stale indefinitely, until key is next written
  solved with two-second hold-off, just used on cold clusters
    after C1 delete(), cold ignores set()s for two seconds
    by then, delete() will propagate via DB to warm cluster

Race 3:
  k starts with value v1
  C1 is in a slave region
  C1 updates k=v2 in master DB
  C1 delete(k) -- local region
  C1 get(k), miss
  C1 read local DB  -- sees v1, not v2!
  later, v2 arrives from master DB
  solved by "remote mark"
    C1 delete() marks key "remote"
    get()/miss yields "remote"
      tells C1 to read from *master* region
    "remote" cleared when new data arrives from master region

Q: aren't all these problems caused by clients copying DB data to mc?
   why not instead have DB send new values to mc, so clients only read mc?
     then there would be no racing client updates &c, just ordered writes
A:
  1. DB doesn't generally know how to compute values for mc
     generally client app code computes them from DB results,
       i.e. mc content is often not simply a literal DB record
  2. would increase read-your-own writes delay
  3. DB doesn't know what's cached, would end up sending lots
     of values for keys that aren't cached

PNUTS does take this alternate approach of master-updates-all-copies

FB/mc lessons for storage system designers?
  cache is vital to throughput survival, not just a latency tweak
  need flexible tools for controlling partition vs replication
  need better ideas for integrating storage layers with consistency



## LEC 17: Disconnected Operation, Eventual Consistency

6.824 2018 Lecture 17: Eventual Consistency, Bayou

"Managing Update Conflicts in Bayou, a Weakly Connected Replicated
Storage System" by Terry, Theimer, Petersen, Demers, Spreitzer,
Hauser, SOSP 95. And some material from "Flexible Update Propagation
for Weakly Consistent Replication" SOSP 97 (sections 3.3, 3.5, 4.2,
4.3).

Why are we reading this paper?
  It explores an important and interesting problem space.
  It uses some specific techniques worth knowing.

Big points:
  * Disconnected / weakly connected operation is often valuable.
    iPhone sync, Dropbox, git, Amazon Dynamo, Cassandra, &c
  * Disconnected operation implies eventual (weak) consistency.
    And it takes work (i.e. ordering) to even get that.
  * Disconnected writable replicas lead to update conflicts.
  * Conflict resolution generally has to be application-specific.

Technical ideas to remember:
  Log of operations is equivalent to data.
  Log helps eventual consistency (merge, order, and re-execute).
  Log helps conflict resolution (write operations easier than data).
  Causal consistency via Lamport-clock timestamps.
  Quick log comparison via version vectors.

Paper context:
  Early 1990s
  Dawn of PDAs, laptops, tablets
    Clunky but clear potential
  They wanted devices to be useful regardless of connectivity.
    Much like today's smartphones, tablets, laptops.

Let's build a conference room scheduler
  Only one meeting allowed at a time (one room).
  Each entry has a time and a description.
  We want everyone to end up seeing the same set of entries.

Traditional approach: one server
  Server executes one client request at a time
  Checks for conflicting time, says yes or no
  Updates DB
  Proceeds to next request
  Server implicitly chooses order for concurrent requests

Why aren't authors satisfied with a central server?
  They want full disconnected operation.
    So need DB replica in each device.
    Modify on any device, as well as read.
    "Sync" devices to propagate DB changes (Bayou's anti-entropy).
  They want to be able to use point-to-point connectivity.
    Sync via bluetooth to colleague in next airplane seat.

Why not merge DB records? (Bayou doesn't do this)
 Allow any pair of devices to sync (synchronize) their DBs.
 Sync could compare DBs, adopt other device's changed records.
 Need a story for conflicting entries, e.g. two meetings at same time.
   User may not be available to decide at time of DB merge.
   So need automatic reconciliation.

There are lots of possible conflict resolution schemes.
  E.g. adopt latest update, discard others.
  But we don't want people's calendar entries to simply disappear!
 
Idea for conflicts: update functions
  Application supplies a function, not just a DB write.
  Function reads DB, decides how best to update DB.
  E.g. "Meet at 9 if room is free at 9, else 10, else 11."
    Rather than just "Meet at 9"
  Function can make reconciliation decision for absent user.
  Sync exchanges functions, not DB content.

Problem: can't just run update functions as they arrive
  A's fn: staff meeting at 10:00 or 11:00
  B's fn: hiring meeting at 10:00 or 11:00
  X syncs w/ A, then B
  Y syncs w/ B, then A
  Will X put A's meeting at 10:00, and Y put A's at 11:00?

Goal: eventual consistency
  OK for X and Y to disagree initially
  But after enough syncing, all devices' DBs should be identical

Idea: ordered update log
  Ordered log of update functions at each device.
  Syncing == ensure both devices have same log (same updates, same order).
  DB is result of applying update functions in order.
  Same log => same order => same DB content.
  Note we're relying here on equivalence of two state representations:
    DB and log of operations.
    Raft also uses this idea.

How can all devices agree on update order?
  Assign a timestamp to each update when originally created.
  Timestamp: <T, I>
  T is creating device's wall-clock time.
  I is creating device's ID.
  Ordering updates a and b:
    a < b if a.T < b.T or (a.T = b.T and a.I < b.I)

Example:
 <10,A>: staff meeting at 10:00 or 11:00
 <20,B>: hiring meeting at 10:00 or 11:00
 What's the correct eventual outcome?
   the result of executing update functions in timestamp order
   staff at 10:00, hiring at 11:00

What DB content before sync?
  A's DB: staff at 10:00
  B's DB: hiring at 10:00
  This is what A/B users will see before syncing.

Now A and B sync with each other
  Each sorts new entries into its log, order by timestamp
  Both now know the full set of updates
  A can just run B's update function
  But B has *already* run B's operation, too soon!

Roll back and replay
  B needs to to "roll back" DB, re-run both ops in the right order
  The "Undo Log" in Figure 4 allws efficient roll-back

Big point: the log holds the truth; the DB is just an optimization

Now DBs will be eventually consistent.
  If everyone syncs enough,
  and no-one creates new updates,
  every device will have the same ordered log,
  and everyone's DB will end up with identical content.

We now know enough to answer The Question.
  initially A=foo B=bar
  one device: copy A to B
  other device: copy B to A
  dependency check?
  merge procedure?
  why do all devices agree on final result?
  
Will update order be consistent with wall-clock time?
  Maybe A went first (in wall-clock time) with <10,A>
  Device clocks unlikely to be perfectly synchronized
  So B could then generate <9,B>
  B's meeting gets priority, even though A asked first

Will update order be consistent with causality?
  What if A adds a meeting, 
    then B sees A's meeting,
    then B deletes A's meeting.
  Perhaps
    <10,A> add
    <9,B> delete -- B's clock is slow
  Now delete will be ordered before add!
  So: design so far is not causally consistent.

Causal consistency means that if operation X might have caused
  or influenced operation Y, then everyone should order X before Y.

Bayou uses "Lamport logical clocks" for causal consistency
  Want to timestamp writes s.t.
    if device observes E1, then generates E2, then TS(E2) > TS(E1)
  So all devices will order E1, then E2
  Lamport clock:
    Tmax = highest timestamp seen from any device (including self)
    T = max(Tmax + 1, wall-clock time) -- to generate a timestamp
  Note properties:
    E1 then E2 on same device => TS(E1) < TS(E2)
    BUT
    TS(E1) < TS(E2) does not imply E1 came before or caused E2

Logical clock solves add/delete causality example
  When B sees <10,A>,
    B will set its Tmax to 10, so
    B will generate <11,B> for its delete

Irritating that there could be a long-delayed update with lower TS
  That can cause the results of my update to change
    User can never be sure if meeting time is final!
    Entries are "tentative"
  Would be nice if each update eventually became "stable"
    => no changes in update order up through that point
    => effect of write function now fixed, e.g. meeting time won't change
    => don't have to roll back, re-run committed updates
  We'd like to know when a write is stable, and tell the user

Idea: a fully decentralized "commit" scheme (Bayou doesn't do this)
  <10,A> is stable if I'll never see a new update w/ TS <= 10
  Once I've seen an update w/ TS > 10 from *every* device
    I'll never see any new TS < 10 (sync sends updates in TS order)
    Then <10,A> is stable
  Why doesn't Bayou use this decentralized commit scheme?

Idea: Bayou's "primary replica" to commit updates.
  One device is the "primary replica".
  Primary sees updates via sync in the ordinary way.
  Primary marks each received update with a Commit Sequence Number (CSN).
    That update is committed.
    So a complete timestamp is <CSN, logical-time, device-id>
    Uncommitted updates come after all committed updates
      i.e. have infinite CSN
  CSN notifications are synced between devices.
 
Why does the commit / CSN scheme eventually yield stability?
  Primary assigns only increasing CSNs.
  Device logs order all updates with CSN before any w/o CSN.
  So once an update has a CSN, the set of previous updates is fixed.

Will commit order match tentative order?
  Often.
  Syncs send in log order ("prefix property")
    Including updates learned from other devices.
  So if A's update log says
    <-,10,X>
    <-,20,A>
  A will send both to primary, in that order
    Primary will assign CSNs in that order
    Commit order will, in this case, match tentative order

Will commit order *always* match tentative order?
  No: primary may see newer updates before older ones.
  A has just: <-,10,A> W1
  B has just: <-,20,B> W2
  If C sees both, C's order: W1 W2
  B syncs with primary, W2 gets CSN=5.
  Later A syncs w/ primary, W1 gets CSN=6.
  When C syncs w/ primary, C will see order change to W2 W1
    <5,20,B> W2
    <6,10,A> W1
  So: committing may change order.
  
How Bayou syncs (this is anti-entropy)?
  A sending to B
  Need a quick way for B to tell A what to send
  Prefix property simplifies syncing (i.e. sync is always in log order)
    So it's meaningful for B to say "I have everything up to ..."
  Committed updates are easy:
    B sends its highest CSN to A
    A sends log entries between B's highest CSN and A's highest CSN
  What about tentative updates?
  A has:
    <-,10,X>
    <-,20,Y>
    <-,30,X>
    <-,40,X>
  B has:
    <-,10,X>
    <-,20,Y>
    <-,30,X>
  At start of sync, B tells A "X 30, Y 20"
    I.e. for each device, highest TS B has seen from that device.
    Sync prefix property means B has all X updates before 30,
      all Y before 20
  A sends all X's updates after <-,30,X>,
    all Y's updates after <-,20,Y>, &c
  "X 30, Y 20" is a version vector -- it summarizes log content
    It's the "F" vector in Figure 4
    A's F: [X:40,Y:20]
    B's F: [X:30,Y:20]

It's worth remembering the "version vector" idea
  used in many systems
  typically a summary of state known by a participant
  one entry per participant
    meaning "I have seen all updates from Pi through update number Vi"

Devices can discard committed updates from log.
  (a lot like Raft snapshots)
  Instead, keep a copy of the DB as of the highest known CSN.
  Roll back to that DB when replaying tentative update log.
  Never need to roll back farther.
    Prefix property guarantees seen CSN=x => seen CSN<x.
    No changes to update order among committed updates.

How do I sync if I've discarded part of my log?
 (a lot like Raft InstallSnapshot RPC)
 Suppose I've discarded all updates with CSNs.
 I keep a copy of the stable DB reflecting just discarded entries.
 If syncing to device X, and its highest CSN is less than mine:
   Send X my complete DB.
 In practice, Bayou devices keep the last few committed updates.
   To reduce chance of having to send whole DB during sync.

How could we cope with a new server Z joining the system?
  Could it just start generating writes, e.g. <-,1,Z> ?
  And other devices just start including Z in VVs?
  If A syncs to B, A has <-,10,Z>, but B has no Z in VV
    A should pretend B's VV was [Z:0,...]

What happens when Z retires (leaves the system)?
  We want to stop including Z in VVs!
  How to announce that Z is gone?
    Z sends update <-,?,Z> "retiring"
  If you see a retirement update, omit Z from VV
  How to deal with a VV that's missing Z?
  If A has log entries from Z, but B's VV has no Z entry:
    e.g. A has <-,25,Z>, B's VV is just [A:20, B:21]
    Maybe Z has retired, B knows, A does not
    Maybe Z is new, A knows, B does not
  Need a way to disambiguate: Z missing from VV b/c new, or b/c retired?

Bayou's retirement plan
  Z joins by contacting some server X
  Z's ID is <Tz,X>
    Tz is X's logical clock as of when Z joined
  X issues <-,Tz,X>:"new server ID=<Tz,X>"

How does ID=<Tz,X> scheme help disambiguate new vs forgotten?
  Suppose Z's ID is <20,X>
  A syncs to B
    A has log entry from Z <-,25,<20,X>>
    B's VV has no Z entry -- has B never seen Z,
      or already seen Z's retirement?
  One case:
    B's VV: [X:10, ...]
    10 < 20 implies B hasn't yet seen X's "new server Z" update
  The other case:
    B's VV: [X:30, ...]
    20 < 30 implies B once knew about Z, but then saw a retirement update

In a few lectures: Dynamo, a real-world DB with eventual consistency

Bayou FAQ

Q: A lot of Bayou's design is driven by the desire to support
disconnected operation. Is that still important today?

A: Disconnected and weakly connected operation continue to be
important, though in different guises. Dropbox, Amazon's Dynamo,
Cassandra, git, and smartphone syncing are all real-world systems with
high-level goals and properties similar to Bayou's: they allow
immediate reads and writes of a local replica even when disconnected,
they provide eventual consistency, and they have to cope with write
conflicts.

Q: Doesn't widely-available wireless Internet mean everyone is
connected all the time?
 
A: Wireless connectivity doesn't seem to have reduced the need for
disconnected operation. Perhaps the reason is that anything short of
100% connectivity means that software has to be designed to handle
periods of disconnection. For example, my laptop has WiFi but not
cellular data; there are times when I want to be able to use my laptop
yet I either don't want to pay for WiFi, or there isn't a nearby
access point. For git, I often do not want to see others' changes
right away even if my laptop is connected. I want to be able to use my
smartphone's contacts list and calendar regardless of whether it's on
the Internet.

Q: Bayou supports direct synchronization of one device to another,
e.g. over Bluetooth or infrared, without going through the Internet or
a server. Is that important?

A: Perhaps not; I think syncing devices via an intermediate server on
the Internet is probably good enough.

Q: Does anyone use Bayou today? If not, why are we reading this paper?

A: No-one uses Bayou today; it was a research prototype intended to
explore a new architecture. Bayou uses multiple ideas that are worth
knowing: eventual consistency, conflict resolution, logging operations
rather than data, use of timestamps to help agreement on order,
version vectors, and causal consistency via Lamport logical clocks.
 
Q: Has the idea of applications performing conflict resolution been
used in other distributed systems?
 
A: There are synchronization systems that have application-specific
conflict resolution. git merges different users' changes to different
lines in the same file. When someone syncs their iPhone with their
Mac, the calendars are merged in a way that understands about the
structure of calendars. Similarly, Dropbox has an API that allows
applications to intervene when Dropbox detects conflicting updates to
the same file. However, I'm not aware of any system other than Bayou
that syncs a log of update functions (other systems typically sync the actual
file content).
 
Q: Do companies like Dropbox use protocols similar to Bayou?
 
A: I doubt Dropbox uses anything like Bayou. I suspect Dropbox moves
file content around, rather than having a log of writes. On the other
hand Dropbox is not as flexible as Bayou at application-specific
resolution of conflicting updates to the same file.
 
Q: What does it mean for data to be weakly consistent?
 
A: It means that clients may see that different replicas are not
identical, though the system tries to keep them as identical as it
can.

Q: Is eventual consistency the best you can do if you want to support
disconnected operation?

A: Yes, eventual consistency (or slight improvements like causal
consistency) is the best you can do. If we want to support reads and
writes to disconnected replicas of the data (as Bayou does), we have
to tolerate users seeing stale data, and users causing conflicting
writes that are only evident after synchronization. It's nice that
it's even possible to get eventual consistency in such a system!
 
Q: It seems like writing dependency checks and merge procedures for a
variety of operations could be a tough interface for programmers to
handle. Is there anything I'm missing there?
 
A: It's true that programmer work is required. But in return Bayou
provides a very slick solution to the difficult problem of resolving
conflicting updates to the same data.

Q: Is the primary replica a single point of failure?

A: Sort of. If the primary is down or unreachable, everything works
fine, except that writes won't be declared stable. For example, the
calendar application would basically work, though users would have to
live in fear that new calendar entries with low timestamps could
appear and change the displayed schedule.
 
Q: How do dependency checks detect Write-Write conflicts? The paper
says "Such conflicts can be detected by having the dependency check
query the current values of any data items being updated and ensure
that they have not changed from the values they had at the time the
Write was submitted", but I don't quite understand in this case what
the expected result of the dependency check is.
 
A: Suppose the application wants to modify calendar entry "10:00", but
only if there's no entry already. When the application runs, there's
no entry. So the application will produce this Bayou write operation:

  dependency_check = { check that the value for "10:00" is nil }
  update = { set the value for "10:00" to "staff meeting" }
  mergeproc = { ... }

If no other write modifies 10:00, then the dependency_check will succeed
on all servers, and they will all set the value to "staff meeting".

If some other write, by a different client at about the same time, sets
the 10:00 entry to "grades meeting", then that's a write/write conflict:
two different writes want to set the same DB entry to different values.
The dependency check will detect the conflict when synchronization
causes some servers to see both writes. One write will be first in the
log (because it has a lower timestamp); its dependency check will
succeed, and it will update the 10:00 entry. The write that's second in
the log will fail the dependency check, because the DB value for 10:00
is no longer nil.

That is, checking that the DB entry has the same value that it had when
the application originally ran does detect the situation in which a
conflicting write was ordered first in the log.
 
Q: When are dependency checks called?
 
A: Each server executes new log entries that it hears during
synchronization. For each new log entry, the server calls the entry's
dependency check; if the check succeeds, the server applies the entry's
update to its DB.
 
Q: If two clients make conflicting calendar reservations on partitioned servers,
do the dependency checks get called when those two servers communicate?
 
A: If the two servers synchronize, then their logs will contain each
others' reservations, and at that point both will execute the checks.
 
Q: It looks like the logic in the dependency check would take place when
you're first inserting a write operation, but you wouldn't find any
conflicts from partitioned servers.
 
A: Whenever a server synchronizes, it rolls back its DB to the earliest point at
which synchronization modified its log, and then re-executes all log entries
after that point (including the dependency checks).
 
Q: What are anti-entropy sessions?
 
A: This refers to synchronization between a pair of devices, during which
the devices exchange log entries to ensure they have identical logs (and
thus identical DB content).
 
Q. What is an epidemic algorithm?
 
A: A communication scheme in which pairs devices exchange data with
each other, including data they have heard from other devices. The
"epidemic" refers to the fact that, eventually, new data will spread
to all devices via these pairwise exchanges.
 
Q: Why are Write exchange sessions called anti-entropy sessions?
 
A: Perhaps because they reduce disorder. One definition of "entropy" is a
measure of disorder in a system.
 
Q: In order to know if writes are stabilized, does a server have to
contact all other servers?
 
A: Bayou has a special primary that commits (stabilizes) writes by assigning
them CSNs. You only have to contact the primary to know what's been committed.
 
Q: How much time could it take for a Write to reach all servers?
 
A: At worst, a server might never see updates, because it is broken.
At best, a server may synchronize with other servers frequently, and
thus may see updates quickly. There's not much concrete the paper can
say about this, because it depends on whether users turn off their
laptops, whether they spend a lot of time without network
connectivity, &c.
 
Q: In what case is automatic resolution not possible? Does it only depend
on the application, or is it the case that for any application, it's
possible for automatic resolution to fail?
 
A: It usually depends on the application. In the calendar example, if
I supply only two possible time slots e.g. 10:00 and 11:00, and those
slots are already taken, Bay can't resolve the conflict. But suppose
the data is "the number of people who can attend lunch on friday", and
the updates are increments to the number from people who can attend.
Those updates can always succeed -- it's always possible to add 1 to
the count.

Q: What are examples of good (quick convergence) and not-so-good anti-entropy
policies?
 
A: A example good situation is if all servers synchronize frequently
with a single server (or more generally if there's a path between
every two servers made up of frequently synchronizing pairs). A bad
situation is if some servers don't synchronize frequently, or if there
are two groups of servers, frequent synchronization within each group,
but rare synchronization between the groups.
 
Q: I don't understand why "tentative deletion may result in a tuple that appears
in the committed view but not in the full view." (very beginning of page 8)
 
A: The committed view only reflects writes that have been committed.
So if there's a tentative operation that deletes record X, record X
will be in the committed view, but not in the full view. The full view
reflects all operations (including the delete), but the committed view
reflects only committed operations (thus not the delete).
 
Q: Bayou introduces a lot of new ideas, but it's not clear which ideas
are most important for performance.
 
A: I suspect Bayou is relatively slow. Their goal was not performance, but
rather new functionality: a new kind of system that could support shared mutable
data despite intermittent network connectivity.
 
Q: What kind of information does the Undo Log contain? (e.g. does it
contain a snapshot of changed files from a Write, or the reverse
operation?) Or is this more of an implementation detail?
 
A: I suspect that the undo log contains an entry for every write, with the value
of DB record that the write modified *before* the modification. That allows
Bayou to roll back the DB in reverse log order, replacing each modified DB
record with the previous version.
 
Q: How is a particular server designated as the primary?
 
A: I think a human chooses.
 
Q: What if communication fails in the middle of an anti-entropy session?
 
A: Bayou always sends log entries in order when synchronizing, so it's OK
for the receiver to add the entries it received to its log, even though it
didn't hear subsequent entries from the sender.
 
Q: Does Bayou cache?
 
A: Each Bayou server (i.e. device like a laptop or iPhone) has a complete copy
of all the data. So there's no need for an explicit cache.

Q: What are the session guarantees mentioned by the paper?

A: Bayou allows an application to switch servers, so that a client
device can talk to a Bayou server over the network rather than having
to run a full Bayou server. But if an application switches servers
while it is running, the new server might not have seen all the writes
that the application sent to the previous servers. Session guarantees
are a technique to deal with this. The application keeps a session
version vector summarizing all the writes it has sent to any server.
When it connects to a new server, the application first checks that
the new server is as least as up to date as the session version
vector. If not, the application finds a new server that is up to date.

## LEC 18: Guest lecturer: Frank Dabek of Google

Preparation: Read The Tail at Scale	may 2	may 3
Hacking day, no lecture	may 4
may 7	may 8

## LEC 19: Peer-to-peer, DHTs

6.824 2018 Lecture 19: P2P, DHTs, and Chord

Reminders:
  course evaluations
  4B due friday
  project reports due friday
  project demos next week
  final exam on May 24th

Today's topic: Decentralized systems, peer-to-peer (P2P), DHTs
  potential to harness massive [free] user compute power, network b/w
  potential to build reliable systems out of many unreliable computers
  potential to shift control/power from organizations to users
  appealing, but has been hard in practice to make the ideas work well

Peer-to-peer
  [user computers, files, direct xfers]
  users computers talk directly to each other to implement service
    in contrast to user computers talking to central servers
  could be closed or open
  examples:
    bittorrent file sharing, skype, bitcoin

Why might P2P be a win?
  spreads network/caching costs over users
  absence of central server may mean:
    easier/cheaper to deploy
    less chance of overload
    single failure won't wreck the whole system
    harder to attack

Why don't all Internet services use P2P?
  can be hard to find data items over millions of users
  user computers not as reliable as managed servers
  if open, can be attacked via evil participants

The result is that P2P has been limited to a few niches:
  [Illegal] file sharing
    Popular data but owning organization has no money
  Chat/Skype
    User to user anyway; privacy and control
  Bitcoin
    No natural single owner or controller

Example: classic BitTorrent
  a cooperative, popular download system
  user clicks on download link for e.g. latest Linux kernel distribution
    gets torrent file w/ content hash and IP address of tracker
  user's BT app talks to tracker
    tracker tells it list of other users w/ downloaded file
  user't BT app talks to one or more users w/ the file
  user's BT app tells tracker it has a copy now too
  user's BT app serves the file to others for a while
  the point:
    provides huge download b/w w/o expensive server/link

But: the tracker is a weak part of the design
  makes it hard for ordinary people to distribute files (need a tracker)
  tracker may not be reliable, especially if ordinary user's PC
  single point of attack by copyright owner, people offended by content

BitTorrent can use a DHT instead of a tracker
  this is the topic of today's readings
  BT apps cooperatively implement a "DHT"
    a decentralized key/value store, DHT = distributed hash table
  the key is the torrent file content hash ("infohash")
  the value is the IP address of an BT app willing to serve the file
    Kademlia can store multiple values for a key
  app does get(infohash) to find other apps willing to serve
    and put(infohash, self) to register itself as willing to serve
    so DHT contains lots of entries with a given key:
      lots of peers willing to serve the file
  app also joins the DHT to help implement it

Why might the DHT be a win for BitTorrent?
  more reliable than single classic tracker
    keys/value spread/cached over many DHT nodes
    while classic tracker may be just an ordinary PC
  less fragmented than multiple trackers per torrent
    so apps more likely to find each other
  maybe more robust against legal and DoS attacks

How do DHTs work?

Scalable DHT lookup:
  Key/value store spread over millions of nodes
  Typical DHT interface:
    put(key, value)
    get(key) -> value
  weak consistency; likely that get(k) sees put(k), but no guarantee
  weak guarantees about keeping data alive

Why is it hard?
  Millions of participating nodes
  Could broadcast/flood each request to all nodes
    Guaranteed to find a key/value pair
    But too many messages
  Every node could know about every other node
    Then could hash key to find node that stores the value
    Just one message per get()
    But keeping a million-node table up to date is too hard
  We want modest state, and modest number of messages/lookup

Basic idea
  Impose a data structure (e.g. tree) over the nodes
    Each node has references to only a few other nodes
  Lookups traverse the data structure -- "routing"
    I.e. hop from node to node
  DHT should route get() to same node as previous put()

Example: The "Chord" peer-to-peer lookup system
  Kademlia, the DHT used by BitTorrent, is inspired by Chord

Chord's ID-space topology
  Ring: All IDs are 160-bit numbers, viewed in a ring.
  Each node has an ID, randomly chosen, or hash(IP address)
  Each key has an ID, hash(key)

Assignment of key IDs to node IDs
  A key is stored at the key ID's "successor"
    Successor = first node whose ID is >= key ID.
    Closeness is defined as the "clockwise distance"
  If node and key IDs are uniform, we get reasonable load balance.

Basic routing -- correct but slow
  Query (get(key) or put(key, value)) is at some node.
  Node needs to forward the query to a node "closer" to key.
    If we keep moving query closer, eventually we'll hit key's successor.
  Each node knows its successor on the ring.
    n.lookup(k):
      if n < k <= n.successor
        return n.successor
      else
        forward to n.successor
  I.e. forward query in a clockwise direction until done
  n.successor must be correct!
    otherwise we may skip over the responsible node
    and get(k) won't see data inserted by put(k)

Forwarding through successor is slow
  Data structure is a linked list: O(n)
  Can we make it more like a binary search?
    Need to be able to halve the distance at each step.

log(n) "finger table" routing:
  Keep track of nodes exponentially further away:
    New state: f[i] contains successor of n + 2^i
    n.lookup(k):
      if n < k <= n.successor:
        return successor
      else:
        n' = closest_preceding_node(k) -- in f[]
        forward to n'

for a six-bit system, maybe node 8's finger table looks like this:
  0: 14
  1: 14
  2: 14
  3: 21
  4: 32
  5: 42

Why do lookups now take log(n) hops?
  One of the fingers must take you roughly half-way to target

Is log(n) fast or slow?
  For a million nodes it's 10 hops (since a hop is only needed to correct a bit).
  If each hop takes 50 ms, lookups take half a second.
  If each hop has 10% chance of failure, it's a couple of timeouts.
  So: good but not great.
    Though with complexity, you can get better real-time and reliability.

Since lookups are log(n), why not use a binary tree?
  A binary tree would have a hot-spot at the root
    And its failure would be a big problem
  The finger table requires more maintenance, but distributes the load

How does a new node acquire correct tables?
  General approach:
    Assume system starts out w/ correct routing tables.
    Add new node in a way that maintains correctness.
    Use DHT lookups to populate new node's finger table.
  New node m:
    Sends a lookup for its own key, to any existing node.
      This yields m.successor
    m asks its successor for its entire finger table.
  At this point the new node can forward queries correctly
  Tweaks its own finger table in background
    By looking up each m + 2^i

Does routing *to* new node m now work?
  If m doesn't do anything,
    lookup will go to where it would have gone before m joined.
    I.e. to m's predecessor.
    Which will return its n.successor -- which is not m.
  We need to link the new node into the successor linked list.

Why is adding a new node tricky?
  Concurrent joins!
  Example:
    Initially: ... 10 20 ...
    Nodes 12 and 15 join at the same time.
    They can both tell 10+20 to add them,
      but they didn't tell each other!
  We need to ensure that 12's successor will be 15, even if concurrent.

Stabilization:
  Each node keeps track of its current predecessor.
  When m joins:
    m sets its successor via lookup.
    m tells its successor that m might be its new predecessor.
  Every node m1 periodically asks successor m2 who m2's predecessor m3 is:
    If m1 < m3 < m2, m1 switches successor to m3.
    m1 tells m3 "I'm your new predecessor"; m3 accepts if closer
      than m3's existing predecessor.
  Simple stabilization example:
    initially: ... 10 <==> 20 ...
    15 wants to join
    1) 15 tells 20 "I'm your new predecessor".
    2) 20 accepts 15 as predecessor, since 15 > 10.
    3) 10 asks 20 who 20's predecessor is, 20 answers "15".
    4) 10 sets its successor pointer to 15.
    5) 10 tells 15 "10 is your predecessor"
    6) 15 accepts 10 as predecessor (since nil predecessor before that).
    now: 10 <==> 15 <==> 20
  Concurrent join:
    initially: ... 10 <==> 20 ...
    12 and 15 join at the same time.
    * both 12 and 15 tell 20 they are 20's predecessor; when
      the dust settles, 20 accepts 15 as predecessor.
    * now 10, 12, and 15 all have 20 as successor.
    * after one stabilization round, 10 and 12 both view 15 as successor.
      and 15 will have 12 as predecessor.
    * after two rounds, correct successors: 10 12 15 20

To maintain log(n) lookups as nodes join,
  Every one periodically looks up each finger (each n + 2^i)

What about node failures?
  Nodes fail w/o warning.
  Two issues:
    Other nodes' routing tables refer to dead node.
    Dead node's predecessor has no successor.
  Recover from dead next hop by using next-closest finger-table entry.
  Now, lookups for the dead node's keys will end up at its predecessor.
  For dead successor
    We need to know what dead node's n.successor was
      Since that's now the node responsible for the dead node's keys.
    Maintain a _list_ of r successors.
    Lookup answer is first live successor >= key

Dealing with unreachable nodes during routing is important
  "Churn" is high in open p2p networks
  People close their laptops, move WiFi APs, &c pretty often
  Fast timeouts?
  Explore multiple paths through DHT in parallel?
    Perhaps keep multiple nodes in each finger table entry?
    Send final messages to multiple of r successors?
    Kademlia does this, though it increases network traffic.

Geographical/network locality -- reducing lookup time
  Lookup takes log(n) messages.
    But messages are to random nodes on the Internet!
    Will often be very far away.
  Can we route through nodes close to us on underlying network?
  This boils down to whether we have choices:
    If multiple correct next hops, we can try to choose closest.

Idea: proximity routing
  to fill a finger table entry, collect multiple nodes near n+2^i on ring
  perhaps by asking successor to n+2^i for its r successors
  use lowest-ping one as i'th finger table entry

What's the effect?
  Individual hops are lower latency.
  But less and less choice as you get close in ID space.
  So last few hops are likely to be long. 
  Though if you are reading, and any replica will do,
    you still have choice even at the end.

Any down side to locality routing?
  Harder to prove independent failure.
    Maybe no big deal, since no locality for successor lists sets.
  Easier to trick me into using malicious nodes in my tables.

What about security?
  Can someone forge data? I.e. return the wrong value?
    Defense: key = SHA1(value)
    Defense: key = owner's public key, value signed
    Defense: some external way to verify results (Bittorrent does this)
  Can a DHT node claim that data doesn't exist?
    Yes, though perhaps you can check other replicas
  Can a host join w/ IDs chosen to sit under every replica of a given key?
    Could deny that data exists, or serve old versions.
    Defense: require (and check) that node ID = SHA1(IP address)
  Can a host pretend to join millions of times?
    Could break routing with non-existant hosts, or control routing.
    Defense: node ID = SHA1(IP address), so only one node per IP addr.
    Defense: require node to respond at claimed IP address.
             this is what trackerless Bittorrent's token is about
  What if the attacker controls lots of IP addresses?
    No easy defense.
  But:
    Dynamo gets security by being closed (only Amazon's computers).
    Bitcoin gets security by proving a node exists via proof-of-work.

How to manage data?
  Here is the most popular plan.
    [diagram: Chord layer and DHT layer]
    Data management is in DHT above layer, using Chord.
  DHT doesn't guarantee durable storage
    So whoever inserted must re-insert periodically
    May want to automatically expire if data goes stale (bittorrent)
  DHT replicates each key/value item
    On the nodes with IDs closest to the key, where looks will find them
    Replication can help spread lookup load as well as tolerate faults
  When a node joins:
    successor moves some keys to it
  When a node fails:
    successor probably already has a replica
    but r'th successor now needs a copy

Summary
  DHTs attractive for finding data in large p2p systems
    Decentralization seems good for high load, fault tolerance
  But: log(n) lookup time is not very fast
  But: the security problems are difficult
  But: churn is a problem, leads to incorrect routing tables, timeouts
  Next paper: Amazon Dynamo, adapts these ideas to a closed system.

References

Kademlia: www.scs.stanford.edu/~dm/home/papers/kpos.pdf
Accordion: www.news.cs.nyu.edu/~jinyang/pub/nsdi05-accordion.pdf
Promiximity routing: https://pdos.csail.mit.edu/papers/dhash:nsdi/paper.pdf
Evolution analysis: http://nms.csail.mit.edu/papers/podc2002.pdf
Sybil attack: http://research.microsoft.com/pubs/74220/IPTPS2002.pdf

http://www.bittorrent.org/beps/bep_0005.html

Chord paper FAQ

Q: Is hashing across machines a good way to get load balanced
sharding? Why not explicitly divide up the key space so it's evenly
split?

A: If you could have a centralized server that assigns keys to shards
then an exact division is a great plan. Many systems do just that
(e.g., GFS or the shard master in lab 4). If you cannot have a central
server, then you need another plan for load balancing, and consistent
hashing is such a plan.

Q: Does BitTorrent use Chord?

A: The Bittorrent P2P Tracker uses Kademlia. Kademlia and Chord are
similar. Bittorrent itself doesn't use Chord or Kademlia.

Q: If you want to add fault-tolerance to a Chord-based system should
you replicate each Chord node using Raft?

A: Typical Chord-based applications don't need strong consistency, and
have weak durability requirements (e.g., often the client must refresh
the data periodically to ensure it isn't lost). So Raft seems too
heavy-weight. I know of only one design (Scatter) that combines Chord
and Paxos, where segments of the ring form a Paxos group to get
stronger guarantees. Google "Scatter SOSP" if you are curious.

Q: What if Chord DHT nodes are malicious?

A: Chord (and most peer-to-peer systems) cannot handle malicious
participants. An open Chord system is vulnerable to a Sybil attack: in an open
Chord system, an attacker can become a participant and create many chord nodes
under the attacker's control, and take over the whole system.  There are DHTs
that try to handle such attacks but it is challenging in a peer-to-peer setting
(e.g.,see http://pdos.csail.mit.edu/papers/whanau-nsdi10.pdf).

Chord and application on top of it provide some security measures, however.  For
example, node IDs are typically the SHA-1 of the IP address of a node, so that
attacker must have control of the IP address being used.  Application typically
advertise data in Chord under the key that corresponds to the SHA-1 of the data;
so when when application retrieves the data, it can check that is the right
data.

Q: Is Chord used anywhere in practice?

A: We don't know. Clearly Kademlia and Amazon's Dynamo are strongly influenced by
Chord. Rumor has it that Cisco uses Chord in some of its products.

Q: Could the performance be improved if the nodes knew more about
network locality?

A: Yes. The total latency for a lookup can be improved using proximity
routing (e.g., see
https://pdos.csail.mit.edu/papers/dhash:nsdi/paper.pdf).

Q: Is it possible to design a DHT in which lookups take less than
log(N) hops?

A: There are a number of O(1) hops DHTs, but they require more
bandwidth. Accordion is one that dynamically adjusts between O(1) and
O(log N): www.news.cs.nyu.edu/~jinyang/pub/nsdi05-accordion.pdf

Q: Does consistent hashing of keys still guarantee load balanced nodes if keys
are not evenly distributed?

A: Chord hashes the keys provided by the application using a SHA1 so that the
keys are well distributed in the key space.

Q: In the case of concurrent joins and failures, Chord pauses when a get fails
to find the key it was looking for. If there's constant activity, how can Chord
distinguish between the system not being stable and the key not actually
existing?

A: The idea is not to wait until the system is stable, because there might never
be a stable system. Instead, the plan is to just retry after a certain period of
time, because stabilization may have fix the routing info needed for that
lookup. With good chance, Chord will go back to case 1 or 2 mentioned in that
paragraph of the paper.

Q: If I introduce a malicious peer in Chord that keeps returning wrong
values or inexistent addresses how disruptive can it be to the whole
DHT? How does Bittorrent deal with particular issue?

A: A malicious node usually can't forge data, since the data is
usually signed or protected with a cryptographic hash. Bittorrent
files are protected in this way -- the original torrent file contains
the hash of the desired file, and the client hashes the file it
eventually gets to check that the content is correct.

Still, a malicious node can deny that content exists, or route lookups
incorrectly.

If it's just a few badly behaved nodes, then the system can cope by
replicating data at a few different nodes near the key in the DHT's ID
space. Lookups can try a few nearby nodes.

An attacker could pretend to be millions of separate nodes. If all of
the nodes only routed queries to other fake nodes, and they all denied
that the desired data existed, they could successfully prevent clients
from downloading. Bittorrent has a weak defense against this attack --
this is what the "token" mechanism in the reading refers to. Basically a
node is not allowed to join the Kademlia DHT unless it can prove that it
really receives packets sent to its claimed IP address. The idea is that
an attacker probably only controls a handful of IP addresses, so this
token defense will hopefully limit the attacker to joining the DHT only
a handful of times.

Some real-life attacks have managed to get control of hundreds or
millions of IP addresses (by manipulating the Internet routing system),
so the token defense it not bullet-proof.

Q: Why isn鈥檛 there a danger of improper load balancing if some keys
are simply used more than others?

A: That danger does exist. I think if it was a problem in practice,
system designers would replicate or cache data in the DHT nodes.

Bittorrent effectively does this in the way it uses Kademlia -- there
are typically many peers serving a given torrent, and each inserts an
entry into the Kademlia DHT declaring that it is willing to serve pieces
of the torrent. The inserts go to Kademlia nodes with IDs near the key
ID, not necessarily to the Kademlia node that's the real home of the key
ID. When a new client wants to find out about a torrent, its DHT lookup
stops as soon as it encounters any entries for the key, so it might not
have to bother the real DHT home node for the key.

System built on Chord (e.g. CFS) replicate a key/value pair at the r
nodes after the key's successor on the ring. get()s choose a random one
of those r nodes, to avoid hot-spots.

## LEC 20: Dynamo

6.824 2018 Lecture 15: Dynamo
=============================

Dynamo: Amazon's Highly Available Key-value Store
DeCandia et al, SOSP 2007

Why are we reading this paper?
  Database, eventually consistent, write any replica
     Like Bayou, with reconciliation
     Like Parameter Server, but geo-distributed
     A surprising design.
  A real system: used for e.g. shopping cart at Amazon
  More available than PNUTS, Spanner, FB MySQL, &c
  Less consistent than PNUTS, Spanner, FB MySQL &c
  Influential design; inspired e.g. Cassandra
  2007: before PNUTS, before Spanner

Their Obsessions
  SLA, e.g. 99.9th percentile of delay < 300 ms
  constant failures
  "data centers being destroyed by tornadoes"
  "always writeable"

Big picture
  [lots of data centers, Dynamo nodes]
  each item replicated at a few random nodes, by key hash

Why replicas at just a few sites? Why not replica at every site?
  with two data centers, site failure takes down 1/2 of nodes
    so need to be careful that *everything* replicated at *both* sites
  with 10 data centers, site failure affects small fraction of nodes
    so just need copies at a few sites

Where to place data -- consistent hashing
  [ring, and physical view of servers]
  node ID = random
  key ID = hash(key)
  coordinator: successor of key
    clients send puts/gets to coordinator
  replicas at successors -- "preference list"
  coordinator forwards puts (and gets...) to nodes on preference list

Consequences of mostly remote access (since no guaranteed local copy)
  most puts/gets may involve WAN traffic -- high delays
    the quorums will cut the tail end --- see below
  but can survive data centers going down

Why consistent hashing?
  Pro
    naturally somewhat balanced
    decentralized -- both lookup and join/leave
  Con (section 6.2)
    not really balanced (why not?), need virtual nodes
    hard to control placement (balancing popular keys, spread over sites)
    join/leave changes partition, requires data to shift

Failures
  Tension: temporary or permanent failure?
    node unreachable -- what to do?
    if temporary, store new puts elsewhere until node is available
    if permanent, need to make new replica of all content
  Dynamo itself treats all failures as temporary

Consequences of "always writeable"
  always writeable => no master! must be able to write locally.
     idea 1: sloppy quorums
  always writeable + failures = conflicting versions
     idea 2: eventual consistency
        idea 1 avoids inconsistencies when there are no failures

Idea #1: sloppy quorum
  try to get consistency benefits of single master if no failures
    but allows progress even if coordinator fails, which PNUTS does not
  when no failures, send reads/writes through single node
    the coordinator
    causes reads to see writes in the usual case
  but don't insist! allow reads/writes to any replica if failures

Temporary failure handling: quorum
  goal: do not block waiting for unreachable nodes
  goal: put should always succeed
  goal: get should have high prob of seeing most recent put(s)
  quorum: R + W > N
    never wait for all N
    but R and W will overlap
    cuts tail off delay distribution and tolerates some failures
  N is first N *reachable* nodes in preference list
    each node pings successors to keep rough estimate of up/down
    "sloppy" quorum, since nodes may disagree on reachable
  sloppy quorum means R/W overlap *not guaranteed*

coordinator handling of put/get:
  sends put/get to first N reachable nodes, in parallel
  put: waits for W replies
  get: waits for R replies
  if failures aren't too crazy, get will see all recent put versions

When might this quorum scheme *not* provide R/W intersection?

What if a put() leaves data far down the ring?
  after failures repaired, new data is beyond N?
  that server remembers a "hint" about where data really belongs
  forwards once real home is reachable
  also -- periodic "merkle tree" sync of key range

Idea #2: eventual consistency
  accept writes at any replica
  allow divergent replicas
  allow reads to see stale or conflicting data
  resolve multiple versions when failures go away
    latest version if no conflicting updates
    if conflicts, reader must merge and then write
  like Bayou and Ficus -- but in a DB

Unhappy consequences of eventual consistency
  May be no unique "latest version"
  Read can yield multiple conflicting versions
  Application must merge and resolve conflicts
  No atomic operations (e.g. no PNUTS test-and-set-write)

How can multiple versions arise?
  Maybe a node missed the latest write due to network problem
  So it has old data, should be superseded by newer put()s
  get() consults R, will likely see newer version as well as old

How can *conflicting* versions arise?
  N=3 R=2 W=2
  shopping cart, starts out empty ""
  preference list n1, n2, n3, n4
  client 1 wants to add item X
    get() from n1, n2, yields ""
    n1 and n2 fail
    put("X") goes to n3, n4
  n1, n2 revive
  client 3 wants to add Y
    get() from n1, n2 yields ""
    put("Y") to n1, n2
  client 3 wants to display cart
    get() from n1, n3 yields two values!
      "X" and "Y"
      neither supersedes the other -- the put()s conflicted

How should clients resolve conflicts on read?
  Depends on the application
  Shopping basket: merge by taking union?
    Would un-delete items removed
    Weaker than Bayou (which gets deletion right), but simpler
  Some apps probably can use latest wall-clock time
    E.g. if I'm updating my password
    Simpler for apps than merging
  Write the merged result back to Dynamo

Programming API
  All objects are immutable
  - get(k) may return multiple versions, along with "context"
  - put(k, v, context)
    creates a new version of k, attaching context
  The context is used to merge and keep track of dependencies, and
  detect how conflicts.  It consists of a VV of the object.

Version vectors
  Example tree of versions:
    [a:1]
      |
      +-------|
           [a:1,b:2]
    VVs indicate v2 supersedes v1
    Dynamo nodes automatically drop [a:1] in favor of [a:1,b:2]
  Example:
    [a:1]
      |
      +-------|
      |    [a:1,b:2]
      |
    [a:2]
    Client must merge

Won't the VVs get big?
  Yes, but slowly, since key mostly served from same N nodes
  Dynamo deletes least-recently-updated entry if VV has > 10 elements

Impact of deleting a VV entry?
  won't realize one version subsumes another, will merge when not needed:
    put@b: [b:4]
    put@a: [a:3, b:4]
    forget b:4: [a:3]
    now, if you sync w/ [b:4], looks like a merge is required
  forgetting the oldest is clever
    since that's the element most likely to be present in other branches
    so if it's missing, forces a merge
    forgetting *newest* would erase evidence of recent difference

Is client merge of conflicting versions always possible?
  Suppose we're keeping a counter, x
  x starts out 0
  incremented twice
  but failures prevent clients from seeing each others' writes
  After heal, client sees two versions, both x=1
  What's the correct merge result?
  Can the client figure it out?

What if two clients concurrently write w/o failure?
  e.g. two clients add diff items to same cart at same time
  Each does get-modify-put
  They both see the same initial version
  And they both send put() to same coordinator
  Will coordinator create two versions with conflicting VVs?
    We want that outcome, otherwise one was thrown away
    Paper doesn't say, but coordinator could detect problem via put() context

Permanent server failures / additions?
  Admin manually modifies the list of servers
  System shuffles data around -- this takes a long time!

The Question:
  It takes a while for notice of added/deleted server to become known
    to all other servers. Does this cause trouble?
  Deleted server might get put()s meant for its replacement.
  Deleted server might receive get()s after missing some put()s.
  Added server might miss some put()s b/c not known to coordinator.
  Added server might serve get()s before fully initialized.
  Dynamo probably will do the right thing:
    Quorum likely causes get() to see fresh data as well as stale.
    Replica sync (4.7) will fix missed get()s.

Is the design inherently low delay?
  No: client may be forced to contact distant coordinator
  No: some of the R/W nodes may be distant, coordinator must wait

What parts of design are likely to help limit 99.9th pctile delay?
  This is a question about variance, not mean
  Bad news: waiting for multiple servers takes *max* of delays, not e.g. avg
  Good news: Dynamo only waits for W or R out of N
    cuts off tail of delay distribution
    e.g. if nodes have 1% chance of being busy with something else
    or if a few nodes are broken, network overloaded, &c

No real Eval section, only Experience

How does Amazon use Dynamo?
  shopping cart (merge)
  session info (maybe Recently Visited &c?) (most recent TS)
  product list (mostly r/o, replication for high read throughput)

They claim main advantage of Dynamo is flexible N, R, W
  What do you get by varying them?
  N-R-W
  3-2-2 : default, reasonable fast R/W, reasonable durability
  3-3-1 : fast W, slow R, not very durable, not useful?
  3-1-3 : fast R, slow W, durable
  3-3-3 : ??? reduce chance of R missing W?
  3-1-1 : not useful?

They had to fiddle with the partitioning / placement / load balance (6.2)
  Old scheme:
    Random choice of node ID meant new node had to split old nodes' ranges
    Which required expensive scans of on-disk DBs
  New scheme:
    Pre-determined set of Q evenly divided ranges
    Each node is coordinator for a few of them
    New node takes over a few entire ranges
    Store each range in a file, can xfer whole file

How useful is ability to have multiple versions? (6.3)
  I.e. how useful is eventual consistency
  This is a Big Question for them
  6.3 claims 0.001% of reads see divergent versions
    I believe they mean conflicting versions (not benign multiple versions)
    Is that a lot, or a little?
  So perhaps 0.001% of writes benefitted from always-writeable?
    I.e. would have blocked in primary/backup scheme?
  Very hard to guess:
    They hint that the problem was concurrent writers, for which
      better solution is single master
    But also maybe their measurement doesn't count situations where
      availability would have been worse if single master

Performance / throughput (Figure 4, 6.1)
  Figure 4 says average 10ms read, 20 ms writes
    the 20 ms must include a disk write
    10 ms probably includes waiting for R/W of N
  Figure 4 says 99.9th pctil is about 100 or 200 ms
    Why?
    "request load, object sizes, locality patterns"
    does this mean sometimes they had to wait for coast-coast msg?

Puzzle: why are the average delays in Figure 4 and Table 2 so low?
  Implies they rarely wait for WAN delays
  But Section 6 says "multiple datacenters"
    You'd expect *most* coordinators and most nodes to be remote!
    Maybe all datacenters are near Seattle?
    Maybe because coordinators can be any node in the preference list?
      See last paragraph of 5
    Maybe W-1 copies in N are close by?

Wrap-up
  Big ideas:
    eventual consistency
    always writeable despite failures
    allow conflicting writes, client merges
  Awkward model for some applications (stale reads, merges)
    this is hard for us to tell from paper
  Maybe a good way to get high availability + no blocking on WAN
  Parameter Server uses similar ideas for ML applications
    no single master, conflicting writes okay
  No agreement on whether eventual consistency is good for storage systems

Dynamo FAQ

Q: What's a typical number of nodes in a Dynamo instance? Large enough that
vector clocks will become impractical large?

A: I don't know how big Amazon's Dynamo instances are, but I expect they are
sizeable. As long as there are no failures the same node will do the puts and
thus the VV stays small (1 item). With weird failure patterns it may grow large,
but if the VV grows larger than 10 items, Dynamo throws out the
least-recently-used entry. (This means that there could be cases that Dynamo
thinks there is a conflict, but if it would have remembered all entries, there
wasn't an actual conflict.)

Q: How can deleted items resurface in a shopping cart (Section 4.4)?

A: Suppose there are two copies of the shopping cart, each in a different data
center. Suppose both copies are identical and have one item in them. Now suppose
a user deletes the item, but the second copy is unreachable, maybe because of a
network partition. Dynamo will update the first copy, but not the second; the
user will see that the shopping cart is empty. Now the network reconnects, and
the two copies must be reconciled. The application using Dynamo is responsible
for doing the reconciliation since it knows the semantics of the objects. In the
case of a shopping cart, the application will take the union of the shopping
carts' contents (which is less sophisticated than Bayou but simpler). Now if the
user looks at the shopping cart again, the deleted item has re-appeared.

Q: How does Dynamo recover from permanent failure -- what is anti-entropy using
Merkle trees?

A: In this paper, anti-entropy is a big word for synchronizing two replicas. To
determine what is different between two replicas, Dynamo traverses a Merkle
representation of the two replicas. If a top-level node matches, then Dynamo
doesn't descend that branch. If a node in the tree don't match, they copy the
branch of the new version to the old one. The use of Merkle trees allows the
authors to copy only the parts of the tree that are different. Wikipedia has a
picture to illustrate: https://en.wikipedia.org/wiki/Merkle_tree.

Q: What are virtual nodes?

A: They are a scheme to balance the keys nicely across all nodes. If you throw N
balls in B bins, you get on average B/N balls in a bin but some bins may have
many balls. To ensure that all bins are close to the average, you can place
multiple virtual bins in each real bin. The average of multiple random variables
has lower variance than a single random variable.

Q: Will Dynamo's use of a DHT prevent it from scaling?

A: It is pretty-well understood now how to build DHTs that scale to large number
of nodes, even O(1) DHTs (e.g., see
http://www.news.cs.nyu.edu/~jinyang/pub/nsdi05-accordion.pdf). The Dynamo
solution as described is an O(1) DHT, with ok scalability. I think the authors
figured that they will use a solution from the literature once they actually
have a scaling problem.

Q: What's a gossip-based protocol?

A: Any system that doesn't have a master that knows about all participants in
the system typically has a protocol to find other members; often such protocols
are called gossip protocols, because the participants need to gossip (gather
information from other nodes) to determine who is part of the system. More
generally, gossip refers to information spreading over a whole system via pairs
of computers exchanging what they know. You can learn more about gossip
protocols from Wikipedia: https://en.wikipedia.org/wiki/Gossip_protocol.

## LEC 21: Peer-to-peer: Bitcoin

6.824 2018 Lecture 20: Bitcoin

Bitcoin: A Peer-to-Peer Electronic Cash System, by Satoshi Nakamoto, 2008

bitcoin:
  a digital currency
  a public ledger to prevent double-spending
  no centralized trust or mechanism <-- this is hard!
  malicious users ("Byzantine faults")

why might people want a digital currency?
  might make online payments easier
  credit cards have worked well but aren't perfect
    insecure -> fraud -> fees, restrictions, reversals
    record of all your purchases

what's hard technically?
  forgery
  double spending
  theft

what's hard socially/economically?
  why do Bitcoins have value?
  how to pay for infrastructure?
  monetary policy (intentional inflation &c)
  laws (taxes, laundering, drugs, terrorists)

idea: signed sequence of transactions
  (this is the straightforward part of Bitcoin)
  there are a bunch of coins, each owned by someone
  every coin has a sequence of transaction records
    one for each time this coin was transferred as payment
  a coin's latest transaction indicates who owns it now

what's in a transaction record?
  pub(user1): public key of new owner
  hash(prev): hash of this coin's previous transaction record
  sig(user2): signature over transaction by previous owner's private key
  (BitCoin is much more complex: amount (fractional), multiple in/out, ...)

transaction example:
  Y owns a coin, previously given to it by X:
    T7: pub(Y), hash(T6), sig(X)
  Y buys a hamburger from Z and pays with this coin
    Z sends public key to Y
    Y creates a new transaction and signs it
    T8: pub(Z), hash(T7), sig(Y)
  Y sends transaction record to Z
  Z verifies:
    T8's sig() corresponds to T7's pub()
  Z gives hamburger to Y

Z's "balance" is set of unspent transactions for which Z knows private key
  the "identity" of a coin is the (hash of) its most recent xaction
  Z "owns" a coin = Z knows private key for "new owner" public key in latest xaction

can anyone other than the owner spend a coin?
  current owner's private key needed to sign next transaction
  danger: attacker can steal Z's private key, e.g. from PC or smartphone

can a coin's owner spend it twice in this scheme?
  Y creates two transactions for same coin: Y->Z, Y->Q
    both with hash(T7)
  Y shows different transactions to Z and Q
  both transactions look good, including signatures and hash
  now both Z and Q will give hamburgers to Y

why was double-spending possible?
  b/c Z and Q didn't know complete set of transactions

what do we need?
  publish log of all transactions to everyone, in same order
    so Q knows about Y->Z, and will reject Y->Q
    a "public ledger"
  ensure Y can't un-publish a transaction

why not rely on CitiBank, or Federal Reserve, to publish transactions?
  not everyone trusts them
  they might be tempted to reverse or restrict

why not publish transactions like this:
  1000s of peers, run by anybody, no trust required in any one peer
  peers flood new transactions over "overlay"
  transaction Y->Z only acceptable if majority of peers think it is valid
    i.e. they don't know of any Y->Q
    hopefully majority overlap ensures double-spend is detected
  how to count votes?
    how to even count peers so you know what a majority is?
    perhaps distinct IP addresses?
  problem: "sybil attack"
    IP addresses are not secure -- easy to forge
    attacker pretends to have 10,000 computers -- majority
    when Z asks, attacker's majority says "we only know of Y->Z"
    when Q asks, attacker's majority says "we only know of Y->Q"
  voting is hard in "open" p2p schemes

the BitCoin block chain
  the goal: agreement on transaction log to prevent double-spending
  the block chain contains transactions on all coins
  many peers
    each with a complete copy of the whole chain
    proposed transactions flooded to all peers
    new blocks flooded to all peers
  each block:
    hash(prevblock)
    set of transactions
    "nonce" (not quite a nonce in the usual cryptographic sense)
    current time (wall clock timestamp)
  new block every 10 minutes containing xactions since prev block
  payee doesn't believe transaction until it's in the block chain

who creates each new block?
  this is "mining"
  all peers try
  requirement: hash(block) has N leading zeros
  each peer tries nonce values until this works out
  trying one nonce is fast, but most nonces won't work
    it's like flipping a zillion-sided coin until it comes up heads
    each flip has an independent (small) chance of success
    mining a block *not* a specific fixed amount of work
  it would likely take one CPU months to create one block
  but thousands of peers are working on it
  such that expected time to first to find is about 10 minutes
  the winner floods the new block to all peers

how does a Y->Z transaction work w/ block chain?
  start: all peers know ...<-B5
    and are working on B6 (trying different nonces)
  Y sends Y->Z transaction to peers, which flood it
  peers buffer the transaction until B6 computed
  peers that heard Y->Z include it in next block
  so eventually ...<-B5<-B6<-B7, where B7 includes Y->Z

Q: could there be *two* different successors to B6?
A: yes, in (at least) two situations:
   1) two peers both get lucky (unlikely, given variance of block time)
   2) network partition
  in both cases, the blockchain temporarily forks
    peers work on whichever block they heard about before
    but switch to longer chain if they become aware of one

how is a forked chain resolved?
  each peer initially believes whichever of BZ/BQ it saw first
  tries to create a successor
  if many more saw BZ than BQ, more will mine for BZ,
    so BZ successor likely to be created first
  if exactly half-and-half, one fork likely to be extended first
    since significant variance in mining success time
  peers always switch to mining the longest fork, re-inforcing agreement

what if Y sends out Y->Z and Y->Q at the same time?
  i.e. Y attempts to double-spend
  no correct peer will accept both, so a block will have one but not both

what happens if Y tells some peers about Y->Z, others about Y->Q?
  perhaps use network DoS to prevent full flooding of either
  perhaps there will be a fork: B6<-BZ and B6<-BQ

thus:
  temporary double spending is possible, due to forks
  but one side or the other of the fork highly likely to disappear
  thus if Z sees Y->Z with a few blocks after it,
    it's very unlikely that it could be overtaken by a
    different fork containing Y->Q
  if Z is selling a high-value item, Z should wait for a few
    blocks before shipping it
  if Z is selling something cheap, maybe OK to wait just for some peers
    to see Y->Z and validate it (but not in block)

can an attacker modify a block in the middle of the block chain?
  not directly, since subsequent block holds block's hash

could attacker start a fork from an old block, with Y->Q instead of Y->Z?
  yes -- but fork must be longer in order for peers to accept it
  so if attacker starts N blocks behind, it must generate N+M+1
    blocks on its fork before main fork is extended by M
  i.e. attacker must mine blocks *faster* than the other peers
  with just one CPU, will take months to create even a few blocks
    by that time the main chain will be much longer
    no peer will switch to the attacker's shorter chain
  if the attacker has 1000s of CPUs -- more than all the honest
    bitcoin peers -- then the attacker can create the longest fork,
    everyone will switch to it, allowing the attacker to double-spend

there's a majority voting system hiding here
  peers cast votes by mining to extend the longest chain

summary:
  if attacker controls majority of CPU power, can force honest
    peers to switch from real chain to one created by the attacker
  otherwise not

validation checks:
  peer, new xaction:
    no other transaction spends the same previous transaction
    signature is by private key of pub key in previous transaction
    then will add transaction to txn list for next block to mine
  peer, new block:
    hash value has enough leading zeroes (i.e. nonce is right, proves work)
    previous block hash exists
    all transactions in block are valid
    peer switches to new chain if longer than current longest
  Z:
    (some clients rely on peers to do above checks, some don't)
    Y->Z is in a block
    Z's public key / address is in the transaction
    there's several more blocks in the chain
  (other stuff has to be checked as well, lots of details)

where does each bitcoin originally come from?
  each time a peer mines a block, it gets 12.5 bitcoins (currently)
  it puts its public key in a special transaction in the block
  this is incentive for people to operate bitcoin peers

Q: what if lots of miners join, so blocks are created faster?

Q: 10 minutes is annoying; could it be made much shorter?

Q: are transactions anonymous?

Q: if I steal bitcoins, is it safe to spend them?

Q: can bitcoins be forged, i.e. a totally fake coin created?

Q: what can adversary do with a majority of CPU power in the world?
   can double-spend and un-spend, by forking
   cannot steal others' bitcoins
   can prevent xaction from entering chain

Q: what if the block format needs to be changed?
   esp if new format wouldn't be acceptable to previous s/w version?

Q: how do peers find each other?

Q: what if a peer has been tricked into only talking to corrupt peers?
   how about if it talks to one good peer and many colluding bad peers?

Q: could a brand-new peer be tricked into using the wrong chain entirely?
   what if a peer rejoins after a few years disconnection?
   a few days of disconnection?

Q: how rich are you likely to get with one machine mining?

Q: why does it make sense for the mining reward to decrease with time?

Q: is it a problem that there will be a fixed number of coins?
   what if the real economy grows (or shrinks)?

Q: why do bitcoins have value?
   e.g., 1 BTC appears to be around $8700 on may 14 2018.

Q: will bitcoin scale well?
   as transaction rate increases?
     claim CPU limits to 4,000 tps (signature checks)
     more than Visa but less than cash
   as block chain length increases?
     do you ever need to look at very old blocks?
     do you ever need to xfer the whole block chain?
     merkle tree: block headers vs txn data.
   sadly, the maximum block size is limited to one megabyte

Q: could Bitcoin have been just a ledger w/o a new currency?
   e.g. have dollars be the currency?
   since the currency part is pretty awkward.
   (settlement... mining incentive...)

key idea: block chain
  public ledger is a great idea
  decentralization might be good
  mining is a clever way to avoid sybil attacks
  tieing ledger to new currency seems awkward, maybe necessary

https://michaelnielsen.org/ddi/how-the-bitcoin-protocol-actually-works/

Bitcoin FAQ

Q: I don't understand why the blockchain is so important. Isn't the
requirement for the owner's signature on each transaction enough to
prevent bitcoins from being stolen?

A: The signature is not enough, because it doesn't prevent the owner
from spending money twice: signing two transactions that transfer the
same bitcoin to different recipients. The blockchain acts as a
publishing system to try to ensure that once a bitcoin has been spent
once, lots of participants will know, and will be able to reject a
second spend.

Q: What does Bitcoin need to define a new currency? Wouldn't it be more
convenient to use an existing currency like dollars?

A: The new currency (Bitcoins) allows the system to reward miners with
freshly created money; this would be harder with dollars because it's
illegal for ordinary people to create fresh dollars. And using dollars
would require a separate settlement system: if I used the blockchain
to record a payment to someone, I still need to send the recipient
dollars via a bank transfer or physical cash.

Q: Why is the purpose of proof-of-work?

A: It makes it hard for an attacker to convince the system to switch
to a blockchain fork in which a coin is spent in a different way than
in the main fork. You can view proof-of-work as making a random choice
over the participating CPUs of who gets to choose which fork to
extend. If the attacker controls only a few CPUs, the attacker won't
be able to work hard enough to extend a new malicious fork fast enough
to overtake the main blockchain.

Q: Could a Bitcoin-like system use something less wasteful than
proof-of-work?

A: Proof-of-work is hard to fake or simulate, a nice property in a
totally open system like Bitcoin where you cannot trust anyone to
follow rules. There are some alternate schemes; search the web for
proof-of-stake or look at Algorand and Byzcoin, for example. In a
smallish closed system, in which the participants are known though
not entirely trusted, Byzantine agreement protocols could be used, as
in Hyperledger.

Q: Can Alice spend the same coin twice by sending "pay Bob" and "pay
Charlie" to different subsets of miners?

A: The most likely scenario is that one subset of miners finds the
nonce for a new block first. Let's assume the first block to be found
is B50 and it contains "pay Bob". This block will be flooded to all
miners, so the miners working on "pay Charlie" will switch to mining a
successor block to B50. These miners validate transactions they place
in blocks, so they will notice that the "pay Charlie" coin was spent in
B50, and they will ignore the "pay Charlie" transaction. Thus, in this
scenario, double-spend won't work.

There's a small chance that two miners find blocks at the same time,
perhaps B50' containing "pay Bob" and B50'' containing "pay Charlie". At
this point there's a fork in the block chain. These two blocks will be
flooded to all the nodes. Each node will start mining a successor to one
of them (the first it hears). Again the most likely outcome is that a
single miner will finish significantly before any other miner, and flood
the successor, and most peers will switch that winning fork. The chance
of repeatedly having two miners simultaneously find blocks gets very
small as the forks get longer. So eventually all the peers will switch
to the same fork, and in fork there will be only one spend of the coin.

The possibility of accidentally having a short-lived fork is the reason
that careful clients will wait until there are a few successor blocks
before believing a transaction.

Q: It takes an average of 10 minutes for a Bitcoin block to be
validated. Does this mean that the parties involved aren't sure if the
transaction really happened until 10 minutes later?

A: The 10 minutes is awkward. But it's not always a problem. For
example, suppose you buy a toaster oven with Bitcoin from a web site.
The web site can check that the transaction is known by a few servers,
though not yet in a block, and show you a "purchase completed" page.
Before shipping it to you, they should check that the transaction is
in a block. For low-value in-person transactions, such as buying a cup
of coffee, it's probably enough for the seller to ask a few peers to
check that the bitcoins haven't already been spent (i.e. it's
reasonably safe to not bother waiting for the transaction to appear in
the blockchain at all). For a large in-person purchase (e.g., a car),
it is important to wait for sufficiently long to be assured that the
block will stay in the block chain before handing over the goods.

Q: What can be done to speed up transactions on the blockchain?

A: I think the constraint here is that 10 minutes needs to be much
larger (i.e. >= 10x) than the time to broadcast a newly found block to
all peers. The point of that is to minimize the chances of two peers
finding new blocks at about the same time, before hearing about the
other peer's block. Two new blocks at the same time is a fork; forks
are bad since they cause disagreement about which transactions are
real, and they waste miners' time. Since blocks can be pretty big (up
to a megabyte), and peers could have slow Internet links, and the
diameter of the peer network might be large, it could easily take a
minute to flood a new block. If one could reduce the flooding time,
then the 10 minutes could also be reduced.

Q: The entire blockchain needs to be downloaded before a node can
participate in the network. Won't that take an impractically long time
as the blockchain grows?

A: It's true that it takes a while for a new node to get all the
transactions. But once a given server has done this work, it can save
the block chain, and doesn't need to fetch it again. It only needs to
know about new blocks, which is not a huge burden. I think most
ordinary users of Bitcoin won't run full Bitcoin nodes; instead they
will one way or another trust some full nodes.

Q: Is it feasible for an attacker to gain a majority of the computing
power among peers? What are the implications for bitcoin if this happens?

A: It may be feasible; some people think that big cooperative groups
of miners have been close to a majority at times:
http://www.coindesk.com/51-attacks-real-threat-bitcoin/

If >50% of compute power are controlled by a single user (or by a clique
of colluding users), they can extend the blockchain from any block they
like, and can double-spend money. Hence, Bitcoin's security would be
broken if this happened.

Q: From some news stories, I have heard that a large number of bitcoin
miners are controlled by a small number of companies.

A: This is true. See here: https://blockchain.info/pools. It looks like
three mining pools together hold >51% of the compute power today, and
two come to 40%.

Q: Are there any ways for Bitcoin mining to do useful work, beyond simply
brute-force calculating SHA-256 hashes?

A: Maybe -- here are two attempts to do what you suggest:
https://www.cs.umd.edu/~elaine/docs/permacoin.pdf
http://primecoin.io/

Q: There is hardware specifically designed to mine Bitcoin. How does
this type of hardware differ from the type of hardware in a laptop?

A: Mining hardware has a lot of transistors dedicated to computing
SHA256 quickly, but is not particularly fast for other operations.
Ordinary server and laptop CPUs can do many things (e.g. floating
point division) reasonably quickly, but don't have so much hardware
dedicated to SHA256 specifically. Some Intel CPUs do have instructions
specifically for SHA256; however, they aren't competitive with
specialized Bitcoin hardware that massively parallelizes the hashing
using lots of dedicated transistors.

Q: The paper estimates that the disk space required to store the block
chain will by 4.2 megabytes per year. That seems very low!

A: The 4.2 MB/year is for just the block headers, and is still the
actual rate of growth. The current 60+GB is for full blocks.

Q: Would the advent of quantum computing break the bitcoin system?

A: Here's a plausible-looking article:
http://www.bitcoinnotbombs.com/bitcoin-vs-the-nsas-quantum-computer/
Quantum computers might be able to forge bitcoin's digital signatures
(ECDSA). That is, once you send out a transaction with your public key
in it, someone with a quantum computer could probably sign a different
transaction for your money, and there's a reasonable chance that the
bitcoin system would see the attacker's transaction before your
transaction.

Q: Bitcoin uses the hash of the transaction record to identify the
transaction, so it can be named in future transactions. Is this
guaranteed to lead to unique IDs?

A: The hashes are technically not guaranteed to be unique. But in
practice the hash function (SHA-256) is believed to produce different
outputs for different inputs with fantastically high probability.

Q: It sounds like anyone can create new Bitcoins. Why is that OK?
Won't it lead to forgery or inflation?

A: Only the person who first computes a proper nonce for the current
last block in the chain gets the 12.5-bitcoin reward for "mining" it. It
takes a huge amount of computation to do this. If you buy a computer
and have it spend all its time attempting to mine bitcoin blocks, you
will not make enough bitcoins to pay for the computer.

Q: The paper mentions that some amount of fraud is admissible; where
does this fraud come from?

A: This part of the paper is about problems with the current way of
paying for things, e.g. credit cards. Fraud occurs when you buy
something on the Internet, but the seller keeps the money and doesn't
send you the item. Or if a merchant remembers your credit card number,
and buys things with it without your permission. Or if someone buys
something with a credit card, but never pays the credit card bill.

Q: Has there been fraudulent use of Bitcoin?

A: Yes. I think most of the problems have been at web sites that act
as wallets to store peoples' bitcoin private keys. Such web sites,
since they have access to the private keys, can transfer their
customers' money to anyone. So someone who works at (or breaks into)
such a web site can steal the customers' Bitcoins.

Q: Satoshi's paper mentions that each transaction has its own
transaction fees that are given to whoever mined the block. Why would
a miner not simply try to mine blocks with transactions with the
highest transaction fees?

A: Miners do favor transactions with higher fees. You can read about
typical approaches here:
https://en.bitcoin.it/wiki/Transaction_fees
And here's a graph (the red line) of how long your transaction waits
as a function of how high a fee you offer:
https://bitcoinfees.github.io/misc/profile/

Q: Why would a miner bother including transactions that yield no fee?

A: I think many don't mine no-fee transactions any more.

Q: How are transaction fees determined/advertised?

A: Have a look here:
https://en.bitcoin.it/wiki/Transaction_fees
It sounds like (by default) wallets look in the block chain at the
recent correlation between fee and time until a transaction is
included in a mined block, and choose a fee that correlates with
relatively quick inclusion. I think the underlying difficulty is that
it's hard to know what algorithms the miners use to pick which
transactions to include in a block; different miners probably do
different things.

Q: What are some techniques for storing my personal bitcoins, in
particular the private keys needed to spend my bitcoins? I've heard of
people printing out the keys, replicating them on USB, etc. Does a
secure online repository exist?

A: Any scheme that keeps the private keys on a computer attached to
the Internet is a tempting target for thieves. On the other hand, it's
a pain to use your bitcoins if the private keys are on a sheet of
paper. So my guess is that careful people store the private keys for
small amounts on their computer, but for large balances they store the
keys offline.

Q: What other kinds of virtual currency were there before and after
Bitcoin (I know the paper mentioned hashcash)? What was different
about Bitcoin that led it to have more success than its predecessors?

A: There have been many proposals for digital cash systems, none with
any noticeable success. And of course Bitcoin has inspired many
competing and variant schemes, again none (as far as I can tell) with
much success. It's tempting to think that Bitcoin has succeeded
because its design is more clever than others: that it has just the
right blend of incentives and decentralization and ease of use. But
there are too many failed yet apparently well-designed technologies
out there for me to believe that.

Q: What happens when more (or fewer) people mine Bitcoin?

A: Bitcoin adjusts the difficulty to match the measured compute power
devoted to mining. So if more and more computers mine, the mining
difficulty will get harder, but only hard enough to maintain the
inter-block interval at 10 minutes. If lots of people stop mining, the
difficulty will decrease. This mechanism won't prevent new blocks from
being created, it will just ensure that it takes about 10 minutes to
create each one.

Q: Is there any way to make Bitcoin completely anonymous?

A: Have a look here: https://en.wikipedia.org/wiki/Zerocoin

Q: If I lose the private key(s) associated with the bitcoins I own,
how can I get my money back?

A: You can't.

Q: What do people buy and sell with bitcoins?

A: There seems to be a fair amount of illegal activity that exploits
Bitcoin's relative anonymity (buying illegal drugs, demanding ransom).
You can buy some ordinary (legal) stuff on the Internet with Bitcoin
too; have a look here:
http://www.coindesk.com/information/what-can-you-buy-with-bitcoins/
It's a bit of a pain, though, so I don't imagine many non-enthusiasts
would use bitcoin in preference to a credit card, given the choice.

Q: Why is bitcoin illegal in some countries?

A: Here are some guesses.

Many governments adjust the supply of money in order to achieve
certain economic goals, such as low inflation, high employment, and
stable exchange rates. Widespread use of bitcoin may make that harder.

Many governments regulate banks (and things that function as banks) in
order to prevent problems, e.g. banks going out of business and
thereby causing their customers to lose deposits. This has happened to
some bitcoin exchanges. Since bitcoin can't easily be regulated, maybe
the next best thing is to outlaw it.

Bitcoin seems particularly suited to certain illegal transactions
because it is fairly anonymous. Governments regulate big transfers of
conventional money (banks must report big transfers) in order to track
illegal activity; but you can't easily do this with bitcoin.

Q: Why do bitcoins have any value at all? Why do people accept it as
money?

Because other people are willing to sell things in return for
bitcoins, and are willing to exchange bitcoins for ordinary currency
such as dollars. This is a circular argument, but has worked many
times in the past; consider why people view baseball trading cards as
having value, or why they think paper money has value.

Q: How is the price of Bitcoin determined?

A: The price of Bitcoin in other currencies (e.g. euros or dollars) is
determined by supply and demand. If more people want to buy Bitcoins
than sell them, the price will go up. If the opposite, then the price
will go down. There is no single price; instead, there is just recent
history of what prices people have been willing to buy and sell at on
public exchanges. The public exchanges bring buyers and sellers
together, and publish the prices they agree to:

  https://bitcoin.org/en/exchanges

Q: Why is the price of bitcoin so volatile?

A: The price is driven partially by people's hopes and fears. When
they are optimistic about Bitcoin, or see that the price is rising,
they buy so as not to miss out, and thus bid the price up further.
When they read negative news stories about Bitcoin or the economy in
general, they sell out of fear that the price will drop and cause them
to lose money. This kind of speculation happens with many goods;
there's nothing special about Bitcoin in this respect. For example:

  https://en.wikipedia.org/wiki/Tulip_mania


## LEC 22: Project demos

Q: Why are we reading this paper?

A: Because it is a fun, intriguing read.  This paper was "written" (see
https://pdos.csail.mit.edu/archive/scigen/) by a 6.824 TAs many years ago, and
assigning it has become a 6.824 tradition. We enjoy reading the answers to the
posted question.  The assignment also has a slightly serious undertone: be
careful with what you read and believe.

## Exam

6.824 2020 Midterm Exam

*** MapReduce (1)

This is a 90-minute exam. Please write down any assumptions you make.
You are allowed to consult 6.824 papers, notes, and lab code. If you
have questions, please ask them on Piazza in a private post to the
staff. Please don't discuss the exam with anyone.


Recall the MapReduce paper by Dean and Ghemawat. You can see example
application map and reduce functions in Section 2.1.

Suppose you have to find the maximum of a very long list of numbers.
The input numbers are split up into a set of 100 files stored in GFS,
each file with many numbers, one number per line. The output should be
a single number. Design a map and a reduce function to find the
maximum.

Explain what your application's map function does.

Answer: find the maximum of its input file, and emit
just one item of output, with key="" and value=<the maximum>.

Explain what your application's reduce function does.

Answer: Find the maximum of the set of values passed, and emit it as
the output.

For a single execution of your application, how many times will
MapReduce call your reduce function?

Answer: One.

*** MapReduce (2)

Alyssa has implemented MapReduce for 6.824 Lab 1. Her worker code for
map tasks writes intermediate output files with names like "mr-{map
task index}-{reduce task index}", using this code:

outputName := fmt.Sprintf("mr-%d-%d", mapIndex, reduceBucket)
file, _ := os.Create(outputName)
//
// write data to file, piece by piece ...
//
file.Close()

Note that Alyssa has ignored the advice to write to a temporary file
and then call os.Rename(). Other than this code, Alyssa's MapReduce
implementation is correct.

Reminders: os.Create() truncates the file if it already exists. In the
lab (unlike the original MapReduce paper) the intermediate files are
in a file system that is shared among all workers.

Explain why the code shown above is not correct. Describe a scenario
in which a MapReduce job produces incorrect results due to the above
code.

Answer: Suppose the master starts a map task on worker W1, but W1 is
slow, so after a while the master starts the same map task on W2. W2
finishes, writes its intermediate output file, and tells the master it
has finished. The master starts a reduce task that reads the
intermediate output file. However, just as the reduce task is starting
to read, the original map task execution of W1 is finishing, and W1
calls os.Create(). The os.Create() will truncate the file so that it
has no content. Now the reduce task will see an empty file, rather
than the correct intermediate output.

*** GFS

Consider the paper "The Google File System" by Ghemawat et al.

Suppose that, at the moment, nobody is writing to the GFS file named
"info". Two clients open "info" and read it from start to finish at
the same time. Both clients' cached meta-data information about file
"info" is correct and up-to-date. Are the two clients guaranteed to
see the same content? If yes, explain how GFS maintains this
guarantee; if no, describe an example scenario in which the clients
could see different content.

Answer: No, the two clients may see different content. Two
chunkservers with the same version of the same chunk may store
different content for that chunk. This can arise if a client
previously issued a record append to the file, and the primary asked
the secondaries to execute the append, but one of the secondaries
didn't receive the primary's request. The primary doesn't do anything
to recover from this; it just returns an error to the writing client.
Thus if the two clients read from different chunkservers, they may see
different bytes.

*** Raft (1)

Ben Bitdiddle is working on Lab 2C. He sees that Figure 2 in the Raft
paper requires that each peer remember currentTerm as persistent
state. He thinks that storing currentTerm persistently is not
necessary. Instead, Ben modifies his Raft implementation so that when
a peer restarts, it first loads its log from persistent storage, and
then initializes currentTerm from the Term stored in the last log
entry. If the peer is starting for the very first time, Ben's code
initializes currentTerm to zero.

Ben is making a mistake. Describe a specific sequence of events in
which Ben's change would lead to different peers committing different
commands at the same index.

Answer: Peer P1's last log entry has term 10. P1 receives a
VoteRequest for term 11 from peer P2, and answers "yes". Then P1
crashes, and restarts. P1 initializes currentTerm from the term in its
last log entry, which is 10. Now P1 receives a VoteRequest from peer
P3 for term 11. P1 will vote for P3 for term 11 even though it
previously voted for P2.

*** Raft (2)

Bob starts with a correct Raft implementation that follows the paper's
Figure 2. However, he changes the processing of AppendEntries RPCs so
that, instead of looking for conflicting entries, his code simply
overwrites the local log with the received entries. That is, in the
receiver implementation for AppendEntries in Figure 2, he effectively
replaces step 3 with "3. Delete entries after prevLogIndex from the
log." In Go, this would look like:

rf.log = rf.log\[0:args.PrevLogIndex+1\]
rf.log = append(rf.log, args.Entries...)

Bob finds that because of this change, his Raft peers sometimes
commit different commands at the same log index. Describe a specific
sequence of events in which Bob's change causes different peers to
commit different commands at the same log index.

Answer:

(0) Assume three peers, 1 2 and 3.
(1) Server 1 is leader on term 1
(2) Server 1 writes "A" at index 1 on term 1.
(3) Server 1 sends AppendEntries RPC to Server 2 with "A", but it is delayed.
(4) Server 1 writes "B" at index 2 on term 1.
(5) Server 2 acknowledges ["A", "B"] in its log
(6) Server 1 commits/applies both "A" and "B"
(7) The delayed AppendEntries arrives at Server 2, and 2 updates its log to ["A"]
(8) Server 3, which only got the first entry ["A"], requests vote on term 2
(9) Server 2 grants the vote since their logs are identical.
(10) Server 3 writes "C" at index 2 on term 2.
(11) Servers 2 and 3 commit/apply this, but it differs from what Server 2 committed.

*** ZooKeeper

Section 4.3 (typo: should be 4.4) of the ZooKeeper paper says that, in
some circumstances, read operations may return stale values. Consider
the Simple Locks without Herd Effect example in Section 2.4. The
getChildren() in step 2 is a read operation, and thus may return
out-of-date results. Suppose client C1 holds the lock, client C2
wishes to acquire it, and client C2 has just called getChildren() in
step 2. Could the fact that getChildren() can return stale results
cause C2 to not see C1's "lock-" file, and decide in step 3 that C2
holds the lock? It turns out this cannot happen. Explain why not.

Answer: ZooKeeper does promise that each of a client's operations
executes at a particular point in the overall write stream, and that
each of a client's operations executes at a point at least as recent
as the previous operation. This means that the getChildren() will read
from state that is at least as up to date as the client's preceding
create(). The lock holder must have executed its create() even
earlier. So the getChildren() is guaranteed to see the lock file
created by the lock holder.

*** CRAQ

Refer to the paper "Object Storage on CRAQ" by Terrace and Freedman.

Item 4 in Section 2.3 says that, if a client read request arrives and
the latest version is dirty, the node should ask the tail for the
latest committed version number. Suppose, instead, that the node
replied with that dirty version (and did not send a version query to
the tail). This change would cause reads to reflect the most recent
write that the node is aware of. Describe a specific sequence of
events in which this change would cause a violation of the paper's
goal of strong consistency.

(Note that this question is not the same as the lecture question
posted on the web site; this exam question asks about returning the
most recent dirty version, whereas the lecture question asked about
returning the most recent clean version.)

Answer: Suppose the chain consists of servers S1, S2, S3. The value
for X starts out as 1. Client C1 has issued a write that sets X to 2,
and the write has reached S1, but not S2 or S3. Client C2 reads X from
S1 and sees value 2. After C2's read completes, client C2 reads X
again, this time from S3, and sees value 1. Since the order of written
values was 1, then 2, and the reads observed 2, then 1, there is no
way to fit the writes and the two reads into an order that obeys real
time. So the result is not linearizable.

Note that two clients seeing different values if they read at the same
time is not a violation of linearizability. Both reads are concurrent
with the write, so one read can be ordered before the write, and the
other after the write. It is only if the reads are *not* concurrent
with each other, and the second one yields an older value than the
first one, that there is a violation.

*** Frangipani

Consider the paper "Frangipani: A Scalable Distributed File System" by
Thekkath et al.

Aviva and Chetty work in adjacent cubicles at Yoyodyne Enterprises.
They use desktop workstations that share a file system using
Frangipani. Each workstation runs its own Frangipani file server
module, as in the paper's Figure 2. Aviva and Chetty both use the file
/project/util.go, which is stored in Frangipani.

Chetty reads /project/util.go, and her workstation's Frangipani caches
the file's contents. Aviva modifies /project/util.go on her
workstation, and notifies Chetty of the change by yelling over the
cubicle wall. Chetty reads the file again, and sees Aviva's changes.

Explain the sequence of steps Frangipani takes that guarantees that
Chetty will see Aviva's changes next time Chetty reads the file,
despite having cached the old version.

Answer: Before Aviva's workstation (WA) can modify the file, WA must
get an exclusive lock on the file. WA asks the lock service for the
lock; the lock service asks Chetty's workstation (WC) to release the
lock. Before it releases the lock, deletes all data covered by the
lock from its cache, including the cached file content. Now WA can
modify the file. Next time WC tries to read the file, it must acquire
the lock, which causes the lock server to ask WA to release it, which
causes WA to write any modified content from the file to Petal.
Because WC deleted the file content when it released the lock, it must
re-read it from Petal after acquiring the lock, and thus will see the
updated content.

*** 6.824

Which papers should we omit in future 6.824 years,
because they are not useful or are too hard to understand?

[ ] MapReduce
[ ] GFS                  xx
[ ] VMware FT            xxx
[ ] The Go Memory Model  xxxxxxxxxxx
[ ] Raft
[ ] ZooKeeper            xxxx
[ ] CRAQ                 xxxxxxx
[ ] Aurora               xxxxxxxxxxxxxxxxxxxxxx
[ ] Frangipani           xxxxxxxxxxxx

Which papers did you find most useful?

[ ] MapReduce            xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
[ ] GFS                  xxxxxxxxxxxxxxxxxxxxxxxxxxxx
[ ] VMware FT            xxxxxxxxxxxxxxxxxxx
[ ] The Go Memory Model  xxxxxxxxxx
[ ] Raft                 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
[ ] ZooKeeper            xxxxxxxxxxxxxxxxx
[ ] CRAQ                 xxxxxxxxxxxxxxxxxxx
[ ] Aurora               xxxxxxxxxx
[ ] Frangipani           xxxxxxxxxxxx

What should we change about the course to make it better?

Answer (sampled): More office hours. More background material, e.g.
how databases work. Better lab testing and debugging tools. More
lecture focus on high-level concepts. More lecture focus on paper
technical details. Labs that don't depend on each other.

6.824 2020 Final Exam

*** Transactions and Serializability

This is a 120-minute exam. Please write down any assumptions you make. You are
allowed to consult 6.824 papers, notes, and lab code. If you have questions,
please ask them on Piazza in a private post to the staff. Please don't discuss
the exam with anyone.

Alyssa has built a storage system that provides its clients with
transactions. Alyssa won't tell us how her system works, but she does
guarantee that the transactions are serializable. Following Section
9.1.6 of the reading for Lecture 12, serializable means that, if you
run a bunch of concurrent transactions and they produce some results
(output and updated database records), there exists some serial order
of those transactions that would, if followed, yield the same results.

You start three concurrent transactions at about the same time:

T1:
  begin()
  put(y, 2)
  end()

T2:
  begin()
  put(x, 99)
  put(y, 99)
  put(z, 99)
  end()

T3:
  begin()
  tmpx = get(x)
  tmpy = get(y)
  tmpz = get(z)
  print tmpx, tmpy, tmpz
  end()

x, y, and z are identifiers of different database records. All three
start out with value 0. put() and get() write and read database
records, and begin() and end() indicate the start and finish of a
transaction. There is no activity in the system other than these three
transactions, and no failures occur.

If T3 prints 99, 2, 99, is that a serializable result? Explain why or
why not.

Answer: Yes, it is a serializable result. The serial order that yields
that result is T2, T1, T3.

---

If T3 prints 0, 2, 99, is that a serializable result? Explain why or
why not.

Answer: No, it is not a serializable result. One way to demonstrate
this is to try all six possible serial orders, and observe that none
of them yield this result. Another is to observe that T3 saw x=0, so
T3 must appear before T2 in any equivalent serial order; and T3 saw
z=99, so T3 must appear after T2; but both cannot be true.

*** Two-Phase Commit

Recall the Two-Phase Commit protocol for distributed transactions
from Lecture 12 and Section 9.6.3 of the reading for Lecture 12.

A common criticism of two-phase commit is that it causes workers to
hold locks for a long time; for example, in Figure 9.37 (page 9-89),
each worker must hold its locks while waiting to receive the COMMIT
message.

What would go wrong if each worker released its locks after replying
to the PREPARE message?

Answer: Suppose some worker W1 is part of transaction T1. When W1
receives a PREPARE message, it doesn't yet know whether T1 will commit
or abort. If W1 releases its locks after the PREPARE, and lets other
transactions see values written by the not-yet-committed transaction
T1, and T1 aborts, then those other transactions will use values that
never existed. If W1 releases its locks but doesn't update values
until T1 commits, then another transaction may aquire T1's locks and
read some values as they existed before T1, and some as updated by T1,
which is not generally serializable.

*** Spanner

Refer to the paper "Spanner: Google's Globally-Distributed Database,"
by Corbett et al.

The start of Section 4.1.2 says that all of a read/write
transactions's writes are assigned the same timestamp (the timestamp
assigned to the Paxos write that commits the transaction).

Suppose, instead, that each write is assigned a timestamp equal to
TT.now().latest at the time when the client transaction code
calls the Spanner library function to modify the data.

What aspect of Spanner would this break, and why?

Answer: Read-only transactions would no longer provide
serializability. Spanner executes read-only transactions as of a
particular time-stamp. In Spanner as described by the paper, each
read-write transaction's writes all occur at the same time-stamp, and
thus are either all before or all after each read-only transaction. So
time-stamp order determines the equivalent serial order. But if a
read-write transaction's writes have different time-stamps, then a
read-only transaction could have a time-stamp that's in the middle of
the read-write transaction's writes, and thus see some but not all of
the read-write transation's writes. In that case there will not
generally be any equivalent serial order.

*** FaRM

Ben is using FaRM (see the paper "No compromises: distributed
transactions with consistency, availability, and performance" by
Dragojevic et al). Here is Ben's transaction code:

  begin()
  if x > y:
    y = y + 1
  else:
    x = x + 1
  end()

x and y are objects stored in FaRM. begin() and end() mark the start
and end of the transaction.


Ben starts two instances of this transaction at exactly the same time
on two different machines. The two transactions read x and y at exactly
the same time, and send LOCK messages at exactly the same time. There
is no other activity in the system, and no failures occur. x and y
both start out with value zero.

Explain what the outcome will be, and how the FaRM commit protocol
(Figure 4) arrives at that outcome.

Answer: The outcome is that FaRM will tell one transaction that it has
committed, and the other that it has aborted. The resulting values
will be x=1 and y=0. FaRM arrives at this outcome because the server
for x will process the two LOCK messages in one order or the other,
and grant the lock on x to only the first LOCK; the server will
respond with a failure message to the second LOCK.

(The scenario in the previous question would work despite this change:
while both transactions' VALIDATE(x) would succeed, one of the
transactions' LOCK(y) would fail, either due to y already being
locked, or due to y's version number having changed.)


---

Ben thinks that some transactions might execute faster if he modified
FaRM to run Figure 4's VALIDATE phase before the LOCK phase. It turns
out that this change can cause incorrect results in some situations.
Outline such a situation. Your answer can involve transactions other
than the one shown above.

Answer: Consider these two transactions:

T1:
  if x == 0:
    y = 1

T2:
  if y == 0:
    x = 1

x and y start as zero. If T1 and T2 run at exactly the same time, and
send VALIDATE first, then both VALIDATEs will succeed. Both LOCKs will
succeed as well, since T1 and T2 lock different objects. So both
transactions will commit, leaving x=1 and y=1. But that's not the
result obtained by either serial order: T1 then T2 yields x=0 y=1, and
T2 then T1 yields x=1 y=0. So Ben's change will break serializability
in some situations.


*** Spark

Ben is using Spark to compute PageRank on a huge database of web
links. He's using PageRank code much like that shown in Section 3.2.2
of "Resilient Distributed Datasets: A Fault-Tolerant Abstraction for
In-Memory Cluster Computing," by Zaharia et al.

Ben's PageRank job takes hours to run, and he wants to find out which
parts of the job take the most time. He inserts code that gets the
wall-clock time before and after each line of the PageRank shown in
Section 3.2.2, and prints the difference (i.e. prints how long that
line took to execute). He sees that each line in the "for" loop in
Section 3.2.2 takes only a fraction of a second to execute, and that
the whole loop itself takes less than a second, even though the entire
Spark job takes hours. Explain why the for loop takes so much less
time than the whole job.

Answer: The for loop contains only transformations, not actions. Spark
doesn't start the computation until a call to an action. Calls to
transformations merely build up a description of a lineage graph,
which takes very little time.

---

Ben thinks that telling Spark to cache the "ranks" RDD may make his
PageRank job complete in less time, since ranks is used in each
iteration of the "for" loop. He adds a call to "ranks.persist()" after
each of the two assignments to ranks. However, he finds that his
PageRank job takes the same amount of time to complete despite this
change. Explain.

Answer: Each iteration of the for loop generates a distinct new ranks
RDD. Thus each ranks RDD is only used once. Thus there's no advantage
in caching it.

*** Memcache / Facebook

Ben runs a small web site. He has a bunch of web servers, a single
memcached server, and a single MySQL DB. The web servers store data
persistently in the MySQL DB, and cache database records in memcached.
Thus Ben's web servers act as clients to the memcached and MySQL
servers.

Though Ben has read "Scaling Memcache at Facebook," by Nishtala et al,
he ignores all the ideas in that paper. For example, Ben's setup has
nothing like Figure 6's invalidates or McSqueal, and no leases.

Ben has programmed his web servers to use memcached and MySQL as
follows. When a web server needs a data item, it first tries to read
it from memcached; if that read misses, the web server reads the data
item from MySQL. However, his web servers do not insert data read from
MySQL into memcached. Instead, when one of Ben's web servers writes
data, it always inserts (or updates) the data in both memcached and
MySQL.

Here's pseudo-code describing how Ben's web server's use memcached and
MySQL:

  read(k):
    v = get(k) // is the key/value in memcached?
    if v is nil {
      // miss -- not in memcached
      v = fetch from DB
    }
    return v

  write(k,v):
    send k,v to DB
    set(k,v) // insert the new key/value into memcached

Ben reasons that, since every write updates both MySQL and memcached,
the two will always agree except during the small window of time
between the two lines of code in write(). Explain why Ben is wrong.

Answer: Suppose clients C1 and C2 both write different values to key K
at about the same time. If C1's write arrives at the DB before C2's
write, but C1's write arrives at memcached *after* C2's write, then
memcached will disagree with the DB. This disagreement will last until
the next time a client writes K, which may be a long time.

*** COPS

You're using COPS (see "Don't Settle for Eventual: Scalable Causal
Consistency for Wide-Area Storage with COPS," by Lloyd et al.). You
have four data centers, and clients and a COPS Data Store Cluster at
each data center.

Four clients, each at a different data center, each start executing
code that uses COPS at about the same time. Here's the code each
client executes:

Client C1:
  put(x, 1)
  put(y, 2)

Client C2:
  tmp_y = get(y)
  put(z, 100 + tmp_y)

Client C3:
  tmp_x = get(x)
  tmp_y = get(y)
  tmp_z = get(z)
  print "x=", tmp_x
  print "y=", tmp_y
  print "z=", tmp_z

Client C4:
  tmp_z = get(z)
  tmp_y = get(y)
  tmp_x = get(x)
  print "x=", tmp_x
  print "y=", tmp_y
  print "z=", tmp_z

Note that C3 and C4 read x, y, and z in opposite orders.

All values start as zero. The clients start executing these code
sequences at about the same time. The system uses COPS, not COPS-GT
(that is, there are no transactions). There is no other activity in
the system, just these four clients. There are no failures.

Which of the following are possible outputs from C3? Mark all that apply.

[ ] x=1 y=2 z=102
[ ] x=1 y=0 z=102
[ ] x=0 y=2 z=102
[ ] x=1 y=2 z=0
[ ] x=1 y=0 z=0
[ ] x=0 y=2 z=0

Answer: All of the above are possible.

---

Which are possible outputs from C4? Mark all that apply.

[ ] x=1 y=2 z=102
[ ] x=1 y=0 z=102
[ ] x=0 y=2 z=102
[ ] x=1 y=2 z=0
[ ] x=1 y=0 z=0
[ ] x=0 y=2 z=0

Answer: 
x=1 y=2 z=102
x=1 y=2 z=0
x=1 y=0 z=0

*** Lab 3

Explain what would go wrong if clients sent Lab 3 Get() RPCs to
followers, and followers responded with the values in their replica of
the data (without talking to the leader, and without adding anything
to the log).

Answer:

Clients could observe stale reads if they contacted a follower that hadn't seen
the latest Put()/Append().

----

Alyssa wonders whether one could use Spanner's TrueTime (the API in
Table 1 of the Spanner paper) to help design a correct version of Lab
3 in which followers can respond to client Get() RPCs. Outline a
design for this. You can assume that both clients and servers have
TrueTime, and that the key/value service is continuously busy with
both Put() and Get() operations.

Answer:

Have clients supply their local timestamp `req_ts = TT.now()` along with their
Get() requests.

On the KV server side, for all operations that go through the log (i.e. Put()
and Append()), have the leader include a local timestamp `op_ts = TT.now()` in
the Command put into the Raft log.

To serve a Get() (on a leader or follower) with a timestamp `req_ts`, wait
until the local state machine has processed an entry from the Raft log with
timestamp `op_ts` such that `op_ts.earliest > req_ts.latest`, and then serve
the Get() by reading from the local key/value map.

*** 6.824

Which papers should we omit in future 6.824 years, because they are
not useful or are too hard to understand?

[ ] 6.033 two-phase commit   XXXXXX
[ ] Spanner 
[ ] FaRM                     XXXXXX
[ ] Spark                    XX
[ ] Memcached at Facebook    XX
[ ] COPS                     XXXXXXXXXXX
[ ] Certificate Transparency XXXXXXXXXXXXX
[ ] Bitcoin                  X
[ ] BlockStack               XXXXXXXXXXXXXXXXXXX
[ ] AnalogicFS               XXXXXXXXXXXXXX

---

Which papers did you find most useful?

[ ] 6.033 two-phase commit   XXXXXXXXXXXXXXXXXX
[ ] Spanner                  XXXXXXXXXXXXXXXXXXXXXXXXXXXXX
[ ] FaRM                     XXXXXXXXXXXXXXXXXXX
[ ] Spark                    XXXXXXXXXXXXXXXXXXXXXXXX
[ ] Memcached at Facebook    XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
[ ] COPS                     XXXXXXXXXXXX
[ ] Certificate Transparency XXXXXXXXXXXX
[ ] Bitcoin                  XXXXXXXXXXXXXXXXX
[ ] BlockStack               XXXX
[ ] AnalogicFS               XXXX