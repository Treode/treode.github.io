---
layout: page
title: Read & Write
order: 3
redirect_from: /read-write-scan.html
redirect_from: /store/read-write-scan.html

---

This page walks through the RESTful API for reading and writing by using `curl`.


## Prerequisites

If you haven&#700;t already, [setup a server][get-server].


## Write a Row

The server will support GET, POST, PUT and DELETE for paths of the form `/<table>/<key>`. Let&#700;s PUT a value for `apple`.

<pre class="highlight">
curl -i -w'\n' -XPUT -d@-  http://localhost:7070/fruit/apple &lt;&lt; EOF
"sour"
EOF
<span class="go">HTTP/1.1 200 OK
Value-TxClock: 1436030358171001
Server-TxClock: 1436030358172001
Content-Length: 0</span>
</pre>

We PUT the string `"sour"`; TreodeDB will accept any well-formed JSON value.

TreodeDB provides the time that the value was written in the header `Value-TxClock`. This timestamp counts the approximate number of microseconds since the Unix epoch. If you store it, you must store the complete value; do not round to milliseconds, seconds or a coarser resolution. Take note of the `Value-TxClock` from your run, as it will be different from above, and we will use it later in the examples.

TreodeDB also provides the logical time of the cell in `Server-TxClock`. The server adds this to all responses regarding a row so that the client may implement a logical clock. This allows the client to handle skew between its clock and server's. Like the `Value-TxClock`, you must handle this value in microsecond resolution. If you compare it to the system time, be sure to convert that to microseconds. Multiplying the system's millisecond time by 1000 is fine; you do not need a clock with microsecond resolution.


## Read a Row

Use GET to read the value for `apple`.

<pre class="highlight">
curl -i -w'\n' http://localhost:7070/fruit/apple
<span class="go">HTTP/1.1 200 OK
Date: Sat, 4 Jul 10:19:52 2015 PDT
Last-Modified: Sat, 4 Jul 10:19:18 2015 PDT
Read-TxClock: 1436030392784000
Value-TxClock: 1436030358171001
Server-TxClock: 1436030392784000
Vary: Read-TxClock
Content-Type: application/json
Content-Length: 6

"sour"</span>
</pre>

The method returns the time that the value was read as-of in the header `Read-TxClock`, and it returns the time that the value was written in the header `Value-TxClock`. It also returns those times in the coarser HTTP format to integrate properly with intermediate HTTP caches. Be sure to use the finer resolution TxClock values for transactions.


## Write Conditionally

Suppose two clients read the value `"sour"` for `apple`, a both try to change it at about the same time. TreodeDB can use the timestamp as a precondition for a write. Suppose the first client rewrites the value to `"green"`.

<pre class="highlight">
curl -i -w'\n' -XPUT -d@- \
    -H'Condition-TxClock: 1436030358171001' \
    http://localhost:7070/fruit/apple &lt;&lt; EOF
"green"
EOF
<span class="go">HTTP/1.1 200 OK
Value-TxClock: 1436030453060001
Server-TxClock: 1436030453062001
Content-Length: 0</span>
</pre>

The first client includes the header `Condition-TxClock`. The row has not been updated since that time, so the write succeeds. Next, suppose a second client tries to rewrite it to `"delicious"`.

<pre class="highlight">
curl -i -w'\n' -XPUT -d@- \
    -H'Condition-TxClock: 1436030358171001' \
    http://localhost:7070/fruit/apple << EOF
"delicious"
EOF
<span class="go">HTTP/1.1 412 Precondition Failed
Value-TxClock: 1436030453060001
Server-TxClock: 1436030454571001
Content-Length: 0</span>
</pre>

The second client also includes the precondition time. This time through, the row has been updated since then, so the write returns `412 Precondition Failed`, and it provides the write time for the newest value.

The `Condition-TxClock` header also works with the HTTP DELETE method. The row will be deleted only if it has not been changed since the precondition time. The POST method ignores the `Condition-TxClock` header; it will set the row if it had never had a value, or if its latest write was a DELETE, regardless of when the write occurred.


## Read Conditionally

The `Condition-TxClock` header also works for reading. The value will be returned if it has changed since that time.

<pre class="highlight">
curl -i -w'\n' \
    -H'Condition-TxClock: 1436030358171001' \
    http://localhost:7070/fruit/apple
<span class="go">HTTP/1.1 200 OK
Date: Sat, 4 Jul 10:22:45 2015 PDT
Last-Modified: Sat, 4 Jul 10:20:53 2015 PDT
Read-TxClock: 1436030565123000
Value-TxClock: 1436030453060001
Server-TxClock: 1436030462593001
Vary: Read-TxClock
Content-Type: application/json
Content-Length: 7

"green"</span>
</pre>

If the value has not changed since that time, the read will return `304 Not Modified`.

<pre class="highlight">
curl -i -w'\n' \
    -H'Condition-TxClock: 1436030453060001' \
    http://localhost:7070/fruit/apple
<span class="go">HTTP/1.1 304 Not Modified
Read-TxClock: 1436030463135001
Server-TxClock: 1436030463135001
Content-Length: 0</span>
</pre>

A client may use this feature to freshen its cache, transferring potentially large JSON objects only when necessary.


## Read from the Past

TreodeDB saves overwritten values back to some configurable point in time. A client may include the `Read-TxClock` header to retrieve past values.

<pre class="highlight">
curl -i -w'\n' \
    -H'Read-TxClock: 1436030359000000' \
    http://localhost:7070/fruit/apple
<span class="go">HTTP/1.1 200 OK
Date: Sat, 4 Jul 10:19:19 2015 PDT
Last-Modified: Sat, 4 Jul 10:19:18 2015 PDT
Read-TxClock: 1436030359000000
Value-TxClock: 1436030358171001
Server-TxClock: 1436030467824001
Vary: Read-TxClock
Content-Type: application/json
Content-Length: 6</span>
</pre>

Without a `Read-TxClock` in the GET request, the operation gets the most recent value.


## Write a Batch

TreodeDB can write multiple rows as an atomic unit, that is all or nothing. It also supports the conditional write on a batch; the write will succeed if none of the rows have been updated since the precondition time.

<pre class="highlight">
curl -i -w'\n' —XPOST d@-\
    -H'Condition-TxClock: 1436030454000000' \
    http://localhost:7070/batch-write &lt;&lt; EOF
[ {"table": "fruit", "key": "apple", "op": "update", "value": "granny smith"},
  {"table": "fruit", "key": "grape", "op": "update", "value": "cabernet"} ]
EOF
<span class="go">HTTP/1.1 200 OK
Value-TxClock: 1436030718013001
Server-TxClock: 1436030718013001
Content-Length: 0</span>
</pre>

An array of JSON objects represents the batch. Each element of the array names the table and key to affect, and an operation to perform on that key. The operation `delete` should be clear enough. The operation, `hold` allows a row to participate in the precondition without effecting a change.

The operations `create` and `update` will both set the key's value. `create` will set it if the key was never written, or if the latest write was a `delete`, even if that `delete` occurred after `Condition-TxClock`. `update` will set the key if it was never written, or if the latest write occurred on or before `Condition-TxClock`. In other words, `create` depends on the existence of the key and not its write time, and `update` depends on its write time rather than its existence.


## Check a Write

Suppose that the client looses the connection to the server while waiting for the response to a write. It happens. The client can reconnect to any server in the cell and learn if the write completed. To do so, the client must include a `Transaction-ID` in the write request:

<pre class="highlight">
curl -i -w'\n' —XPOST -d@-\
    -H'Condition-TxClock: 1436030454000000' \
    -H'Transaction-ID: D12322BEF164E7142CAA92642B48ED81' \
    http://localhost:7070/batch-write &lt;&lt; EOF
[ {"table": "fruit", "key": "apple", "op": "update", "value": "jonathan"},
  {"table": "fruit", "key": "grape", "op": "update", "value": "merlot"} ]
EOF
<span class="go">HTTP/1.1 200 OK
Value-TxClock: 1436030745604001
Server-TxClock: 1436030745604001
Content-Length: 0</span>
</pre>

And then include that same ID in the request to check the status.

<pre class="highlight">
curl -i -w'\n' \
    -H'Transaction-ID: D12322BEF164E7142CAA92642B48ED81' \
    http://localhost:7070/tx-status
<span class="go">HTTP/1.1 200 OK
Value-TxClock: 1436030745604001
Server-TxClock: 1436030748765001
Content-Length: 0</span>
</pre>

A `Transaction-ID` must be unique, and the clients are responsible for ensuring this. Choosing a random value of 128 bits or more will likely do the trick, though you could use other methods to make a unique ID. It is encoded into the `Transaction-ID` header as a hexadecimal string.


[get-server]: /getting-started "Getting Started"
