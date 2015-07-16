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

You can scan a past snapshot of the table:

<pre class="highlight">
curl -i -w'\n' http://localhost:7070/fruit?until=1436030359000000
<span class="go">HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked

[ {"key": "apple", "time": 1436030358171001, "value": "sour"} ]</span>
</pre>

And you can see all changes over a window of time:

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


[get-server]: /getting-started "Getting Started"

[read-write]: /read-write "Read & Write"