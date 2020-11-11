6.824 2020 Lecture 2: Infrastructure: RPC and threads

Today:
  Threads and RPC in Go, with an eye towards the labs

Why Go?
  good support for threads
  convenient RPC
  type safe
  garbage-collected (no use after freeing problems)
  threads + GC is particularly attractive!
  relatively simple
  After the tutorial, use https://golang.org/doc/effective_go.html

Threads
  a useful structuring tool, but can be tricky
  Go calls them goroutines; everyone else calls them threads

Thread = "thread of execution"
  threads allow one program to do many things at once
  each thread executes serially, just like an ordinary non-threaded program
  the threads share memory
  each thread includes some per-thread state:
    program counter, registers, stack

Why threads?
  They express concurrency, which you need in distributed systems
  I/O concurrency
    Client sends requests to many servers in parallel and waits for replies.
    Server processes multiple client requests; each request may block.
    While waiting for the disk to read data for client X,
      process a request from client Y.
  Multicore performance
    Execute code in parallel on several cores.
  Convenience
    In background, once per second, check whether each worker is still alive.

Is there an alternative to threads?
  Yes: write code that explicitly interleaves activities, in a single thread.
    Usually called "event-driven."
  Keep a table of state about each activity, e.g. each client request.
  One "event" loop that:
    checks for new input for each activity (e.g. arrival of reply from server),
    does the next step for each activity,
    updates state.
  Event-driven gets you I/O concurrency,
    and eliminates thread costs (which can be substantial),
    but doesn't get multi-core speedup,
    and is painful to program.
    
Threading challenges:
  shared data 
    e.g. what if two threads do n = n + 1 at the same time?
      or one thread reads while another increments?
    this is a "race" -- and is usually a bug
    -> use locks (Go's sync.Mutex)
    -> or avoid sharing mutable data
  coordination between threads
    e.g. one thread is producing data, another thread is consuming it
      how can the consumer wait (and release the CPU)?
      how can the producer wake up the consumer?
    -> use Go channels or sync.Cond or WaitGroup
  deadlock
    cycles via locks and/or communication (e.g. RPC or Go channels)

Let's look at the tutorial's web crawler as a threading example.

What is a web crawler?
  goal is to fetch all web pages, e.g. to feed to an indexer
  web pages and links form a graph
  multiple links to some pages
  graph has cycles

Crawler challenges
  Exploit I/O concurrency
    Network latency is more limiting than network capacity
    Fetch many URLs at the same time
      To increase URLs fetched per second
    => Need threads for concurrency
  Fetch each URL only *once*
    avoid wasting network bandwidth
    be nice to remote servers
    => Need to remember which URLs visited 
  Know when finished

We'll look at two styles of solution [crawler.go on schedule page]

Serial crawler:
  performs depth-first exploration via recursive Serial calls
  the "fetched" map avoids repeats, breaks cycles
    a single map, passed by reference, caller sees callee's updates
  but: fetches only one page at a time
    can we just put a "go" in front of the Serial() call?
    let's try it... what happened?

ConcurrentMutex crawler:
  Creates a thread for each page fetch
    Many concurrent fetches, higher fetch rate
  the "go func" creates a goroutine and starts it running
    func... is an "anonymous function"
  The threads share the "fetched" map
    So only one thread will fetch any given page
  Why the Mutex (Lock() and Unlock())?
    One reason:
      Two different web pages contain links to the same URL
      Two threads simultaneouly fetch those two pages
      T1 reads fetched[url], T2 reads fetched[url]
      Both see that url hasn't been fetched (already == false)
      Both fetch, which is wrong
      The lock causes the check and update to be atomic
        So only one thread sees already==false
    Another reason:
      Internally, map is a complex data structure (tree? expandable hash?)
      Concurrent update/update may wreck internal invariants
      Concurrent update/read may crash the read
    What if I comment out Lock() / Unlock()?
      go run crawler.go
        Why does it work?
      go run -race crawler.go
        Detects races even when output is correct!
  How does the ConcurrentMutex crawler decide it is done?
    sync.WaitGroup
    Wait() waits for all Add()s to be balanced by Done()s
      i.e. waits for all child threads to finish
    [diagram: tree of goroutines, overlaid on cyclic URL graph]
    there's a WaitGroup per node in the tree
  How many concurrent threads might this crawler create?

ConcurrentChannel crawler
  a Go channel:
    a channel is an object
      ch := make(chan int)
    a channel lets one thread send an object to another thread
    ch <- x
      the sender waits until some goroutine receives
    y := <- ch
      for y := range ch
      a receiver waits until some goroutine sends
    channels both communicate and synchronize
    several threads can send and receive on a channel
    channels are cheap
    remember: sender blocks until the receiver receives!
      "synchronous"
      watch out for deadlock
  ConcurrentChannel master()
    master() creates a worker goroutine to fetch each page
    worker() sends slice of page's URLs on a channel
      multiple workers send on the single channel
    master() reads URL slices from the channel
  At what line does the master wait?
    Does the master use CPU time while it waits?
  No need to lock the fetched map, because it isn't shared!
  How does the master know it is done?
    Keeps count of workers in n.
    Each worker sends exactly one item on channel.

Why is it not a race that multiple threads use the same channel?

Is there a race when worker thread writes into a slice of URLs,
  and master thread reads that slice, without locking?
  * worker only writes slice *before* sending
  * master only reads slice *after* receiving
  So they can't use the slice at the same time.

When to use sharing and locks, versus channels?
  Most problems can be solved in either style
  What makes the most sense depends on how the programmer thinks
    state -- sharing and locks
    communication -- channels
  For the 6.824 labs, I recommend sharing+locks for state,
    and sync.Cond or channels or time.Sleep() for waiting/notification.

Remote Procedure Call (RPC)
  a key piece of distributed system machinery; all the labs use RPC
  goal: easy-to-program client/server communication
  hide details of network protocols
  convert data (strings, arrays, maps, &c) to "wire format"

RPC message diagram:
  Client             Server
    request--->
       <---response

Software structure
  client app        handler fns
   stub fns         dispatcher
   RPC lib           RPC lib
     net  ------------ net

Go example: kv.go on schedule page
  A toy key/value storage server -- Put(key,value), Get(key)->value
  Uses Go's RPC library
  Common:
    Declare Args and Reply struct for each server handler.
  Client:
    connect()'s Dial() creates a TCP connection to the server
    get() and put() are client "stubs"
    Call() asks the RPC library to perform the call
      you specify server function name, arguments, place to put reply
      library marshalls args, sends request, waits, unmarshalls reply
      return value from Call() indicates whether it got a reply
      usually you'll also have a reply.Err indicating service-level failure
  Server:
    Go requires server to declare an object with methods as RPC handlers
    Server then registers that object with the RPC library
    Server accepts TCP connections, gives them to RPC library
    The RPC library
      reads each request
      creates a new goroutine for this request
      unmarshalls request
      looks up the named object (in table create by Register())
      calls the object's named method (dispatch)
      marshalls reply
      writes reply on TCP connection
    The server's Get() and Put() handlers
      Must lock, since RPC library creates a new goroutine for each request
      read args; modify reply
 
A few details:
  Binding: how does client know what server computer to talk to?
    For Go's RPC, server name/port is an argument to Dial
    Big systems have some kind of name or configuration server
  Marshalling: format data into packets
    Go's RPC library can pass strings, arrays, objects, maps, &c
    Go passes pointers by copying the pointed-to data
    Cannot pass channels or functions

RPC problem: what to do about failures?
  e.g. lost packet, broken network, slow server, crashed server

What does a failure look like to the client RPC library?
  Client never sees a response from the server
  Client does *not* know if the server saw the request!
    [diagram of losses at various points]
    Maybe server never saw the request
    Maybe server executed, crashed just before sending reply
    Maybe server executed, but network died just before delivering reply

Simplest failure-handling scheme: "best effort"
  Call() waits for response for a while
  If none arrives, re-send the request
  Do this a few times
  Then give up and return an error

Q: is "best effort" easy for applications to cope with?

A particularly bad situation:
  client executes
    Put("k", 10);
    Put("k", 20);
  both succeed
  what will Get("k") yield?
  [diagram, timeout, re-send, original arrives late]

Q: is best effort ever OK?
   read-only operations
   operations that do nothing if repeated
     e.g. DB checks if record has already been inserted

Better RPC behavior: "at most once"
  idea: server RPC code detects duplicate requests
    returns previous reply instead of re-running handler
  Q: how to detect a duplicate request?
  client includes unique ID (XID) with each request
    uses same XID for re-send
  server:
    if seen[xid]:
      r = old[xid]
    else
      r = handler()
      old[xid] = r
      seen[xid] = true

some at-most-once complexities
  this will come up in lab 3
  what if two clients use the same XID?
    big random number?
    combine unique client ID (ip address?) with sequence #?
  server must eventually discard info about old RPCs
    when is discard safe?
    idea:
      each client has a unique ID (perhaps a big random number)
      per-client RPC sequence numbers
      client includes "seen all replies <= X" with every RPC
      much like TCP sequence #s and acks
    or only allow client one outstanding RPC at a time
      arrival of seq+1 allows server to discard all <= seq
  how to handle dup req while original is still executing?
    server doesn't know reply yet
    idea: "pending" flag per executing RPC; wait or ignore

What if an at-most-once server crashes and re-starts?
  if at-most-once duplicate info in memory, server will forget
    and accept duplicate requests after re-start
  maybe it should write the duplicate info to disk
  maybe replica server should also replicate duplicate info

Go RPC is a simple form of "at-most-once"
  open TCP connection
  write request to TCP connection
  Go RPC never re-sends a request
    So server won't see duplicate requests
  Go RPC code returns an error if it doesn't get a reply
    perhaps after a timeout (from TCP)
    perhaps server didn't see request
    perhaps server processed request but server/net failed before reply came back

What about "exactly once"?
  unbounded retries plus duplicate detection plus fault-tolerant service
  Lab 3

Go FAQ

Q: Why does 6.824 use Go for the labs?

A: Until a few years ago 6.824 used C++, which worked well. Go works a
little better for 6.824 labs for a couple reasons. Go is garbage
collected and type-safe, which eliminates some common classes of bugs.
Go has good support for threads (goroutines), and a nice RPC package,
which are directly useful in 6.824. Threads and garbage collection
work particularly well together, since garbage collection can
eliminate programmer effort to decide when the last thread using an
object has stopped using it. There are other languages with these
features that would probably work fine for 6.824 labs, such as Java.

Q: Do goroutines run in parallel? Can you use them to increase
performance?

A: Go's goroutines are the same as threads in other languages. The Go
runtime executes goroutines on all available cores, in parallel. If
there are fewer cores than runnable goroutines, the runtime will
pre-emptively time-share the cores among goroutines.

Q: How do Go channels work? How does Go make sure they are
synchronized between the many possible goroutines?

A: You can see the source at https://golang.org/src/runtime/chan.go,
though it is not easy to follow.

At a high level, a chan is a struct holding a buffer and a lock.
Sending on a channel involves acquiring the lock, waiting (perhaps
releasing the CPU) until some thread is receiving, and handing off the
message. Receiving involves acquiring the lock and waiting for a
sender. You could implement your own channels with Go sync.Mutex and
sync.Cond.

Q: I'm using a channel to wake up another goroutine, by sending a
dummy bool on the channel. But if that other goroutine is already
running (and thus not receiving on the channel), the sending goroutine
blocks. What should I do?

A: Try condition variables (Go's sync.Cond) rather than channels.
Condition variables work well to alert goroutines that may (or may
not) be waiting for something. Channels, because they are synchronous,
are awkward if you're not sure if there will be a goroutine waiting at
the other end of the channel.

Q: How can I have a goroutine wait for input from any one of a number
of different channels? Trying to receive on any one channel blocks if
there's nothing to read, preventing the goroutine from checking other
channels.

A: Try creating a separate goroutine for each channel, and have each
goroutine block on its channel. That's not always possible, but when
it works it's often the simplest approach.

Otherwise try Go's select.

Q: When should we use sync.WaitGroup instead of channels? and vice versa?

A: WaitGroup is fairly special-purpose; it's only useful when waiting
for a bunch of activities to complete. Channels are more
general-purpose; for example, you can communicate values over
channels. You can wait for multiple goroutines using channels, though it
takes a few more lines of code than with WaitGroup.

Q: I need my code to perform a task once per second. What's the
easiest way to do that?

A: Create a goroutine dedicated to that periodic task. It should have
a loop that uses time.Sleep() to pause for a second, and then do the
task, and then loop around to the time.Sleep().

Q: How do we know when the overhead of spawning goroutines exceeds
the concurrency we gain from them?

A: It depends! If your machine has 16 cores, and you are looking for
CPU parallelism, you should have roughly 16 executable goroutines. If
it takes 0.1 second of real time to fetch a web page, and your network
is capable of transmitting 100 web pages per second, you probably need
about 10 goroutines concurrently fetching in order to use all of the
network capacity. Experimentally, as you increase the number of
goroutines, for a while you'll see increased throughput, and then
you'll stop getting more throughput; at that point you have enough
goroutines from the point of view of performance.

Q: How would one create a Go channel that connects over the Internet?
How would one specify the protocol to use to send messages?

A: A Go channel only works within a single program; channels cannot be
used to talk to other programs or other computers.

Have a look at Go's RPC package, which lets you talk to other Go
programs over the Internet:

  https://golang.org/pkg/net/rpc/

Q: What are some important/useful Go-specific concurrency patterns to know?

A: Here's a slide deck on this topic, from a Go expert:

https://talks.golang.org/2012/concurrency.slide

Q: How are slices implemented?

A: A slice is an object that contains a pointer to an array and a start and
end index into that array. This arrangement allows multiple slices to
share an underlying array, with each slice perhaps exposing a different
range of array elements.

Here's a more extended discussion:

  https://blog.golang.org/go-slices-usage-and-internals

I use slices often, and arrays never. A Go slice is more flexible than
a Go array since an array's size is part of its type, whereas a
function that takes a slice as argument can take a slice of any
length.

Q: What are common debugging tools people use for Go?

Q: fmt.Printf()

As far as I know there's not a great debugger for Go, though gdb can be
made to work:

https://golang.org/doc/gdb

In any case, for most bugs I've found fmt.Printf() to be an extremely
effective debugging tool.

Q: When is it right to use a synchronous RPC call and when is it right to
use an asynchronous RPC call?

A: Most code needs the RPC reply before it can proceed; in that case it
makes sense to use synchronous RPC.

But sometimes a client wants to launch many concurrent RPCs; in that
case async may be better. Or the client wants to do other work while it
waits for the RPC to complete, perhaps because the server is far away
(so speed-of-light time is high) or because the server might not be
reachable so that the RPC suffers a long timeout period.

I have never used async RPC in Go. When I want to send an RPC but not
have to wait for the result, I create a goroutine, and have the
goroutine make a synchronous Call().

Q: Is Go used in industry?

A: You can see an estimate of how much different programming languages
are used here:

https://www.tiobe.com/tiobe-index/