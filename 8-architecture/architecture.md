---
layout: documentation
title: "Architecture FAQ"
short_title: Architecture FAQ
docs_active: architecture
permalink: docs/architecture/
alias: docs/advanced-faq/
js: faq_index
---

{% infobox %}
__Want to learn more about the basics?__

* Read the [ten-minute guide](/docs/guide/javascript/) to get started with using RethinkDB.
* [Read the FAQ](/faq/).
* Jump into the [cookbook](/docs/cookbook/javascript/) and see dozens of examples of common RethinkDB queries.
{% endinfobox %}


{% toctag %}

# Sharding and replication

## How does RethinkDB partition data into shards? ##

RethinkDB uses a range sharding algorithm parameterized on the table's
primary key to partition the data. When the user states they want a
given table to use a certain number of shards, the system examines the
statistics for the table and finds the optimal set of split points to
break up the table evenly. All sharding is currently done based on the
table's primary key, and cannot be done based on any other attribute
(in RethinkDB the primary key and the shard key are effectively the
same thing).

For example, if a given table contains a thousand JSON documents whose
primary keys are uniformly distributed, alphabetical, upper-case strings
and the user states they want two shards, RethinkDB will likely pick
split point 'M' to partition the table. Every document with a primary
key less than or equal to 'M' will go into the first shard, and every
document with a primary key greater than 'M' will go into the second
shard. The split point will be picked such that each shard contains
close to five hundred keys, and the shards will automatically be
distributed across the cluster.

Even if the primary keys contain unevenly distributed data (such as
human last names, where some keys are likely to occur much more
frequently than others), the system will still pick a correct split
point to ensure that each shard has a roughly similar number of
documents (there are many more Smiths in the phone book than
Akhmechets).

Split points will not automatically be changed after table creation,
which means that if the primary keys are unevenly distributed, shards may
become unbalanced. However, the user can manually rebalance shards when
necessary, as well as reconfigure tables with new sharding and
replication settings. Users cannot set split points for shards manually.

Internally this approach is more difficult to implement than the more
commonly used consistent hashing, but it has significant advantages
because it allows for an efficient implementation of range queries.

## What governs the location of shards and replicas in the cluster? ##

Sharding and replication is configured through _table configurations,_
which let you simply specify the number of shards and replicas per table
or for all tables within a database. Users do not need to manually
associate servers with tables. RethinkDB uses a set of heuristics to
attempt to satisfy table configurations in an optimal way. It will copy
data for new replicas from an available server, evenly distribute replicas
of the data across the cluster, try to distribute the load evenly, and so
on.

For a more fine-grained mechanism, replicas can be associated with
servers using _server tags._ Every server may be assigned one or more
tags, and every table may have a specified number of replicas assigned
to a server tag. For instance, if you have six servers, you might assign
two the tag `us_west`, two the tag `us_east`, and two the tag `london`,
and further assign all four servers in the United States the tag `us`.
Then tables might have their configuration set with `reconfigure` to
group replicas in specific ways:

```py
r.table('a').reconfigure(shards=2, replicas={'us_east':2, 'us_west':2,
    'london':2}, primary_replica_tag='us_east')
r.table('b').reconfigure(shards=2, replicas={'us':2, 'london':1},
    primary_replica_tag='london')
```

In the second example, the two replicas in the `us` group may be on any
of the four servers in the United States.

Note that server tags cannot be configured through RethinkDB's web
administration dashboard. They may be created and assigned through ReQL
commands and scripts.

RethinkDB keeps an internal directory tracking the current state of the
cluster: how many servers are accessible, what data is currently stored
on each server, etc. The data structures that keep track of the
directory are automatically updated when the cluster changes. For
example, if a server dies due to power failure, the directory is updated
to represent this change.

## How does multi-datacenter support work? ##

Starting with RethinkDB 1.16, the earlier concept of "data centers" has
been replaced by server tags, described above. Servers in a given data
center could all be given a tag such as `us_east` or `us_west`, and a
table can be configured to have replicas associated with specific server
tags (e.g., 2 replicas on servers tagged with `us_east` and 3 on servers
tagged with `us_west`).

RethinkDB uses the same protocol for communication within a datacenter
as it does across datacenters. Because the architecture is immediately
consistent and does not require quorums on individual document reads
and writes, the latency issues commonly associated with
cross-datacenter quorums on Dynamo-style systems do not arise in
RethinkDB.

## Does RethinkDB automatically reshard the database without the user's request? ##

The short answer is no. The longer answer is that the clustering
system is designed with three main principles in mind:

- Common operations such as scaling up and down, rebalancing shards,
  and increasing/decreasing replication count should easily be
  performed in a click of a button.
- In cases where it matters, the system should give administrators
  fine-tuned control, such as pinning specific primary and secondary replicas to
  specific servers in the cluster.
- Information about the cluster and all operations on the cluster
  should be programmatically accessible.

We felt that performing automatic maintenance operations on the
cluster (such as adding shards) is a higher-level component, and that
it's crucial to have a really good implementation of the lower-level
components done first. As a result, the clustering system is organized
into three layers:

- The first layer implements the distributed infrastructure, placing
  copies of data on specific servers, routing queries, etc.
- The second layer builds on the first and implements various
  automation mechanisms (e.g. automatically determining how to split
  shards, where to place copies of the data, automatically picking
  optimal primary replicas, etc.)  This is the layer that compiles goals
  specified by the user into blueprints.
- The third layers builds on the previous two and provides the user
  with command line and web-based tools to control the cluster.

Invoking this functionality automatically without the user's request
is the next layer in this hierarchy. Currently the user can control
the system via the web UI, manually via the command line, or by
writing scripts to call the command line tools to perform server
automation.

We're exploring best practices to determine whether it's possible to
build a really good general purpose automation layer that controls the
cluster by automatically enforcing user-specified rules (such as
resharding the system when the shard balance drops below a certain
threshold).

# CAP theorem

## Is RethinkDB immediately or eventually consistent? ##

Every shard in RethinkDB is assigned to a single authoritative primary
replica. All reads and writes to any key in a given shard always get routed
to its respective primary, where they're ordered and evaluated. Data always
remains immediately consistent and conflict-free, and a read that follows an
acknowledged write is always guaranteed to see the write. However, neither
reads nor writes are guaranteed to succeed if the primary replica is
unavailable.

RethinkDB supports both up-to-date and out-of-date reads. By default,
all read queries are executed up-to-date, which means that every read
operation for a given shard is routed to the primary replica for that shard and
executed in order with other operations on the shard. In this default
mode, the client always sees the latest, consistent, artifact-free
view of the data.

The programmer can also mark a read query to be ok with out-of-date
data. In this mode, the query isn't necessarily routed to the shard's
primary, but is likely to be routed to its closest replica. Out-of-date
queries are likely to have lower latency and have stronger
availability guarantees, but don't necessarily return the latest
version of the data to the client.

## What CAP theorem tradeoffs are made in RethinkDB? ##

The essential tradeoff exposed by the CAP theorem is this: in case of
network partitioning, does the system maintain availability or data
consistency? (Jumping ahead, RethinkDB chooses to maintain data
consistency).

Dynamo-based systems such as Cassandra and Riak choose to maintain
stronger availability. In these systems if there is a network
partition, the clients can write to the same row on both sides of the
netsplit. In exchange for the write availability, applications built
on top of these systems must deal with various complexities such as
clock skew, conflict resolution code, conflict repair operations,
performance issues for highly contested keys, and latency issues
associated with quorums.

Authoritative systems such as RethinkDB and MongoDB choose to
maintain data consistency. Building applications on top of
authoritative primary systems is much simpler because all of the issues
associated with data inconsistency do not arise. In exchange, these
applications will occasionally experience availability issues.

In RethinkDB, if there is a network partition, the behavior of the
system from any given client's perspective depends on which side of the
netsplit that client is on. If the client is on the same side of the
netsplit as the majority of voting replicas for the shard the client is
trying to reach, it will continue operating without any problems. If the
client is on the side of the netsplit with half or fewer of the voting
replicas for the shard the client is trying to reach, the client's
up-to-date queries and write queries will encounter a failure of
availability. For example, if the client is running an up-to-date range
query that spans multiple shards, the primaries for all shards must be
on the same side of the netsplit as the client, or the client will
encounter a failure of availability.

If the programmer marks a read query to be ok with out-of-date data,
RethinkDB will route the query to the closest available replica
instead of routing it to the primary. In this case the client will see
the data as long as there are replicas of the data on its side of
the netsplit. However, in this case the data has the risk of being out
of date. This is usually ok for reports, analytics, cached data, or
any scenario in general where having the absolute latest information
isn't imperative.

## How is cluster configuration propagated? ##

Updating the state of a cluster is a surprisingly difficult problem in
distributed systems. At any given point different (and potentially)
conflicting configurations can be selected on different sides of a
netsplit, different configurations can reach different nodes in the
cluster at unpredictable times, etc.

RethinkDB uses the [Raft algorithm][ra] to store and propagate cluster
configuration in most cases, although there are some situations it uses
semilattices, versioned with internal timestamps. This architecture
turns out to have sufficient mathematical properties to address all the
issues mentioned above (this result has been known in distributed
systems research for quite a while).

[ra]: https://en.wikipedia.org/wiki/Raft_(computer_science)

# Indexing

## How does RethinkDB index data? ##

When the user creates a table, they have the option of specifying the
attribute that will serve as the primary key (if the primary key
attribute isn't specified, it defaults to 'id'). When the user inserts
a document into the table, if the document contains the primary key
attribute, its value is used to index the document. Otherwise, a random
unique ID is generated for the index automatically.

The primary key of each document is used by RethinkDB to place the
document into an appropriate shard, and index it within that shard
using a B-Tree data structure. Querying documents by primary key is
extremely efficient, because the query can immediately be routed to
the right shard and the document can be looked up in the B-Tree.

## Does RethinkDB support secondary and compound indexes? ##

RethinkDB supports both secondary and compound indexes, as well as
indexes that compute arbitrary expressions. You
can see examples of how to use the secondary index API
[here](/docs/secondary-indexes).

# Availability and failover

## What happens when a server becomes unreachable? ##

If your cluster has at least three servers, then in most cases RethinkDB
will be able to perform automatic failover and maintain table availability.

- If the primary replica for a table fails, as long as more than half of the
table's voting replicas and more than half of the voting replicas for each
shard remain available, one of the voting replicas will become the new
primary replica.

- If half or more of the voting replicas for a shard are lost (including the
case of a two-server cluster losing one server), the cluster will need to be
repaired manually using the [emergency repair][er] option of `reconfigure`.

For more details, read about [Failover][f].

[er]: /api/javascript/reconfigure/#emergency-repair-mode
[f]:  /docs/failover

## What are availability and performance impacts of sharding and replication? ##

RethinkDB maintains availability if the user increases or decreases
the number of replicas in the cluster. In most cases, the replication
process should not have a strong performance impact on the real-time
system.

RethinkDB may or may not maintain availability if the user modifies
the number of shards. In many cases availability will be maintained,
but currently it cannot be guaranteed. We're exploring different
solutions to remove this limitation.

# Query execution

## How does RethinkDB execute queries? ##

When a node in the cluster receives a query from the client, it
evaluates the query in the following way.

First, the query is transformed into an execution plan that consists
of a stack of internal logical operations. The operation stack fully
describes the query in a data structure useful for efficient
execution. The bottom-most node of the stack usually deals with data
access&mdash; it can be a lookup of a single document, a short range scan
bounded by an index, or even a full table scan. Nodes closer to the
top usually perform transformations on the data -- mapping the values,
running reductions, grouping, etc. Nodes can be as simple as
projections (i.e. returning a subset of the document), or as complex
as entire stacks of stacks in case of subqueries.

Each node in the stack has a number of methods defined on it. The
three most important methods define how to execute a subset of the
query on each server in the cluster, how to combine the data from
multiple servers into a unified resultset, and how to stream data to
the nodes further up in small chunks.

As the client attempts to stream data from the server, these stacks
are transported to every relevant server in the cluster, and each
server begins evaluating the topmost node in the stack, in parallel
with other servers. On each server, the topmost node in the stack
grabs the first chunk of the data from the node below it, and applies
its share of transformations to it. This process proceeds recursively
until enough data is collected to send the first chunk to the
client. The data from each server is combined into a single
resultset, and forwarded to the client. This process continues as the
client requests more data from the server.

The two most important aspects of the execution engine is that every
query is completely parallelized across the cluster, and that queries
are evaluated lazily. For instance, if the client requests only one
document, RethinkDB will try to do just enough work to return this
document, and will not process every shard in its entirety. This
allows for large, complicated queries to execute in a very efficient
way.

The full query execution process is fairly complex and nuanced. For
example, some operations cannot be parallelized, some queries cannot
be executed lazily (which has implications on runtime and RAM usage),
and implementations of some operations could be significantly
improved. We will be adding tools to help visualize and understand
query execution in a user-friendly way, but at the moment the best way
to learn more about it is to ask us or to look at the code.

## How does the atomicity model work? ##

Write atomicity is supported on a per-document basis -- updates to a
single JSON document are guaranteed to be atomic. RethinkDB is
different from other NoSQL systems in that atomic document updates
aren't limited to a small subset of possible operations -- any
combination of operations that can be performed on a single document is
guaranteed to update the document atomically. For example, the user
may wish to update the value of attribute A to a sum of the values of
attributes B and C, increment the value of attribute D by a fixed
number, and append an element to an array in attribute E. All of these
operations can be applied to the document atomically in a single
update operation.

However, RethinkDB does come with some restrictions regarding which
operations can be performed atomically. Operations that cannot be
proven deterministic cannot update the document in an atomic
way. Currently, values obtained by executing JavaScript code, random
values, and values obtained as a result of a subquery
(e.g. incrementing the value of an attribute by the value of an
attribute in a different document) cannot be performed atomically. If
an update or replace query cannot be executed atomically, by default
RethinkDB will throw an error. The user can choose to set the flag on
the update operation in the client driver to execute the query in a
non-atomic way. Note that non-atomic operations can only be detected when
they involve functions (including `row()`) being passed to `update` or
`replace`; a non-atomic `insert` operation will not throw an error.

In addition, like most NoSQL systems, RethinkDB does not support
updating multiple documents atomically.

## How are concurrent queries handled? ##

To efficiently perform concurrent query execution RethinkDB implements
block-level multiversion concurrency control (MVCC). Whenever a write
operation occurs while there is an ongoing read, RethinkDB takes a
snapshot of the B-Tree for each relevant shard and temporarily
maintains different versions of the blocks in order to execute read
and write operations concurrently. From the perspective of the
applications written on top of RethinkDB, the system is essentially
lock-free&mdash; you can run an hour-long analytics query on a live system
without blocking any real-time reads or writes.

RethinkDB does take exclusive block-level locks in case multiple
writes are performed on documents that are close together in the
B-Tree. If the contested block is cached in memory, these locks are
extremely short-lived. If the blocks need to be loaded from disk,
these locks take longer. Typically this does not present performance
problems because the top levels of the B-Tree are all cached along with
the frequently used blocks, so in most cases writes can be performed
essentially lock-free.

# Data storage

## How is data stored on disk? ##

The data is organized into B-Trees, and stored on disk using a
log-structured storage engine built specifically for RethinkDB and
inspired by the architecture of BTRFS. The storage engine has a number
of benefits over other available options, including an incremental,
fully concurrent garbage compactor, low CPU overhead and very
efficient multicore operation, a number of SSD optimizations,
instantaneous recovery after power failure, full data consistency in
case of failures, and support for multiversion concurrency control.

The storage engine is used in conjunction with a custom, B-Tree-aware
caching engine which allows file sizes many orders of magnitude
greater than the amount of available memory. RethinkDB can operate on
a terabyte of data with about ten gigabytes of free RAM.

## How does RethinkDB handle data corruption? ##

It relies on the underlying storage system to ensure data consistency.
RethinkDB does not perform additional checksums on stored data. It is,
however, compatible with file systems which do guarantee data integrity,
such as ZFS.

## Which file systems are supported? ##

RethinkDB supports most commonly used file systems. It optionally
supports direct disk I/O for greater efficiency, but this is not enabled
by default.

## How can I perform a backup of my cluster? ##

RethinkDB ships with simple tools to perform a hot backup of a running
cluster. See the [backup instructions](/docs/backup/) for more
details.
