---
layout: post
title:  "Six Months of AWS Aurora"
date: 2017-11-25 21:00:00 -0800
categories: engineer
---

AWS [Aurora](https://aws.amazon.com/rds/aurora/), as AWS describes, is a MySQL and PostgreSQL compatible relational database built for the cloud.

We moved one of our largest database to AWS Aurora in April 2017 and experienced a series of interesting events with it. So I think it's time for me to write something about it, (right before reInvent 2017!!).

Note this post will be focus on AWS Aurora MySQL, I haven't used Aurora PostgreSQL yet.

First of all, I have to say that I really like the design of Aurora, it innovatively replacing the storage layer of MySQL with a multi-tenant scale-out storage layer and did lots of other smart things which you can find out more from this [paper](http://www.allthingsdistributed.com/files/p1041-verbitski.pdf) so I won't spend too much time there.

Although Aurora implements its storage layer in a distributed fault tolerant fashion, it still behaves like your favourite MySQL database and it is a perfect drop-in scalable replacement for your MySQL stores.

Aurora still only has one writer but you can add up to 15 readers to one single cluster, because storage layer is shared between writer and readers, the replica lag is minimized in most of the cases :-)

Those are enough good words, what I really want to share in this post are the issues we met with Aurora, some of them got solved eventually but some of them are ongoing.

## Metadata Corruption

This happened short after we moved to Aurora and were doing our first Aurora maintenance - upgrade Aurora version.

And tragically we found out, after we shutdown Aurora to restart it, it couldn't standup anymore!

It took AWS and us quite some time to find out, this tragic was due to an ALTER TABLE statements we did before that corrupted the metadata written to the storage layer. The cluster was healthy before the restart because Aurora caches everything in its memory and didn't load the corrupted metadata until a restart was initiated.

It is really an edge case but we are unfortunate enough to hit. AWS later fixed this in Aurora [1.15](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/AuroraMySQL.Updates.20171024.html), the fix is described as
```
Fixed InnoDB metadata inconsistency during ALTER TABLE RENAME query which improved stability. Example: When columns of table t1(c1, c2) are renamed cyclically to t1(c2,c3) within the same ALTER statement.
```

## Replica Restarts

As I mentioned and Auora proud of itself, writer and readers are sharing one distributed storage layer and writer is only passing page caches to readers which significantly reduces the IO between instances thus Aurora can provide much better write and read performance than traditional MySQL.

This characteristic of Aurora gives the readers an advantage that when they see a hugh replication lag, they can restart to recover. Because no data is passed from master, everything is in the storage layer, if a reader ever sees a 100 seconds replication lag, there must be something wrong by itself and the quickest way to recover itself, according to Aurora, is restart...

This reader auto recovery mechanism has been torturing us. When our writer workload went really high, readers restart because it couldn't tell if it's something wrong in themselves or it's because writer failed to pass the freaking page caches in time. To be honest, neither do we.

Because AWS couldn't figure out why their readers keep restarting, our services have to be more fault tolerant. So queries will be retried automatically when we found high replication lag and receive an connection error from Aurora. Circuit breaker wraps the connection so it won't cause catastrophic damages, writer needs to slow down a bit when this happens etc.

## Reader Endpoint Load Balancing

Aurora provides a reader endpoint which helps you balance the load to its reader instances, it behaves like a TCP ELB which probably really is what it is. It works very well in the ideal cases, load came, load balanced, load returned. Easy job, isn't it?

But the fact that readers can restart broke the idealism. MySQL connections are persistent and clearly reader endpoing (ELB) balances the workload in a request based basis, so it will not dispatch requests to the newly restarted instances for quite sometime because clients still hold connections to other instances and it won't create new connections until it reaches the limit of the connection pool so no load can actually be balanced by the ELB!

So we have to force timeout Aurora connections from client side periodically, basically refresh the connection to reader instances to ensure the load will be balanced.

If you are not familiar with it, Go's `sql` package has a function `SetConnMaxLifetime` could help you with it.

## Query Performance

OK, this is something really interesting. The tables we have in Aurora are quite large and that's the main reason we moved from RDS MySQL to Aurora, however, we haven't deleted the old table in RDS MySQL yet just for the sake of safety guard. And we have a particular query pattern which behaves very well in RDS MySQL but poorly in Aurora which is quite surprising because Aurora claims it has much better performance and in most of the cases it does.

I can't share too much about the query itself here but it contains an IN clause and a time range constraint, for some reasons Aurora keeps choosing the wrong index for this query and we are planning to modify the value of `innodb_persistent_stats_sample_pages` then see if it helps Aurora to figure better indexes, this is still a work in progress and I will try to share the optimization results later in the blog.


In summary, Aurora is good product and I think we will keep using it, I just hope AWS will give enough and long-lasting support to it because it is still a young database and needs lots of improvements.

And finally, I hope AWS will announce Aurora launching in Singapore during reInvent, we have been using it in Mumbai from Singapore for quite some time and the latency penalty is not just a bit :-(
