---
layout: page
title: Overview
order: 1
redirect_from: /store
redirect_from: /store/
redirect_from: /store/index.html

---

TreodeDB is a distributed key-value store that provides atomic writes, and it does so in a way that
works well for RESTful services.  TreodeDB is a library for building your own server.  It offers a
Scala API for read, write and scan; you add layers for data-modeling, client protocol and security.

![arch][arch]


## Programming Interface

TreodeDB maintains tables of rows, each row being a key and value, and it keeps past values back to
some configurable point in time.  A long identifies the table, and the keys and values are arrays of
bytes. TreodeDB primarily provides read, write and scan; the full interface includes details to
implement [optimistic transactions][occ], which services HTTP ETags quite nicely.  We provide a
rough overview here, and get deeper later in the [walkthroughs][rws].  You may also want to consult
the [API docs][api-docs].

```
package com.treode.store

import com.treode.async.{Async, BatchIterator}

trait Store {

  def read (rt: TxClock, ops: ReadOp*): Async [Seq [Value]]

  def write (xid: TxId, ct: TxClock, ops: WriteOp*): Async [TxClock]

  def scan (table: TableId, start: Bound [Key], window: Window, slice: Slice, batch: Batch): BatchIterator [Cell]

}
```

The `read` method requires a time `rt` to read as-of, and you would typically supply `TxClock.now`.
The method returns the values, each with a timestamp.  The maximum timestamp may be used as a
precondition for a later write, and that corresponds to the HTTP response header `ETag`.  The method
ensures that the timestamp of each returned value is the most recent that is less than or equal to
`rt`, and it arranges that all writes (past, in-progress or future) will occur at a timestamp
strictly greater than `rt`.

The `write` method requires a transaction identifier `xid` and a condition timestamp `ct`.  The
transaction identifier is an arbitrary array of bytes which must be universally unique, and you are
responsible for ensuring this.  The write will succeed only if the the greatest timestamp on all
rows in the write are less than or equal to `ct`, which corresponds to the HTTP request header `If-
Unmodified-Since`.

The `scan` method allows the client to retrieve some or all rows of table, and to retrieve the
different versions of each row through a window of time.  The `slice` parameter allows the client to
retrieve a selected partition of the table in a manner that follows TreodeDB's replica placement.

TreodeDB does not cache reads.  You can provide more meaningful caching that is aware of the model
and stores the deserialized and derived objects.  Also, you can provide caching at the edge closer
to the client.  If TreodeDB cached rows, it would be redundant with the caches that should be in
place and just waste memory.

## Testing

TreodeDB includes a stub for testing.  It runs entirely in memory on one node only, so it's fast and
dependency-lite, and that makes it great for unit tests.  Otherwise, the StubStore works identically
to the live one.  Treode's asynchronous libraries provide some useful tools for testing asynchronous
code; more information can be found in the [API docs][api-docs].


```
package com.treode.store.stubs

import com.treode.async.Scheduler
import com.treode.store.Store

class StubStore extends Store {

  def scan (table: TableId): Seq [Cell]
}

object StubStore {

  def apply () (implicit scheduler: Scheduler): StubStore
}
```

## Administering

You may attach new disks to a live server and drain existing disks from it.  You may install new
machines in the cluster or remove old ones and issue a new `Atlas`, which describes how to
distribute data across the cluster. The walkthroughs on [disks][manage-disks] and
[peers][manage-peers] have more details.

```
package com.treode.store

import java.nio.file.Path
import com.treode.async.Async
import com.treode.disk.{DriveAttachment, DriveDigest}

object Store {

  trait Controller {

    def cohorts: Seq [Cohort]

    def cohorts_= (cohorts: Seq [Cohort])

    def drives: Async [Seq [DriveDigest]]

    def attach (items: DriveAttachment*): Async [Unit]

    def drain (paths: Path*): Async [Unit]

  }}
```

Much like the Store interface, TreodeDB provides the Scala API for the controller.  You decide what
is an appropriate network interface and what are appropriate security mechanisms.  Then you add a
layer over TreodeDB to connect your remote interface to the API.


[api-docs]: http://oss.treode.com/docs/scala/store/{{site.vers}} "API Docs"

[arch]: /img/architecture.png "Architecture"

[manage-disks]: /managing-disks "Managing Disks"

[manage-peers]: /managing-peers "Managing Peers"

[occ]: http://en.wikipedia.org/wiki/Optimistic_concurrency_control "Optimistic Concurrency Control"

[online-forum]: https://forum.treode.com "Online forum for Treode Users and Developers"

[rws]: /read-write-scan "Read,Write, Scan"

[sbt]: //www.scala-sbt.org/ "Simple Build Tool"

[sbt-install]: //www.scala-sbt.org/0.13/tutorial/Setup.html "Install SBT"

[server-jar]: https://oss.treode.com/jars/com.treode.store/{{site.vers}}/server.jar

[stackoverflow]: http://stackoverflow.com "Stack Overflow"

[stackoverflow-read]: http://stackoverflow.com/questions/tagged/treode "Read questions on Stack Overflow tagged with treode"

[stackoverflow-ask]: http://stackoverflow.com/questions/ask?tags=treode "Post a question on Stack Overflow tagged with treode"

[store-github]: https://github.com/Treode/store "TreodeDB on GitHub"
