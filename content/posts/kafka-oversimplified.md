---
title: "Kafka Oversimplified Concepts"
date: 2022-04-16T14:41:02+02:00
summary: "Important concepts in simplified manner, so you know what to expect"
tags:
- Kafka
categories:
- Kafka
ShowToc: true
TocOpen: true
---

> Unfinished article


{{< figure
	src="/media/kafka/queue_logo.png#center"
	alt="Queue, like any other queue in a world"
	height="250px"
>}}
So, Kafka Topic is a queue... Kinda queue... More like distributed set of queues.  
**Distributed set of queues with routing and some consistency guarantees.**


### Partitioning
In **Kafka** terms, partition is separate queue.
Then you have Topic, which is a group of partitions.

That predefined number of partitions is spread across nodes.
{{< figure
	src="/media/kafka/partitions_5_3.png"
	alt="5 Kafka partitions across 3 nodes"
	title="5 partitions across 3 nodes"
>}}
Real number of partitions is usually way higher, to allow even load balancing.



How do we know the partition for each specific message?


{{< figure
	src="/media/kafka/partitioning.png#center"
	height="200px"
	alt="Queue, like any other queue in a world"
>}}
Each message is sent to random (or specificly selected) partition.

### Log Segments and Offsets
Each partition is stored as append-only immutable log file.  
New messages are appended, replicated, and read by offsets. Old log files are deleted later.  
{{< figure

	src="/media/kafka/partitions_log_offsets.png"
	alt="Queue, like any other queue in a world"
	height="200px"
>}}
**Offsets**:
1. **Current** position, where new messages are written;
2. "**HighWatermark**" position that was replicated (if needed);
3. **Read** position
> **Here** your custom software received, process, and commits messages
4. **Commit** position

old log file is deleted after retention window


**Once again, Kafka creates log files for each partition separately, so**
{{< figure

	src="/media/kafka/partitions_3_log_offsets.png"
	alt="Queue, like any other queue in a world"
	height="200px"
>}}



### Message Order
Understanding internals, one may ask how does Kafka keeps topic message ordering?  
{{< figure
	src="/media/kafka/queue_arrow.png#center"
	alt="Queue, like any other queue in a world"
	height="150px"
>}}
**It does not. Messages are ordered ONLY within a partition**. If you need them ordered, make sure they appear in same partition. You can set it manually, but it's easier to use message key.
```python
partition = hash(key) % len(patitions)
```
Usually `user_id`, `transaction_id`, `content_id` or suchlike fields are set as message key.



### Consistency

Usually you have to choose between duplicated messages (at least once) or lost messages (at most once)
{{< figure
	src="/media/kafka/consistency_tradeoff.png#center"
	alt="There is no magic, you either loose message or get it duplicated"
	title="There is no magic, you either loose message or get it duplicated"
	height="200px"
>}}

Kafka claims to support "Exactly once" mode, via two phase commit, but that's a separate topic of discussion.









