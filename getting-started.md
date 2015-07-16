---
layout: page
title: Getting Started
order: 2
redirect_from: /store/getting-started.html
---

## Download the Server

Retrieve the [server.jar][server-jar].

```sh
curl https://oss.treode.com/jars/{{site.vers}}/server-{{site.vers}}.jar -o server.jar
```

Throughout the discussions, we&#700;ll create database files. To keep it tidy, you may want to make a temporary directory and move the jar there.

## Initialize the Database

Next, initialize the database file, which we&#700;ll call `store.3kv`.

<pre class="highlight">
java -jar server.jar init -host 0xF47F4AA7602F3857 -cell 0x3B69376FF6CE2141 store.3kv
<span class="go">Jul 04, 2015 10:17:18 AM com.treode.disk.DiskEvents changedDisks
INFO: Attached disks: "store.3kv"</span>
</pre>

We'll discuss the flags `-host` and `-cell` in [managing peers][manage-peers]. We&#700;ll say more about `init` in [managing disks][manage-disks].

## Run the Server

Now start the server.

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

We&#700;ll discuss the `-solo` flag in [managing peers][manage-peers], and the `serve` command in [managing disks][manage-disks].


## Upload a Schema

Each table is identified by a 64 bit number, but we want to use symbolic names in our URLs. The schema maps names to IDs.

```
table <name> {
    id: <number>;
};

...more tables...
```

Upload the schema by performing an HTTP PUT to `/admin/treode/schema`.

<pre class="highlight">
curl -i -w'\n' -XPUT -d@- http://localhost:9990/admin/treode/schema &lt;&lt; EOF
table fruit {
    id: 1;
};
EOF
<span class="go">HTTP/1.1 200 OK
Content-Length: 0</span>
</pre>

We will use the `fruit` table in the examples for [read and write][read-write], and [scan][scan].


## Get in Touch

If you have any troubles, we are here to help.

- [Online Forum][online-forum] on Discourse
- \#treode on StackOverflow:
  [Browse questions asked][stackoverflow-read] or [post a new one][stackoverflow-ask]
- In person, we accept two forms of payment: margaritas and spicy food.



[manage-disks]: /managing-disks "Managing Disks"

[manage-peers]: /managing-peers "Managing Peers"

[online-forum]: https://forum.treode.com "Forum for Treode Users and Developers"

[read-write]: /read-write "Read & Write"

[scan]: /scan "Scanning"

[server-jar]: https://oss.treode.com/jars/{{site.vers}}/server-{{site.vers}}.jar

[stackoverflow-read]: http://stackoverflow.com/questions/tagged/treode "Read questions on Stack Overflow tagged with treode"

[stackoverflow-ask]: http://stackoverflow.com/questions/ask?tags=treode "Post a question on Stack Overflow tagged with treode"
