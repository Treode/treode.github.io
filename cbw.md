---
layout: page
title: Conditional Batch Write
order: 8

---

Reading about the CAP theorem, eventually consistent databases and transactional ones, it sometimes feels like I'm being asked to choose between one or the other, but we can integrate the two approaches. We can extend optimistic transactions through eventually consistent caches. The key lies in understanding the technique, and building data storage that supports it.

_This originally appeared as a blog entry titled "Eventual Consistency and Transactions Working Together." We could no longer support that blog, so we copied this key article here._


## The Database Contract

Let's start with the storage layer, with what our database will do. We combine [optimistic concurrency control][wikipedia-occ] (OCC), [multiversion concurrency control][wikipedia-mvcc] (MVCC) and [Lamport clocks][wikipedia-lamport-clocks], and we call this combination the Conditional Batch Write. We expose an interface that looks like a hash table, with some twists: the interface has timestamps, and it provides a batch write.

$$
\begin{array}{l}
\textit{// Read key as of time $T_R$; return value at $T_V \leq T_R$} \\\
\textit{// Arrange for future write to be at time $T_W > T_R$} \\\
\textsf{read}\, (T_R, table, key) \rightarrow (value, T_V) \\\
\textrm{} \\\
\textit{// The various write operations} \\\
\begin{array}{rl}
Op \: := & \textsf{Create}\, (table, key, value) \\\
| & \textsf{Hold}\, (table, key) \\\
| & \textsf{Update}\, (table, key, value) \\\
| & \textsf{Delete}\, (table, key) \\\
\end{array} \\\
\textit{// Write multiple rows if all remain unchanged since time $T_C$} \\\
\textsf{write}\, (T_C, ops) \rightarrow T_W
\end{array}
$$

The read function allows us to read the database as of some point in time, \\(T_R\\). That might be now, or it might be in the past. We reject a time in the future---though we might allow a few seconds leeway for clock skew. When we read as of some time, the database returns the value written most recently before that time---that's multiversion concurrency control. Underneath the covers, read arranges that any update to that row (key-value pair) will occur at a timestamp strictly greater than the one we used for reading---that's the Lamport clock.

The write function allows us to update the database in an atomic batch. That means all the rows will be updated, or none of them; the write operation never applies just some of the updates. The write operation needs a condition timestamp \\(T_C\\), and it will apply the batch only if all those rows are unchanged since \\(T_C\\)---that's optimistic concurrency control. It will write the updated rows with a new timestamp \\(T_W\\) that is strictly greater than any timestamp previously used to read those rows---again, that's the Lamport clock.


## A Transaction with OMVCC

What does this API give us? The quintessential example of a transaction is to transfer money from one account to another, so let's express that using this API. Let's transfer \$100 from Acct1 to Acct2, if Acct1 has at least \$100.

$$
\begin{array}{l}
\textsf{retry}\; \{ \\\
\quad T_R \leftarrow \textsf{Now}\, \\\
\quad (a, T_a) \leftarrow \textsf{Read}\, (T_R, \textsf{Accts}, 1) \\\
\quad (b, T_b) \leftarrow \textsf{Read}\, (T_R, \textsf{Accts}, 2) \\\
\quad \textsf{Check}\; a \geq 100 \\\
\quad T_W \leftarrow \textsf{Write}\, ( \\\
\quad \qquad \textsf{max}\, (T_a, T_b), \\\
\quad \qquad \textsf{Update}\, (\textsf{Accts}, 1, a - 100), \\\
\quad \qquad \textsf{Update}\, (\textsf{Accts}, 2, b + 100)) \\\
\}\; \textsf{while stale}
\end{array}
$$

We read the two balances for the bank accounts using the same timestamp \\(T_R\\), and we get the most recent value before or at \\(T_R\\). In other words, the timestamps \\(T_a\\) and \\(T_b\\) are less than or equal to \\(T_R\\). After checking that Acct1 has sufficient funds for the transfer, we execute the transfer as a batch write. We attach the condition that the balances must be unchanged since \\(\textsf{max}\,(T_a, T_b)\\), and if that check passes, then the database executes the two updates.

It's possible that a concurrent transaction updated one of the balances while our transaction was doing its work. Remember that Read arranges for subsequent writes to occur at a timestamp strictly after \\(T_R\\). Thus the concurrent transaction will have written at a timestamp greater than \\(T_R\\), which we will call \\(T_2\\). We have

$$
\begin{array}{rl}
& T_A \leq T_R \\\
& T_B \leq T_R \\\
\therefore & \textsf{max}\,(T_A, T_B) \leq T_R \\\
& T_R < T_2 \\\
\therefore & \textsf{max}\,(T_A, T_B) < T_2
\end{array}
$$

If a concurrent transaction has updated one of the balances, then the write condition will fail. The first transaction can resolve this by starting over. It can reread the balances to get the freshest values, and then proceed with its checks and modifications a second time---this is optimistic concurrency control.


## Preserving an Invariant

Let's consider a snapshot of time. In fact, let's consider a sequence of snapshots. Suppose we have five accounts, and each opens with \$100 at time \\(T=1\\), so we have \$500 total across all the accounts. When money is transferred from one account to another, that total remains the same. For example, if we transfer \$7 from Acct1 to Acct4, the \$7 debit balances the \$7 credit, and the total is still \$500. We could even witness two transfers happen simultaneously, and this would be true.

$$
\begin{array}{rrrrrl}
\textsf{Acct1} & \textsf{Acct2} & \textsf{Acct3} & \textsf{Acct4} & \textsf{Acct5} & \\\
$100 \;@ 1 & $100 \;@ 1 & $100 \;@ 1 & $100 \;@ 1 & $100 \;@ 1 & \textit{// Opening total \$500} \\\
$93 \;@ 2 & $100 \;@ 1 & $100 \;@ 1 & $107 \;@ 2 & $100 \;@ 1 & \textit{// \$7 from 1 to 4} \\\
$102 \;@ 3 & $88 \;@ 3 & $91 \;@ 3 & $119 \;@ 3 & $100 \;@ 1 & \textit{// \$12 from 2 to 4, \$9 from 3 to 1} \\\
$102 \;@ 3 & $103 \;@ 4 & $91 \;@ 3 & $119 \;@ 3 & $85 \;@ 4 & \textit{// \$15 from 5 to 2} \\\
$102 \;@ 3 & $103 \;@ 4 & $94 \;@ 5 & $119 \;@ 3 & $82 \;@ 5 & \textit{// \$3 from 5 to 3}
\end{array}
$$

We could choose a read timestamp \\(T_R\\), read the latest balance on or before that timestamp for all accounts, total our numbers and get \$500. We could do this for any read timestamp. Note, the C in ACID stands for consistency, and it means what we have just demonstrated: invariants are maintained. The C in CAP also stands for consistency, but in that context it means something like "replicas are synchronized". Beware of overloaded terms.


## A Cache with OMVCC

So far we have explored transactions, so where's the eventual consistency? Get ready.

We need the cache to track a key, its value and some bookkeeping. That bookkeeping needs to include the timestamp \\(T_v\\) from the value, and the timestamp \\(T_R\\) that we used to read the value. If we had used \\(T=4\\) to read Acct4 and Acct5, and then later we had used \\(T=6\\) to read Acct1 and Acct2, then our cache would have:

$$
\begin{array}{llll}
\textsf{Table:Key} & \textsf{Value} & T_v & T_r \\\
\textsf{Acct}:1 & $102 & 3 & 6 \\\
\textsf{Acct}:2 & $103 & 4 & 6 \\\
\textsf{Acct}:4 & $119 & 3 & 4 \\\
\textsf{Acct}:5 & $85 & 4 & 4
\end{array}
$$

In the example transaction earlier, we chose a timestamp for \\(T_R\\) and used it for all reads. Then we found the maximum of the value timestamps, and we used that as condition timestamp \\(T_C\\) for the write. This is approach is simple to implement and manage, but we have more flexibility at our disposal, and that comes in handy here.

We need to compute the maximum of the value timestamps, and the minimum of the read timestamps across our cache. We'll call these \\(\textsf{max}\, (\overline{T_v})\\) and \\(\textsf{min}\, (\overline{T_r})\\), respectively. If \\(\textsf{max}\, (\overline{T_v}) \leq \textsf{min}\, (\overline{T_r})\\) then we have a consistent snapshot. It may be a partial snaspshot; the source database may have more data. It may be a stale snapshot; the source database may have new data. Nonetheless, our cache is a consistent snapshot---that is, using the ACID "invariants are maintained" meaning of consistent.

For our example cache above, \\(\textsf{max}\, (\overline{T_v}) = 4\\) and \\(\textsf{min}\, (\overline{T_r}) = 4\\). Since \\(4 \leq 4\\), our cache above is a consistent snapshot. In fact, compare the data of our cache against the fourth snapshot from the previous section. The values are the same. This happened even though we read them at different times. When we read Acct1 and Acct2 at time 6, we got lucky that neither had been updated since time 4, and we were able to use a simple rule to determine that we got lucky like that.

Since we have a consistent snapshot, we can attempt an optimistic write using the cached data. If our transfer involves Acct5, it will fail because our cache has stale data for Acct5, but the database API and client are ready for that case, being they were written with optimistic concurrency control in mind. On the other hand, if our transfer does not involve the stale data of Acct5, then it may succeed, and it will have done so without returning to the database for the read. It will have contacted the database only for the write.

## Putting This into Practice

To use OMVCC in practice, we need a database that serves the API, and we need caches that track the timestamps.

For the database, we designed [TreodeDB][treodedb] this way. TreodeDB is open-source. It implements the OMVCC API. It can be sharded to scale to petabytes. It can be replicated to tolerate failures of individual machines, or racks, or datacenters. If you replicate Treode across multiple datacenters, the latency can be high (1 to 2 seconds), but that's alleviated by the caches. TreodeDB is in its early stages, so there's work to be done, but the prototype is there.

For the caches, the [HTTP Spec][http] nearly has this already. That's because HTTP was designed with optimistic concurrency control in mind. We need to bend the rules slightly, because HTTP was not designed with batch updates in mind. Nonetheless, we can find ways to make this work with existing CDNs. Moreover, if the technique becomes widespread, the HTTP Spec and CDNs will adapt.

You can have transactions, which can simplify application development. You can have eventual consistency, which can serve data faster and more reliably than a transactional database. You can marry the two in a clean and simple way. We invite you to have a look at [TreodeDB][treodedb] and to ponder how this might help your next application. We also invite you to come back to our blog, as we have more to say.



[http]: http://tools.ietf.org/html/rfc7232 "HTTP/1.1: Conditional Requests"

[treodedb]: https://github.com/Treode/store "TreodeDB on GitHub"

[wikipedia-mvcc]: http://en.wikipedia.org/wiki/Multiversion_concurrency_control "Multiversion Concurrency Control"

[wikipedia-occ]: http://en.wikipedia.org/wiki/Optimistic_concurrency_control "Optimistic concurrency control"

[wikipedia-lamport-clocks]: http://en.wikipedia.org/wiki/Lamport_timestamps "Lamport Timestamps"
