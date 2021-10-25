## Logical decoding (Draft)

Introduced in Postgres since version 9.4, allows abuse of internal consistency mechanism, for some advanced functionality.
Built on top of existing features. Which we need to understand first.


#### Terms 

- **WAL** (White Ahead Log) [doc](https://www.postgresql.org/docs/14/wal-intro.html) - Seqential log of all transactions, which guarantees consistency.
- **LSN** (Log Sequence Number) [doc](https://www.postgresql.org/docs/14/datatype-pg-lsn.html) - location in a WAL file.
- **Slot** (Replication Slot) [doc](https://www.postgresql.org/docs/10/warm-standby.html#STREAMING-REPLICATION-SLOTS) - Keeps WAL from deleting untill every replica confirmed, that data is safely stored. By keeping track of 3 LSN values ( Write, Flush and Apply )


#### Consistency
In order to keep DB fault tollerant, and keep speed at usable level, engineers ended up with interesting approach.
Each transaction, before it's applied, goes to WAL file. From this point, even in case of crash, database can replay log file and restore state. After all the changes were persisted on disk (in a proper places), WAL file can be removed.

Why not to write on disk directly? Huge performance penalty, during random access can explain it.

#### Replication
Having WAL file, replication is as easy as sending WAL over a network to another replica. Having replica in kinda "restore" mode.
No need of keeping track of each change separately. 

This however, require some additional information in WAL, making it larger in size.

We are not much interested in **syncronous replication**, when each transaction wait confirmation from replica.

#### Asynchronous Replication
Having replica in asynchronous mode, means that there always be some lag. And in order to keep track of this lag Postgres uses **Slots**.  
If for any reason, replica goes offline or simply stops responding, database preserves WAL file.




Introduced in Postgres since version 9.4, allows to abuse internal consistency mechanism (WAL file), for advanced usage.
Built on top of existing functionality. Which we need to understand first.
