6.824 2020 Lecture 20: Blockstack

why are we looking at Blockstack?
  it touches on three questions that interest me:
    how to build a naming system -- a PKI -- a critical missing piece
    a non-crypto-currency use of a blockchain
    a very different architecture for web sites, might someday be better
  Blockstack is a real system, developed by a company
  it does have some users and some apps written for it
  but view it as an exploration of how things could be different and better
    not as "this is definitely how things should be"

what's a decentralized app?
  apps built in a way that moves ownership of data into users's hands
    and out of centrally-controlled web sites
  there are many recent (and older) explorations of this general vision.
  the success (and properties) of Bitcoin has prompted a lot of recent activity

old: a typical (centralized) web site
  [user browsers, net, site's web servers w/ app code, site's DB]
  users' data hidden behind proprietary app code
  e.g. blog posts, gmail, piazza, reddit comments, photo sharing,
    calendar, medical records, &c
  this arrangement has been very successful
    it's easy to program
  why is this not ideal?
    users have to use this web site's UI if they want to see their data
    web site sets (and changes!) the rules for who gets access
    web site may snoop, sell information to advertisers
    web site's employees may snoop for personal reasons
    disappointing since it's often the user's own data!
  a design view of the problem:
    the big interface division is between users and app+data
    app+data integration is convenient for web site owner
    but HTML as an interface is UI-oriented.
      and is usually not good about giving users control and access to data

new: decentralized apps
  [user apps, general-purpose cloud storage service, naming/PKI]
  this architecture separates app code from user data
    the big interface division is between user+app and data
    so there's a clearer notion of a user's data, owned/controlled by user
    much as you own the data on your laptop, or in your Athena account
  requirements for the storage system
    in the cloud, so can be accessed from any device
    general-purpose, like a file system
    paid for and controlled by user who owns the data
    sharing between users, modulo permissions, for multi-user apps
    sharing between a user's apps, modulo permissions
    similar to existing services like Amazon S3

what's the point?
  easier for users to switch apps, since data not tied to apps (or web sites)
  easier to have apps that look at multiple kinds of data
    calendar/email, or backup, or file browser
  privacy vs snooping (assuming end-to-end encryption)

how might decentralized applications work?
  here's one simple possibility.
  app: a to-do list shared by two users
    [UI x2, check-box list, "add" button]
  both contribute items to be done
  both can mark an item as finished
  a public storage system,
    key/value data owned by each of U1 and U2
  users U1 and U2 run apps on their computers
    maybe as JavaScript in browsers
    the apps read other user's public data, write own user's data
  the app doesn't have any associated server, it just uses the storage system
  each user creates a file with to-do items
    and a file with "done" marks
  each user's UI code periodically scans the other user's to-do files
  the point:
    the service is storage, independent of any application.
    so users can switch apps, write their own, add encryption to
    prevent snooping, delete their to-do lists, back them up,
    integrate with e-mail app, &c

what could go wrong?
  decentralization is painful:
    per-user FS-like storage much less flexible than dedicated SQL DB
    no trusted server to e.g. look at auction bids w/o revealing
    cryptographic privacy/authentication makes everything else harder
    awkward for users as well as programmers
  current web site architecture works very well
    easy to program
    central control over software+data makes changes (and debugging) easy
    good solutions for performance, reliability
    easy to impose application-specific security
    successful revenue model (ads)

now for Blockstack
  
why does Blockstack focus on naming?
  names correspond to human users, e.g. "robertmorris"
  name -> location (in Gaia) of user's data, so multiple users can interact
  name -> public key, for end-to-end data security
    so I can check I've really retrieved your authentic data
    so I can encrypt my data so only you can decrypt it
    since storage system is not trusted
  lack of a good global PKI has been damaging to many otherwise good security ideas
    so Blockstack started with names

Blockstack claims naming is hard, summarized by "Zooko's triangle":
  1. unique (global) i.e. each name has the same meaning to everyone
  2. human-readable
  3. decentralized
  claim: all three would be valuable (debatable...)
  claim: any two is easy; all three is hard

example for each pair of properties?
  unique + human-readable : e-mail addresses
  unique + decentralized : randomly chosen public keys
  human-readable + decentralized : my contact list

why is all three hard?
  can we add the missing property to any of our three schemes?
  no, all seem to be immediate dead ends

summary of how Blockstack gets all three?
  Bitcoin produces an ordered chain of blocks
  Blockstack embeds name-claiming records in Bitcoin blocks
  if my record claiming "rtm" is first in Bitcoin chain, I own it
  unique (== globally the same)?
  human-readable?
  decentralized?

is this kind of name space good for decentralized apps?
  is unique (== global) valuable?
    yes: I may be able to remember names I already know.
    yes: I can give you a name, and you can use it.
    yes: I can look at an ACL and guess what it means.
    no: human-readable names aren't likely to be very meaningful if chosen from global pool
        e.g. robert_morris_1779 -- is that me? or someone else?
        how about "rtm@mit.edu"?
    no: how can I find your Blockname name?
        how can I verify that a Blockstack name is really you?
  other (possibly bad) ideas:
    only public keys, don't bother with human-readable names
      each person keeps separate "contact list" with names they understand
      naturally decentralized
      not "unique" thus no need for Bitcoin
    central entity that reliably verifies human identity

what are all the pieces in Blockstack?
  client, browser, application, blockstack.js
  Blockstack Browser (meant to run on client machine)
  Bitcoin's block-chain
  Blockstack servers
    read Bitcoin chain
    interpret Blockstack naming records to update DB
    serve naming RPCs from clients
    name -> pub key + zone hash
  Atlas servers -- store "zone records"
    a name record in bitcoin maps to a zone record in Atlas
    zone record indicates where my Gaia data is stored
    keyed by content-hash, so items are immutable
    you can view Atlas as just reducing the size of Blockstack's Bitcoin transactions
    Atlas keeps the full DB in every server
  Gaia servers
    separate storage area for each user (i.e. end-users)
    key -> value
    backed by Amazon S3, Dropbox, &c
      Gaia makes them all look the same
    most users use Gaia storage provided by Blockstack
    user's profile contains user's public key, per-app public keys
    user can have lots of other files, containing app data
    apps can sign and/or encrypt data in Gaia
  S3, Dropbox, &c
    back-ends for Gaia

NAME CREATION

how does one register a Blockstack name?
  (https://docs.blockstack.org/core/wire-format.html)
  the user does it (by running Blockstack software)
  user must own some bitcoin
  two bitcoin transactions: preorder, registration
  preorder transaction
    registration fee to "burn" address
    hash(name)
  registration transaction
    name (not hashed)
    owner public key
    hash(zonefile)
  Blockstack info hidden inside the transactions, Bitcoin doesn't look at it
    but Bitcoin signatures/hashes cover this Blockstack info

why *two* transactions?
  front-running

why the registration fee? after all there's no real cost.

what if a client tries to register a name that's already taken?

what if two clients try to register same name at same time?

is it possible for an attacker to change a name->key binding?
  after all, anyone can submit any bitcoin transaction they like

is it possible for Blockstack to change a name->key binding?

STORAGE

how does the client know where to fetch data from?
  starting with owning user's name, and a key
  apps probably use well known keys, e.g. "profile" or "todo-list"
  bitcoin/blockstack, hash(zone), gaia address

how does the client check that it got the right data back from Gaia?

how does the client know data from Gaia is fresh (the latest version)?
  owner signed the data when writing
  where can others get the owner's public key, to check signature?

how does Gaia know whether to let a client write/change/delete?

what about encryption for privacy?
  if only the owner should see the data?
  if one other user should see the data, in addition to the owner?
  if just 6.824 students should see the data?

PRIVATE KEYS

never leaves user's device(s)
  so you don't have to trust anything other than your device and Blockstack's software
  each of your devices has a copy of your master private key

"master" private key only seen by Blockstack Browser
  too sensitive to let apps see or use it
  protected by pass-phrase, then in clear while user is active

Blockstack Browser hands out per-app private keys
  so each app has more or less separate encrypted storage
  makes it hard for one user's different apps to cooperate
    sometimes that's what you want
    sometimes you do want sharing among your own apps

DISCUSSION

here are some questions to chew on.
  about naming
  about decentralized applications
you can view them as criticism.
or as areas for further development.

Q: could blockstack be used as a PKI for e-mail, to map rtm@mit.edu to my public key?
   blockstack names vs e-mail addresses?
   what does a blockstack name mean?

Q: why is PKI hard in general?
   lost pass-phrases and keys
     recovery (mother's maiden name? SMS? e-mail?)
   what does a name mean? connection to "real" identity?
   how to go from intuitive notion of who I want to talk to, to name?
   some progress, e.g. Keybase

Q: for naming and PKI, is there strong value in decentralization?
   can we have a centralized but secure naming system?
     who can we all trust for a global-scale system?
   indeed what value can a central authority realistically deliver?
   would adoption be easier with decentralization?

Q: could blockstack use a scheme like Certificate Transparency instead of Bitcoin?
   CT can't resolve conflicts, only reveal them.
     different CT logs may have different order
       so CT can't say which came first
     it's Bitcoin mining that resolves forks and forces agreement
   the fee aspect of Blockstack seems critical vs spam &c, relies on cryptocurrency
   in general, open block-chains only seem to make sense w/ cryptocurrency

Q: is Blockstack convenient for programmers?
   all code in client, no special servers
     hard to have data that's specific to the app, vs each user
     indices, vote counts, front-page rankings for Reddit or Hacker News
   SQL queries
   cryptographic access control, groups, revocation, &c
   hard to both look at other users' secrets, and keep the secrets
     e.g. for eBay
   maybe only worthwhile if users are enthusiastic...

Q: is decentralized user-owned storage good for user privacy?
   is it better than trusting Facebook/Google/&c web sites to keep data private?
     vs other users, hackers, their own employees?
   can Blockstack storage providers watch what you access?
   what if app, on your computer, snoops on you?
     after all, it's presumably still Facebook or whoever writing the app.
   is cryptographic access control really feasible?
   you still have to trust the provider to preserve your data
     and to serve up the most recent version
     if you trust them that much, why not trust them to keep it secret too?

Q: is decentralized user-owned storage good for user control?
   do users want to switch applications a lot for the same data?
   do users want to use same data in multiple applications?
   does either even work in general, given different app formats?

Q: will users be willing to pay for their own Gaia storage?

CONCLUSION

what do I take away from Blockstack?
  I find the overall decentralization vision attractive.
  the whole thing rests on a PKI -- any progress here would be great
    a general-purpose mapping from all users to their public keys would be very useful
  surprising that we can have decentralized human-readable name allocation
    but unclear whether decentralized human-readable names are a good idea
  separating cloud data from applications sounds like a good idea
    but developers will hate it (e.g. no SQL).
    not clear users will know or care.
    not clear whether users will want to pay for storage.
  end-to-end encryption for privacy sound like a good idea
    private key management is a pain, and fragile
    encryption makes sharing and access control very awkward
  you still have to trust vendor software; not clear it's a
    huge win that it's running on your laptop rather than
    vendor's server.

all that said, it would be fantastic if Blockstack or something
  like it were to be successful.

6.824 Blockstack FAQ

Q: The whitepaper is light on details. How can I find out more about
how Blockstack works?

A: The authors' earlier USENIX paper explains in more detail how to
use Bitcoin as the basis of a decentralized naming system:

https://www.usenix.org/system/files/conference/atc16/atc16_paper-ali.pdf

Blockstack's May 2019 white paper, while also light on technical
details, has a fuller explanation of what they are trying to achieve:

https://blockstack.org/whitepaper.pdf

Blockstack's web site has lots of documentation and tutorial material
on how to use their system to build applications.

Q: Could Blockstack use technology similar to Certificate Transparency
logs rather than Bitcoin? After all, Bitcoin is expensive to operate,
and slow.

A: CT doesn't have a resolution scheme to decide who really owns a
name if two people register it; it only publishes the fact that a
conflict exists. Bitcoin/Blockstack choose one registration as the
winner. CT can't really use the same ordering idea because CT has
multiple logs, but the logs may be different; two records may appear
in opposite orders in two different logs. It is Bitcoin's expensive
mining that allows Bitcoin to cause all the replicas of the Bitcoin
blockchain to reach agreement without much risk of malicious forks; CT
has nothing like mining to force agreement.

Malicious people would likely submit lots of name registrations to
CT-based personal naming system, and consume every useful name.
Blockstack/Bitcoin defend against this by charging fees for names, but
CT has no corresponding defense. (When CT is used for web
certificates, people have to buy a valid web certificate from a CA
before they can register is with CT, but it's not clear how to build
something similar for personal names.)

My guess is that open (public) block-chains with no associated
crypto-currency can't have strong enough fork-resolution to support
Blockstack-like naming, because they can't have fees or mining (or
even proof-of-stake, since there's no money at stake). But that's just
a guess.

Q: Would decentralization actually change user experience significantly?

A: As things stand now it's likely to make the user experience worse.
Blockstack has nice properties in principle with respect to privacy
and user control, but it's not clear that most users care much about
this. In the sense that they probably wouldn't be willing to sacrifice
much in the way of features or convenience. And, at least so far, it's
much harder for developers to develop applications with complex
features in Blockstack than in traditional centralized web sites.

Q: Does the kind of decentralization delivered by Blockstack improve
user privacy?

A: The main way in which Blockstack could improve security is that
data is stored encrypted in the storage system. So that, even though
the storage is probably run by the same outfits that store your data
in the existing web (Amazon AWS, Microsoft Azure, Google Cloud), they
can't read the data. And the encrypted stored data is resistant to
malicious people exploiting bugs to break into the storage service
software.

But there are a bunch of major caveats. 1) You still have to run
application software, which someone else writes: e.g. if you run
Facebook's Blockstack-based application, you have to trust it not to
snoop on you or do anything else bad. 2) The technology for sharing
encrypted data is (so far) fairly awkward; it's a pain to write
Blockstack-based applications that share data among users but encrypt
it for privacy. Managing the encryption keys is the problem. 3) The
track record for users being able to keep private keys safe is pretty
bad; in other private-key systems, e.g. Bitcoin wallets, users
routinely lose their keys, lose the devices containing keys, run
malware that steals their keys, &c.

Q: What if every email address were associated with a public key?

A: It's a good idea, but no scheme for this has ever gotten much
traction. A name for what you're looking for is PKI (Public Key
Infrastructure). Blockstack's naming system is a PKI.

A deep problem is how to find someone else's public key in a secure way
in a large open system like the Internet. You could imagine a global
registry, but even if the registry is entirely trustworthy, there's no
reliable way for them to check ownership of e-mail addressses or
identities. So they cannot really know whether the person asking to
register a public key for "rtm@mit.edu" is really rtm@mit.edu. So you
cannot reliably find a public key for rtm@mit.edu. And of course if the
central registry turns out to have corrupt employees, or is required to
comply with laws, they may intentionally give you bad information. This
has caused people to search for decentralized PKI solutions, like
Blockstack, but these have even weaker properties with respect to
guaranteeing that the person who registered the name rtm@mit.edu really
is rtm@mit.edu.

Even in more limited situations -- e.g. MIT's Kerberos, which is a PKI
-- it's very hard to achieve real security. If I don't already know
someone's e-mail address, it's not clear how I can find that out
reliably. Even if I know your e-mail address, it's not clear even MIT
can be sure that the person who registered your e-mail address is
really you. Suppose someone tells IS&T that they are you, that they
have lost their password or private key, and that they need to re-gain
access to their account. Will that person be able to take over your
account? Quite possibly, and this kind of failure happens all the
time.

There are a bunch of modern messaging systems that encrypt in much the
way you have in mind (Keybase, Signal), but key discovery is still a
weak point for them. Of the ones I'm familiar with, Keybase has the
best story.

Q: What problems does Blockstack create for application developers?

A: My impressions from having used and built systems like Blockstack
is that they are significantly harder to develop for than the
traditional web site arrangement.

The storage interface is restricted to key/value. So there are no
powerful queries such as you get from SQL databases. And queries take a
relatively long time (each must cross the Internet).

Security has to be enforced with cryptography. For data that should
only be read by the data's owner, that's not a problem. But for shared
data, it's pretty awkward to arrange for the required encryption. You
could encrypt the data once for each permitted reader; or encrypt the
data once with a unique key and encrypt that key for each permitted
user. If you want to support big groups (e.g. everyone in EECS),
that's another level of complexity, particularly if the person who
controls the group membership is different from the person who owns
the data. If you need to retract access (e.g. if someone leaves EECS)
that's even more complex.

Many web sites need to use data that the user shouldn't be able to
see. For example, when I use eBay, the software I run needs to be able
to tell me if I'm currently winning the auction, which requires the
software to know what other people have bid. But I can't be allowed to
know other bids directly, or if I'm not winning. But if the eBay
software runs on my computer (as it would with Blockstack), I can
modify it to tell me anything the software knows. The real eBay solves
this by running the software on their own servers, and storing user
bids in their own database, but the point of Blockstack is to reject
that arrangement.

Many web sites have data that's really the property of the web site
itself, not any user. Blockstack has no obvious way to support that.

Many web sites maintain public data that's not owned by any user, and
incorporates many users' data. Indices and "like" counts, for example,
or the article rankings for the front pages of Reddit and Hacker News.
Blockstack has a scheme for indices specifically, but it's
centralized, and it's not clear how to build other shared constructs.

Q: What is the consistency hash about?

A: Blockstack can suffer forks in its own sequence of transactions,
for example if different Blockstack peers run different software
versions that have different rules for accepting transactions. If
Blockstack weren't careful it might never notice such a fork. But
these forks could hide malicious behavior -- if (perhaps by exploiting
software version differences) an attacker cause some Blockstack peers
to see a given transaction, but others to ignore it. Blockstack uses
the Consistency Hashes to ensure that any such fork is noticed --
after a fork, each peer will only pay attention to subsequent
transactions on the same fork, and will ignore transactions on other
forks. This is tricky on top of Bitcoin -- Blockstack cannot simply
put the hash of the previous transaction in each of its transactions,
because a given Blockstack peer is likely not aware of all the recent
transactions. Thus Blockstack uses a looser scheme involving a kind of
skip-list of previous hashes, so that peers can decide how close they
are to having seen identical histories.  