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
