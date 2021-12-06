---
title: "Postgres and Elasticsearch Realtime Sync"
date: 2021-11-22T14:11:47+01:00
draft: true
summary: "The power of sequential WAL decoding."
---

### TL;DR; Yes, it works.
You work with Postgress as database. Stream WAL (with logical decoding) into Elasticsearch. And use Elastic for search.
Data will be almost realtime and always consistent.

Bellow I would describe some details, pitfalls, decidions.

