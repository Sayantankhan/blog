---
layout: post
title: "Locking in Distributed System"
date: 2025-11-09
tags: [Lock, Distributed System, ]
categories: ["Lock", "consensus", "distributed-systems"]
---


# 🔐 Locking in Distributed Systems: Why, How, and the Trade-offs

In the **Distributed System world**, as systems grow, we often need a safer way to **ingest or update records** without corrupting shared data.  
In the modern era, where **many instances of a service run in parallel**, it becomes hard to maintain a **lock just for a single instance**, as it doesn’t guarantee **global exclusivity**.

For example, imagine multiple replicas of a service processing the same queue item or record simultaneously — without coordination, you risk **duplicate processing**, **data loss**, or **inconsistent states**.

However, it’s worth noting that **locking comes at a cost**.  
Locking mechanisms consume time, network bandwidth, and sometimes block other operations — reducing concurrency.  
That’s why in distributed systems, **we try to avoid locking when possible**, but in certain critical scenarios, it’s an **unavoidable factor** for maintaining correctness.

The core idea behind locking is **mutual exclusion** — ensuring that shared resources are modified by **only one process** at a time.

---

## 🏗️ Ways of Locking

Distributed locking can be achieved in multiple ways, depending on the system’s **consistency**, **availability**, and **latency** requirements.

---

## 1. 🗄️ Centralized Coordinator — Redis-Based Locking

A common and lightweight approach is to use **Redis** as a centralized lock manager.

### 🔧 How It Works

Redis provides atomic operations (`SETNX`, `GET`, `DEL`) that help acquire and release locks safely.

#### Example: Acquire Lock

```bash
SET resource_name my_unique_value NX PX 30000
```

- NX ensures the key is set only if not already present.
- PX 30000 sets a 30-second expiry to prevent deadlocks.

The command will set the key only if it does not already exist (NX option), with an expire of 30000 milliseconds (PX option). This value must be unique across all clients and all lock requests.

Basically the Unique value is used in order to release the lock in a safe way, with a lua script that tells Redis: remove the key only if it exists and the value stored at the key is exactly the one I expect to be. 

```lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

The "lock validity time(PX)" is the time we use as the key's time to live. It is both the auto release time, and the time the client has in order to perform the operation required before another client may be able to acquire the lock again, without technically violating the mutual exclusion guarantee, which is only limited to a given window of time from the moment the lock is acquired.

#### ❌  Cons

While Redis is simple and fast, it comes with a few trade-offs that need careful consideration.
First, a single Redis instance can easily become a single point of failure — if it goes down, all locks are effectively lost.
Even with replication or clustering, there’s a window where locks may not synchronize correctly.

Another issue arises from clock drift — since locks rely on time-based expiry, small timing differences between servers can cause locks to expire too early or too late, potentially allowing multiple clients to hold the same lock.

Redis also provides eventual consistency, which means it’s not ideal for scenarios that demand strict, immediate guarantees of correctness.

### 🧩 Understanding Redlock

To overcome the single-point-of-failure issue in Redis, Salvatore Sanfilippo (the creator of Redis) proposed the Redlock algorithm — a way to safely implement distributed locks using multiple independent Redis instances.

The idea is simple: instead of relying on one Redis server, you create locks on N Master Redis nodes (commonly 5) using the same key and value. Those nodes are totally independent, so we don’t use replication or any other implicit coordination system.

In order to acquire the lock, the client performs the following operations:

#### 🕒 1. Start with a Local Timestamp
The client first records the current time in milliseconds. This helps calculate how long it takes to acquire the lock across all Redis instances.

This timestamp will later be used to compute how much of the lock’s lifetime has already elapsed during acquisition.

#### 🔐 2. Try Acquiring the Lock on All Redis Nodes

The client attempts to set the same key with a unique random value on all N Redis instances, one by one.

Each SET command will use:
- NX → only set the key if it does not already exist (to ensure exclusivity)
- PX → automatically expire the lock after a defined period (to avoid deadlocks)

```bash
SET resource_name unique_value NX PX 10000
```
To avoid long blocking times if a Redis node is down or slow, each attempt uses a short timeout, typically between 5–50 ms — much smaller than the lock’s expiration time (e.g., 10 seconds).
This ensures the system stays responsive even if one or more Redis nodes fail.

#### ⏱️ 3. Measure Total Acquisition Time

After attempting to acquire the lock on all nodes, the client calculates how much time has elapsed since step 1. This ensures that the total time spent trying to obtain the lock doesn’t consume too much of its validity period.

#### 4. Check for Quorum and Validity

The lock is considered successfully acquired only if both conditions are met:
- The client was able to acquire the lock on a majority of Redis nodes
(e.g., at least 3 out of 5), and
- The total time taken to acquire it is less than the lock’s validity time.

If successful, the remaining validity of the lock is adjusted by subtracting the time spent acquiring it.
For instance, if the lock was set for 10 seconds and took 1 second to acquire, the effective validity becomes 9 seconds.

#### ❌ 5. Handle Lock Acquisition Failure

If the client fails to acquire a quorum of locks (less than N/2 + 1) or if too much time has passed, the attempt is considered unsuccessful. 

In that case, the client must release any partial locks it may have obtained.
This is done by sending a DEL command with the same random value to all Redis nodes, even those it suspects didn’t register the lock — ensuring no stale locks are left behind.

This cleanup step ensures safety and prevents “orphaned” locks from persisting across nodes.


```java
// java implementation with Redisson Client
import org.redisson.Redisson;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.redisson.api.RedissonReactiveClient;
import org.redisson.config.Config;
import org.redisson.RedissonRedLock;


try {
    List<RedissonClient> clients = new ArrayList<>(REDIS_ADDRESSES.length);
    
    // Create client for Redis nodes
    for (String addr : REDIS_ADDRESSES) {
        Config cfg = new Config();
        cfg.useSingleServer()
            .setAddress(addr)
            .setTimeout(200)
            .setConnectTimeout(1000);
        
        RedissonClient rc = Redisson.create(cfg);
        clients.add(rc);
    }

    // Create RLock per client using the same lock name
    String lockName = "resource:order:{orderid}";
    List<RLock> rlocks = new ArrayList<>(clients.size());
    for (RedissonClient client : clients) {
        rlocks.add(client.getLock(lockName));
    }

    // Compose RedissonRedLock from the RLocks
    RedissonRedLock redLock = new RedissonRedLock(rlocks.toArray(new RLock[0]));

    // Try to acquire the lock
    long waitTimeMs = 200;     // try up to 200ms for quorum
    long leaseTimeMs = 10_000; // lock TTL 10s
    boolean locked = false;

    try {
        locked = redLock.tryLock(waitTimeMs, leaseTimeMs, TimeUnit.MILLISECONDS);
        if (locked) {
            System.out.println("Lock acquired (Redlock). Lease: " + leaseTimeMs + " ms");

            Thread.sleep(1000);
        } else {
            System.out.println("Failed to acquire Redlock (quorum not obtained).");
        }
    } catch(InterruptedException ie) {
        Thread.currentThread().interrupt();
        System.err.println("Interrupted while waiting for lock: " + ie.getMessage());
    } finally{
        if (locked && redLock.isHeldByCurrentThread()) {
            try {
                redLock.unlock();
            } catch (IllegalMonitorStateException imse) {
                imse.printStackTrace();
            }
        
        }
    }
} finally {
    for (RedissonClient client : clients) {
        try { client.shutdown(); } catch (Exception ignored) {}
    }
}
```
---

## 2. 🤝 Consensus-Based Locking (Paxos / Raft / ZAB)

When strict consistency is essential, systems use consensus algorithms like Paxos or Raft.

🔧 How It Works

- A leader node is elected using consensus.
- The leader handles lock requests, and all state changes are replicated to followers.
- The cluster maintains a single source of truth for lock ownership.

Systems like zookeeper, etcd and Consul rely on consensus internally to offer strong, fault-tolerant locking.

### 2.1.  🧭 Coordination Services — Zookeeper/Consul Based Locking

Zookeeper is purpose-built and battle-tested for distributed coordination — ideal for leader election, configuration, and locks.

#### 🦓 Zookeeper Locking
- Clients create ephemeral sequential znodes under `/locks/`.
- The client with the lowest sequence number owns the lock.
- If the client disconnects, its node is removed automatically.

They automatically handle lock cleanup — if a client holding a lock disconnects or crashes, the associated ephemeral node or session expires, and the lock is automatically released, preventing deadlocks.

All coordination data, such as lock ownership and session metadata, is persistently stored and replicated across the quorum, so even if one or two nodes fail, the cluster retains the correct state.
Beyond just locking, these systems also come with a rich ecosystem of features like leader election, configuration management, and service discovery, making them a go-to choice for building reliable distributed systems.

##### ❌ Cons

The downside of using Zookeeper is the additional latency introduced by consensus coordination — every write or lock operation must be replicated across multiple nodes before it’s considered complete.

Operating and maintaining these clusters can also be complex, since they require careful configuration, monitoring, and quorum management to stay healthy.

Finally, for smaller systems or lightweight workloads, these tools can feel costly and heavy, as you’re maintaining a full distributed coordination service just for locking or leader election — something that might be overkill compared to simpler solutions like Redis or database-based locks.

```java
// zookeeper implementation with curator [zookeeper client]
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;

/**
 *
 * Demonstrates using Apache Curator's InterProcessMutex (ZooKeeper-backed) for distributed locks.
 *
 * Important:
 * - Ensure a ZooKeeper ensemble is running and reachable at the connect string.
 * - Use ephemeral sequential znodes (handled by InterProcessMutex).
 * - Tune sessionTimeoutMs and connectionTimeoutMs according to your environment.
 */

try {
    final String zkConnect = "127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183";
    
    final int baseSleepMs = 500;
    final int maxRetries = 5;
    final int sessionTimeoutMs = 60_000;
    final int connectionTimeoutMs = 10_000;

    final String lockPath = "/locks/order-resource";
    final String dataPath = "/app/config/order";

    ExponentialBackoffRetry retryPolicy = new ExponentialBackoffRetry(baseSleepMs, maxRetries);

    CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString(zkConnectString)
                .retryPolicy(retryPolicy)
                .sessionTimeoutMs(60_000)
                .connectionTimeoutMs(15_000)
                .build();
    
    // Add a connection state listener to react to LOST / SUSPENDED / RECONNECTED events
    ConnectionStateListener conStateListner => (c, state) -> {
        if (state == ConnectionState.LOST) {
            System.err.println("[CONN] State LOST -> ephemeral nodes may be gone; treat locks as invalid.");
        } else if (state == ConnectionState.SUSPENDED) {
            System.err.println("[CONN] State SUSPENDED -> connectivity hiccup; operations may be delayed.");
        } else if (state == ConnectionState.RECONNECTED) {
            System.out.println("[CONN] State RECONNECTED -> client reattached to ensemble.");
        } else if (state == ConnectionState.READ_ONLY) {
            System.out.println("[CONN] State READ_ONLY -> server in read-only mode.");
        } else {
            System.out.println("[CONN] State changed: " + state);
        }
    };

    client.getConnectionStateListenable().addListener(conStateListner);

    client.start();

    // Wait up to connectionTimeoutMs for initial connection
    boolean connected = client.blockUntilConnected(connectionTimeoutMs, TimeUnit.MILLISECONDS);

    if (!connected) {
        System.err.println("Could not connect to ZooKeeper ensemble within " + connectionTimeoutMs + "ms. Exiting.");
        return;
    }

    System.out.println("Connected to ZooKeeper ensemble: " + zkConnect);

    // Create lock (InterProcessMutex uses ephemeral sequential children under the given path)
    InterProcessMutex lock = new InterProcessMutex(client, lockPath);

    System.out.println("Attempting to acquire lock at " + lockPath);
    boolean acquired = lock.acquire(5, TimeUnit.SECONDS);
    if (!acquired) {
        System.out.println("Failed to acquire lock within timeout. Exiting.");
        return;
    }

    System.out.println("Lock acquired");
    try {
        byte[] payload = "hello-from-service".getBytes(StandardCharsets.UTF_8);
        try {
            if (client.checkExists().forPath(dataPath) == null) {
                client.create().creatingParentsIfNeeded().forPath(dataPath, payload);
            } else {
                client.setData().forPath(dataPath, payload);
            }
        } catch(org.apache.zookeeper.KeeperException.NodeExistsException nee) {
            // race: if someone else created concurrently, set data
            client.setData().forPath(dataPath, payload);
        }

        // Immediately read the value back with freshness guarantee:
        // call sync() first so a follower (if connected to one) will catch up with leader
        client.sync().forPath(dataPath);
        byte[] read = client.getData().forPath(dataPath);
        String readStr = read == null ? null : new String(read, StandardCharsets.UTF_8);
        System.out.println("Read after sync(): " + readStr);

        // simulate short work
        Thread.sleep(500);
    } finally {
        // release the lock
        try {
            if (lock.isAcquiredInThisProcess()) {
                lock.release();
            }
        } catch (IllegalMonitorStateException imse) {
            imse.printStackTrace();
        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }
} catch (InterruptedException ie) {
    Thread.currentThread().interrupt();
    ie.printStackTrace();
} catch (Exception e) {
    e.printStackTrace();
} finally {
    // Graceful shutdown
    try {
        if (client != null) {
            client.close();
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

## 3. 💡 Optimistic Locking — a non-blocking approach

Optimistic locking assumes **conflicts are rare** and avoids taking locks up-front.  

Instead, readers/writers proceed without blocking and check for conflicts at commit time using a **version field**, **timestamp**, or vector clock. If a conflict is detected, the operation is retried, merged, or rejected.

This makes optimistic locking ideal for high-throughput systems where contention is low, or where you can tolerate occasional retries or merges.

### 🧩 How it works
1. Read the record along with its `version` (or timestamp).
2. Make changes locally.  
3. When writing, include the original `version` in the `WHERE` clause (or compare timestamps).  
4. If the version matches, update succeeds and you increment the `version`.
5. If the version has changed (another writer committed), your update affects 0 rows → detect conflict and decide to retry, abort, or merge.

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  amount NUMERIC,
  status VARCHAR(32),
  version INT NOT NULL DEFAULT 1
);

-- read
SELECT amount, status, version FROM orders WHERE id = 42;

-- attempt update using optimistic check
WITH u AS (
  UPDATE orders
  SET amount = 120.00, version = version + 1
  WHERE id = 42 AND version = 7
  RETURNING id, amount, status, version
)
SELECT 'updated'   AS result, u.id, u.amount, u.status, u.version FROM u
UNION ALL
SELECT 'conflict'  AS result, o.id, o.amount, o.status, o.version
FROM orders o
WHERE o.id = 42
  AND NOT EXISTS (SELECT 1 FROM u);
```
##### Example output (when update succeeds and version was 7):

| result  | id | amount | status | version |
| ------- | -- | ------ | ------ | ------- |
| updated | 42 | 120.00 | OPEN   | 8       |

##### Example output (when version = 8):

| result   | id | amount | status | version |
| -------- | -- | ------ | ------ | ------- |
| conflict | 42 | 100.00 | OPEN   | 8       |


### ⚠️ Conflict Handling in Optimistic Locking
In optimistic locking, we accept the risk that **conflicts can occur** — two or more writers may modify the same record concurrently before realizing it. Unlike pessimistic locks that block other writers upfront, optimistic systems detect conflicts **after the fact**, during the commit or merge phase.

When a conflict is detected, the system needs a **resolution strategy** to decide which version of the data “wins” or how to merge concurrent updates.  

This is where approaches like **Last-Write-Wins (LWW)**, **version vectors**, or **CRDTs (Conflict-Free Replicated Data Types)** come into play.

#### Last-Write-Wins (LWW) — simple timestamp approach

In some distributed systems, when concurrent updates occur, the **latest timestamp wins** — the update with the most recent `lastUpdatedAt` (or vector clock) overwrites older ones. This is known as **Last-Write-Wins (LWW)** conflict resolution.

For example:
- **Amazon DynamoDB** (and systems inspired by it, like **Cassandra**) use timestamp-based LWW resolution for concurrent writes — the record with the newer timestamp wins.
- **Riak KV** supports LWW as one of its conflict-resolution strategies (simpler but less safe than sibling resolution or CRDTs).
- **MongoDB’s sharded clusters** can apply LWW semantics internally during replication (based on operation timestamps).
- **Redis (Active-Active / CRDT mode)** uses LWW timestamps for simple key overwrites when full CRDT merge semantics aren’t required.
- **CouchDB / Cloudant** rely on revision trees but can be configured to behave in LWW fashion when conflicts aren’t explicitly handled.
- Even some **mobile sync frameworks** (like Firebase Realtime Database or Realm Sync) use timestamp-based reconciliation as a lightweight default when clients reconnect.

While this approach is simple and fast, it’s important to note that LWW **can silently drop concurrent updates** and is vulnerable to **clock skew** between replicas — making it suitable only when such data loss is acceptable or when conflicts are rare.

To reduce errors caused by clock drift, some systems replace raw timestamps with **vector clocks** or a **globally synchronized logical clock** to capture causal ordering safely.

📊 Example:
Imagine two users updating the same profile record concurrently:

| Replica | Timestamp (ms) | New Name | Result        |
|----------|----------------|-----------|----------------|
| Node A   | 1700010000000  | "Alice"  |                |
| Node B   | 1700010001000  | "Alicia" | ✅ wins (later timestamp) |

When both updates reach the coordinator, **Node B’s** version is chosen as the winner because its timestamp is newer.

#### 🔁 CRDTs — Conflict-Free Replicated Data Types

While LWW simply overwrites data based on timestamps, **CRDTs** (Conflict-Free Replicated Data Types) go a step further by allowing **automatic, lossless merging** of concurrent updates across replicas — without any locks or coordination.

CRDTs are built around mathematical properties of **commutativity**, **associativity**, and **idempotence**, ensuring that replicas eventually converge to the same state no matter the order of operations or message delays.

They are particularly useful in **geo-replicated systems**, **offline-first apps**, and **multi-master databases**, where multiple nodes can update the same data simultaneously.

Common CRDT types:
- **G-Counter (Grow-only Counter)** — counts only increments; merges by summing.
- **PN-Counter (Positive-Negative Counter)** — supports increments and decrements using two G-Counters.
- **G-Set / OR-Set** — sets that support add/remove operations without losing elements.
- **LWW-Register / LWW-Element-Set** — combines timestamps with merge semantics for last-write-wins.
- **Map CRDTs** — dictionary-like CRDTs where each value is itself a CRDT (used in Redis, Riak, and Akka Distributed Data).

✅ Benefits:
- No coordination required — all replicas can accept writes independently.  
- Guaranteed eventual convergence — all nodes reach the same state once all updates propagate.  
- Great fit for real-time collaborative applications, multi-region databases, and mobile sync.

❌ Drawbacks:
- More complex implementation and storage overhead.  
- Merge semantics depend on data type — not all business logic fits CRDT patterns.  
- Some data types (like ordered lists or financial transactions) are harder to model as CRDTs safely.

---
## Conclusion — Choosing the Right Lock in a Distributed System

Each approach that has been explored has a clear purpose and trade-off:

- **Centralized Locks (e.g., Redis)** — fast, simple, and good for lightweight coordination, but limited by single-node reliability.
- **Consensus-Based Locks (Zookeeper, etcd, Consul)** — provide strong correctness and automatic failover through quorum replication and leader election. Ideal when consistency is critical.
- **Optimistic Locking** — avoids blocking entirely, detects conflicts at commit time, and retries when needed. Great for high-throughput, low-contention workloads.
- **Last-Write-Wins (LWW)** — resolves conflicts using timestamps; simple but can silently drop concurrent writes.
- **CRDTs (Conflict-Free Replicated Data Types)** — go beyond simple overwrites, allowing concurrent updates to merge automatically while guaranteeing eventual convergence.

In a distributed system, **locking isn’t just about exclusion** — it’s about deciding *where and how* to coordinate safely under partial failures, latency, and concurrency. Locks, consensus, and optimistic strategies are all tools in that balancing act. The art lies in choosing **the simplest mechanism** that guarantees the level of correctness your system actually needs — nothing more, nothing less.


---
## 📎 References:
- [Distributed Locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [Consensus Models: Paxos vs Raft](https://www.notion.so/Consensus-Paxos-vs-Raft-1f2e2fbb597c4bd886218da47e14f3a9)
- [ZooKeeper: Distributed Process Coordination](https://www.oreilly.com/library/view/zookeeper/9781449361297/)
