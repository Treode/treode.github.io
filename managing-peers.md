---
layout: page
title: Managing Peers
order: 6
redirect_from: /store/managing-peers.html
---

TreodeDB allows you to control the placement of a shard&#700;s replicas. The database locates replicas by hashing the table ID and row key onto an array of shards, and each shard lists the peers that host a copy of the shard&#700;s data. We use the term *peer* to designate a JVM process that is running a TreodeDB server. Each one is usually but not necessarily hosted on its own machine. We may casually call them servers, hosts or machines.

![Atlas][atlas]

There are four shards in this example atlas, and there are six peers in the cell. We have arranged it so that each peer appears in two shards, and no pair appears into two shards, but you need not do it that way. You have considerable freedom as there are only a few constraints: the number of shards must be a power of two, the number of distinct peers in each shard must be odd, and all peers must be able to connect to each other.  There are no other constraints. You may list a peer in any number of shards; one peer may appear with another peer in multiple shards or not; two peers may or may not run on the same machine, rack, bay, colo or datacenter; peers may have different processors, memory, disk and network speed; and so on.

You will face performance and reliability tradeoffs when laying out shards. For example, locating replicas on one rack will speed response time but put all replicas at risk of loosing power or network together. You will face tradeoffs when growing your cluster. For example, you may want to rebalance replicas after adding each machine, or you may want to delay rebalancing them until you have added a whole rack. You already have enough constraints to juggle when managing large clusters, so TreodeDB&#700;s atlas is flexible.

## Hailing Existing Servers

Suppose we've setup one server, launched a whizzy new app, and found that customers took an interest.  We may not have enough load to justify a big cluster yet, but we at least want to our service remain available if one machine crashes, so we will expand to three servers.  First, we start two more servers.  These servers do not need to start out alone; they can receive a warm welcome into the cluster.

<pre class="highlight">
java -jar server.jar init -host 0xE2D69225128DB874 -cell 0x3B69376FF6CE2141 store-host2.3kv
<span class="go">Jul 04, 2015 12:12:07 PM com.treode.disk.DiskEvents changedDisks
INFO: Attached disks: "store-host2.3kv"</span>

java -jar server.jar serve \
    -admin.port=:9991 \
    -httpAddr=:7071 \
    -peerAddr=:6279 \
    -hail=0xF47F4AA7602F3857=localhost:6278 \
    store-host2.3kv
<span class="go">…
I 0704 19:13:53.216 THREAD32: Reattaching disks: "store-host2.3kv"
I 0704 19:13:53.389 THREAD35: Accepting peer connections to Host:E2D69225128DB874 for Cell:3B69376FF6CE2141 on 0.0.0.0/0.0.0.0:6279
I 0704 19:13:53.565 THREAD32: Connected to Host:F47F4AA7602F3857 at /127.0.0.1:55841 : localhost/127.0.0.1:6278
I 0704 19:17:56.417 THREAD32: Connected to Host:4FC3013EE2AE1737 at /127.0.0.1:6279 : localhost/127.0.0.1:55887</span>
</pre>

You supply the host ID and address of several different peers, separated by commas.  The new server will contact those servers and exchange information, including the atlas.  You only need to hail one server that&#700;s up, but more is okay. Generally you would hail several to ensure that at least one of them responds.

As a result of this new server hailing the existing one, it is now looped into all the gossip that flows through the cell.  In particular, it is aware of the atlas.

<pre class="highlight">
curl -w'\n' -i http://localhost:9991/admin/treode/atlas
<span class="go">HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 52

[{"state":"settled","hosts":["0xF47F4AA7602F3857"]}]</span>
</pre>

## All servers are equally functional

This second server can now handle GET and PUT requests too.  Let&#700;s do a PUT on this new server, and then a GET of that value on the original.

<pre class="highlight">
curl -w'\n' -i -XPUT -d@- \
    http://localhost:7071/fruit/grape &lt;&lt; EOF
"merlot"
EOF
<span class="go">HTTP/1.1 200 OK
HTTP/1.1 200 OK
Value-TxClock: 1436037340625001
Content-Length: 0</span>

curl -w'\n' -i http://localhost:7070/fruit/grape
<span class="go">HTTP/1.1 200 OK
Date: Sat, 4 Jul 12:16:01 2015 PDT
Last-Modified: Sat, 4 Jul 12:15:40 2015 PDT
Read-TxClock: 1436037361627000
Value-TxClock: 1436037340625001
Vary: Read-TxClock
Content-Type: application/json
Content-Length: 8

"merlot"</span>
</pre>

In a TreodeDB cell, every host can handle read, write and scan.  Some hosts may be located closer to the replicas.  In small reads and writes, which change a little bit of data for several rows, and thus several shards, and thus many machines, proximity to replicas may be a moot issue.  For scans, which move large amounts of data, perhaps from just one shard, proximity may become more necessary and feasible.  We'll return to this point again later.  The take-away now is: a remote client performing a read or write can connect to any server in the cell.


## Settled, Issuing and Moving

We now have two of our three servers running.  Let&#700;s start the third.

<pre class="highlight">
java -jar server.jar init -host 0x4FC3013EE2AE1737 -cell 0x3B69376FF6CE2141 store-host3.3kv
<span class="go">Jul 04, 2015 12:17:15 PM com.treode.disk.DiskEvents changedDisks
INFO: Attached disks: "store-host3.3kv"</span>

java -jar server.jar serve \
    -admin.port=:9992 \
    -httpAddr=:7072 \
    -peerAddr=:6280 \
    -hail=0xF47F4AA7602F3857=localhost:6278 \
    store-host3.3kv
<span class="go">…
I 0704 19:17:55.882 THREAD33: Reattaching disks: "store-host3.3kv"
I 0704 19:17:56.053 THREAD33: Accepting peer connections to Host:4FC3013EE2AE1737 for Cell:3B69376FF6CE2141 on 0.0.0.0/0.0.0.0:6280
I 0704 19:17:56.228 THREAD36: Connected to Host:F47F4AA7602F3857 at /127.0.0.1:55886 : localhost/127.0.0.1:6278
I 0704 19:17:56.417 THREAD33: Connected to Host:E2D69225128DB874 at /127.0.0.1:55887 : localhost/127.0.0.1:6279</span>
</pre>

The atlas still directs the reads and writes to one replica.  We&#700;ve started three servers to give us tolerance of one failure, but we need to update the atlas before we have that.

<pre class="highlight">
curl -w'\n' -i -XPUT -d@- \
    -H'content-type: application/json' \
    http://localhost:9990/admin/treode/atlas &lt;&lt; EOF
[ { "hosts": ["0xF47F4AA7602F3857", "0xE2D69225128DB874", "0x4FC3013EE2AE1737"] } ]
EOF
<span class="go">HTTP/1.1 200 OK
Content-Length: 0</span>
</pre>

The servers in the Treode cell constantly gossip.  We PUT the new atlas on the server at admin port 9990, and if we check the server at 9992.

<pre class="highlight">
curl -w'\n' http://localhost:9992/admin/treode/atlas
<span class="go">[ { "state": "settled",
    "hosts": ["0xF47F4AA7602F3857","0xE2D69225128DB874","0x4FC3013EE2AE1737"]
  } ]</span>
</pre>

What&#700;s `settled`?  Our cluster is small and has little data, so we hardly had a chance to see the atlas move through two other states.  If our cluster was large, like 10,000 machines, we&#700;d have a minute to witness this atlas:

<pre class="highlight">
curl -w'\n' http://localhost:9990/admin/treode/atlas
<span class="go">[ { "state": "issuing",
    "origin": ["0xF47F4AA7602F3857"],
    "target": ["0xF47F4AA7602F3857","0xE2D69225128DB874","0x4FC3013EE2AE1737"]
  } ]</span>
</pre>

During this time, TreodeDB is not yet migrating data from old to new nodes.  It is only enlisting the help of readers and writers in performing the move.  When a quorum of every shard in the atlas (in this case just the one) becomes aware of the move, we will see the shard change state.

<pre class="highlight">
curl -w'\n' http://localhost:9990/admin/treode/atlas
<span class="go">[ { "state": "moving",
    "origin": ["0xF47F4AA7602F3857"],
    "target": ["0xF47F4AA7602F3857","0xE2D69225128DB874","0x4FC3013EE2AE1737"]
  } ]</span>
</pre>

At this time, the original nodes of the shard are sending their data to the new nodes.  When they have completed this process, we will see the shard change state again:

<pre class="highlight">
curl -w'\n' http://localhost:9990/admin/treode/atlas
<span class="go">[ { "state": "settled",
    "hosts": ["0xF47F4AA7602F3857","0xE2D69225128DB874","0x4FC3013EE2AE1737"]
  } ]</span>
</pre>

These state changes happen independently for each shard.  When you want to change a larger atlas, you can change several shards or just one at a time.  It&#700;s up to you.  You can also issue an updated atlas that cancels or changes a move, even when it&#700;s in progress.

## Parallel Scans

Our atlas maps all keys to one shard at the moment. As our database grows, we&#700;ll need to map slices of data to different machines.  By the time our fledgeling business has that much data, it probably has money to afford a large cluster, and we could build an atlas that maps many shards, each to three or five machines.  However, to keep this walkthrough manageable, let&#700;s work with an atlas of two shards, each of one machine.

<pre class="highlight">
curl -w'\n' -i -XPUT -d@- \
    http://localhost:9990/admin/treode/atlas &lt;&lt; EOF
[ { "hosts": ["0xF47F4AA7602F3857"] },
  { "hosts": ["0xE2D69225128DB874"] } ]
EOF
<span class="go">HTTP/1.1 200 OK
Content-Length: 0</span>
</pre>

Also, let&#700;s add a few more rows to the table.

<pre class="highlight">
curl -XPUT -d'"ripe"' http://localhost:7070/fruit/banana
curl -XPUT -d'"tangy"' http://localhost:7070/fruit/kiwi
curl -XPUT -d'"sweet"' http://localhost:7070/fruit/orange
curl -XPUT -d'"firm"' http://localhost:7070/fruit/kiwi
curl -XPUT -d'"watery"' http://localhost:7070/fruit/orange
</pre>

Now we have a few rows to scan.

<pre class="highlight">
curl -w'\n' http://localhost:7070/fruit
<span class="go">[ { "key": "apple", "time": 1403130358674001, "value": "green"},
  { "key": "banana", "time": 1403131260143001, "value": "ripe"},
  { "key": "grape", "time": 1403131390837001, "value": "merlot"},
  { "key": "kiwi", "time": 1403131390935001, "value": "firm"},
  { "key": "orange", "time": 1403131391668001, "value": "watery"} ]</span>
</pre>

If this was a large table, we might want to scan pieces of it in parallel.  For information on the API to do this, see [Reading, Writing and Scanning][rws].  Here we are going to discuss how slices of a table relate to the atlas.

![slices][slices]

In a scan, we can slice the table as much as we want, as long as its a power of two.  Functionally, the number of slices need not be related to the number of shards.

Depending on the size of the table, scanning a slice may require gathering a large amount of data from the peers in a shard.  It can be helpful if at least one of those peers is the local machine, as that will reduce network traffic. The `/peers` endpoint let&#700;s you discover which hosts hold data for the slice.

<pre class="highlight">
curl -w'\n' http://localhost:7070/peers?slice=0\&amp;nslices=2
<span class="go">[ { "weight": 1, "addr": "localhost:7070", "sslAddr": null } ]</span>

curl -w'\n' http://localhost:7070/peers?slice=1\&amp;nslices=2
<span class="go">[ { "weight": 1, "addr": "localhost:7071", "sslAddr": null } ]</span>
</pre>

This tells us that for slice #0 we should probably contact `localhost:7070`, and for slice #1 we should probably contact `localhost:7071`.  If we had multiple hosts per shard, we would see them all listed here.  If our choice for `nslices` was less than the number of shards, then a given slice would include multiple shards, and we would see `weight` count how many shards a given host served.

Knowing this, we can issue scans for slices to appropriate hosts.

<pre class="highlight">
curl -w'\n' http://localhost:7070/fruit?slice=0\&amp;nslices=2
<span class="go">[ { "key": "banana", "time": 1403131260143001, "value": "sweet"},
  { "key": "grape", "time": 1403131390837001, "value": "merlot"},
  { "key": "kiwi", "time": 1403131390935001, "value": "firm"} ]</span>

curl -w'\n' http://localhost:7071/table/0x1?slice=1\&amp;nslices=2
<span class="go">[ { "key": "apple", "time": 1403130358674001, "value": "green"},
  { "key": "orange", "time": 1403131391668001, "value": "watery"} ]</span>
</pre>

## Next

Now that you know how to use TreodeDB, build something.


[atlas]: /img/atlas.png "Atlas"

[managing-disks]: /managing-disks "Managing Disks"

[rws]: /read-write-scan "Reading, Writing and Scanning"

[slices]: /img/slices.png "Slices"

[split-brain]: http://en.wikipedia.org/wiki/Split-brain_(computing) "Split Brain"
