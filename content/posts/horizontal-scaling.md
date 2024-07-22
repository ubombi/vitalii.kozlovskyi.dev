---
title: "Horizontal Scaling"
summary: |
  Scale pragmatically. Understand your data, choose technologies wisely,
  and design for future growth without overengineering. Keep services stateless,
  use NoSQL only when truly needed, and prioritize solving current problems while
  preparing for future scaling. The goal is to delay scaling as long as possible
  while staying ready for when it becomes necessary.
Type: "posts"
tags:
- Scaling
- Kafka
- Cassandra
- Postgres
- SQL
- NoSQL
categories:
- Scaling
date: 2024-07-22T14:48:41+02:00
draft: false
ShowToc: true
---

Software products, especially SaaS, PaaS, or any other \_aaS
have a potential of explosive growth. It does not happen often though,
and most start-ups fail or barely break even.
  
Nowadays, every application is expected to be global. And most of companies
dream of acquiring 100% of their market niche. Which is not a wrong thing to do.

{{< figure
 src="/media/scaling/poster.png"
 alt="Joke about scaling application like scaling image in editor"
 caption="Scaling your product isn't as simple as resizing an image, but it doesn't have to be overwhelming."
>}}

> **TL;DR:** Scale pragmatically. Understand your data, choose technologies wisely,
> and design for future growth without overengineering. Keep services stateless,
> use NoSQL only when truly needed, and prioritize solving current problems while
> preparing for future scaling. The goal is to delay scaling as long as possible
> while staying ready for when it becomes necessary.

## Think big, implement incrementally

Planning a global service, which can handle at least two time of
the whole world population is a good mental exercise, but never a priority.

Avoid premature optimization. Solve current problems efficiently,
not hypothetical future issues. It's not clever to throw money at a problem
that does not even exists yet.

Plan ahead, but focus on solving immediate challenges.

### Be Strategic: Avoid Scaling Pitfalls

> Some architectural decisions can cause significant issues if not carefully considered.
  
{{< figure
 src="/media/scaling/time_bomb.png#center"
 alt="image-icon of dynamite and a clock" height="200px"
>}}
Follow best practices to avoid common pitfalls in scaling.

More pragmatic approach is focusing on current scale and reasonable growth,
while avoiding common mistakes, so we can easily improve it later.

### Design for FUTURE Growth

We can carefully cut most of the corners, and get back to them as needed.
With userbase growth, product evolves in all directions.

{{< figure
 src="/media/scaling/planning.png#center"
 alt="drawing with multiple rakes that person can step on"
 caption="Plan your scaling path carefully to avoid future obstacles."
>}}

Handling a server load may become quite a challenge, but managing teams
is even harder task. And somehow we need to be prepared for both.

## How?

### Know Your Data

Always keep up-to-date relational diagram, for all the information that company
works with. And make sure, it's accessible within and across teams.
This schema is rather informational and not enforceable.

{{< figure
 src="/media/scaling/data_model.png#center"
 alt="drawing of ER diagram, splited into three parts"
 caption=""
>}}

#### Define the Ownership

Each model should have an service/team that owns it. One that is
responsible, for managing this data, and is a source-of truth, for
denormalized values in other parts.

Making this surgical cut _(when needed)_ based on data model is important
for a few reasons:

- Reduces dependencies, making teams and products more autonomous.
- Allows quick DB replacement, without dealing with dependencies.
- Reduces amount fo cross-service communication.
- Defines who is responsible to keep denormalized cache in sync with owner.

#### Understand Consistency Requirements

Assume, that any relation-field can be eventually consistent or even materialized
by default. If not, mark them accordingly.

{{< figure
 src="/media/scaling/product_page.png#center"
 alt="drawing of a product page in web shop"
 caption="allowing purchase of out-of-stock item can cause user frustration and company losses"
>}}
For example, `"product"` page needs consistent `"stock".quantity`. And if we separate
product catalog and warehouse management, this place will require some attention.
At the same time for search or filtering by "in-stock" eventual consistency is
usually enough.

On the other hand, we can assume that `"news"` article `"author".name` can be
denormalized _(materialized)_ in a news table, to avoid unnecessary DB dependency or
cross-service communication in a read-heavy flow.
Fetching it once, during article creation, should be enough.

Difference - how it impacts end user:

- Old author's lastname in year old article? - no one cares
- Once in a lifetime non-available product is shown in search? - barely noticeable
- You bought a product, which is not in stock? - that's moderately annoying

#### Know Your Partition Keys

> **TL;DR:** Make sure each `SELECT`/`UPDATE` has at least one field with WHERE closure,
> like `foo = 'val'` or `bar IN ('val1', 'val2', 'valN')` and avoid full table scans.

NoSQL won't save you. If data gets too large to fit into one computer,
it must be spread across multiple of them. In simple words,

You may deploy a database per region, per client-company or any other business attribute.
If clients are somewhat independent, we can skip attribute and onboard them into
current server until it gets full, and then setup a new server for future clients.
Etc.

Partitioning is important, if we need to work with multiple data entities at once,
with a dataset that can't fit in a single server, in a cost efficient manner.

- List **client** devices
- Manage **company** employees
- List **employee** companies
- Find layers in **city**
- Show bandwidth by **device group** and **hour**

Appropriate sharding key, depends on both the data and access pattern.
Partitioning by reminder of Primary Key hash divided by cluster size,
is the trick that most NoSQL databases use.

Do you consider PostgreSQL or ElasticSearch to be horizontally scalable?
They can be. Even SQLite can be with data partitioning.

### NoSQL: When and Why

> **TL;DR:** Use them as soon as you actually **need** them, but not before.  
> Not want but need!  
> Choose wisely.  

So much misunderstanding there. It's sad to see, the powerful technology
being chosen mostly by it's side-effect function.

It's true, that most of them are designed with scaling in mind and are quite
good at it, otherwise people will use relational DBs. But each of them solves
a specific problem in efficient manner. And fails drastically if used for different
purpose.

Do you need a write performance at all costs?  
Maybe keep historical data for occasional analysis?  
How about realtime alerting, or counters?  
Will user benefit from lower read latency, do you need it at all costs?  
Can we go offline or lose some data, without causing a disaster?  
What about consistency?... So many questions.

You can't beat [CAP Theorem](https://en.wikipedia.org/wiki/CAP_theorem), Choose
tradeoffs wisely based on your business requirements. Remember,
eventual consistency is often sufficient for many use cases and allows
for better scalability.

Cassandra, Mongo, BigTable, Victoria Metrics, ClickHouse, DynamoDB, CouchBase...
Do not limit yourself with technology choice from the beginning.
> There is a high chance you're wasting investors' money.

At some point, knowing you data, access patterns and load, you will clearly
understand what you need. When migrating _(specific part)_ becomes economically
reasonable, you introduce new technology to the project.

### Distributed Systems

> **TL;DR:** Keep your _(micro)_ services **stateless**.
> You do not want to support and debug distributed synchronization.

#### Services

Cache invalidation is hard, but distributed consensus and distributed
synchronization is even more so. It is painful. Unless you are writing
a database or a queue, keep the state out of it.

This makes your application to be easily scalable, by simply deploying more.
This makes code-base way simpler and smaller, focused on business requirements.

Even if stateless implementation is less efficient, and may cost a bit more, it will
work better in a long run.

#### Distributed Transactions

Try dividing product, that will avoid them. Or keep them to minimum.
If not possible, it's still okay, but require a bit more work.

## Conclusion

Balance future-proofing with solving current problems. Let your data guide
technology choices and design decisions. This approach enables effective
future scaling without unnecessary complexity or cost now.

Delay scaling as long as possible, but be prepared for when it becomes necessary.
