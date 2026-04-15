# D05 — Distributed Systems Internals
**Track: Deep Dive | Consensus, CAP in practice, failure detection**

---

## 1. Why Distributed Systems Are Hard

Three fundamental problems that don't exist on a single machine:

```mermaid
graph TB
    subgraph Problems["The Three Problems"]
        Clock["Partial Ordering / Clocks<br/>No global clock in distributed systems<br/>Event A on Node 1 vs Event B on Node 2:<br/>which happened first?<br/>Solution: Logical Clocks (Lamport, Vector)"]
        Failure["Partial Failure<br/>Node crashes mid-operation<br/>Did Node 2 receive my message?<br/>Network split — is it dead or partitioned?<br/>Solution: Consensus protocols"]
        Consistency["Consistency vs Availability<br/>CAP theorem:<br/>During partition, sacrifice one<br/>CP or AP — your choice"]
    end
```

---

## 2. The CAP Theorem — Real Implementation Choices

```mermaid
graph TD
    CAP_Real["CAP In Real Systems"]

    CP_Systems["CP Systems<br/>'I will not answer if I'm not sure'"]
    CP_Ex["Examples:<br/>ZooKeeper: refuses reads during partition<br/>HBase: blocks writes until quorum<br/>etcd (Kubernetes): leader election stalls"]

    AP_Systems["AP Systems<br/>'I'll answer but maybe stale'"]
    AP_Ex["Examples:<br/>Cassandra: always answers (may be old)<br/>DynamoDB (default): eventually consistent<br/>DNS: cached stale for TTL duration"]

    CA_Reality["CA in Practice:<br/>Only possible with single node<br/>Or network with zero partitions<br/>= impossible in real production"]

    CAP_Real --> CP_Systems & AP_Systems & CA_Reality
    CP_Systems --> CP_Ex
    AP_Systems --> AP_Ex
```

**Practical design choice:** For banking (financial consistency critical): CP. For e-commerce shopping cart (slight inconsistency acceptable): AP. For DNS (stale for TTL is fine): AP. For distributed locks: CP (must be consistent, can sacrifice availability).

---

## 3. Consensus — How Distributed Nodes Agree

**Problem:** 5 nodes need to agree on a value (e.g., "is Node 3 the leader?"). Any node could be slow, crashed, or on the wrong side of a network partition.

### Raft Consensus Algorithm

Raft is the consensus algorithm used by etcd (Kubernetes control plane), Consul, CockroachDB.

```mermaid
sequenceDiagram
    participant L as Leader (Node 1)
    participant F1 as Follower (Node 2)
    participant F2 as Follower (Node 3)
    participant F3 as Follower (Node 4)

    Note over L,F3: Normal operation: heartbeats
    L->>F1: Heartbeat (AppendEntries, empty)
    L->>F2: Heartbeat
    L->>F3: Heartbeat

    Note over L,F3: Client writes value "X"
    L->>F1: AppendEntries: [term=3, index=47, entry="X"]
    L->>F2: AppendEntries: [term=3, index=47, entry="X"]
    L->>F3: AppendEntries: [term=3, index=47, entry="X"]

    F1->>L: ACK
    F2->>L: ACK
    Note over L: Majority (3/5 including self) confirmed
    L->>L: Commit entry 47
    L->>F1: Commit notification
    L->>F2: Commit notification
    L->>F3: Eventually (F3 was slow)
    L->>Client: Success

    Note over F3: Node 1 crashes
    F3->>F3: Election timeout expires
    F3->>F1: RequestVote: term=4, I want to be leader
    F3->>F2: RequestVote: term=4
    F1->>F3: Vote granted (F3's log is up-to-date)
    F2->>F3: Vote granted
    Note over F3: F3 becomes new leader (2 votes = majority of remaining)
```

**Quorum:** With N nodes, Raft requires (N/2 + 1) nodes to confirm writes. With 5 nodes, 3 must confirm. This tolerates 2 simultaneous failures. With 3 nodes, 2 must confirm — tolerates 1 failure.

---

## 4. Eventual Consistency — What It Actually Means

```mermaid
sequenceDiagram
    participant Client1 as Client 1 (Delhi)
    participant NodeA as Cassandra Node A (Mumbai)
    participant NodeB as Cassandra Node B (Singapore)
    participant Client2 as Client 2 (Singapore)

    Client1->>NodeA: WRITE user_name="Alice" (timestamp: T1)
    NodeA->>NodeA: Write locally
    NodeA->>Client1: ACK (write successful)
    Note over NodeA: Background async replication to NodeB

    Client2->>NodeB: READ user_name for user_id=1
    Note over NodeB: Replication not yet arrived from NodeA
    NodeB->>Client2: user_name="Bob" (OLD VALUE — stale read)

    Note over NodeA,NodeB: Replication completes (T1+100ms)
    NodeB->>NodeB: user_name="Alice" (updated)

    Client2->>NodeB: READ user_name for user_id=1
    NodeB->>Client2: user_name="Alice" (consistent now)

    Note over Client2: This is "eventual" consistency:<br/>Given enough time with no new writes,<br/>all nodes will converge to same value
```

---

## 5. Distributed System Failure Modes

```mermaid
graph TB
    Failures["Distributed System Failure Types"]

    CF["Crash Failure<br/>Node stops responding<br/>All messages lost<br/>Detection: timeout<br/>Example: EC2 instance crash"]
    OF["Omission Failure<br/>Node up but drops messages<br/>Network packet loss<br/>Example: EC2 network issues"]
    TF["Timing Failure<br/>Response too slow<br/>Breaks timing assumptions<br/>Example: EC2 on noisy host, GC pause"]
    BF["Byzantine Failure<br/>Node sends incorrect data<br/>Worst case<br/>Example: compromised server<br/>Blockchain uses BFT consensus"]

    Failures --> CF & OF & TF & BF
```