6.824 Lecture 18: Certificate Transparency, Equivocation

why are we looking at Certificate Transparency?
   so far we have assumed closed systems
     where all participants are trustworthy
     e.g. Raft peers
   what if the system is open -- anyone can use it?
   and there's no universally-trusted authority to run the system?
   you have to build useful systems out of mutually suspicious pieces
     this makes trust and security top-level distributed systems issues
   the most basic is "am I talking to the right computer?"
     this is the (almost unsolvable) problem that CT helps address
   this material ties backwards to consistency
     since CT is all about ensuring all parties see the same information
   this material ties forwards to block-chain and Bitcoin
     CT is a non-crypto-currency use of a block-chain-like design
     and it's deployed and used a lot

before certificates (i.e. before 1995)
  man-in-the-middle (MITM) attacks were a concern on the web
  [browser, internet, gmail.com, fake server, stolen password]
  not too hard to redirect traffic:
    DNS isn't very secure, can fake DNS information for gmail.com to browser
    network routers, routing system, WiFi not always very secure

basic certificate and CA scheme
  maps DNS names to public keys
    user is assumed to know the DNS name of who they want to talk to
  [Google, CA, gmail.com server, browser, https connection]
  a certificate contains:
    DNS name e.g. "gmail.com"
    public key of that server
    identity of CA
    signature with CA's private key
  browsers contain list of public keys of all acceptable CAs
  when browser connects with https:
    server sends certificate
    browser checks CA signature (using list of acceptable CA public keys)
    browser challenges server to prove it has private key
  now MITM attacks are harder:
    assume user clicks on https://gmail.com
    attacker's fake gmail.com server must have certificate for gmail.com
    
why certificates haven't turned out to be perfect
  it's not clear how to decide who owns a DNS name
    if I ask CA for a certificate for "x.com", how does CA decide?
    turns out to be hard, even for well-known names like microsoft.com
  worse: there are over 100 CAs in browsers' lists
    not all of them are well-run
    not all of them employ only trustworthy employees
    not all of them can resist demands from country's government
  and: any CA can issue a certificate for any name
    so the overall security is limited by the least secure CA
  result:
    multiple incidents of "bogus" certificates
    e.g. certificate for gmail.com, issued to someone else
    hard to prevent
    hard even to detect that a bogus certificate is out there!

why not an online DB of valid certs?
  DB service would detect and reject bogus certificates
  browsers would check DB before using any cert
  - how to decide who owns a DNS name?
    if you can't decide, you can't reject bogus certificates.
  - must allow people to change CAs, renew, lose private key, &c
    these all look like a 2nd certificate for an existing name!
  - who would run it?
    there's no single authority that everyone in the world trusts.

how does Certificate Transparency (CT) approach this problem?
  it's really an audit system; it doesn't directly prohibit anything
    makes sense since the name ownership rules aren't well defined
  the main effect is to ensure that the existence of all certificates is public
  [gmail.com, CA, CT log server, browser, gmail's monitor]
  the basic action:
    gmail.com asks CA for a certificate
    CA issues cert to gmail.com
    CA registers certificate with CT log server (typically more than one)
    log server adds certificate to log
    browser connects to gmail.com
    gmail.com provides certificate to browser
    browser asks CT log server if cert is in the log
    meanwhile:
      gmail.com's Monitor periodically fetches the entire CT log
      scans the log for all certs that say "gmail.com"
      complains if there are other than the one it knows about
      since those must be bogus
  thus:
     if browsers and monitors see the same log,
     and monitors raise an alarm if there's a bogus cert in the log,
     and browsers require that each cert they use is in the log,
     then browsers can feel safe using any cert that's in the log.

the hard part: how to ensure everyone sees the same log?
  when the log operator may be malicious,
    and conspiring with (other) malicious CAs
  critical: no deletion (even by malicious log operator)
    otherwise: log could show a bogus cert to browser,
      claim it is in the log, then delete after browser uses it,
      so monitors won't notice the bogus cert.
    (really "no undetected deletion")
  critical: no equivocation (i.e. everyone sees same content)
    otherwise: log server could show browser a log with the
      bogus cert, and show the monitor a log without
      the bogus cert.
    (really "no undetected equivocation")
  then
    if a CA issues a bogus certificate,
    it must add it to central log in order for clients to accept it,
    but then it can't be deleted,
    so name owner's Monitor will eventually see it. 

how can we have an immutable/append-only fork-free log w/ untrusted server?

step one: Merkle Tree
  [logged certs, tree over them, root]
  log server maintains a log of certificates
  let's pretend always a power of two log entries, for simplicity
  H(a,b) means cryptographic hash of a+b
    key property: cannot find two different inputs that produce the same hash
  binary tree of hashes over the log entries -- Merkle Tree
  for each hash value, only one possible sequence of log entries
    if you change the log in any way, the root hash will change too
  STH is Signed Tree Head -- signed by log server
    so it can't later deny it issued that root hash
  once a log server reveals an STH to me,
    it has committed to specific log contents.

how log server appends records to a Merkle Tree log
  assume N entries already in the log, with root hash H1
  log server waits for N new entries to arrive
  hashes them to H2
  creates new root H3 = H(H1, H2)

how a log server proves that a certificate is in the log under a given STH
  "proof of inclusion" or "RecordProof()" or "Merkle Audit Proof"
  since browser doesn't want to use the cert if its not in the log
    since then Monitors would not see it
    and there's be no protection against the cert being bogus
  the proof shows that, for an STH, a certificate, and a log position, that 
    the specified certificate has to be at that position.
  the browser asks the log server for the current STH.
    (the log server may lie about the STH; we'll consider that later.)
  consider a log with just two records, a and b.
    STH = H(H(a), H(b))
    initially browser knows STH but not a or b.
    browser asks "is a in the log?"
      server replies "0" and z=H(b) -- this is the proof
      browser checks H(H(a), z) = STH
    browser asks "is x in the log?" -- but we know x is not in the log under STH
      log server wants to lie and say yes.
        it wants to do this to trick the browser into using
        bogus cert x without a Monitor seeing x in the log.
      browser knows STH.
      so log server needs to find a y such that
        H(H(x), H(y)) = STH = H(H(a), H(b))
        for x != a
      but cryptographic hash guarantees this property:
        infeasible to produce any pair of distinct messages M1 鈮� M2 with
        identical hashes H(M1) = H(M2).
  you can extend this to bigger trees by providing the "other"
    hashes all the way up to the root.
  the proofs are smallish -- log(N) hashes for a log with N elements.
    important; we don't want browsers to have to e.g. download the whole log.
    there are millions of certificates out there.

what if the log server cooks up a *different* log for the browser,
    that contains a bogus cert, and sends the STH for that log
    only to the browser (not to any Monitors)?
  i.e. the server lies about what the current STH is.
  the log server can prove to the browser that the bogus cert
    is in that log.
  this is a fork attack -- or equivocation.
  [diagram -- linear log, forked]
  forks are possible in CT.
    but that is not the end of the story.

how to detect forks/equivocation?
  browsers and monitors should compare STHs to make sure they
    are all seeing the same log.
  called "gossip".
  how can we tell if a given pair of STHs are evidence of a fork?
  they can differ legitimately, if one is from a later version
    of the log than the other!

merkle consistency proof or TreeProof()
  given two STHs H1 and H2, is the log under H1 a prefix of the log under H2?
  clients ask the log server to produce a proof
  if proof works out, the log server hasn't forked them
  if no proof, log server may have forked them -- shown them different logs
  the proof:
    for each root hash Hz = H(Hx,Hy) as the tree grew from H1 to H2,
    the right-hand hash -- Hy
  clients can compute H(H(H1,Hy1),Hy2) ... and check if equal to H2

why does the log consistency proof mean the clients are seeing the same log?
  what if H2 is derived from different log entries in the region that H1 covers?
    i.e. the log server has forked the two clients
  suppose H2 is the very next larger power-of-two-size log from H1
  [draw tree]
  the clients, who both know H1 and H2, are expecting this proof:
    x such that H2 = H(H1, x)
  because the log server forked the clients, we know
    H2 = H(Hz, y) where Hz != H1
  so the cheating log server would have to find x such that
    H(H1, x) = H2 = H(Hz, y) where H1 != Hz
  but cryptographic hashes promise this isn't likely:
    not practical to find two different inputs that produce the same output
  so a consistency proof is convincing evidence that H1 is a prefix
    of H2, and thus that the log server hasn't forked the two clients

so, if browsers and Monitors do enough gossiping and require consistency proofs,
  they can be pretty sure that they are all seeing the same log,
  and thus that Monitors are checking the same set of certificates that
  web browsers see.

one last proof issue
  what if  browser encounters bogus cert C1,
    gets a valid proof of inclusion from the log server,
    but it's in a forked log intended to trick the browser
  [diagram of fork]
  browser will go ahead and use the bogus C1
  but next time the browser talks to log server,
    log server provides it with the main fork and its STH again
  now when browser compares STHs with Monitors, it will
    look like nothing bad happened

browsers use log consistency proofs to prevent switching forks
  browser remembers (persistently) last STH it got from log server
  each time browser talks to log server, it gets a new STH
  browser demands proof that old STH represents a log prefix of new STH
  thus, once log server reveals a certificate to browser,
    future STHs it shows browsers must also reflect that certificate
  [diagram]
  so, if a log server forks a browser, it can't then switch forks
    this is called "fork consistency"
    once forked, always forked
  so, next time browser gossips with a Monitor, the fork will be revealed
  so, log servers are motivated to not fork

what should happen if someone notices a problem?
  failure to provide log consistency proofs:
    i.e. evidence of a fork attack
    this suggests the log service is corrupt or badly managed
    after investigation, the log server would probably be taken off
      browers' approved lists
  failure to provide proof of inclusion despite a signed SCT
    this may be evidence of an attack
      i.e. showing a victim browser a certificate,
      but omitting it from logs where Monitors would see it.
    or perhaps the log server is slow about adding certificates to its log
  bogus certificate in a CT log
    e.g. a certificate for mit.edu, but MIT didn't ask for it
    human must talk to responsible CA and find out the story
    often simply a mistake -> add cert to revocation list
    if happens a lot, or is clearly malicious
      browser vendors may delete CA from their approved list

how can CT be attacked?
  window between when browser uses a cert and monitors check log
  no-one monitoring a given name
  not always clear what to do if there's a problem
    who owns a given name?
    maybe the bogus cert was an accident?
  lack of gossip in current CT design/implementations
  privacy/tracking: browsers asking for proofs from log server

what to take away
   the key property: everyone sees the same log, despite malice
     both browsers and DNS name owners
     if a browser sees a bogus cert, so will the DNS name owner
     a consistency property!
   auditing is worth considering when prevention isn't possible
   equivocation
   fork consistency
   gossip to detect forks
  we will see much of this again in the next lecture, about Bitcoin

6.824 Certificate Transparency FAQ


Q: What's the difference between Monitors and Auditors?

A: An "Auditor" runs in each web browser. When the browser visits an
https web site, and receives a certificate from the https server, the
browser asks the relevant log server(s) for a proof that the
certificate is present in the log. The browser also stores
(persistently) the last STH (signed tree head) it has seen, and asks
the log server for a consistency proof that the server's newest STH
describes a log that's an extension of the last log the browser has an
STH for.

A "Monitor" checks that the names the monitor knows about have only
correct certificates in the log. For example, MIT could run a monitor
that periodically looks in the CT logs for all certificates for
*.mit.edu, and checks against a list of valid certificates that the
monitor knows about. Since MIT is running the monitor, it's reasonable
for the monitor to have a list of all MIT certificates (or at least know
which CA should have issued them).

Another possibility is for each CA to run a monitor, that periodically
checks the logs to make sure that DNS names for which the CA has issued
certificates do not suddenly appear to have certificates from some other
CA.

A Monitor fetches the entire log, and knows (for some DNS names) how to
decide if a certificate is legitimate. A monitor also checks that the
STH from the log server matches the log content provided by the log
server.

An Auditor (browser) only checks that the certificates it uses are in
the log, and that each new version of the log contains the previous
version it knew about as a prefix. So an Auditor doesn't have the
complete log contents, only some STHs, some proofs of inclusion, and
some proofs of log consistency.

If Auditors and Monitors see the same log contents, then the fact that
Monitors are checking for bogus certificates affords Auditors (browsers)
protection against bogus certificates. If a bogus web server gives a
bogus certificate (e.g. claiming falsely to be mit.edu) to a browser,
and the browser only uses the certificate if it's in the CT log, and
MIT's monitor sees the same log content as the browser, then MIT's
monitor will detect the bogus certificate.

Much of the detailed design (the Merkle trees and proofs) are about
ensuring that Auditors and Monitors do indeed see the same log contents,
and that log servers cannot change log contents between when Auditors
look and when Monitors look.


Q: Why is it important for CT to ensure that the log is immutable and
append-only -- that certificates cannot be dropped from the log?

A: Suppose a malicious CA creates a bogus certificate for gmail.com. It
creates a web site that looks like gmail.com and tricks your browser
into going there. It conspires with a corrupt CT log operator to
convince your browser that the bogus gmail.com certificate is in the
log. Now your browser will accept the bad web site, which may capture
your password, log into the real gmail as you, fetch and record your
mail, and show you your inbox as if it were real.

Afterwards, the malicious CA and CT log operator need to cover their
tracks. If they can cause the bogus certificate for gmail.com to
disappear from the CT log without anyone noticing, then they can
probably get away with their crime.

However, since the CT system makes it perhaps impossible to delete a
certificate from a log once anyone has seen it, this attack is likely to
be detected.


Q: How can someone impersonate a website with a fake SSL certificate?

A: Suppose an attacker wants to steal Ben Bitdiddle's gmail password.

Let's assume that the attacker has a correct-looking certificate for
gmail.com, signed by a legitimate CA. This is the scenario CT is aimed
at. So let's assume CT does not exist.

One piece of the attack is to trick Ben's browser into connecting to the
attacker's web server instead of the real gmail.com's servers. One way
to do this is to generate fake DNS replies when Ben's browser asks the
DNS system to look up gmail.com -- replies that indicate the IP address
of the attacker's servers rather than the real gmail.com servers. DNS is
only modestly security in this respect; there have been successful
attacks along these lines in the past. The attacker could snoop on Ben's
local WiFi network or the MIT campus net, intercept Ben's DNS requests,
and supply incorrect replies.

Once Ben's browser has connected to the attacker's web server, using
https, the attacker's web server will send the browser the bogus
gmail.com certificate. Since it's signed by a legitimate CA, Ben's
browser will accept it. The attacker's web server can produce a page
that looks just like a gmail.com login page, and Ben will type his
password.


Q: What do monitors do when they find a suspicious certificate?

A: I think in practice some human has to look into what happened --
whether the certificate is actually OK, or is evidence of some CA's
employee or an entire CA being either malicious or poorly run, or was
just a one-off accident. Most situations are likely to be benign (e.g.
someone legitimately got a new certificate for their web site). Even
an incorrectly-issued certificate may well be just an accident, and
once detected should be placed on a certificate revocation list so
browsers will no longer accept it. If the problem seems to be deeper
(malice or serious negligence at a CA) you'd then report the problem
to the major browser vendors, who would talk to the CA in question,
and if they didn't get a satisfactory promise to improve, might drop
the CA's public key from the browser list of acceptable CAs.


Q: Is the existing CA strategy for authentication still necessary? It
seems like the log could serve as the authority itself.

A: It's an attractive idea! One thing people are worried about is
that, without CAs to act as a filter, the CT system could be
overwhelmed with garbage certificate submissions. Another is that the
CAs do perform some verification on certificate requests. Another is
that browsers and web servers already use certificates as they exist
today; switching to a different scheme would be a giant pain. I
suspect that just as the CA scheme ultimately had more serious
problems than most people imagined with it was introduced in the
1990s, so would CT if we had to entirely rely on it.


Q: How can a Monitor know which certificates in the CT log are
legitimate, and which are bogus?

A: One way or another a Monitor has to have special knowledge about
the set of DNS names for which it checks for bogus certificates.

Some companies (e.g. Google and Facebook, probably others) run their
own Monitors that look for certificates that mention DNS names that
they own. Such monitors are supplied with a list of the certificates
that the owner considers legitimate, or of the CAs that are authorized
to issue certificates for the owner.

A CA can run a Monitor that checks, for each DNS name the CA has
issued certificates for, that no other CA has issued a certificate. So
that the CA's monitor is protecting the CA's customers.

There are also monitoring services. I think you tell them what
certificates or CAs are legitimate for your DNS names, and the service
periodically checks the CT logs for bogus certificates for your names.

https://www.facebook.com/notes/protect-the-graph/introducing-our-certificate-transparency-monitoring-tool/1811919779048165/

https://blog.cloudflare.com/introducing-certificate-transparency-monitoring/

https://sslmate.com/certspotter/howithelps


Q: How does the browser/Auditor learn the index of the certificate for
which it needs an inclusion proof?

A: The browser (Auditor) doesn't actually supply the index, it
supplies a hash of the certificate it wants to look up. The CT log
server replies with a proof of inclusion (plus the certificate's index
in the log). So the CT log server must have a big side-table that maps
hashes of certificates to their position in the log.

See section 4.5 of this document for details:

https://tools.ietf.org/html/rfc6962


Q: How can I check if a given web site's certificate is in the CT logs?

A: https://transparencyreport.google.com/https/certificates


Q: How can I watch the browser checking CT logs?

A: Here are directions for Chrome:

  http://www.certificate-transparency.org/certificate-transparency-in-chrome


Q: Has CT ever actually caught a bogus certificate?

A: Here's a success story:

https://security.googleblog.com/2015/09/improved-digital-certificate-security.html


Q: Won't checking the CT logs be a performance bottleneck?

A: I do now know how much a problem performance is.

Replicas of a given log can be served by many servers to increase
throughput. A log operator would have a master copy of the log, and
push updates to the replicas. I don't know if CT implementations
actually do this.

Once a browser (Auditor) fetches a log inclusion proof for an https
site's certificate once, the browser can cache the proof forever.

Here's a post that touches on some performance measures for CT:

https://blog.cloudflare.com/a-tour-through-merkle-town-cloudflares-ct-ecosystem-dashboard/


Q: Isn't it possible for a log server to "fork" its log, i.e. show
different logs to different viewers?

A: That is indeed a possible attack, at least temporarily.

The way to think about this is that the CT design aims to improve the
current CA system, while still being practical to deploy. It doesn't aim
for perfection (which no-one really knows how to achieve). In the pre-CT
system, there was basically no way to ever discover bogus certificates.
The ones that were discovered were found accidentally. With CT, you
can't spot them right away, so they can be used, but monitors will
eventually (perhaps within a day) realize either that there's a bogus
certificate in the logs, or that a log operator has forked their log.


Q: How are incorrectly-issued certificates revoked?

A: https://www.ssl.com/article/how-do-browsers-handle-revoked-ssl-tls-certificates/


Q: Are the different CT logs supposed to be identical?

A: Different CT logs are likely to be different. To achieve its goals,
CT only needs a certificate to be in a single well-behaved log. The
certificate (really the "SCT") contains the ID of the log it is in, so
browsers know which log service to talk to. Since browsers insist that
the certificate be in the log, and monitors inspect all logs, it's
likely that a bogus certificate that's in even a single log will be
quickly detected.

Because any given log service may fail or become malicious, there are a
modest number of independent log services (dozens), and CAs typically
insert each new certificate in multiple logs (e.g. five).

Q: Gossipping doesn't seem to be defined or implemented. Is CT still
secure even without gossip?

A: Without gossip, a conspiring CA and CT log could fork a browser and
trick it into using a bogus certificate (which can happen even with
gossip), and perhaps never be noticed (which gossip could prevent).
But, because CT is fork-consistent, the malicious CA/CT would have to
forever maintain and extend the log they show to the victim, to
include all certificates the victim expects. This would have to be a
special and unique log for that victim. If the malicious CA/CT didn't
continue to do this, the victim would notice that many certificates
that work for other people don't work for the victim, and perhaps the
victim would realize something was wrong. This is a somewhat weak
story, but it does seem to be the case that exploiting lack of gossip
would be awkward and risky.