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

