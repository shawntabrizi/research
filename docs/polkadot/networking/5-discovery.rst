====================================================================

**Authors**: Ximin Luo

**Created**: 05.11.2019

====================================================================

=========
Discovery
=========

As with many real-world secure p2p networks, nodes should deal with each other
as cryptographic identities, and it should not matter too much what their
physical (network) address(es) are since these can easily be faked by the
surrounding environment. Nevertheless for the communications to get through to
the other side, we need to associate the correct addresses with the correct
identities, and so we have an abstract "address book" service that provides
this association. [#]_ In the p2p space, this is commonly referred to as "node
discovery" or simply "discovery".

Typically in real-world p2p systems this service is implemented by a
decentralised DHT, and the most commonly-used solution today is the Kademlia
DHT. So this document has a lot of focus on that, but we also discuss
non-Kademlia solutions and combined hybrid solutions.

.. [#] As always, the communication contents are protected by cryptography, so
   the worst attack then is simply to drop the communication, which we must
   assume doesn't happen often enough to disrupt the whole network.


Background
==========

Kademlia is a DHT (distributed hash table) that is one of the more popular ones
to implement in real-world p2p systems. Much of this is due to the fact that
the mathematics of it is relatively simple to implement, involving essentially
only the XOR operation, and its various bookkeeping tasks are also reasonably
straightforward, mostly involving comparing timestamps.

It has some weaknesses that are typical of generic DHT systems, namely being
susceptible to the sybil attack - global identity flooding - and the eclipse
attack - targeted identity flooding.

In general these attacks can be sufficiently defended against, by making use of
context-specific information that is outside of the model of more general
analyses that do not take into account such information. For example, polkadot
does have the idea of more trusted users (e.g. validators) that we can make use
of in the DHT layer as well. We make some concrete proposals below.

S-Kademlia is a commonly-cited extension to Kademlia that offers a couple of
basic security mechanisms on top of Kademlia:

1. A proposal to increase the cost of generating new IDs. This is general and
   not specific to Kademlia, and there has been newer work on this as well.

2. A bug fix to the distance algorithm for nodes very close to yourself.

3. A security fix for the lookup algorithm where it's trivial for a malicious
   node to "override" the responses of non-malicious nodes.

This last one (3) is often described as "disjoint paths" but the current author
believes that this is misleading phrasing and is better described as "disjoint
first hop". In original Kademlia, the results of all first-hop queries are
accumulated and the closest k (to the eventual target) are selected for the
second hop. Here obviously, a malicious first-hop peer can trivially control
the second hop, by returning fake nodes that seem very close to the eventual
target. S-Kademlia fixes this by mandating that second-hop peers are chosen
from the results from *different* first-hop peers. There is no enforcement of
disjointness beyond the second hop - and in fact in a working routing algorithm
the paths are naturally supposed to converge (i.e. become non-disjoint) in the
good case with no attackers, so it does not make sense to "enforce
disjointness" or talk about "disjoint paths" as a good thing.

Note also that this defence is effective against an eclipse attack on yourself,
but not effective against an eclipse attack around the eventual target, since
your queries (with disjoint first hops) are likely to converge towards the
target, which is what they are supposed to do with *any* working routing
algorithm, eventually likely landing on the eclipse attackers who will
presumably redirect you away from the correct eventual target node.


Design topics
=============

Authentication and resource usage
---------------------------------

See :doc:`6-authentication` for some background on the general topic of
authentication. Kademlia natively does not have a concept of authentication,
but it is fairly straightforward to fit one in.

Specifically, Kademlia natively already treats recently-seen peers as more
trustworthy than others - they have higher priority when being added into its
k-buckets. So, we can generalise this idea to cover peers that have been marked
as more trustworthy by a higher-layer application.

For polkadot, this means that in the relay chain validators should pre-allocate
some space in their k-buckets *for other validators* such that this space
cannot be overridden by other non-validator nodes. This functionality should be
configurable via a generic API offered by the kademlia implementation that does
not specifically reference polkadot or specific authenticated roles, so it can
continue to be useful even if we (for example) add extra types of authenticated
roles to the polkadot relay chain. Details are given in the next section.

This ensures that a certain amount of resources is always allocated to those
"more trustworthy" peers. It does not protect against an attack by the "more
trustworthy" peers themselves, e.g. if a group of validators themselves decides
to attack the kademlia network. However, it is intended that other mechanisms
will be able to detect such attacks, and furthermore the slashing mechanisms in
place elsewhere will be able to discourage these types of attacks, whereas the
slashing mechanism by its nature cannot work against unauthenticated peers.

Proposal: authenticated stratified k-buckets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. The kademlia implementation should offer an API for the higher-layer
application to define different "authenticated roles" for peers that can
potentially be inserted into k-buckets, along with the maximum fraction of the
whole k-bucket that peers belonging to that role are allowed to occupy. Roles
must be sortable. So for example a typical configuration might look like:

- ``{ role 2: 0.5, role 1: 0.3 }``

The sum of the fractions should <= 1, with the remainder being allocated to the
default unauthenticated role ("role 0"). So in the above example, role 0 would
be left with a fraction of 0.2.

This should be called sporadically e.g. at the start of the process run.

2. The kademlia implementation should offer an API for the higher-layer
application to define which cryptographic identities belong to which roles. As
described in :any:`proposal-fresh-authentication-signals` this should come
with an expiry time and be refreshed continually. For example a typical API
call might look like:

- ``{ role 2: member_of { some set }, role 1: some predicate, expires: 1 hour }``

When the k-bucket is not full, these configurations have no effect and the
behaviour is the same as ordinary Kademlia. That is, a new node is simply added
to the k-bucket, and an existing node has their timestamp refreshed.

When the k-bucket is full however, this configuration serves to affect which
other node is potentially evicted in favour of the new node. In ordinary
Kademlia it is the node with the oldest timestamp that is nominated for
eviction - a PING is sent and if they don't reply then they are evicted in
favour of the new node. In our new proposal, the node to maybe-evict is instead
selected as follows:

1. from lowest to highest role (starting with the unauthenticated role 0):

   a. if the k-bucket contains more than k*fraction entries for that role, then

      i. select the oldest node *in that role* for eviction

2. if the above loop did not select any node, then

   a. select the oldest node in *the same role* as the new node for eviction

This should serve to guarantee that:

1. If resources are available (i.e. the k-bucket is not full) then everyone is
   added, without prejudice.

2. If resources are constrained then higher-priority roles are added with
   greater priority than lower-priority roles, but without starving
   lower-priority roles of the guarantees they are given according to the
   fractional resource limits configuration.


Parachain extensions
--------------------

WIP, TODO

Something about using multiple DHTs so parachains can run mostly independently?

Coral’s clustering algorithms may be useful here.

Allow validators to also store their addresses in an on-chain address book.
This has different security tradeoffs and different validators may want to
choose differently, so it is good to offer both on-chain address book and
off-chain DHT as options.


Implementation topics
=====================

This section discusses some implementation topics not commonly mentioned in
research papers, that any “real world” implementation has to deal with. We will
approach these issues in a methodical and principled way so that implementors
are not forced to come up with their own hacky solutions.

Real-world network addresses
----------------------------

Most networking papers (including Kademlia and S-Kademlia) talk about node
addresses as a trivial fact that each node inherently knows. In both the real
internet and a more precise model of an abstract addressing scheme, this is not
realistic at all. Existing Kademlia implementations therefore are forced to
devise their own hacky non-standard methods to detect node addresses, most of
which are quite insecure, and some of which are unable to deal with multiple
addresses, and some of which get confused by local or NAT addresses. We present
something that should be much more robust in all possible real-world scenarios.

Starting from first principles, an address is simply a piece of information
used to receive things from others. You do not need to know your own address in
order to send things to other people. If your house is teleported around at
will by your landlord, you have no way of knowing what your address is unless
you re-explore your surroundings by asking other people. Likewise, the analogue
of this happens all the time on the internet - sometimes because the node owner
did in fact decide to move the machine, but at other times because the ISP
reconfigured their network outside of the node owner’s control. This might even
happen within an organisation that wants to e.g. keep the networking department
well-separated from the applications department.

On the internet, from addresses are added automatically as part of the delivery
of a packet. However these may not always be correct, i.e. these addresses may
not actually have the ability to receive replies. However one can flip this
around slightly to create a more robust and “obviously correct” protocol:

1. If node A sends to address B some unpredictable string, and

2. Node A later receives (from any address) the same unpredictable
   string, then

3. Node A knows that address B can be observed by some alive node, that
   can reply with the aforementioned from address.

Adding signatures or some other cryptographic authentication makes the security
more precise:

1. If node A sends to address AddrB some unpredictable string, and

2. Node A later receives (from any address) the same unpredictable
   string signed (or otherwise authenticated) by node B, then

3. Node A knows that node B is able to observe (some) information sent
   to AddrB, at least in the recent past and hopefully recent future.

Note that AddrB may or may not be the “actual address” for node B - whatever
“actual address” means - it is possible there is a proxy in the way that is
translating addresses, but this doesn’t matter too much - we have gained some
concrete evidence that node B is *reachable* via AddrB, and that is all that
matters for most protocols, and certainly for Kademlia. And in some sense,
everywhere on the internet is proxied by routers anyway.

Proposal: multiple addresses
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Out of this we can amend the Kademlia protocol as follows. The original paper
describes each nodeId as being associated with a single address and a single
timestamp. Our amendment associates each nodeId with a list of (address, seen)
pairs. The “seen” type describes how we know about that address, and is itself
a pair (verification-method, timestamp) where verification-method is either
“Untrusted” or “ExplicitReply”, where “ExplicitReply” sorts higher than
“Untrusted”. The overall “seen” value for the given nodeId, is simply the
highest “seen” value over all of its (address, seen) pairs.

When sending an outgoing request, we try all of a node’s “ExplicitReply”
addresses first, from most-recent to least-recent, in sequence. If these all
timeout (or don’t exist) then we try the “Untrusted” addresses, in parallel as
reasonable, e.g. 3 at once. If these all timeout then we fail the overall
sending request, which is what higher-level functionality should wait on.

This gracefully deals with the issue of NATs and firewalls since we simply use
whatever address works, without being too fussed about which one is “the
correct one”. This is very similar to how ICE (RFC 5245) does things.

The “unpredictable string” we mentioned above is simply the same as Kademlia’s
“request IDs” that is already part of the protocol. We store the ones we
generate, and match these against incoming replies. We also store the timestamp
of the original send, and use that as the timestamp associated with the
ExplicitReply, so that all the timestamps used are *local* values according to
our own clock. We cannot compare our own timestamps with others’ timestamps.

We do have to embed the sending address in the body of the request packet, and
the replying node should read this and re-embed this in the body of the reply
packet. This is necessary to distinguish which address(es) actually succeeded,
when sending a packet to the same node but different addresses in parallel.
Also, packet bodies should be authenticated, otherwise NATs and malicious MITMs
could rewrite this information. (The replying node could also fake its own
address in the reply, but this would simply be attacking itself.)

When we receive a reply, we update the relevant “seen” entries as prescribed by
original Kademlia, but extended in the obvious way to deal with multiple
addresses. I.e. either update the relevant address’s “seen” value to
(ExplicitReply, timestamp) or insert it anew up to some bounded limit, if it
doesn’t already exist in the list.

When we receive a FIND_NODE request, we only send the addresses and not our own
“verification-method” markers, since these should not be trusted by others. In
the future we may come up with an heuristic to make use of this information but
that is a hard problem so we omit it for now to avoid implementors attempting
their own hacky solutions.

When we receive a FIND_NODE reply, we add all those addresses as “Untrusted”.
If these overflow our limit, we might use some heuristic to choose which ones
to retain, e.g. one per IP block or something like that. Again this is
difficult to do well and could be the target of spam attacks, but we’ll defer
this problem for the future. At the very least, an “Untrusted” address should
not cause an “ExplicitReply” address to be ejected. The only way for an
“ExplicitReply” address to be dropped is if we send a PING to that explicit
address and it fails to reply. A (semi-)exception is mentioned below.

One special case to note here is that of finding out your own address. The
subprotocol mentioned above works in theory when checking yourself, and works
in practice under well-configured simple networks without NAT, but does not
work well in badly-configured networks or those under NAT. In these cases,
others can reach you via your “public” addresses, but you are not able to reach
yourself under those addresses and must use a local address. So under the above
proposal, all of your own public addresses you would store as “Untrusted” and
not “ExplicitReply”. However, since we strip out this information when sending
FIND_NODE replies to others, this should still remain robust since others will
get a chance to test these addresses for themselves. We just have to be careful
that e.g. we don’t fill up our own address-list with only ExplicitReply local
addresses that are unusable for others. In other words, for our own address
list, we should ensure that there is enough reserved space for “Untrusted”
addresses that others can use to test.

Drop policy
-----------

Nodes are dropped immediately if we PING and they time out. We may want to be a
bit more generous, especially since Kademlia has an explicitly stated design
intention of preferring known-long-term nodes to unknown nodes.

We should collect real-world logs about how often nodes drop vs whether this is
short-term or long-term.

Using stream-based sockets
--------------------------

Kademlia as commonly described runs over UDP, and includes within it a
subprotocol for checking whether nodes are alive. Above, we extended this so
that it supports multiple addresses across complex network topologies.

Some real-world Kademlia implementations on the other hand use TCP instead of
UDP. The main advantage of this is that TCP includes a handshake already so
that the alive-checking subprotocol (PING, PONG) is not required, and therefore
the Kademlia implementation can be made simpler. Depending on how TCP is used,
there are different downsides to this approach however:

1. If a new TCP connection is opened on every request (FIND_NODE,
   FIND_VALUE, STORE), then this increases latency by several round trips,
   i.e. at least 4x.

2. If TCP connections are kept open on a per-peer basis, this consumes
   OS resources (TCP ports) which might be better-used for other applications
   or other parts of the same application. Furthermore, care must be taken to
   ensure that relevant keepalives and heartbeats are set on the TCP connection
   to ensure it stays alive, otherwise one throws away the advantage of keeping
   these open in the first place. This is preferable to (1) in most real-world
   situations.

In (2) the disadvantage can be mitigated by using QUIC which does not need to
consume many OS ports, instead consuming just one per usage-context and
multiplexing through it multiple connections indexed by 64-bit connection IDs
handled internally by QUIC. These cannot run out, at least before the machine
itself runs out of memory.

In this case, the tradeoff remains that one has to keep these connections alive
using QUIC pings which are done on a time-interval basis, rather than Kademlia
native pings which are done in response to various network observations. The
resulting simplification however is probably worth it in overall “real-world”
development terms, assuming that one already has a good QUIC implementation
(which at the time of writing, end 2019, is actually a bit of a stretch):

Proposal: multiple addresses, with QUIC
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

-  We can omit PING/PONG entirely

-  When we receive information about new nodes, we don’t have to check
   old nodes whether they are alive or not since this is already taken care of
   by the QUIC keepalive logic. Instead, we simply attempt to open up a QUIC
   connection to each new node that we know about, and add them to the k-bucket
   if this connection attempt succeeds.

-  When doing this in conjunction with the “multiple addresses” proposal
   above, we do not need to embed the send address in the body of the request,
   since any discrepancies between to/from addresses are handled by QUIC or
   QUIC-TLS.

-  QUIC is able to keep connections alive even if one endpoint changes
   IP address. Whenever we receive something from a node’s connection,
   we should check the current from-address of the connection and update
   the relevant (verification-method, timestamp) tag for this address
   specifically. This is effectively the same as in the “multiple
   address” proposal above, just re-worded for clarity to be more
   specific to QUIC.

As per the “drop policy” section above, if an existing QUIC connection times
out, the analogous thing to do here would be to remove them immediately from
the k-bucket. However it may be more prudent to attempt to re-establish the
connection at least for a short while, before switching to a different node, to
better effect the “long-term nodes are preferred” design intention of Kademlia.


Other links
===========
