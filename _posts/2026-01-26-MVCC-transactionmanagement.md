---
layout: post
title: "Multi-Version Concurrency Control (MVCC): How Modern Databases Scale Concurrency"
date: 2026-01-26
tags:
  - MVCC
  - distributed-systems
  - Concurrency Control
  - Transaction Management
  - system-design
  - DBMS
math: true
---

Modern databases are expected to do something almost magical: serve thousands of concurrent transactions while behaving as if they ran one-by-one. The mechanism that makes this illusion work—without killing performance—is **Multi-Version Concurrency Control (MVCC).**

MVCC is the dominant transaction management scheme in today’s database systems, from classic disk-based engines to modern in-memory databases. But while the idea sounds simple, scaling MVCC efficiently—especially on multi-core, in-memory systems—is far from trivial.

This post breaks MVCC down from first principles, walks through how transactions actually interact with versions, and compares the major MVCC flavors used in real systems.

This blog is inspired by PostgreSQL’s official documentation and research material on MVCC and heap storage design.

---

## Why MVCC Exists
Traditional locking-based concurrency control forces transactions to wait. Reads block writes, writes block reads, and throughput collapses under contention. MVCC takes a different route:

> Instead of blocking, keep multiple versions of the same data and let transactions read the version they’re allowed to see.

This allows:
- Readers to proceed without blocking writers
- Writers to proceed without blocking readers
- High parallelism without sacrificing serializability

But this flexibility comes with real engineering challenges.

---
## Disk-Based vs In-Memory Databases
Before diving into MVCC internals, it’s important to understand the environment:

### Disk-Based Databases
- Optimized to minimize I/O
- Locks and metadata often stored separately
- Page layout and buffer management dominate performance

### In-Memory Databases
- Data lives entirely in Memory itself
- CPU cache behavior becomes critical
- Synchronization overhead (locks, atomics, cache-line bouncing) can dominate

Systems using MVCC, there is no one “standard” implementation. MVCC designs that work well for disk-based systems can perform poorly in in-memory, multi-core environments due to synchronization costs. There are several design choices that
have different trade-offs and performance behaviors.

---

## What MVCC Actually Manages
An MVCC implementation isn’t just “multiple versions”. It is a coordinated system of:
- **Concurrency control protocol** – Who can read/write which version?
- **Version storage** – Where and how versions are stored
- **Garbage collection** – When old versions can be removed
- **Index management** – How indexes point to the correct versions

## The Core Idea of MVCC
For each logical row (tuple), the database maintains multiple physical versions. Each transaction:
- Sees a consistent snapshot of the database
- Operates only on versions valid for its timestamp
- Never reads uncommitted data (in most designs)

This is what enables high concurrency without blocking.

----

## MVCC Flavors in Real Systems

### Tuple-Based MVCC
The most widely used form of MVCC is tuple-based versioning, where each update to a row creates a new physical version of that row. Reads never block writes because readers simply choose the version that is visible to them.

#### Append-Only (O2N) Versioning

In append-only MVCC, updates never overwrite existing data. Instead, every update produces a brand-new version, while the old versions remain intact until they are explicitly removed by garbage collection. This model is famously used by PostgreSQL.

The main appeal of append-only MVCC is its simplicity. Visibility rules are straightforward: a transaction only needs to check timestamps to determine whether a version is visible. This makes read performance excellent, especially for long-running queries that need a stable snapshot.

The downside is that old versions accumulate quickly. Over time, version chains can grow long, increasing memory usage and slowing down scans. This is why background cleanup mechanisms like VACUUM are not optional in such systems—they are essential for performance and correctness.

#### Delta / Undo-Log Based MVCC

Another common approach is delta-based or undo-log–based MVCC. Instead of storing a full new copy of a tuple for every update, the system records only the changes needed to undo that update.This design is used by systems such as MySQL (InnoDB) and Oracle.

The advantage here is efficiency. Because only deltas are stored, memory overhead is lower and garbage collection tends to be faster. However, reads may become more expensive, since reconstructing an older version may require walking backward through multiple undo records.

In short, append-only favors fast reads, while undo-based designs trade some read complexity for better space efficiency.

### Hybrid MVCC

Some systems blur the line between transactional and analytical workloads by combining MVCC with time-travel semantics.
#### Time-Travel / Snapshot-Based MVCC
In this model, versions are indexed explicitly by timestamps, making it easy to query historical states of the database. This approach is particularly well-suited for analytics, auditing, and long-running queries that require consistent historical views.

A notable example is SAP HANA, which leverages in-memory storage and timestamped versions to support both OLTP and OLAP workloads efficiently.


## How MVCC Coordinates Transactions

The DBMS assigns a transaction T a unique, monotonically increasing timestamp as its identifier (Tid) when they first enter the system. The concurrency control protocols use this identifier to mark the tuple versions that a transaction accesses. Some protocols also use it for the serialization order of transactions.

At the physical level, each tuple version carries metadata that allows the database to coordinate concurrent access:

```
//=================================================================
//  txn-id | begin-ts | end-ts | pointer | .... |     Columns     |
//=================================================================
// <------------ Header -----------------------><-----Content----->
```

The header is where MVCC really lives. The **txn-id** field acts as the tuple’s write lock. A value of 0 means the tuple is currently unlocked. Most systems use a 64-bit transaction ID so this field can be updated atomically using a **compare-and-swap (CAS)** instruction, ensuring that only one writer can modify a version at a time.

The begin-ts and end-ts fields define the visibility window of the version. A transaction with timestamp T can see the version if:
```python
begin-ts ≤ T < end-ts
```
When **end-ts is set to infinity**, it simply means the version is still valid and has not been superseded.

Finally, the **pointer** field links versions together, forming a version chain. This allows the system to navigate between older and newer versions of the same logical row. There are two specific kindof version chains - O2N [Old to New] and N2O [New to Old]

---

## Concurrency Control Protocols in MVCC

Every DBMS includes a concurrency control protocol that coordinates the execution of concurrent transactions
This protocol determines 
- whether to allow a transaction to access or modify a particular tuple version in the database at runtime, 
- whether to allow a transaction to commit its modifications.

There are  four core concurrency control protocols for MVCC DBMSs
![](/assets/img/ConcurrencyProtocols.png)

### Timestamp Ordering (MVTO)
Timestamp Ordering is one of the earliest and most conceptually clean MVCC protocols. The core idea is simple: the database decides the serialization order of transactions in advance, based purely on their timestamps.

When a transaction enters the system, it is assigned a unique, monotonically increasing identifier (Tid). This timestamp represents the transaction’s position in the global execution order. Once assigned, the system never violates this order.

To support this protocol, tuple versions carry slightly more metadata than in basic MVCC. In addition to the write lock (**txn-id**) and visibility timestamps (**begin-ts**, **end-ts**), each version also tracks the timestamp of the last transaction that successfully read it, commonly called **read-ts**.

#### Reading Under MVTO
When a transaction T attempts to read a logical tuple A, the database searches through A’s version chain to find a physical version whose visibility window contains T’s timestamp—that is, a version where:

```
begin-ts ≤ Tid < end-ts
```

If such a version Ax is found, T is allowed to read it only if the version is not write-locked by another active transaction. In practice, this means the txn-id field must either be zero or already equal to T’s own identifier. MVTO strictly forbids reading uncommitted data, so this check is non-negotiable.

After successfully reading Ax, the database updates the version’s read-ts field—but only if Tid is larger than the current value. If read-ts already reflects a newer transaction, the system avoids updating it and may force T to read an older version instead. This detail is crucial, because read-ts directly influences whether future updates are allowed.

#### Writing Under MVTO
Writes in MVTO are more restrictive than reads. A transaction is only ever allowed to update the latest version of a tuple.

Suppose transaction T wants to update tuple B. Let Bx be the most recent committed version. The database permits T to proceed only if two conditions are met. First, no other active transaction may hold Bx’s write lock. Second, T’s timestamp must be strictly greater than Bx’s read-ts. This ensures that no “younger” transaction has already observed the version that T is trying to overwrite.

If both conditions hold, the database creates a new version Bx+1 and marks it as being owned by T by setting its txn-id to Tid. The actual visibility of this new version is deferred until commit time.

When transaction T commits, the database finalizes the version chain. The newly created version Bx+1 becomes visible by setting its begin-ts to Tid and its end-ts to infinity. At the same time, the old version Bx has its end-ts set to Tid, marking the precise moment when it stops being visible to future transactions. This clean handoff between versions ensures that every transaction observes a database state consistent with the predefined timestamp order.

#### Why MVTO Matters (and Where It Hurts)
MVTO’s biggest strength is its conceptual clarity. Because serialization order is fixed in advance, reasoning about correctness becomes straightforward, and many anomalies are impossible by construction.

However, this rigidity is also its weakness. Transactions that arrive “too late” are often forced to abort, even if conflicts are mild. Under high contention or skewed workloads, abort rates can rise sharply, making MVTO less attractive for modern, highly concurrent systems.

For Example a simple row A whose value is 100. Transaction $T_1$ (doing update the value) enters the system first and receives timestamp 10. A moment later, transaction $T_2$ (just reading the value) enters and receives timestamp 20. According to MVTO, the serialization order is now fixed: $T_1$ must appear before $T_2$, no matter what happens next. $T_2$ happens to run faster. It reads row A first. The database finds the correct version of A, confirms that it is visible to $T_2$, and allows the read. As part of this read, the system records that the latest read of this version was done by a transaction with $T_2$'s timestamp.

When $T_1$ attempts the write, the database checks the metadata on the current version of A and notices that a newer transaction, $T_2$, has already read it. If $T_1$ were allowed to write now, the system would be forced to pretend that $T_1$ executed after $T_2$, even though $T_1$’s timestamp says otherwise. That would violate the core rule of MVTO. So the database aborts $T_1$.

---
### Optimistic Concurrency Control (MVOCC)

Optimistic Concurrency Control is motivated by the observation that, in many workloads, transaction conflicts are rare. Based on this assumption, the DBMS allows transactions to execute without acquiring locks during reads or updates, reducing the time spent holding synchronization primitives and improving concurrency. In its multi-version form, OCC is adapted to leverage version metadata directly. Unlike classic OCC, the DBMS does not require a private workspace for each transaction, since version visibility rules already prevent transactions from accessing versions they should not see.

Execution under MVOCC is divided into three logical phases. A transaction begins in the read phase, where it performs reads and updates on the database. To read a logical tuple A, the DBMS locates a physical version Ax whose visibility window—defined by begin-ts and end-ts—includes the transaction’s timestamp. If the transaction decides to update a tuple, it must operate on the latest version, provided that the version is not write-locked. In that case, the DBMS creates a new version, such as Bx+1, and marks it with the transaction’s identifier, while keeping it invisible to other transactions.

When the transaction requests to commit, it enters the validation phase. At this point, the DBMS assigns the transaction a commit timestamp ($T_{commit}$), which establishes its position in the serialization order. The system then checks whether any tuple in the transaction’s read set was modified by a transaction that has already committed. If such a conflict is detected, the transaction is aborted.

If validation succeeds, the transaction proceeds to the write phase. The DBMS installs the newly created versions by setting their begin-ts to $T_{commit}$ and their end-ts to infinity, thereby making them visible to subsequent transactions.

Under MVOCC, transactions are restricted to updating only the most recent version of a tuple. However, a transaction cannot observe a newly created version until the transaction that produced it commits. As a result, a transaction may execute using an outdated snapshot and only discover during validation that its execution is no longer serializable, at which point it must abort.

> 💡 **Note:** Mostly InMemory Databases use MVOCC. Because all data resides in memory, the cost of aborting and retrying a transaction is significantly lower than in disk-based systems. This makes optimistic execution practical: transactions can proceed without acquiring locks, and conflicts are resolved later during validation. <br /> <br />
In-memory systems also tend to favor short transactions and high parallelism. Under these conditions, the assumption that conflicts are rare often holds, allowing MVOCC to outperform more rigid protocols like MVTO or lock-heavy schemes such as MV2PL. By avoiding fine-grained locking and reducing synchronization during execution, MVOCC minimizes cache-line contention and improves scalability on multi-core hardware.
<br /><br />
However, this advantage comes with a trade-off. When conflicts do occur, transactions may be aborted late, after performing substantial work. While this cost is acceptable in memory-resident systems, it would be far more expensive in disk-based databases, where redo, undo, and I/O amplification make late aborts undesirable.

---
### Two-Phase Locking (MV2PL)

MV2PL extends classical two-phase locking to a multi-version environment in order to guarantee serializability. Under this protocol, a transaction must acquire the appropriate lock on the current version of a logical tuple before it is allowed to read or modify it. Unlike disk-based systems, where locks are typically stored separately to avoid being paged out, in-memory systems embed locking metadata directly within tuple headers to reduce overhead.

In MV2PL, the tuple’s txn-id field serves as the write lock, while a separate **read-cnt** field tracks the number of active transactions that have read the tuple. These fields may be packed into a single 64-bit value, allowing the DBMS to update both atomically using a single compare-and-swap operation.

To read a tuple A, the DBMS first locates a visible version by comparing the transaction’s timestamp with the version’s begin-ts. If a valid version is found and its txn-id is zero, indicating that no transaction holds the write lock, the DBMS increments the version’s **read-cnt**. A transaction is permitted to update a version only when both the **read-cnt** and txn-id fields are zero, ensuring exclusive access.

When a transaction commits, the DBMS assigns it a commit timestamp ($T_{commit}$), updates the begin-ts of the newly created versions, and releases all read and write locks held by the transaction.

A critical aspect of MV2PL is how it handles deadlocks. Rather than detecting and resolving deadlocks after they occur, MV2PL could employ a **no-wait policy**. If a transaction cannot immediately acquire a required lock, it is aborted. This approach has been shown to scale better under high contention by avoiding blocking chains and deadlock detection overhead.

> 💡 **Note :** Database like MySQL, Postgress, OracleDB  use MV2PL. Most disk-based database systems rely on MV2PL or closely related locking schemes.
In disk-oriented engines, the cost of aborting a transaction late is high due to disk I/O, logging, and buffer management overhead. Two-phase locking avoids this by detecting conflicts early and forcing transactions to wait or abort before significant work is done.
<br /><br />
Disk-based systems also benefit from the predictability of locking. Since transactions may be long-running and I/O-bound, holding locks provides stronger guarantees about progress and reduces wasted computation. Embedding MVCC metadata alongside 2PL-style locking further ensures serializability while maintaining compatibility with page-based storage and logging mechanisms.
<br /><br />
As a result, while MV2PL can limit concurrency under high contention, its conservative nature makes it a practical and robust choice for disk-based databases where late aborts would be prohibitively expensive.
<br /><br />
In practice, many disk-based systems using MV2PL allow transactions to wait for locks and rely on deadlock detection via a wait-for transaction graph, rather than enforcing a strict no-wait policy. When a transaction cannot immediately acquire a lock, the DBMS records a dependency edge in this graph, indicating that the transaction is waiting for another to release a lock. The system periodically analyzes the graph to detect cycles, which represent deadlocks, and resolves them by aborting one of the participating transactions.
<br /><br />
While no-wait policies scale better under extreme contention by avoiding blocking chains and graph maintenance overhead, waiting-based approaches significantly reduce abort rates for long-running, I/O-heavy transactions. Because aborting such transactions is expensive in disk-oriented systems, deadlock detection using transaction graphs is generally preferred in production databases.


---
### Serialization Certifiers
Serialization certifiers take a different approach to concurrency control. Instead of preventing anomalies during execution, the DBMS allows transactions to run under weaker isolation levels and then certifies their correctness before commit. Internally, the system tracks transaction dependencies using a serialization graph and aborts transactions only when it detects patterns that could violate serializability, often referred to as dangerous structures.

The earliest and most widely known certifier-based protocol is Serializable Snapshot Isolation (SSI). SSI builds on snapshot isolation and eliminates write-skew anomalies by monitoring dependencies between concurrent transactions. Transactions read visible versions using their identifiers and are allowed to update a version only if its write lock is free. To enforce serializability, the DBMS tracks anti-dependencies, which arise when one transaction reads a version of a tuple that is later overwritten by another transaction. Each transaction maintains metadata indicating the number of incoming and outgoing anti-dependency edges. When the system detects a chain of two consecutive anti-dependencies, it aborts one of the involved transactions to break the cycle.

The Serial Safety Net (SSN) is a more recent certifier-based protocol that generalizes this idea. Unlike SSI, which applies only to snapshot isolation, SSN works with any isolation level at least as strong as READ_COMMITTED. SSN encodes transaction dependency information directly into metadata fields and validates a transaction by computing a low-watermark that summarizes transactions which committed earlier but must be serialized after it. This more precise tracking allows SSN to identify true anomalies while avoiding overly conservative aborts.

By significantly reducing false aborts, SSN is particularly well-suited for workloads dominated by read-only or read-mostly transactions, where aggressive abort behavior can severely impact throughput.

---
### Optimizations and Their Limits in MVCC Systems

Several proposals attempt to improve the efficiency of MVCC protocols by relaxing some of their strict execution rules. One such optimization allows transactions to speculatively read uncommitted versions created by other transactions. The goal is to reduce unnecessary aborts by increasing concurrency, especially when conflicts are unlikely to materialize. To preserve serializability, however, the DBMS must explicitly track read dependencies between transactions. Each worker maintains a dependency counter representing how many uncommitted transactions it depends on. A transaction is permitted to commit only when this counter reaches zero, at which point the DBMS propagates completion by decrementing the counters of dependent transactions.

A related optimization permits transactions to eagerly update versions that have already been read by uncommitted transactions. This approach similarly requires the system to track dependency relationships, ensuring that a transaction commits only after all transactions it depends on have successfully committed.

While these techniques can reduce abort rates for certain workloads, they introduce new challenges. Speculative reads and eager updates make the system vulnerable to cascading aborts, where aborting one transaction forces multiple dependent transactions to abort as well. Moreover, maintaining centralized dependency-tracking data structures often becomes a scalability bottleneck. As core counts increase, contention on these structures can outweigh the benefits of reduced aborts, limiting the DBMS’s ability to scale efficiently on modern multi-core hardware.

## VERSION STORAGE
Under MVCC, every update to a tuple results in the creation of a new physical version rather than overwriting existing data. How these versions are stored—and how the DBMS navigates among them—is defined by the system’s version storage scheme. Each version contains a pointer that links it to other versions of the same logical tuple, forming a latch-free linked list known as a version chain. This chain allows the DBMS to locate the version that is visible to a given transaction based on its timestamp.

Tuple insertion is straightforward: new tuples are added without interacting with existing versions. Deletions are typically implemented by marking the tuple’s visibility metadata, often by updating its begin-ts or setting a deletion flag. Updates, however, always trigger versioning and are where storage design becomes critical.

![](/assets/img/DBVersionStorage.png)

### Append-Only Storage
In append-only storage, all tuple versions reside in the same table or storage space. This approach is used by PostgreSQL and several in-memory systems such as Hekaton, NuoDB, and MemSQL.

When a tuple is updated, the DBMS allocates a new slot, copies the entire content of the current version into it,applies the modification, and links the new version into the version chain. The original version remains untouched until garbage collection. A key design choice in append-only storage is how the version chain is ordered, since maintaining a latch-free doubly linked list is impractical. As a result, chains are unidirectional.

In the oldest-to-newest (O2N) layout, the head of the chain points to the oldest surviving version. One major advantage of this approach is that indexes never need to be updated when a tuple is modified; they continue to point to the same logical tuple entry. However, finding the most recent version may require traversing a long chain. This pointer chasing increases latency and pollutes CPU caches, making performance highly dependent on aggressive pruning of obsolete versions.

Naively copying large attributes on every update is wasteful, especially when the attribute is unchanged. To address this, many systems allow multiple versions to reference the same external data and maintain reference counters so that the data is freed only when no version still depends on it.

> 💡 **Note:** This is where PostgreSQL introduces HOT (Heap-Only Tuple) updates, which avoid updating indexes when changes do not affect indexed columns. In this case, PostgreSQL does not update any index entries. Instead, index entries continue to point to the original tuple (the root tuple), and PostgreSQL follows an internal heap-only version chain within the same page to locate the latest visible version. Combined with autovacuum and pruning, HOT significantly reduces version chain length and mitigates the main performance drawback of O2N storage.
<br /><br />
If the condition fails — for example, an indexed column is updated or the heap page does not have enough free space — PostgreSQL falls back to the regular update

The alternative is newest-to-oldest (N2O) ordering, where the head of the chain always points to the latest version. This optimizes the common case, since most transactions read the most recent version and avoid traversing the chain. The downside is that every update changes the chain head, forcing the DBMS to update all indexes to reference the new version. This makes N2O more expensive for write-heavy workloads with many secondary indexes.

### Time-Travel Storage
In time-travel storage, the DBMS separates current data from historical versions. The main table stores a master version of each tuple, while older or alternate versions are placed in a dedicated time-travel table. Systems differ in what they consider the master version. For example, SQL Server typically stores the latest version as the master, while SAP HANA may keep the oldest version to efficiently support snapshot isolation.

To update a tuple, the DBMS copies the master version into the time-travel table and then modifies the master version in place. Indexes always point to the master version, so version chain updates do not require index maintenance. This makes the scheme particularly efficient for workloads dominated by queries on current data.

The trade-off appears during garbage collection. When historical versions are pruned, the DBMS may need to copy data back into the main table, introducing additional overhead. Like append-only storage, this approach must also handle non-inline attributes carefully to avoid unnecessary duplication.

### Delta (Undo-Log) Storage
Delta storage takes a fundamentally different approach. Instead of storing full tuple copies, the DBMS maintains the current version in the main table and records only the before-images of modified attributes in a separate delta store. This delta store is known as the rollback segment in MySQL and Oracle, and a similar design is used in HyPer.

When a transaction updates a tuple, the DBMS first writes the original values of the modified attributes into the delta storage and then performs an in-place update on the master tuple. This design is especially efficient for updates that touch only a small subset of columns, as it minimizes memory allocation and copying.

However, reads become more expensive. To reconstruct a consistent snapshot, the DBMS may need to traverse multiple delta records, potentially one per modified attribute. For read-heavy workloads that access many columns, this can introduce significant overhead compared to append-only or time-travel schemes.

>💡 **Note :**  These version storage schemes exhibit different performance characteristics under OLTP workloads, and none of them is universally optimal. Append-only storage performs well for analytical queries that scan large portions of a table because tuple versions are laid out contiguously in memory. This layout minimizes CPU cache misses and benefits from hardware prefetching. However, accessing older versions incurs higher overhead, as the DBMS must traverse the version chain to locate the correct tuple version. Exposing physical versions to index structures also enables more flexible index management, but increases maintenance complexity.
<br /><br />
Across all storage schemes, a common scalability challenge arises from memory allocation. Each transaction allocates space for new versions from centralized structures such as tables or delta stores. As concurrency increases, multiple worker threads contend for these shared allocation points, which can become a bottleneck.
<br /><br />
To mitigate this issue, DBMSs can partition memory allocation by maintaining separate memory spaces for each centralized structure and expanding them in fixed-size chunks. Worker threads then allocate memory from a single space, effectively partitioning the database at the allocation level and eliminating centralized contention hotspots.

----

## GARBAGE COLLECTION
Because MVCC creates a new physical version whenever a transaction updates a tuple, the database will eventually run out of space unless obsolete versions are reclaimed. Beyond storage pressure, excessive versions also degrade performance, as queries spend more time traversing long version chains. As a result, the overall efficiency of an MVCC-based DBMS is highly dependent on the effectiveness of its garbage collection (GC) mechanism.

Most production systems rely on background processes to handle this cleanup. PostgreSQL uses its well-known autovacuum process to remove dead tuples and maintain visibility maps. MySQL (InnoDB) performs cleanup through its purge threads, which remove obsolete undo records once they are no longer visible to any active transaction. In Oracle, version cleanup is handled through background processes such as SMON, which reclaim space from undo segments when it is safe to do so.

At a high level, MVCC garbage collection proceeds in three stages. First, the DBMS identifies expired versions. Next, it unlinks these versions from version chains and index structures. Finally, it reclaims the physical storage occupied by those versions.

>💡 When is a version considered expired?
<br />
A version is expired if it was created by an aborted transaction, or if it is no longer visible to any active transaction. For the latter case, the DBMS checks whether the version’s end-ts is smaller than the timestamps of all currently active transactions.

To perform this check, the DBMS must track the set of active transactions, typically using a centralized data structure. While correct, this approach becomes a scalability bottleneck on multi-core systems due to contention.

In-memory databases often avoid this problem by using epoch-based memory management. Instead of tracking individual transactions, the DBMS groups transactions into coarse-grained epochs. There is always one active epoch and a FIFO queue of older epochs. New transactions are registered to the active epoch, and each epoch maintains a counter of how many transactions are still running. When a transaction completes, it decrements the counter of its associated epoch. Once an epoch’s counter reaches zero, all versions created in that epoch are guaranteed to be invisible and can be safely reclaimed. Epoch transitions are handled either by a background thread or cooperatively by worker threads, eliminating fine-grained synchronization.

Finally, MVCC systems differ in how they locate expired versions during garbage collection. Tuple-level GC examines individual tuple versions and checks their visibility conditions directly. Transaction-level GC, on the other hand, reasons at the transaction granularity and reclaims all versions created by a transaction once it is known that none of them can be visible.

Both approaches are used in practice, with the choice reflecting a trade-off between precision, overhead, and scalability.

> 💡 **Note**: PostgreSQL implements MVCC garbage collection primarily through its VACUUM and autovacuum processes. Unlike some systems that reclaim space immediately, PostgreSQL takes a conservative approach to ensure correctness under concurrent access.
<br /><br />
In PostgreSQL, when a tuple is updated or deleted, the old version is not removed right away. Instead, it is marked as dead by setting its transaction visibility metadata. These dead tuples remain physically present in the table until PostgreSQL can guarantee that no active transaction can still see them.
<br /><br />
To make this determination, PostgreSQL tracks transaction IDs using a global concept called xmin, which represents the oldest active transaction in the system. A tuple version is considered reclaimable only if its deletion or update transaction ID is older than xmin. This ensures that no running transaction can legally access that version.
<br /> <br />
The autovacuum daemon periodically scans tables to identify such dead tuples. Once found, it unlinks them from indexes, removes them from visibility maps, and marks their space as reusable within the table. Importantly, PostgreSQL usually does not return this space to the operating system; instead, it keeps it available for future inserts, which avoids repeated allocations and improves locality.
<br /> <br />
For append-only version chains, PostgreSQL further optimizes updates using HOT (Heap-Only Tuple) updates. If an update does not modify indexed columns, PostgreSQL avoids creating new index entries, which shortens version chains and significantly reduces vacuum overhead. Combined with autovacuum pruning, this helps keep version chains shallow and traversal costs low.

### Tuple-level Garbage Collection
In tuple-level garbage collection, the DBMS determines whether each individual tuple version is still visible to any active transaction. If a version is no longer visible, it can be reclaimed. There are two common ways to implement this.

The most widely used approach is **background vacuuming**. In this design, dedicated background threads periodically scan tables to identify expired versions and reclaim their storage. This method is popular because it is simple to implement and works with all version storage schemes. However, it does not scale well for large databases, especially when only a small number of GC threads are available. Scanning large portions of the database repeatedly becomes expensive.

To improve scalability, some systems register invalidated versions in a **latch-free data structure as transactions execute**. Garbage collection threads then reclaim these versions using epoch-based techniques, avoiding repeated visibility checks. Another practical optimization is to maintain a bitmap of modified (dirty) blocks so that vacuum threads skip blocks that have not changed since the last GC cycle.

An alternative approach is **cooperative cleaning**. Here, garbage collection work is partially shifted to transaction execution. When a transaction traverses a version chain to locate a visible version, it also identifies expired versions along the way and records them in a global structure for later reclamation. This design scales well because GC threads no longer need to search for expired versions themselves. However, cooperative cleaning only works with append-only storage using oldest-to-newest (O2N) version chains.

A known issue with this approach is the so-called dusty corners problem. If no transaction ever traverses the version chain of a particular tuple, its expired versions may never be discovered. Systems that use cooperative cleaning address this by periodically running a full background GC pass, similar to vacuuming, to clean up these neglected regions.

### Transaction-level Garbage Collection
Transaction-level garbage collection operates at a coarser granularity. Instead of inspecting individual tuple versions, the DBMS reasons about entire transactions. A transaction is considered expired when none of the versions it created can be visible to any active transaction.

This approach integrates naturally with epoch-based memory management. Once an epoch ends, all versions created by transactions belonging to that epoch can be safely reclaimed in bulk. Transaction-level GC is simpler than tuple-level GC and works well with transaction-local storage, since all memory allocated by a transaction can be freed at once.

The main drawback is that the DBMS must track the read and write sets of transactions across epochs, rather than relying solely on an epoch membership counter. This additional bookkeeping increases overhead compared to purely epoch-based reclamation.


> 💡 **Note**: In practice, tuple-level garbage collection with background vacuuming remains the most common approach in MVCC-based systems. Increasing the number of dedicated GC threads generally improves cleanup throughput, but it does not eliminate all bottlenecks.
<br /> <br />
A persistent challenge across all GC strategies is the presence of long-running transactions. As long as such a transaction remains active, any versions that could still be visible to it must be retained. This delays garbage collection, leads to version buildup, and can significantly degrade performance—especially in systems with heavy update workloads.

---

##  INDEX MANAGEMENT

In MVCC-based DBMSs, versioning information is kept separate from indexes. An index entry only indicates that some version of a tuple with a given key exists; it does not encode which version is visible to a particular transaction. Conceptually, an index entry is a key–value pair where the key is formed from indexed attributes and the value is a pointer to the tuple. After following this pointer, the DBMS must traverse the tuple’s version chain to locate the version that is visible under the transaction’s snapshot.

Because of this design, indexes in MVCC systems never produce false negatives: if a key exists in the index, some version of the tuple exists. However, false positives are possible, since the indexed tuple version may not be visible to the transaction performing the lookup.

Primary key indexes typically point to the “current” tuple, but how often they need to be updated depends on the version storage scheme. In delta-based designs, the primary key index always points to the master version stored in the main table, so updates do not require index maintenance. In append-only storage, index behavior depends on version-chain ordering. With newest-to-oldest (N2O) ordering, the DBMS must update the primary key index every time a new version is created. If a transaction modifies a tuple’s primary key, the change is applied as a delete followed by an insert in the index.

Secondary indexes are more complex, because both the indexed key and the pointer target may change. MVCC systems generally use one of two strategies for managing secondary index pointers: logical pointers or physical pointers.

### Logical Pointers
With logical pointers, index entries do not reference a specific physical version. Instead, they store a stable identifier for the tuple, and the DBMS uses an indirection layer to map that identifier to the head of the tuple’s version chain. When a tuple is updated, only the mapping entry needs to change, avoiding widespread index updates when the indexed attributes themselves are unchanged.

This approach works with all version storage schemes but requires an extra level of indirection and version-chain traversal during reads. Logical pointers are commonly implemented in two ways. One option is to use the tuple’s primary key itself as the identifier. In this case, a secondary index lookup is followed by a lookup in the primary key index to locate the version-chain head. While simple, this increases storage overhead for large primary keys and adds lookup cost.

An alternative is to assign each tuple a compact, fixed-size identifier and maintain a separate latch-free mapping structure from this identifier to the version-chain head. This reduces index size and avoids expensive primary key lookups, at the cost of maintaining an additional data structure.

### Physical Pointers
With physical pointers, index entries directly reference the physical location of a specific tuple version. This approach is only feasible with append-only storage, where versions are never overwritten and remain addressable. Whenever a tuple is updated, the DBMS inserts the newly created version into all relevant secondary indexes.

The advantage is faster reads: an index lookup points directly to the exact version, eliminating version-chain traversal and extra comparisons. Several MVCC systems, including MemSQL and Hekaton, adopt this design. The downside is higher update cost, since every new version must be reflected in every secondary index.

> 💡 **Note:** Like other MVCC design choices, index management strategies behave differently across workloads. Logical pointers favor write-intensive workloads because secondary indexes are updated only when indexed attributes change, but reads are slower due to indirection and version-chain traversal. Physical pointers favor read-intensive workloads by providing direct access to the correct version, but they significantly increase index maintenance overhead during updates.
<br /><br />
A final implication of MVCC is that index-only scans are generally not possible unless versioning metadata is embedded directly in index entries. The DBMS must inspect tuple-level metadata to determine version visibility. Some systems, such as NuoDB, reduce this overhead by storing version headers separately from tuple data, minimizing the amount of data that must be fetched during visibility checks.

---

## Conclusion

MVCC is not a single algorithm but a broad design space shaped by trade-offs between concurrency, correctness, and performance. While the core idea—maintaining multiple versions to avoid blocking—is simple, the complexity emerges in how systems manage version creation, visibility, indexing, garbage collection, and conflict resolution at scale.

Different concurrency control protocols emphasize different priorities. Timestamp ordering enforces a strict global order but suffers from unnecessary aborts under contention. Optimistic schemes delay conflict detection to maximize parallelism and are well suited for in-memory systems where aborts are cheap. Lock-based approaches favor predictability and early conflict detection, making them a practical choice for disk-oriented databases. Certifier-based protocols further relax execution while selectively preventing only truly dangerous interleavings.

Version storage strategies amplify these trade-offs. Append-only layouts optimize scans and historical access but depend heavily on aggressive pruning. Time-travel storage isolates current and historical data to stabilize indexes. Delta-based schemes minimize write amplification but complicate reads. None of these designs is universally optimal; each reflects assumptions about workload shape, data locality, and hardware costs.

Index management and garbage collection reveal where MVCC becomes most fragile. Indexes must tolerate false positives while preserving correctness, and garbage collection must balance safety against scalability. Long-running transactions, centralized metadata, and version buildup remain persistent challenges, especially on modern multi-core systems.

The common thread across all of these decisions is that MVCC performance depends less on versioning itself and more on how well the system controls its side effects—version chain growth, synchronization overhead, memory contention, and abort rates. Modern systems increasingly favor designs that reduce centralized coordination, exploit hardware locality, and delay decisions until they are unavoidable.

Understanding these trade-offs is essential not only for database designers, but also for practitioners tuning real systems. MVCC is powerful, but only when its complexity is carefully contained.

---
## References
- [An Empirical Evaluation of In-Memory Multi-Version Concurrency Control](https://www.vldb.org/pvldb/vol10/p781-Wu.pdf)
- [The Part of PostgreSQL We Hate the Most](https://www.cs.cmu.edu/~pavlo/blog/2023/04/the-part-of-postgresql-we-hate-the-most.html)