---
layout: page
title: Overview
order: 1
redirect_from: /store
redirect_from: /store/
redirect_from: /store/index.html

---

TreodeDB is a distributed key-value store that provides atomic multirow writes, and it works well for RESTful services. It connects to Spark for analytics, it batches updates for search indexes, and it integrates well with CDNs for speed.

![arch][arch]


## RESTful Transactions

TreodeDB maintains tables of rows, each row being a key and value, and it keeps past values back to some configurable point in time. It provides a type of database transaction called the [conditional batch write][cbw]. The walkthrough on [read, write and scan][rws] explains the details.


## Administration

You may attach new disks to a live server and drain existing disks from it.  You may install new machines in the cluster or remove old ones. The walkthroughs on [disks][manage-disks] and [peers][manage-peers] have more details.


[arch]: /img/architecture.png "Architecture"

[cbw]: https://forum.treode.com/t/eventual-consistency-and-transactions-working-together/36 "Eventual Consistency and Transactions Working Together"

[manage-disks]: /managing-disks "Managing Disks"

[manage-peers]: /managing-peers "Managing Peers"

[rws]: /read-write-scan "Read,Write, Scan"
