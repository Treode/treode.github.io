---
layout: page
title: Getting Started
order: 2

---

## An Example Server

These pages walk through building and operating a RESTful service with TreodeDB.  We have built an
example application using [the Scala Programming Language][scala], [Twitter's Finagle]
[twitter-finagle] and the [Twitter Server][twitter-server].  The walkthroughs use a single `.jar`,
which you can download or build yourself.


### Downloading `server.jar`

Retrieve the prebuilt [server.jar][server-jar]. That was easy.

Throughout the discussions, we'll create database files. To keep it tidy, you may want to make a
temporary directory and move the downloaded jar there. Then proceed to the
[first walk through][rws].

### Building `server.jar`

You will need [SBT][sbt]; [install it][sbt-install] if you haven't already.

Clone the [TreodeDB respository][store-github] and then build the assembly. The example is in the
`examples/finatra` directory, separate from the main code. It even has its own build file.

<pre class="highlight">
git clone git@github.com:Treode/store.git
cd store/examples/finatra
sbt assembly
<div class="go">[info] Packaging /Users/topher/src/store/examples/finatra/target/scala-2.10/server.jar ...
[info] Done packaging.</div>
</pre>

Throughout the discussions, we'll create database files. To keep it tidy, you may want to make a
temporary directory and link the built jar there. Then proceed to the [first walk through][rws].


## Building Your Own Server

While learning about the example server, you may wish we had made different choices about the data
model and service protocol. You may feel an urge to build your own server. We encourage you to act
on this.

The libraries for TreodeDB are available Scala 2.10 and 2.11. Notice the uncommon notation for the
dependency; this gives you stubs for testing.

```
resolvers += Resolver.url (
    "treode-oss",
    new URL ("https://oss.treode.com/ivy")) (Resolver.ivyStylePatterns)

libraryDependencies +=
    "com.treode" %% "store" % "{{site.vers}}" % "compile;test->stubs"
```


## Getting in Touch

If you have any troubles, we are here to help.

- [Online Forum][online-forum] on Discourse
- [Chat Room][online-chat] on HipChat
- \#treode on StackOverflow:
  [Browse questions asked][stackoverflow-read] or [post a new one][stackoverflow-ask]
- In person: Buy one of us a margarita.



[finatra]: //finatra.info/ "Finatra"

[online-chat]: http://www.hipchat.com/giwb5oIkz "Chat Room for Treode Users and Developers"

[online-forum]: https://forum.treode.com "Forum for Treode Users and Developers"

[rws]: /read-write-scan "Read,Write, Scan"

[sbt]: //www.scala-sbt.org/ "Simple Build Tool"

[sbt-install]: //www.scala-sbt.org/0.13/tutorial/Setup.html "Install SBT"

[scala]: //www.scala-lang.org/ "The Scala Programming Language"

[server-jar]: https://oss.treode.com/examples/finagle/{{site.vers}}/finagle-server-{{site.vers}}.jar

[stackoverflow-read]: http://stackoverflow.com/questions/tagged/treode "Read questions on Stack Overflow tagged with treode"

[stackoverflow-ask]: http://stackoverflow.com/questions/ask?tags=treode "Post a question on Stack Overflow tagged with treode"

[store-github]: https://github.com/Treode/store "TreodeDB on GitHub"

[twitter-finagle]: https://twitter.github.io/finagle/ "Twitter's Finagle"

[twitter-server]: http://twitter.github.io/twitter-server/ "Twitter Server"
