---
layout: page
title: Scanning
order: 4

---

This page walks through the RESTful API for scanning by using `curl`.


## Prerequisites

If you haven&#700;t already, [setup a server][get-server], and then [add some data][read-write].


## Scan the Latest

To scan the most recent values of the whole table, GET just `/<table>`.

<pre class="highlight">
curl -i -w'\n' http://localhost:7070/fruit
<span class="go">HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked

[ {"key": "apple", "time": 1436030718013001, "value": "granny smith"},
  {"key": "grape", "time": 1436030718013001, "value": "cabernet"} ]</span>
</pre>


## Scan a Snapshot

You can scan a past snapshot of the table:

<pre class="highlight">
curl -i -w'\n' http://localhost:7070/fruit?until=1436030359000000
<span class="go">HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked

[ {"key": "apple", "time": 1436030358171001, "value": "sour"} ]</span>
</pre>

TreodeDB keeps snapshots to midnight of the previous day.


## Scan a Duration

You can see all changes over a window of time:

<pre class="highlight">
curl -i -w'\n' 'http://localhost:7070/fruit?since=1436030359000000&until=1436030719000000&pick=between'
<span class="go">HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked

[ {"key": "apple", "time": 1436030718013001, "value": "granny smith"},
  {"key": "apple", "time": 1436030453060001, "value": "green"},
  {"key": "grape", "time": 1436030718013001, "value": "cabernet"} ]</span>
</pre>

The `since` parameter defaults to time 0. The `until` parameter defaults the current time. And the `pick` parameter defaults to `latest`.

TreodeDB keeps historical writes to midnight of yesterday. TreodeDB always keeps the latest write, even if that occurred prior midnight of yesterday. But overwritten writes it keeps only until midnight of yesterday.


## Scan a Range

You can limit the scan to a range of keys.

<pre class="highlight">
curl -i -w'\n' 'http://localhost:7070/fruit?start=a&end=d'
<span class="go">HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked

[ {"key": "apple", "time": 1436030718013001, "value": "granny smith"} ]</span>

curl -i -w'\n' 'http://localhost:7070/fruit?start=e&end=h'
<span class="go">HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked

[ {"key": "grape", "time": 1436030718013001, "value": "cabernet"} ]</span>
</pre>


## Scan a Page

You can limit the scan to a count of items. We have very few items in this database, so we will use a very low limit to show how TreodeDB reveals the next page of the scan.

<pre class="highlight">
curl -i -w'\n' http://localhost:7070/fruit?limit=1
<span class="go">HTTP/1.1 200 OK
Link: &lt;http://localhost:7070/fruit?start=grape&time=1436744628483001&limit=1&gt;; rel="next"
Content-Type: application/json
Content-Length: 64

[ {"key": "apple", "time": 1436030718013001, "value": "granny smith"} ]</span>
</pre>

TreodeDB provides the URL for the next request in the `Link` header.


[get-server]: /getting-started "Getting Started"

[read-write]: /read-write "Read & Write"
