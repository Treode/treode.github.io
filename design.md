---
layout: page
title: Design & Philosophy
order: 6

---

## Design

Across nodes, TreodeDB uses mini-transactions, similar to Sinfonia ([ACM][sinfonia-acm], [PDF][sinfonia-pdf]) and Scalaris ([ACM][scalaris-acm], [website][scalaris-web]), and it uses single-decree Paxos to commit or abort a transaction.  To find data, TreodeDB hashes the row key onto an array of server cohorts, called the atlas, which is controlled by the system administrator.  It requires a positive acknowledgement from a quorum of the cohort to complete a read or write.

On a node, TreodeDB uses a tiered table similar to LevelDB, though TreodeDB maintains timestamped versions of the rows.  To lock data when a transaction is in progress, it hashes the row key onto a lock space, whose size can be configured.  The locks use timestamp locking so that readers of older values do not block writers.

TreodeDB is written in an asynchronous style.  The async package offers an Async class which works with Scala's monads, it provides a Fiber class which is a little bit like an actor but does not require explicit message classes.

The disk system provides a write log and write-once pages.  Periodically it scans for garbage pages and compacts existing ones.  The tiered table mechanism hooks into the compaction to collapse levels at the same time the disk system needs to move live pages.

The cluster system allows the registration of listeners on ports.  It delivers a complete message once or not at all.  The algorithms written above the cluster package implement timeouts and retries, not the cluster system.  This matches how academic texts usually describe distributed protocols like Paxos and two-phase commit.



## Philosophy

#### Transactions

Some applications will require the availability and geographic distribution that eventual consistency can provide.  However other applications can benefit from the simplicity atomic writes affords, and they can tolerate, or even desire, delays when the atomic write cannot complete.  There are many open source options for eventually consistent data stores, but systems that provide transactions are proprietary, SQL based, or bound to a single host.  TreodeDB  provides an open source, NoSQL, distributed, transactional store to fill that gap.

#### Optimistic Transactions

Most transactional storage systems use the traditional interface of begin, commit and rollback.  The client opens a transaction and holds it open until complete.  For a web client to hold anything open causes significant problems, especially when handling disconnects.  Furthermore, begin/commit does not match RESTful design.  On the other hand, optimistic concurrency using timestamps integrates easily with the HTTP specification.

#### Separating Storage from Modeling and Protocol

Building a distributed data store that provides transactions is challenging.  Devising a generic data model is also challenging, and and there are many options for the client protocol and security mechanism.  Separating the key-value store as an API allows TreodeDB to support multiple ways to tackle the other issues.  TreodeDB is opinionated about distributed transactional storage, but not about much else.

#### The Cohort Atlas

Systems such as Google's BigTable ([ACM][bigtable-acm], [PDF][bigtable-pdf]) and HBase&trade; ([website][hbase-web]), layout data by key range.  To locate data for a read or write, a reader fetches indirect nodes from peers to navigate the range map, a type of bushy search tree.  This adds time to reads and writes, which the user needs to be quick.  The systems can cache the indirect nodes, but that requires memory that could be put to other uses.  These systems do this ostensibly to collate data for scans, which the user anticipates will take time.  Those queries whose first phase happens to match the key choice will benefit, but queries that require the initial collation by some other property of the row must still juggle the data.  These systems hamper small frequent read-write operations and consume memory to save time for the few data analyses whose first map-reduce phase happens to match the physical layout.

Systems like Amazon's Dynamo ([ACM][dynamo-acm]) and Riak ([website][riak-web]) use a consistent hash to locate data.  This makes finding an item's replicas fast since the system does not need to fetch nodes of a search tree, but it removes control from the system adminstrators.  Ceph ([USENIX][ceph-usenix], [website][ceph-web]) uses a different consistent hash function, one that provides the system administrator more control.  With that it function it builds array of replica groups, and then to locate an item it hashes the identifier onto that array.  TreodeDB borrows the last part of that idea, but it does not prescribe any particular mechanism to generate the replica groups (cohorts).

In summary, hashing the key onto an array of cohorts allows read-write operations to run quickly.  It saves us from the delusion that our choice of collation benefits all data analyses.  Finally, the mechanism affords the system administrator a reasonable level of control over replica placement.



[bigtable-acm]: http://dl.acm.org/citation.cfm?id=1365815.1365816 "Bigtable: A Distributed Storage System for Structured Data (ACM Digital Library)"

[bigtable-pdf]: http://research.google.com/archive/bigtable-osdi06.pdf "Bigtable: A Distributed Storage System for Structured Data (PDF)"

[ceph-usenix]: https://www.usenix.org/legacy/event/osdi06/tech/weil.html "Ceph: A Scalable, High-Performance Distributed File System (USENIX)"

[ceph-web]: http://ceph.com/ "Ceph (Website)"

[dynamo-acm]: http://dl.acm.org/citation.cfm?id=1294281 "Dynamo: amazon's highly available key-value store (ACM Digital Library"

[hbase-web]: http://hbase.apache.org "Apache HBase&trade; (Website)"

[riak-web]: http://basho.com/riak/ "Riak (Website)"

[scalaris-acm]: http://dl.acm.org/citation.cfm?id=1411273.1411280 "Scalaris: reliable transactional p2p key/value store (ACM Digital Library)"

[scalaris-web]: https://code.google.com/p/scalaris "Scalaris (Google Code)"

[sinfonia-acm]: http://dl.acm.org/citation.cfm?id=1629087.1629088 "Sinfonia: A new paradigm for building scalable distributed systems (ACM Digital Library)"

[sinfonia-pdf]: http://www.sosp2007.org/papers/sosp064-aguilera.pdf "Sinfonia: A new paradigm for building scalable distributed systems (PDF)"
