---
title: "What is Postgres Write-Ahead Log"
slug: wtf-is-wal
date: 2022-04-06T20:29:11+02:00
tags:
- Postgres
- WAL
- Logical Decoding
categories:
- Logical Decoding
---

### Write-Ahead Log [docs](https://www.postgresql.org/docs/current/wal-intro.html)
As it states in name, it's log that's written BEFORE any actual changes are made. 

The most important thing, **WAL can be replayed** and will always give same result. 
Being compact binary representation of operations and minimal data used in them, gives us few functions:

- Ability to restore after crash
- Replicate data by transfering log segments and replaying them in replica nodes.
- Replacing multiple random writes with single sequential write.

It can however use all of your disk space, quite fast. Serialized transactions use way more disk space then result of execution. So log files, which were applied and fsync-ed to disk, are beeing garbage collected asap.

Quite a lot of benefits for such a simple concept. But it doesn't end here.

### Streaming (bonus)
Since Postgres 9.0, implemented Streaming Replication allows to continiosly stream WAL (via `wal_sender`) in real time to multiple replicas. Now read replicas (aka standby) could be consistent in sync, near to realtime consistent in async mode, using `wal_receiver`.

But since we are streaming files, how to know where to re-start, after crash? 
How to make sure logs are not deleted before replication is successful?

### Replicaiton Slots
Oversimplified: Slot is a position at which specific replica is replaying WAL is atm, which also keeps Primary WAL from beeing deleted. [attributes](https://www.postgresql.org/docs/current/view-pg-replication-slots.html) [Doc](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION-SLOTS)

Consists of written, flushed and applied positions.

### Logical decoding (bonus)
Data in WAL is in postgres specific binary format, never meant to be used externaly, untill recently.
There are few reasons why would you need to decode that data:

- Migration / Replicaiton between different non-compatible versions.
- Sync / Replication between different databases or services.
- Multimaster setups, where data is replicated both ways.

So, concept of Logical Replicaiton was born, where data is decoded into more generic format, which btw can also be replayed. This however requires additional data to be written in WAL, thus one needs to setup
```conf
wal_level=logical
```
