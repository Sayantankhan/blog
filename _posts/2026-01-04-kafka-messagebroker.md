---
layout: post
title: "Kafka Broker – the heart of Kafka"
date: 2026-01-04
tags:
  - kafka
  - distributed-systems
  - streaming
  - messaging
  - system-design
  - scalability
math: true
---

In modern distributed systems, Apache Kafka plays a crucial role. It is a system designed to handle massive traffic with extremely high throughput while scaling horizontally with ease.

Today, most large-scale systems that demand reliable event ingestion, stream processing, and real-time data pipelines include Kafka as a core component.

In this blog, we will go deep into Kafka’s internals—not just what Kafka does, but how it does it.
We will break down Kafka’s architecture, its storage model, consumer mechanics, replication, and the optimizations that make Kafka fast and reliable at scale.

---

## Kafka Brokers

At the heart of Apache Kafka lies the broker. A Kafka broker is not just a message forwarder—it is a log manager, disk I/O engine, and network server combined.

### What Is a Kafka Broker?

A Kafka broker is a server process responsible for:
- Storing data durably on disk
- Serving read & write requests from clients
- Managing partition leadership
- Replicating data to other brokers

A Kafka cluster is simply a set of brokers, each identified by a unique broker.id. 

> 💡 **Note**: Kafka does not have a master broker.Leadership is per-partition, not per-cluster.

### Broker as a Log Storage Engine
Internally, Kafka brokers store data as append-only logs. Each topic partition on a broker maps to:

```pgsql
/kafka-logs/
 └── topic-1/
     └── partition-0/
         ├── 00000000000000000000.log
         ├── 00000000000000000000.index
         ├── 00000000000000000000.timeindex
```

> 💡 **Note**: us-east-1-consumer-product is the topic created

```bash
[appuser@6dfdd4df6072 /var/lib/kafka/data]$ ls -lrt 
-rw-r--r-- 1 appuser appuser    0 Jan  4 05:28 cleaner-offset-checkpoint
-rw-r--r-- 1 appuser appuser   88 Jan  4 05:28 meta.properties
drwxr-xr-x 2 appuser appuser 4096 Jan  4 05:33 us-east-1-consumer-product-0
-rw-r--r-- 1 appuser appuser   35 Jan  4 05:41 recovery-point-offset-checkpoint
-rw-r--r-- 1 appuser appuser    4 Jan  4 05:41 log-start-offset-checkpoint
-rw-r--r-- 1 appuser appuser   35 Jan  4 05:41 replication-offset-checkpoint

[appuser@6dfdd4df6072 data]$ cd us-east-1-consumer-product-0/

[appuser@6dfdd4df6072 /var/lib/kafka/data/us-east-1-consumer-product-0]$ ls -lrt
-rw-r--r-- 1 appuser appuser 10485760 Jan  4 05:33 00000000000000000000.index
-rw-r--r-- 1 appuser appuser 10485756 Jan  4 05:33 00000000000000000000.timeindex
-rw-r--r-- 1 appuser appuser       43 Jan  4 05:33 partition.metadata
-rw-r--r-- 1 appuser appuser        8 Jan  4 05:33 leader-epoch-checkpoint
-rw-r--r-- 1 appuser appuser       94 Jan  4 05:38 00000000000000000000.log
```

Key properties:
- Writes are sequential disk appends
- Reads are offset-based
- No random writes
- No message mutation

This design is what allows Kafka to scale linearly with disk throughput.

---

### Broker Network Server and Core Responsibilities
Every Apache Kafka broker runs a network server that handles all client and inter-broker communication. Kafka uses a binary TCP-based request–response protocol, and each request is processed independently. The broker does not maintain conversational state with clients, which keeps the network layer lightweight and scalable under high concurrency.

![](/assets/img/kafka-producer-broker.png)
![](/assets/img/kafka-consumer-broker.png)

#### Broker Responsibilities
##### 1️⃣ Partition Leadership
For each partition, exactly one broker acts as the leader. All writes and, by default, reads go through this leader. The leader assigns offsets, appends records to disk, and coordinates replication with follower brokers. Leadership exists at the partition level, allowing load to be evenly distributed across the cluster.

##### 2️⃣ Disk I/O
Kafka brokers are disk-centric systems. Data is written as sequential appends to log files and read back using offset-based fetches. Brokers rely heavily on the operating system’s page cache and avoid random writes or in-place updates. This design keeps disk I/O predictable and fast, even at very high throughput.

##### 3️⃣ Client Coordination
![](/assets/img/kafka-client-coordination.png)
Brokers coordinate with clients only through metadata and protocol-level interactions. They inform producers and consumers about partition leaders, replicas, and cluster state. While brokers facilitate consumer group membership and rebalances, they do not track message processing or consumer progress. Offsets are treated as data and stored internally in Kafka itself.

---

### Kafka Brokers : Why Kafka Is a Log, Not a Queue
At its core, Apache Kafka brokers implement a **distributed append-only log**, not a traditional message queue. 

That single decision explains Kafka’s performance characteristics, its scalability, and even the simplicity of its failure model.

#### Traditional Queue Model
In a traditional message queue, messages have a short life. A producer sends a message, the broker delivers it to a consumer, and once acknowledged, the message disappears. The broker must constantly track who consumed what, handle deletions, and coordinate delivery state across consumers.

This model works well for small systems, but it collapses under large-scale traffic. As throughput increases, the broker becomes overloaded with consumer bookkeeping, memory churn, and synchronization overhead. Replay is hard, fan-out is expensive, and scaling reads independently of writes is nearly impossible.

#### Brokers as Log Managers
Kafka brokers act as log managers, not message dispatchers. A Kafka broker treats incoming data as immutable history. When data is written to Kafka, the broker’s primary responsibility is to append that data to disk in an ordered and durable way. There is no concept of message lifecycle or delivery acknowledgment at the broker level. Instead, brokers expose logs as byte streams that clients can read from arbitrary offsets. This design deliberately keeps brokers simple, predictable, and fast.

##### 1️⃣ Sequential Disk Writes Beat Everything Else
Kafka brokers rely almost entirely on sequential disk writes. When a producer sends data, the broker appends it to the end of a partition log without modifying existing data. This access pattern is extremely efficient on modern disks and works exceptionally well with the operating system’s page cache.

![](/assets/img/kafka-broker-flow.png)

By avoiding random writes, in-place updates, and deletions, Kafka turns disk I/O into a throughput problem rather than a latency problem. Even with commodity hardware, brokers can sustain millions of writes per second because they are always writing forward, never reorganizing data.

##### 2️⃣ Multiple Consumers, Same Data (Fan-out for Free)
Because Kafka brokers do not delete messages after consumption, the same data can be read by multiple consumers independently. Each consumer group maintains its own offset, allowing different systems to process the same stream without interfering with each other.

![](/assets/img/kafka-consumer-read.png)

This makes fan-out essentially free. Analytics pipelines, monitoring services, and real-time processors can all consume the same topic without message duplication or broker-side coordination. The broker simply serves data; it does not care who reads it or how many times it is read.

##### 3️⃣ Replay Is a First-Class Feature
In Kafka, replaying data is not an exception—it is a normal operation. Since brokers retain logs for a configurable period, consumers can rewind their offsets and reprocess historical data whenever needed.

This is particularly powerful for recovering from bugs, rebuilding stateful services, or onboarding new consumers. Unlike traditional queues, where messages vanish after delivery, Kafka treats history as a durable asset rather than a temporary buffer.

##### 4️⃣ Brokers Stay Dumb (and That’s a Good Thing)
Kafka brokers intentionally avoid tracking consumer state or delivery progress. They do not know which messages have been processed, which ones failed, or which consumer is “ahead” or “behind.”

All the intelligence lives on the client side. Consumers decide when to commit offsets, how to handle failures, and how to process data. This design drastically reduces broker complexity and allows Kafka to scale horizontally without becoming a coordination bottleneck.

##### 5️⃣ Brokers Stay Stateless on Purpose
By remaining stateless with respect to consumers, brokers become easier to replace and recover. If a broker fails, another broker can take over leadership for its partitions without reconstructing per-consumer metadata.

This statelessness improves fault tolerance and keeps failure recovery fast and deterministic. Brokers focus on durability and replication, while consumers handle progress tracking and correctness guarantees.

##### 6️⃣ Why Logs Scale and Queues Don’t, and Managing Infinite Logs Without Pain
Queue-based systems struggle at scale because they must constantly manage message deletion, delivery acknowledgment, and consumer bookkeeping. Kafka avoids all of this by modeling data as an append-only log.

To prevent logs from growing indefinitely, brokers split each partition into log segments. These segments are immutable once closed and can be efficiently deleted based on time or size-based retention policies. This allows Kafka to manage infinite streams of data while keeping storage and performance under control.

The combination of append-only writes, segmented logs, and consumer-managed state is what allows Kafka to scale predictably under heavy load.

---

### Replication and Leader Election: Inside the Kafka Broker
Replication and leader election are what turn Apache Kafka from a fast log into a fault-tolerant distributed system. Every Kafka broker is responsible not only for storing data locally, but also for coordinating with other brokers to replicate that data, track health, and recover from failures.

### Kafka Leader Election via ZooKeeper

Before KRaft, Apache Kafka relied on Apache ZooKeeper for cluster coordination and leader election. ZooKeeper does not move data, replicate logs, or handle traffic. Its only job is to act as a source of truth for cluster state. Kafka uses ZooKeeper for who is alive, who is the leader, and who is allowed to lead.

![](/assets/img/kafka-replication.png)

#### 1️⃣ There Is Only ONE Controller

Although Kafka has many brokers, only one broker acts as the controller at any given time.

The controller is responsible for:
- Detecting broker failures
- Triggering partition leader elections
- Updating metadata in ZooKeeper
- Informing other brokers about leadership changes

Here, we have spinned up 3 kafka brokers and during the boot up they got registered with zookeeper 

> /brokers/ids/{broker_id}

```json
/brokers/ids/1
{"features":{},"listener_security_protocol_map":{"INTERNAL":"PLAINTEXT","EXTERNAL":"PLAINTEXT"},"endpoints":["INTERNAL://kafka1:29092","EXTERNAL://localhost:9092"],"jmx_port":-1,"port":29092,"host":"kafka1","version":5,"timestamp":"1767512418493"}

/brokers/ids/2
{"features":{},"listener_security_protocol_map":{"INTERNAL":"PLAINTEXT","EXTERNAL":"PLAINTEXT"},"endpoints":["INTERNAL://kafka2:29093","EXTERNAL://localhost:9093"],"jmx_port":-1,"port":29093,"host":"kafka2","version":5,"timestamp":"1767512420068"}

/brokers/ids/3
{"features":{},"listener_security_protocol_map":{"INTERNAL":"PLAINTEXT","EXTERNAL":"PLAINTEXT"},"endpoints":["INTERNAL://kafka3:29094","EXTERNAL://localhost:9094"],"jmx_port":-1,"port":29094,"host":"kafka3","version":5,"timestamp":"1767512416669"}
```

##### Controller Election
When brokers start, they all try to create the same ZooKeeper node

This node is created as an ephemeral znode.
- The first broker that successfully creates it becomes the controller
- If the controller dies, its ZooKeeper session expires
- The /controller znode disappears
- Another broker immediately wins the race and becomes controller

![](/assets/img/kafka-zoo-controller.png)

📌 ZooKeeper guarantees only one winner. This is **cluster-level leadership, not partition leadership**. This broker id tells which broker has elected for leader for the cluster

#### Partition Leader Election
For every partition, Kafka stores leadership metadata in ZooKeeper:
> /brokers/topics/{topic}/partitions/{partition}/state

![](/assets/img/kafka-zoo-topic-state.png)
![](/assets/img/kafka-zoo-topic-partition-state.png)

**leader_epoch** prevents split-brain writes. If Old leader comes back and tries to write stale data Kafka rejects it because Epoch no longer matches. ZooKeeper is the arbiter of truth here.

> In the image we can see for the topic us-east-1-consumer-product - broker 3 got elected as Leader

#### What Happens When a Leader Broker Dies

Let’s walk through a failure scenario.

- Broker 1 crashes
- ZooKeeper detects session expiration
- Controller receives a watch notification
- Controller checks:
    - Which partitions had broker 1 as leader
    - Which replicas are still in ISR
- Controller selects a new leader from ISR
- ZooKeeper /state is updated
- Brokers watch /state and update their local metadata
- Clients refresh metadata and continue

All of this happens without stopping the cluster.

---

#### Replication: RF and ISR

When a topic is created, Kafka assigns each partition a replication factor (RF). This means the same partition data is stored on multiple brokers. Among these replicas, one broker is elected as the leader, while the others act as followers.

The Replication Factor (RF) defines **how many Kafka brokers store a copy of a partition’s data**.
If a topic has a replication factor of 3, then each partition of that topic is physically stored on three different brokers.

RF is a storage-level guarantee, not a runtime guarantee. It answers only one question:

> How many brokers should hold this data?

It does not decide:
- Who handles reads
- Who handles writes
- Who can become leader at any moment

Followers continuously pull data from the leader and append it to their own logs. A follower is considered reliable only if it keeps up with the leader. These reliable replicas form the **ISR (In-Sync Replica)** set.

![](/assets/img/kafka-topic-settings.png)

If a follower broker is slow, or looses network connectivity, or crashes - it will be removed from ISR. Writes can still continue as long as the leader and enough ISR members remain, depending on the acks setting. This mechanism prevents stale replicas from being promoted and causing data loss.

| Concept            | Meaning                                   |
| ------------------ | ----------------------------------------- |
| Replication Factor | How many replicas *exist*                 |
| ISR                | How many replicas are *currently in sync* |


##### How Kafka Decides ISR Membership
Each follower continuously pulls data from the leader. The leader tracks:
- The follower’s last fetch offset
- The time since its last successful fetch

If a follower does not fetch data within **replica.lag.time.max.ms**, it is removed from ISR.

> 💡 **Note**: This check is leader-driven, not ZooKeeper-driven.

```shell
(venv) sk@skservername:/project/docker/kafka$ sudo docker exec -it kafka1 kafka-topics   --bootstrap-server kafka1:29092   --describe
Topic: us-east-1-consumer-product	TopicId: NSFgkT3aRvK2Hb3PgumiPQ	PartitionCount: 1	ReplicationFactor: 2	Configs: min.insync.replicas=1,cleanup.policy=delete,retention.ms=604800000
	Topic: us-east-1-consumer-product	Partition: 0	Leader: 1	Replicas: 1,2	Isr: 1,2
```

ISR is dynamic.
- Shrink → when replicas fall behind or fail
- Expand → when replicas catch up and rejoin

When a broker comes back, It does not instantly rejoin ISR. It must fully catch up to the leader first. This prevents stale replicas from corrupting leadership.

##### Why ISR Matters for Writes
Producer acknowledgements interact directly with ISR.
```properties
acks=all
```
With this setting:
- Leader waits for all ISR replicas to confirm write
- Only then sends success to producer

If ISR shrinks:
- Write quorum shrinks
- Throughput improves
- Risk increases if leader fails

ISR is Kafka’s real quorum — not RF.

**min.insync.replicas** - This tells kafka at least how many ISR members must exist, Otherwise, writes fail.

##### ISR and Leader Election
Only ISR members are eligible for leadership. If the leader dies: Kafka picks a new leader from ISR. If ISR is empty → no safe leader exists

Kafka can be configured to allow unclean leader election. This allows Out-of-sync replicas to become leader and data loss in exchange for availability. This setting breaks Kafka’s safety guarantees and should be avoided unless you fully accept data loss.
```properties
unclean.leader.election.enable=true
```

## Conclusion

At first glance, Apache Kafka appears complex—brokers, partitions, offsets, ISR, ZooKeeper, leader election. But when viewed from the inside, Kafka is intentionally built on a few very simple ideas, applied consistently.

Kafka works because brokers are not intelligent message routers. They are log servers. They append data sequentially, serve it by offset, and replicate it with clear leadership rules. By pushing state and processing logic to clients, Kafka keeps the core of the system fast, predictable, and scalable.

Replication is not about having copies—it is about trust. ISR defines which replicas are safe, leader election ensures only trusted replicas can write, and acknowledgements enforce durability boundaries. Kafka never guesses; it either knows a replica is safe or it refuses to proceed.

ZooKeeper (in classic Kafka) is not a control plane that “runs Kafka.” It is a consistency anchor—a place where authority is recorded and verified. Kafka’s controller uses it to make deterministic decisions, not dynamic ones.

What makes Kafka powerful is not any single feature, but how these pieces reinforce each other:
- Logs enable replay and fan-out
- Sequential I/O enables throughput
- ISR enables safety
- Leader election enables fast recovery
- Simple brokers enable scale

Overall speaking, Kafka is a distributed commit log, engineered to survive failure while moving data at scale.