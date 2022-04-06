---
title: "Postgres and Elasticsearch Realtime Sync"
date: 2021-11-22T14:11:47+01:00
summary: "The power of sequential WAL decoding."
slug: intro-into-logical-decoding
tags:
- Postgres
- Elasticsearch
- WAL
- Logical Decoding
categories:
- Logical Decoding
---

### TL;DR; Yes, it works.
**Proof: [SearchReplica](https://github.com/pg2es/search-replica)**
You can stream changes from Postgres, and injest the data in any external database, like regular postgres replica does. Using Postgres replication functionality to keep data consistent, and almost realtime.

This would be start of series of posts, regarding pitfalls, mistakes and details of logical decoding.
ToC: TBD;
- [What is WAL, Slot and Logical Decoding]({{< ref "posts/what-is-wal.md" >}})
- [Replication flow]({{< ref "posts/replication-flow.md" >}})

You work with Postgress as database. Stream WAL (with logical decoding) into Elasticsearch. And use Elastic for search.



