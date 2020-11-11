# Fault Tolerance: Raft (2)

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
