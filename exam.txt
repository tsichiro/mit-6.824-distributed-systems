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