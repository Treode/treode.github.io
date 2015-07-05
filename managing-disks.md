---
layout: page
title: Managing Disks
order: 4
redirect_from: /store/managing-disks.html

---

TreodeDB divides creating a new database and opening it into two distinct
steps.

<pre class="highlight">
<span class="c"># Create or overwrite a database.</span>
java -jar server.jar init -host 0xF47F4AA7602F3857 -cell 0x3B69376FF6CE2141 store.3kv

<span class="c"># Open an existing database and serve it.</span>
java -jar server.jar serve -solo store.3kv
</pre>

These are two separate steps since often an install script will initialize the database, and an `/etc/rc.d` script will start the server.

## Creating a new database

You may supply multiple paths for `init`.  Those paths may name files or raw devices;
either way you must specify a disk geometry.  Although disks come in different sizes and block sizes, those provided to `init` will be treated with the same geometry.  You can add disks with different characteristics later.

TreodeDB allocates and frees disk space in segments.  The `-segmentBits` argument specifies the segments size of (2^segmentBits) bytes.  The system will cap its allocation according to `-diskBytes` which gives the total byte size of the disk; it does not need to be a whole multiple of the segment size.  TreodeDB will write the log and pages aligned per `-blockBits`.  Like most storage systems, the database places superblocks in known locations, and the `superBlockBits` arguments allows the superblock to span multiple disk blocks.

The `hostId` and `cellId` identify this peer and the storage cell, where a cell is one or more servers cooperating to provide the same database.  The `hostId` must be unique for every host in the cell, and the `cellId` must be the same across all hosts in the cell.  If you are running multiple cells, you should consider making a unique `cellId` for each one.  A peer will reject connections from another peer if the `cellId` does not match, and this can prevent corruption of data.  Both arguments are 64 bit values.  We aren't very imaginative, so when we need a new `hostId` or `cellId`, we invoke this magic incantation:

    head -c 8 /dev/random | hexdump -e "\"0x\" 8/1 \"%02X\" \"\n\""

You must provide a `hostId` and `cellId`, even when starting a server to run stand alone.  Your business is going to be booming someday, and you'll be glad that you prepared early for running a large cluster.

So let's create a database.

<pre class="highlight">
java -jar server.jar init -host 0xF47F4AA7602F3857 -cell 0x3B69376FF6CE2141 store.3kv
<span class="go">Jul 04, 2015 10:17:18 AM com.treode.disk.DiskEvents changedDisks
INFO: Attached disks: "store.3kv"</span>
</pre>

## Opening an existing database

You may also supply multiple paths to the `serve` method, however it is only necessary to supply one.  All disks attached to a server are listed in the superblock on all disks, so TreodeDB can find all disks when given any one of them.  Without this trick, adding and removing disks would require you to synchronize your startup scripts (or configuration files) with the database&#700;s commands to change the disks.

You also supply two related socket addresses.  The `-bindAddr` is the address on which the server will listen for connections from peers in the storage server, and it often does not specify the host explicitly.  The `-shareAddr` is the address other peers use to connect to this one, and it must be given a concrete host.  In many cases, one would bind on `*:port` and share `hostname:port`, that is one would bind and share the same port number.  In other more specialized cases, one might setup IP chains or other network configuration that would require the two port numbers to differ.

Now let&#700;s open the database we created earlier.

<pre class="highlight">
java -jar server.jar serve -solo store.3kv
<span class="go">I 0703 14:57:51.775 THREAD1: /admin => com.twitter.server.handler.SummaryHandler
&hellip;
I 0704 17:17:20.039 THREAD1: Serving admin http on 0.0.0.0/0.0.0.0:9990
I 0704 17:17:20.054 THREAD1: Finagle version 6.26.0 (rev=f8ea987f8da7dbe34a4fe1cb481446b5a0d34b56) built at 20150625-094005
I 0704 17:17:20.449 THREAD32: Reattaching disks: "store.3kv"
I 0704 17:17:20.633 THREAD32: Accepting peer connections to Host:F47F4AA7602F3857 for Cell:3B69376FF6CE2141 on 0.0.0.0/0.0.0.0:6278
I 0704 17:17:20.886 THREAD1: Tracer: com.twitter.finagle.zipkin.thrift.SamplingTracer</span>
</pre>

## Adding a drive

As your database grows, you will want to add a disk for more space.

<pre class="highlight">
curl -i -w'\n' -XPOST -d@- \
    http://localhost:9990/admin/treode/drives &lt;&lt; EOF
{ "attach":
  [ { "path": "store2.3kv",
      "geometry": {"segmentBits": 30, "blockBits": 13, "diskBytes": 1099511627776}
    } ] }
EOF
<span class="go">HTTP/1.1 200 OK
Content-Length: 0</span>
</pre>

You can list the drives to check that the new disk is included:

<pre class="highlight">
curl -i -w'\n' http://localhost:9990/admin/treode/drives
<span class="go">HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 246

[ { "path": "store2.3kv",
    "geometry": {"segmentBits":30, "blockBits":13, "diskBytes":1099511627776},
    "allocated": 2,
    "draining": false }, 
  { "path": "store.3kv",
    "geometry": {"segmentBits":30, "blockBits":13, "diskBytes":1099511627776},
    "allocated": 2,
    "draining": false } ]</span>
</pre>

## Removing a drive

There will be times when you want to remove a disk too.  Perhaps it's getting old and you want tonproactively replace it before it yields read errors.  Or maybe it has become too small and you want to swap it for a larger one.

<pre class="highlight">
curl -i -w'\n' -XPOST -d@- \
    http://localhost:9990/admin/treode/drives  &lt;&lt; EOF
{ "drain": ["store.3kv"] }
EOF
<span class="go">HTTP/1.1 200 OK
Content-Length: 0</span>
</pre>

Before draining a disk, you should check your startup scripts first (or configuration files, or wherever you list the paths fed to `serve`).  If that includes a disk you want to drain, remove the path from there first, then issue the `drain` command. When TreodeDB has moved all data from the old disk to others, it will update the list of paths in the superblocks.  This way, you don't need to precisely synchronize your scripts and configuration with TreodeDB detaching drives.


If you have lots of data on the disk, you&#700;ll see it listed as draining until it's finally detached:

<pre class="highlight">
curl -i -w'\n' http://localhost:9990/admin/treode/drives
<span class="go">HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 245

[ { "path": "store2.3kv",
    "geometry": {"segmentBits":30, "blockBits":13, "diskBytes":1099511627776},
    "allocated": 2,
    "draining": true
  }, 
  { "path": "store.3kv",
    "geometry": {"segmentBits":30, "blockBits":13, "diskBytes":1099511627776},
    "allocated":3,
    "draining": false
  } ]</span>
</pre>

After only this short tutorial though, you probably have little or no data on the disk, so TreodeDB will detach it quickly.  You will have seen a log message announcing the detachment, and you will see the disk missing in the listing.

## Next

That's all there is to adding and removing drives.  Move on to [manage peers][manage-peers].

[manage-peers]: /managing-peers "Managing Peers"
