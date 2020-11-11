# The Google File System

[The Google File System](the-google-file-system.pdf)

Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung
SOSP 2003

<!-- TOC -->

- [The Google File System](#the-google-file-system)
  - [Why are we reading this paper?](#why-are-we-reading-this-paper)
  - [What is consistency?](#what-is-consistency)
  - ["Ideal" consistency model](#ideal-consistency-model)
  - [Challenges to achieving ideal consistency](#challenges-to-achieving-ideal-consistency)
  - [GFS goals:](#gfs-goals)
  - [High-level design / Reads](#high-level-design--reads)
    - [Writes](#writes)
    - [Record append](#record-append)
  - [Housekeeping](#housekeeping)
  - [Failures](#failures)
  - [Does GFS achieve "ideal" consistency?](#does-gfs-achieve-ideal-consistency)
    - [Authors claims weak consistency is not a big problems for apps](#authors-claims-weak-consistency-is-not-a-big-problems-for-apps)
  - [Performance (Figure 3)](#performance-figure-3)
  - [Summary](#summary)
  - [References](#references)
  - [GFS FAQ](#gfs-faq)

<!-- /TOC -->
## Why are we reading this paper?

- the file system used for map/reduce
- main themes of 6.824 show up in this paper
  - trading consistency for simplicity and performance
  - motivation for subsequent designs
- good systems paper -- details from apps all the way to network
  - performance, fault-tolerance, consistency
- influential
  - many other systems use GFS (e.g., Bigtable, Spanner @ Google)
  - HDFS (Hadoop Distributed File System) based on GFS

## What is consistency?

- A correctness condition
- Important but difficult to achieve when data is replicated
  - especially when application access it concurrently
  - [diagram: simple example, single machine]
  - if an application writes, what will a later read observe?
    - what if the read is from a different application?
  - but with replication, each write must also happen on other machines
  - [diagram: two more machines, reads and writes go across]
  - Clearly we have a problem here.
- Weak consistency
  - read() may return stale data --- not the result of the most recent write
- Strong consistency
  - read() always returns the data from the most recent write()
- General tension between these:
  - strong consistency is easy for application writers
  - strong consistency is bad for performance
  - weak consistency has good performance and is easy to scale to many servers
  - weak consistency is complex to reason about
- Many trade-offs give rise to different correctness conditions
  - These are called "consistency models"
  - First peek today; will show up in almost every paper we read this term

## "Ideal" consistency model

- Let's go back to the single-machine case
- Would be nice if a replicated FS **behaved like a non-replicated file system**
  - [diagram: many clients on the same machine accessing files on single disk]
- If one application writes, later reads will observe that write
- What if two application concurrently write to the same file?
  - Q: what happens on a single machine?
  - In file systems often undefined --- file may have some mixed content
- What if two application concurrently write to the same directory
  - Q: what happens on a single machine?
  - One goes first, the other goes second (use locking)

## Challenges to achieving ideal consistency

- Concurrency -- as we just saw; plus there are many disks in reality
- Machine failures -- any operation can fail to complete
- Network partitions -- may not be able to reach every machine/disk
- Why are these challenges difficult to overcome?
  - Requires communication between clients and servers
    - May cost performance
  - Protocols can become complex --- see next week
    - Difficult to implement system correctly
  - Many systems in 6.824 don't provide ideal
    - GFS is one example

## GFS goals:

- With so many machines, failures are common
  - must tolerate
  - assume a machine fails once per year
  - w/ 1000 machines, ~3 will fail per day.
- High-performance: many concurrent readers and writers
  - Map/Reduce jobs read and store final result in GFS
  - Note: *not* the temporary, intermediate files
- Use network efficiently: save bandwidth
- These challenges difficult combine with "ideal" consistency

## High-level design / Reads

- [Figure 1 diagram, master + chunkservers]
- Master stores directories, files, names, open/read/write
  - But not POSIX
- 100s of Linux chunk servers with disks
  - store 64MB chunks (an ordinary Linux file for each chunk)
  - each chunk replicated on three servers
  - Q: Besides availability of data, what does 3x replication give us?
    - load balancing for reads to hot files affinity
  - Q: why not just store one copy of each file on a RAID'd disk?
    - RAID isn't commodity
    - Want fault-tolerance for whole machine; not just storage device
  - Q: why are the chunks so big?
    - amortizes overheads, reduces state size in the master
- GFS master server knows directory hierarchy
  - for directory, what files are in it
  - for file, knows chunk servers for each 64 MB
  - master keeps state in memory
    - 64 bytes of metadata per each chunk
  - master has private recoverable database for metadata
    - operation log flushed to disk
    - occasional asynchronous compression info checkpoint
    - N.B.: != the application checkpointing in Â§2.7.2
    - master can recovery quickly from power failure
  - shadow masters that lag a little behind master
    - can be promoted to master
- Client read:
  - send file name and chunk index to master
  - master replies with set of servers that have that chunk
    - response includes version # of chunk
    - clients cache that information
  - ask nearest chunk server
    - checks version #
    - if version # is wrong, re-contact master

### Writes

- [Figure 2-style diagram with file offset sequence]
- Random client write to existing file
  - client asks master for chunk locations + primary
  - master responds with chunk servers, version #, and who is primary
    - primary has (or gets) 60s lease
  - client computes chain of replicas based on network topology
  - client sends data to first replica, which forwards to others
    - pipelines network use, distributes load
  - replicas ack data receipt
  - client tells primary to write
    - primary assign sequence number and writes
    - then tells other replicas to write
    - once all done, ack to client
  - what if there's another concurrent client writing to the same place?
    - client 2 get sequenced after client 1, overwrites data
    - now client 2 writes again, this time gets sequenced first (C1 may be slow) writes, but then client 1 comes and overwrites
    - => all replicas have same data (= consistent), but mix parts from C1/C2
      - (= NOT defined)
- Client append (not record append)
  - same deal, but may put parts from C1 and C2 in any order
  - consistent, but not defined
  - or, if just one client writes, no problem -- both consistent and defined

### Record append

- Client record append
  - client asks master for chunk locations
  - client pushes data to replicas, but specifies no offset
  - client contacts primary when data is on all chunk servers
    - primary assigns sequence number
    - primary checks if append fits into chunk
      - if not, pad until chunk boundary
    - primary picks offset for append
    - primary applies change locally
    - primary forwards request to replicas
    - let's saw R3 fails mid-way through applying the write
    - primary detects error, tells client to try again
  - client retries after contacting master
    - master has perhaps brought up R4 in the meantime (or R3 came back)
    - one replica now has a gap in the byte sequence, so can't just append
    - pad to next available offset across all replicas
    - primary and secondaries apply writes
    - primary responds to client after receiving acks from all replicas

## Housekeeping

- Master can appoint new primary if master doesn't refresh lease
- Master replicates chunks if number replicas drop below some number
- Master rebalances replicas

## Failures

- Chunk servers are easy to replace
  - failure may cause some clients to retry (& duplicate records)
- Master: down -> GFS is unavailable
  - shadow master can serve read-only operations, which may return stale data
  - Q: Why not write operations?
  - split-brain syndrome (see next lecture)

## Does GFS achieve "ideal" consistency?

- Two cases: directories and files
- Directories: yes, but...
  - Yes: strong consistency (only one copy)
  - But: master not always available & scalability limit
- Files: not always
  - Mutations with atomic appends
  - record can be duplicated at two offsets
  - while other replicas may have a hole at one offset
  - Mutations without atomic append
    - data of several clients maybe intermingled
    - if you care, use atomic append or a temporary file and atomically rename
- An "unlucky" client can read stale data for short period of time
  - A failed mutation leaves chunks inconsistent
    - The primary chunk server updated chunk
    - But then failed and the replicas are out of date
  - A client may read an not-up-to-date chunk
  - When client refreshes lease it will learn about new version #

### Authors claims weak consistency is not a big problems for apps

- Most file updates are append-only updates
  - Application can use UID in append records to detect duplicates
  - Application may just read less data (but not stale data)
- Application can use temporary files and atomic rename

## Performance (Figure 3)

- huge aggregate throughput for read (3 copies, striping)
  - 125 MB/sec in aggregate
  - Close to saturating network
- writes to different files lower than possible maximum
  - authors blame their network stack
  - it causes delays in propagating chunks from one replica to next
- concurrent appends to single file
  - limited by the server that stores last chunk
- numbers and specifics have changed a lot in 15 years!

## Summary

- case study of performance, fault-tolerance, consistency
  - specialized for MapReduce applications
- what works well in GFS?
  - huge sequential reads and writes
  - appends
  - huge throughput (3 copies, striping)
  - fault tolerance of data (3 copies)
- what less well in GFS?
  - fault-tolerance of master
  - small files (master a bottleneck)
  - clients may see stale data
  - appends maybe duplicated

## References

- [(discussion of gfs evolution)](http://queue.acm.org/detail.cfm?id=1594206)
- [Google's Colossus Makes Search Real-Time By Dumping MapReduce](http://highscalability.com/blog/2010/9/11/googles-colossus-makes-search-real-time-by-dumping-mapreduce.html)

## GFS FAQ

Q: Why is atomic record append at-least-once, rather than exactly once?

It is difficult to make the append exactly once, because a primary would then need to keep state to perform duplicate detection. That state must be replicated across servers so that if the primary fails, this information isn't lost. You will implement exactly once in lab 3, but with more complicated protocols that GFS uses.

Q: How does an application know what sections of a chunk consist of padding and duplicate records?

A: To detect padding, applications can put a predictable magic number at the start of a valid record, or include a checksum that will likely only be valid if the record is valid. The application can detect duplicates by including unique IDs in records. Then, if it reads a record that has the same ID as an earlier record, it knows that they are duplicates of each other. GFS provides a library for applications that handles these cases.

Q: How can clients find their data given that atomic record append writes it at an unpredictable offset in the file?

A: Append (and GFS in general) is mostly intended for applications that read entire files. Such applications will look for every record (see the previous question), so they don't need to know the record locations in advance. For example, the file might contain the set of link URLs encountered by a set of concurrent web crawlers. The file offset of any given URL doesn't matter much; readers just want to be able to read the entire set of URLs.

Q: The paper mentions reference counts -- what are they?

A: They are part of the implementation of copy-on-write for snapshots.
When GFS creates a snapshot, it doesn't copy the chunks, but instead increases the reference counter of each chunk. This makes creating a snapshot inexpensive. If a client writes a chunk and the master notices the reference count is greater than one, the master first
makes a copy so that the client can update the copy (instead of the chunk that is part of the snapshot). You can view this as delaying the copy until it is absolutely necessary. The hope is that not all chunks will be modified and one can avoid making some copies.

Q: If an application uses the standard POSIX file APIs, would it need to be modified in order to use GFS?

A: Yes, but GFS isn't intended for existing applications. It is designed for newly-written applications, such as MapReduce programs.

Q: How does GFS determine the location of the nearest replica?

A: The paper hints that GFS does this based on the IP addresses of the servers storing the available replicas. In 2003, Google must have assigned IP addresses in such a way that if two IP addresses are close to each other in IP address space, then they are also close together in the machine room.

Q: Does Google still use GFS?

A: GFS is still in use by Google and is the backend of other storage systems such as BigTable. GFS's design has doubtless been adjusted over the years since workloads have become larger and technology has changed, but I don't know the details. HDFS is a public-domain clone of GFS's design, which is used by many companies.

Q: Won't the master be a performance bottleneck?

A: It certainly has that potential, and the GFS designers took trouble to avoid this problem. For example, the master keeps its state in memory so that it can respond quickly. The evaluation indicates that for large file/reads (the workload GFS is targeting), the master is not a bottleneck. For small file operations or directory operations, the master can keep up (see 6.2.4).

Q: How acceptable is it that GFS trades correctness for performance and simplicity?

A: This a recurring theme in distributed systems. Strong consistency usually requires protocols that are complex and require chit-chat between machines (as we will see in the next few lectures). By exploiting ways that specific application classes can tolerate relaxed
consistency, one can design systems that have good performance and sufficient consistency. For example, GFS optimizes for MapReduce applications, which need high read performance for large files and are OK with having holes in files, records showing up several times, and
inconsistent reads. On the other hand, GFS would not be good for storing account balances at a bank.

Q: What if the master fails?

A: There are replica masters with a full copy of the master state; an unspecified mechanism switches to one of the replicas if the current master fails (Section 5.1.3). It's possible a human has to intervene to designate the new master. At any rate, there is almost certainly a
single point of failure lurking here that would in theory prevent automatic recovery from master failure. We will see in later lectures how you could make a fault-tolerant master using Raft.
