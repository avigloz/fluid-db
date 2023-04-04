# fluid-db
Proof of concept for a NoSQL DB that replicates data lazily, maintaining only a single true "owner" of client data.

**Currently in an unusable state/code is unavailable.**

The following things will be included here eventually:
- A basic intermediate server (without caching)
- A database implementation (including socket and direct communication replication, and expiration)
- A basic client app

## Explanation

Consider the following scenario, which is pretty common:

>I register for a service, and log in to my account. 
><br>My session info is stored in region A. Later, I decide I want to fly to region B. 
><br>As a user, I expect to not have to log in again.

Most approaches replicate data, which can be done in any number of ways. 
MongoDB, for example, does it in a fairly straightforward, P2P style where the server
a client is currently using (primary) communicates any writes to other nodes (secondary).

While there's benefits to such a system, namely:
- redunancy
- high availability across regions

Data replication of this sort is completely optional, and at scale it's expensive
to operate nodes that are always fully loaded with every user's data,
even if users never communicate with these replicas. 

That's not to say there's not ways to mitigate this issue, but it often requires 
more engineering effort or additional architectural considerations.

## FluidDB approach

### How does FluidDB differ from other NoSQL implementations?

The main differences are as follows:
- No (real) redundancy
- Lazy availibility

### What the #$&^?

Yes. Referring back to the scenario above, let's dive a little deeper into how it would
work in a FluidDB context, starting from when I land in region B:

>After I land in region B, I expect to stay logged in.
><br>However, the node in region B has never heard of me.

At this point, the node in region B can either:
- reference a log included with query which has info the most-recently-used node
- as a fallback, talk to a subset of all nodes until it finds most-recently-written data the user needs in that moment.

In any case, as soon as data is written, it is set to expire (lets say, within 7 days by default).

>So, in order to learn about me, the node in region B pings the node in region A for my session info.
><br>Every other piece of data I need is communicated to and stored in the region B node, as needed.*

Unless explicitly disabled, data dependent on other data (e.g if a reference to a different object is stored on requested object(s)) is collected 
in a depth-first fashion to ensure functionality.

This data still exists in region A, much of it being the most recent version, but eventually some of this
data will be come obsolesced by new data. Keeping it up to date (or at all) is unnecessary unless I return to region A.

### How does data go from region A to region B?

Two of these are mentioned above.

- Using an intermediary node, which optionally acts as a cache for commonly requested data (more costly)
- Direct communication, which is easier but less reliable and can introduce added latency (less costly)
- Socket-based communication, which can be slower, but increases reliability (less costly)

Sometimes more than one node will have the same version of data, because a client replicated the same data to both nodes
but it wasn't updated and hasn't yet expired.

## Why this approach?

Generally and relatively speaking, it's a lot more lightweight and far cheaper. Nodes only store data needed to cater to local clients, and really nothing more (for long).

As long a user's data reliably moves between nodes in a timely manner, a user never has to worry about "losing data" unless they do not make contact (R/W) with any node
prior to their data expiring. Local expiration is reset/extended upon read or writes.

### Considerations

* Of course, this is design-dependent. You may choose to load all client data at once. In any case, your use case should anticipate
that after a certain amount of time, all client-specific data in region A pertaining to me will be deleted. As such, there is no true redundancy
When I return from region B, the node in region A will ask region B for data, and while 

** That said, it may introduce certain issues for inter-client data sharing. Nothing a cache can't solve, though.
