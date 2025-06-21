---
title: "FlexiRaft: Flexible Quorums with Raft in MySQL"
date: 2025-06-20
tags: ["raft", "mysql", "flexiquorum", "distributed-systems", "consensus"]
categories: ["Systems", "consensus", "distributed-systems"]
---

In large-scale distributed environments like Meta, MySQL remains the most widely deployed transactional datastore, managing **petabytes of data**. Ensuring **low latency, high availability, and strong consistency** across global deployments presents a unique challenge.

To address the limitations of traditional semi-synchronous replication and tailor consensus behavior to application needs, Meta introduced **FlexiRaft** — a modification of the Raft algorithm supporting **flexible and dynamic quorums**.

---

## 🚨 Problem with Traditional Stack

Before FlexiRaft, MySQL at Meta used a semi-synchronous replication protocol:

- Writes were accepted by the **primary** (`R0`), acknowledged by one of its **binlog servers (BLS R0.1 or R0.2)`.
- **Secondary replicas** in other regions (`R1`, `R2`, etc.) asynchronously applied the updates.
- **Binlog servers in other regions** received updates **indirectly** from their local MySQL replica (`R1` → `BLS R1.x`).

### Limitations:

- **Error-prone leader election logic** spread across automation tools.
- Manual intervention during outages.
- External discovery system introduced **scalability bottlenecks**.
- Increasing **cross-region traffic**.
- Lack of fine-grained control over commit vs election quorum.

---

## ✅ Raft Adoption and Customization

Raft provides strong leader semantics and a clean model for replication. However, the original Raft lacked **flexibility in quorum configuration**, critical for latency-sensitive applications.

**FlexiRaft** extends Raft by introducing:

### 🔁 **Configurable Commit Quorums**
- Developers can define data commit quorums based on application needs.
- Election quorums are computed automatically to ensure intersection.

### 🔀 **Dynamic Quorums**
- Election/commit quorum depends on **current leader’s region**.
- Enables low-latency writes without full cross-region consensus.
- Safer crash recovery by inferring **voting history**.

### ⚙️ **Automation Improvements**
- All consensus logic lives **inside the MySQL server**.
- Simplified external tooling and orchestration.

---

## 📦 Architecture Overview

Each region consists of:
- One **MySQL replica** (R0, R1, R2).
- Two **binlog servers** (BLS).

Only the **primary (R0)** accepts writes. Other regions asynchronously replicate. Binlog servers maintain recent logs, aiding fast failovers and recovery.

---

## 🧠 Key Quorum Types

| Type | Description |
|------|-------------|
| **Data Commit Quorum** | Set of replicas (MySQL/BLS) that must ack a transaction before commit |
| **Leader Election Quorum** | Set of replicas that must approve a candidate for leadership |
| **Pessimistic Quorum** | Majority in every constituent group — guarantees intersection with any valid quorum |

---

## 🏗️ New Architecture (Refer Figure 1)

A typical deployment has:

- A **Primary replica (Region 0)** that accepts writes.
- **Secondary replicas (Regions 1 and 2)** with **Binlog Servers (BLS)**.
- BLS store recent logs and allow region-local replication without maintaining a full DB copy.

**Binlog Server Tree:**
```plaintext
         Region 0                   Region 1                   Region 2
        -----------               -----------                -----------
        | Primary |               |Secondary|                |Secondary|
        |   R0    |               |   R1    |                |   R2    |
        -----------               -----------                -----------
         /       \                  /    \                     /     \
   BLS R0.1   BLS R0.2         BLS R1.1  BLS R1.2         BLS R2.1  BLS R2.2
```
![FlexiRaft Replication Layout](/assets/img/Replication.png)

- R0 sends binlog data to BLS R0.1 and R0.2 (semi-sync).
- Secondaries (R1, R2) replicate asynchronously.
- BLS in Region 1 and 2 **only sync from their respective regional secondaries**, not from the primary.

---

## 💡 What is FlexiRaft?

A modification of Raft where:

- **Data commit quorum** and **leader election quorum** are explicitly configurable.
- Supports **static** and **dynamic** quorum models.
- Consensus logic fully lives inside the MySQL server — simplifying tooling.

---

## 🛠️ Modes of Operation

### 1. **Static Quorums**
Quorum configuration is fixed across the cluster. Examples:

**Disjunction (either US or Europe group):**
- Commit: Majority in 2 of 5 (G1–G5) OR 2 of 3 (G6–G8)
- Elect: Majority in 4 of 5 AND 2 of 3

**Conjunction (both US and Europe):**
- Commit: Majority in 2 of 5 AND 2 of 3
- Elect: Majority in 4 of 5 AND 2 of 3

### 2. **Dynamic Quorums**
- Commit quorum = Majority in leader's group
- Elect quorum = Derived from leader’s group and voting history

---

## 🧮 Algorithmic Changes

FlexiRaft modifies the Raft election algorithm to handle:

- **Tracking last known leader** and its term
- **Evaluating voting history** to determine safe elections
- **Using pessimistic quorums** only if dynamic methods fail

---

## 🧪 Algorithm

Changing the original Raft algorithm to support the static quorum modes was relatively less intrusive. Instead of waiting on a simple majority of voters, the algorithm would now
evaluate a configurable static quorum instead. The helper function 𝑖𝑠_𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑 returns a couple of boolean values after analyzing all the votes
received by the candidate. The first boolean denotes if the quorum has already been satisfied from the votes received
thus far and the second one denotes if the satisfaction of the quorum is still possible

For supporting the dynamic quorum mode, the leader election algorithm had to undergo significant changes

There are two extra pieces of information that each server would need to store for supporting dynamic quorums.
• Last known leader and its term
• Previous voting history

```python
Input: 𝑟𝑒𝑠𝑝𝑜𝑛𝑠𝑒𝑠 : 𝑠𝑒𝑡 < 𝑅𝑒𝑞𝑢𝑒𝑠𝑡𝑉𝑜𝑡𝑒𝑅𝑒𝑠𝑝𝑜𝑛𝑠𝑒 >, 𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑝𝑒𝑐𝑖𝑓𝑖𝑐𝑎𝑡𝑖𝑜𝑛, 𝑙𝑎𝑠𝑡_𝑘𝑛𝑜𝑤𝑛_𝑙𝑒𝑎𝑑𝑒𝑟
Output: ElectionResult<𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑, 𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑎𝑐𝑡𝑖𝑜𝑛_𝑝𝑜𝑠𝑠𝑖𝑏𝑙𝑒>
```

- ∈ → belongs to
- u → union
- ⊂ → Proper Subset;  A ⊂ B - every element of A is in B, but B has more elements.
- ∨ →  OR
- ∧ →  AND

```python
if 𝑖𝑠_𝑠𝑡𝑎𝑡𝑖𝑐(𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑝𝑒𝑐𝑖𝑓𝑖𝑐𝑎𝑡𝑖𝑜𝑛) then
   <𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑, 𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠 𝑓𝑎𝑐𝑡𝑖𝑜𝑛_𝑝𝑜𝑠𝑠𝑖𝑏𝑙𝑒> ← 𝑖𝑠_𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑 (𝑟𝑒𝑠𝑝𝑜𝑛𝑠𝑒𝑠, 𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑝𝑒𝑐𝑖𝑓𝑖𝑐𝑎𝑡𝑖𝑜𝑛)
   return ElectionResult<𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑, 𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠 𝑓𝑎𝑐𝑡𝑖𝑜𝑛_𝑝𝑜𝑠𝑠𝑖𝑏𝑙𝑒>
end
```

```python
# Otherwise, executing in dynamic mode 
<𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑, 𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑎𝑐𝑡𝑖𝑜𝑛_𝑝𝑜𝑠𝑠𝑖𝑏𝑙𝑒> ← 𝑖𝑠_𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑 (𝑟𝑒𝑠𝑝𝑜𝑛𝑠𝑒𝑠, 𝑝𝑒𝑠𝑠𝑖𝑚𝑖𝑠𝑡𝑖𝑐_𝑞𝑢𝑜𝑟𝑢𝑚)
if 𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑 or 𝑙𝑒𝑎𝑠𝑡_𝑘𝑛𝑜𝑤𝑛_𝑙𝑒𝑎𝑑𝑒𝑟 = 𝑁𝑈𝐿𝐿 then
	# last_known_leader could be NULL before the first successful election
	return ElectionResult<𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑, 𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑎𝑐𝑡𝑖𝑜𝑛_𝑝𝑜𝑠𝑠𝑖𝑏𝑙𝑒>
end

if 𝑐𝑢𝑟𝑟𝑒𝑛𝑡_𝑡𝑒𝑟𝑚 = 𝑙𝑎𝑠𝑡_𝑘𝑛𝑜𝑤𝑛_𝑙𝑒𝑎𝑑𝑒𝑟.𝑡𝑒𝑟𝑚 + 1 then
	𝑞𝑢𝑜𝑟𝑢𝑚 ← 𝑚𝑎𝑗𝑜𝑟𝑖𝑡𝑦(𝑙𝑎𝑠𝑡_𝑘𝑛𝑜𝑤𝑛_𝑙𝑒𝑎𝑑𝑒𝑟.𝑔𝑟𝑜𝑢𝑝)
	<𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑, 𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠 𝑓 𝑎𝑐𝑡𝑖𝑜𝑛_𝑝𝑜𝑠𝑠𝑖𝑏𝑙𝑒> ← 𝑖𝑠_𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑 (𝑟𝑒𝑠𝑝𝑜𝑛𝑠𝑒𝑠, 𝑞𝑢𝑜𝑟𝑢𝑚)
	return ElectionResult<𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑, 𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠 𝑓 𝑎𝑐𝑡𝑖𝑜𝑛_𝑝𝑜𝑠𝑠𝑖𝑏𝑙𝑒>
end

𝑡𝑒𝑟𝑚_𝑖𝑡 ← 𝑙𝑎𝑠𝑡_𝑘𝑛𝑜𝑤𝑛_𝑙𝑒𝑎𝑑𝑒𝑟.𝑡𝑒𝑟𝑚
𝑛𝑒𝑥𝑡_𝑙𝑒𝑎𝑑𝑒𝑟_𝑔𝑟𝑜𝑢𝑝𝑠 ← {𝑙𝑎𝑠𝑡_𝑘𝑛𝑜𝑤𝑛_𝑙𝑒𝑎𝑑𝑒𝑟.𝑔𝑟𝑜𝑢𝑝}
𝑒𝑥𝑝𝑙𝑜𝑟𝑒𝑑_𝑙𝑒𝑎𝑑𝑒𝑟_𝑔𝑟𝑜𝑢𝑝𝑠 ← {𝑙𝑎𝑠𝑡_𝑘𝑛𝑜𝑤𝑛_𝑙𝑒𝑎𝑑𝑒𝑟.𝑔𝑟𝑜𝑢𝑝}
𝑡𝑒𝑟𝑚𝑖𝑛𝑎𝑙_𝑔𝑟𝑜𝑢𝑝𝑠 ← 𝜙 # empty set

# Iterate until the pessimistic quorum is absolutely required
while 𝑒𝑥𝑝𝑙𝑜𝑟𝑒𝑑_𝑙𝑒𝑎𝑑𝑒𝑟_𝑔𝑟𝑜𝑢𝑝𝑠 ⊂ G do 
	# terminal groups include all groups, denoted by G
	𝑠𝑡𝑎𝑡𝑢𝑠, 𝑛𝑒𝑥𝑡_𝑡𝑒𝑟𝑚, 𝑝𝑜𝑡𝑒𝑛𝑡𝑖𝑎𝑙_𝑛𝑒𝑥𝑡_𝑙𝑒𝑎𝑑𝑒𝑟𝑠 ← 𝐺𝑒𝑡𝑃𝑜𝑡𝑒𝑛𝑡𝑖𝑎𝑙𝑁𝑒𝑥𝑡𝐿𝑒𝑎𝑑𝑒𝑟𝑠(𝑡𝑒𝑟𝑚_𝑖𝑡, 𝑛𝑒𝑥𝑡_𝑙𝑒𝑎𝑑𝑒𝑟_𝑔𝑟𝑜𝑢𝑝𝑠)
	switch 𝑠𝑡𝑎𝑡𝑢𝑠 do
		case 𝐴𝐿𝐿_𝐼𝑁𝑇𝐸𝑅𝑀𝐸𝐷𝐼𝐴𝑇𝐸_𝑇𝐸𝑅𝑀𝑆_𝐷𝐸𝐹𝑈𝑁𝐶𝑇 do
			# when the set of all servers which could have potentially won an election without the candidate’s knowledge
			# has been determined
			𝑞𝑢𝑜𝑟𝑢𝑚_𝑟𝑒𝑠𝑢𝑙𝑡𝑠 ← {𝑖𝑠_𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑓𝑖𝑒𝑑 (𝑟𝑒𝑠𝑝𝑜𝑛𝑠𝑒𝑠,𝑚𝑎𝑗𝑜𝑟𝑖𝑡𝑦(𝑔𝑟𝑜𝑢𝑝)) : 𝑔𝑟𝑜𝑢𝑝 ∈ 𝑡𝑒𝑟𝑚𝑖𝑛𝑎𝑙_𝑔𝑟𝑜𝑢𝑝𝑠}
			return < {𝑞𝑟 ∈ 𝑞𝑢𝑜𝑟𝑢𝑚_𝑟𝑒𝑠𝑢𝑙𝑡𝑠} ∧ 𝑞𝑟.𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑖𝑒𝑑, {𝑞𝑟 ∈ 𝑞𝑢𝑜𝑟𝑢𝑚_𝑟𝑒𝑠𝑢𝑙𝑡𝑠} ∨ 𝑞𝑟.𝑞𝑢𝑜𝑟𝑢𝑚_𝑠𝑎𝑡𝑖𝑠𝑓𝑎𝑐𝑡𝑖𝑜𝑛_𝑝𝑜𝑠𝑠𝑖𝑏𝑙𝑒>
		end
		case 𝑊𝐴𝐼𝑇𝐼𝑁𝐺_𝐹𝑂𝑅_𝑀𝑂𝑅𝐸_𝑉𝑂𝑇𝐸𝑆 do
			return 𝐸𝐿𝐸𝐶𝑇𝐼𝑂𝑁_𝑈𝑁𝐷𝐸𝐶𝐼𝐷𝐸𝐷 < 𝑓𝑎𝑙𝑠𝑒, 𝑡𝑟𝑢𝑒 >
		end
		case 𝑃𝑂𝑇𝐸𝑁𝑇𝐼𝐴𝐿_𝑁𝐸𝑋𝑇_𝐿𝐸𝐴𝐷𝐸𝑅𝑆_𝐷𝐸𝑇𝐸𝐶𝑇𝐸𝐷 do
			𝑡𝑒𝑟𝑚_𝑖𝑡 ← 𝑛𝑒𝑥𝑡_𝑡𝑒𝑟𝑚
			𝑛𝑒𝑥𝑡_𝑙𝑒𝑎𝑑𝑒𝑟_𝑔𝑟𝑜𝑢𝑝𝑠 ← { 𝑛𝑒𝑥𝑡_𝑙𝑒𝑎𝑑𝑒𝑟 ∈ 𝑝𝑜𝑡𝑒𝑛𝑡𝑖𝑎𝑙_𝑛𝑒𝑥𝑡_𝑙𝑒𝑎𝑑𝑒𝑟𝑠 } ∪ {𝑛𝑒𝑥𝑡_𝑙𝑒𝑎𝑑𝑒𝑟.𝑔𝑟𝑜𝑢𝑝}
			𝑒𝑥𝑝𝑙𝑜𝑟𝑒𝑑_𝑙𝑒𝑎𝑑𝑒𝑟_𝑔𝑟𝑜𝑢𝑝𝑠 ← 𝑒𝑥𝑝𝑙𝑜𝑟𝑒𝑑_𝑙𝑒𝑎𝑑𝑒𝑟_𝑔𝑟𝑜𝑢𝑝𝑠 ∪ 𝑛𝑒𝑥𝑡_𝑙𝑒𝑎𝑑𝑒𝑟_𝑔𝑟𝑜𝑢𝑝𝑠
		end
	end
end

return 𝐸𝐿𝐸𝐶𝑇𝐼𝑂𝑁_𝑈𝑁𝐷𝐸𝐶𝐼𝐷𝐸𝐷 < 𝑓𝑎𝑙𝑠𝑒, 𝑡𝑟𝑢𝑒 >
```

---

## 📈 Benefits of FlexiRaft

- **Lower tail latency** for writes
- **Cross-region traffic minimized**
- **Customizable fault-tolerance vs performance tradeoffs**
- **Better recovery paths during partial region failures**
- **Simplified tooling and more robust elections**

---

## 📚 Summary

FlexiRaft is a production-hardened, latency-aware extension of Raft tailored for globally distributed MySQL deployments at Meta. It bridges the gap between **strict consensus** and **flexible system-level tradeoffs**, enabling developers to fine-tune the balance of performance, availability, and consistency.

---

📎 References:
- [CIDR 2023 Paper on FlexiRaft](https://www.cidrdb.org/cidr2023/papers/p83-yadav.pdf)
- [Consensus Models: Paxos vs Raft](https://www.notion.so/Consensus-Paxos-vs-Raft-1f2e2fbb597c4bd886218da47e14f3a9)
