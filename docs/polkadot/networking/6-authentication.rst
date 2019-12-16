====================================================================

**Authors**: Ximin Luo

**Created**: 29.11.2019

====================================================================

==============
Authentication
==============

In a public secure decentralised network, it’s important for some level of
service to be provided to unauthenticated users, e.g. for transparency and
auditing purposes. This however opens up avenues of attack for malicious agents
to attack the network. Fortunately, this is straightforward to defend against
at least conceptually - simply constrain the amount of resources that are
allocated to service unauthenticated users.

Extending this to a more general case, we can imagine that, depending on the
needs of the higher layer (e.g. application), there may be several possible
roles available for authentication, where some roles have higher resource
priorities than others.

In a computer system, the possible resource types are typically taken to be
CPU, memory and bandwidth. For now, we’ll only focus on the latter in this
document because it’s the simplest to control directly - and doing so should
also help to implicitly constrain the other two albeit with less accuracy than
a more direct mechanism.

To clarify our terminology, note that there are two things commonly referred to
as "authentication":

1. cryptographic authentication, where messages (ciphertext) are linked to
   (authenticated against) some cryptographic identity such as the entity
   controlling a private key.
2. identity authentication, where cryptographic identities are linked to some
   other source of authority that distinguishes "good" identities from "bad" or
   "unknown" ones.

In the rest of this document we talk solely about (2), which is the harder
problem especially because the set of "good" identities can change over time.
(1) is straightforward and is done automatically by the networking layer via
the underlying transport protocol (e.g. TLS or QUIC) and requires no runtime
configuration to achieve. Both (1) and (2) need to be done in order to achieve
security. Note also that if (1) is done but not (2), then the actual identity
supplied by (1) may be false, since it may be controlled by an active MITM.

General
=======

.. _proposal-fresh-authentication-signals:

Proposal: fresh authentication signals
--------------------------------------

The networking layer receives some input data or signal from a higher layer,
which instructs it how to validate all the different roles of authenticated
peers. (For example, peers must present a certificate group-signed by a recent
set of validators, or belong to a fixed set of identities supplied by the
higher layer.)

This signal must include some period of validity, after which the networking
layer will deauthenticate (disconnect and ignore) any existing authenticated
peers, and reject any further attempts at authentication. In other words, the
higher layer must continually refresh the validity instructions for the
networking layer, otherwise it will eventually revert to only dealing with
unauthenticated peers with a very restricted resource policy.

One effect of this is that, if a node goes offline for a long-enough period of
time, the networking layer will expire its last authentication instructions.
This will result in disconnection of all authenticated peers; this is drastic
but is better than continuing in an authenticated state that is expired. The
higher layer should not let the situation reach this stage, but if it does, it
should run e.g. a synchronisation protocol to retrieve the latest version of
the authentication instructions, and signal this to the networking layer again.

Note that in practise with epoch-based authentication schemes, the higher layer
may want to send two staggered signals per epoch switch - first sending a
signal to include instructions for the next epoch, then waiting a grace period
for the external network to all switch to the next epoch, then a second signal
to exclude instructions for the previous epoch.

Proposal: bandwidth resource allocation
---------------------------------------

Given some roles (or more concretely, “peer connections”) we want to send data
to or receive data from, how do we do this in a “fair” way?

Here’s one basic proposal. Suppose we have a list of roles R, and each role is
associated with a ``priority`` and a relative ``use_limit``, such that
``sum(use_limit[r] for r in R) == 1`` and R is sorted by priority, “most
urgent” first; this defines our resource limits and is to be configured by the
higher layer. Then suppose for every time-interval t we have a list of actual
usages for each role R, written as ``demand[r]``, and we want to calculate what
to actually use for each role, ``to_use[r]`` such that ``sum(to_use) <
total_avail``.

Then the algorithm we propose below (valid for both sending and receiving)
satisfies the following properties:

1. If the ``sum(demand) < total_avail``, then ``to_use == demand``. In
   other words, if demand is lower than availability then demand is
   fully-satisfied, regardless of priorities or use-limits. A good algorithm
   should not special case this explicitly, but simply degenerate to this when
   appropriate.

2. If ``all(demand[r] > use_limit[r] * total_avail for r in R)``, then
   ``all(to_use[r] == use_limit[r] * total_avail for r in R)``. In other words
   if every role demands to use more than is available, then their actual use
   is simply the proportion that is defined by our resource limits. This avoids
   higher-priority roles starving lower-priority roles.

The algorithm is as follows, described as executable Python:

.. include:: bw-alloc.py
   :code: py

In a real networking program, the above algorithm may have to be tweaked a bit.
However this should not be so hard - it should be possible to know what
``demand`` is at any given moment either for sending or for receiving.

-  for sending: in order to deal with backpressure properly you should
   always have a priority-heap of things to send, as opposed to trying to send
   something as soon as it becomes available to send.

   -  (The priority-heap is populated as new things become available to
      send, and is popped when the recipient signals via some control-flow
      mechanism that they are unblocked for receiving again. This priority is
      unrelated to the role-priorities we introduce above for
      ``calculate_to_use``.)

-  for receiving: normal networking implementations have the kernel
   buffer stuff until the application is ready to process it, and any excess is
   dropped. It is normally possible to query how full the buffer is.

At each time-interval, it is straightforward to calculate ``demand`` from
either the application’s send priority-heaps or the kernel’s receive buffers.

As applied to Polkadot
======================

Unauthenticated operation
-------------------------

Collators without any fresh authentication instructions can only:

-  TODO…

Validators without any fresh authentication instructions can only:

-  TODO…

Authenticated roles
-------------------

Collators need to authenticate the following:

-  Parachain validators
-  TODO…

Validators need to authenticate the following:

-  TODO…

XXX
---
