# D04 — Storage Internals
**Track: Deep Dive | How EBS, S3, and HDFS actually store your data**

---

## 1. EBS Architecture — Network Block Device Over NVMe

EBS is not a disk. It is a distributed storage system that presents itself to your EC2 instance as a local NVMe block device.

```mermaid
graph TB
    subgraph EC2_Host["EC2 Physical Host"]
        VM["Guest VM<br/>Sees: /dev/nvme0n1<br/>Reads/writes blocks"]
        NitroStorage["Nitro Storage Controller<br/>NVMe driver<br/>Exposes block device to VM"]
    end

    subgraph Storage_Network["EBS Storage Network (Dedicated, 25Gbps+)"]
        SN["Separate from VPC network<br/>Never competes with instance networking"]
    end

    subgraph EBS_Fleet["EBS Storage Fleet (Multiple AZs)"]
        Primary["Primary EBS Node<br/>SSD/HDD array<br/>Your data blocks"]
        Replica1["Replica 1<br/>Synchronous copy<br/>Same AZ, different rack"]
        Replica2["Replica 2<br/>Synchronous copy<br/>Different rack"]
    end

    VM -->|NVMe command| NitroStorage
    NitroStorage -->|NVMe over Fabric UDP| SN
    SN --> Primary
    Primary -->|Synchronous replicate| Replica1 & Replica2
    Primary -->|ACK| NitroStorage
    NitroStorage -->|ACK| VM
```

### EBS Write Path

```mermaid
sequenceDiagram
    participant App as Application in VM
    participant Kernel as Guest Kernel I/O Stack
    participant Nitro as Nitro Storage Controller
    participant EBS_P as EBS Primary Node
    participant EBS_R as EBS Replicas

    App->>Kernel: write(fd, buf, len)
    Kernel->>Kernel: Buffer in page cache (if buffered I/O)
    Kernel->>Nitro: Block I/O request: write sector X, data Y
    Nitro->>EBS_P: NVMeoF write: LBA X, data Y
    EBS_P->>EBS_R: Synchronous replication (before ACK)
    EBS_R->>EBS_P: ACK
    EBS_P->>Nitro: Write acknowledged
    Nitro->>Kernel: I/O complete
    Kernel->>App: write() returns
```

**Why EBS is slower than local SSD:** The round trip includes: Nitro card serialization + storage network traversal + EBS node processing + replication + ACK back. Total: ~0.5–2ms for gp3, vs ~0.05ms for local NVMe (i3 instances).

### EBS Volume Types

| Type | Media | Max IOPS | Max Throughput | Latency | Use Case |
|------|-------|---------|----------------|---------|----------|
| gp3 | SSD | 16,000 | 1,000 MB/s | ~1ms | Default, general use |
| gp2 | SSD | 16,000 | 250 MB/s | ~1ms | Legacy general use |
| io2 Block Express | SSD | 256,000 | 4,000 MB/s | <0.5ms | Critical DBs, SAP HANA |
| st1 | HDD | 500 | 500 MB/s | ~10ms | Throughput: big data |
| sc1 | HDD | 250 | 250 MB/s | ~10ms | Cold data, cheapest |

---

## 2. S3 Architecture — How Objects Are Actually Stored

S3 is not a file system. It is a **key-value object store** built on AWS's distributed infrastructure.

```mermaid
graph TB
    subgraph S3_Request_Path["S3 Request Path"]
        Client["Client: PUT s3://bucket/key"] --> DNS["Route 53: s3.amazonaws.com"]
        DNS --> S3_Frontend["S3 Frontend (Request Router)<br/>Any of thousands of nodes<br/>Stateless HTTP handlers"]
        S3_Frontend --> Metadata["Metadata Store<br/>DynamoDB-like system<br/>bucket→key→location mapping"]
        S3_Frontend --> DataStore["Data Store<br/>Objects chunked + stored<br/>across storage nodes"]
    end

    subgraph Storage_Layer["Underlying Storage"]
        DataStore --> Node1["Storage Node 1<br/>AZ-a shard"]
        DataStore --> Node2["Storage Node 2<br/>AZ-b shard"]
        DataStore --> Node3["Storage Node 3<br/>AZ-c shard"]
    end
```

### S3 Consistency Model

Pre-2020: S3 was **eventually consistent** for overwrite PUTs and DELETEs. You could read stale data after an update.

Post-2020: S3 is **strongly consistent** (read-after-write) for all operations. After a successful PUT, any GET returns the new object immediately.

**How strong consistency is achieved:** S3 uses a distributed metadata system that tracks object versions with strict serialization. Every write goes through a single metadata writer for that object's partition. Reads consult this metadata to find the latest version. This is expensive to implement at S3 scale — it took AWS years to achieve.

### S3 Durability — 11 Nines

```
How 99.999999999% durability is achieved:

1. Object uploaded to S3 frontend
2. Frontend routes to primary data node
3. Primary node writes to 3+ storage nodes synchronously
4. Each storage node is on different physical hardware, different rack, different AZ
5. Background scrubbing: periodic checksums verify data integrity
6. Background replication: if a node fails, S3 automatically re-replicates to maintain redundancy

To lose data: Need simultaneous failure of ALL copies + erasure coding segments
Probability: ~1 in 10^11 per object per year
```

### S3 Storage Classes — Internal Implementation

```mermaid
graph LR
    Standard["S3 Standard<br/>3 AZ synchronous replication<br/>Millisecond access"] -->|"Lifecycle policy"| IA["Standard-IA<br/>Same 3-AZ durability<br/>Retrieval fee per GB"]
    IA -->|"Lifecycle policy"| GlacierIR["Glacier Instant Retrieval<br/>Milliseconds but expensive retrieval"]
    GlacierIR -->|"Lifecycle policy"| GlacierFlex["Glacier Flexible<br/>1-12 hour retrieval<br/>Data on tape/cold storage"]
    GlacierFlex -->|"Lifecycle policy"| GDA["Glacier Deep Archive<br/>Cheapest<br/>12 hour retrieval<br/>Amazon uses literal tape"]
```

---

## 3. HDFS Write Path — How Hadoop Stores Your 1TB File

```mermaid
sequenceDiagram
    participant Client as HDFS Client
    participant NN as NameNode
    participant DN1 as DataNode 1 (AZ-a)
    participant DN2 as DataNode 2 (AZ-b)
    participant DN3 as DataNode 3 (AZ-c)

    Client->>NN: create("/data/file.txt", replication=3)
    NN->>NN: Allocate Block ID: 1001
    NN->>Client: Write to DN1 pipeline: [DN1, DN2, DN3]

    Note over Client,DN3: Block 1 Write Pipeline
    Client->>DN1: Write 128MB chunk (block 1)
    DN1->>DN2: Pipeline forward
    DN2->>DN3: Pipeline forward
    DN3->>DN2: ACK
    DN2->>DN1: ACK
    DN1->>Client: ACK (write confirmed to 3 nodes)

    Client->>NN: Block 1 complete, start Block 2
    NN->>Client: Write Block 2 to [DN2, DN4, DN5]
    Note over Client: Different nodes for each block — distributes load
```

**NameNode single point of failure:** The NameNode holds ALL metadata in RAM. Loss = cluster death. Mitigations:
- SecondaryNameNode: Periodic checkpoint (not HA — you lose recent changes)
- HA NameNode: Two NameNodes with ZooKeeper-based failover (production standard)

### HDFS Block Placement Strategy

```
Default placement rule (rack-aware):
Block 1 replica 1 → Local rack DataNode (write speed)
Block 1 replica 2 → Different rack DataNode (rack failure tolerance)
Block 1 replica 3 → Same different rack (throughput + redundancy)

Why? Two replicas on same rack = lost to single rack switch failure.
Two replicas on different racks = two network hops for writes, but survives rack failure.
```

---

## 4. Object Storage vs Block Storage vs File Storage

```mermaid
graph TB
    subgraph Block["Block Storage (EBS)"]
        B1["Fixed-size blocks<br/>No metadata beyond address<br/>Random read/write<br/>Must be mounted as filesystem"]
        B2["OS formats: ext4, NTFS, XFS<br/>Fast for databases<br/>Attached to ONE instance"]
    end

    subgraph Object["Object Storage (S3)"]
        O1["Variable-size objects<br/>Rich metadata (user-defined headers)<br/>HTTP PUT/GET/DELETE<br/>No mounting needed"]
        O2["Eventually → strongly consistent<br/>Unlimited scale<br/>Accessed by many clients simultaneously"]
    end

    subgraph File["File Storage (EFS/NFS)"]
        F1["File system abstraction<br/>Directories, paths<br/>POSIX compliance<br/>Mount on multiple instances (NFS)"]
        F2["Shared access: many EC2s<br/>Auto-scaling capacity<br/>Higher latency than EBS"]
    end
```

| Criteria | Block (EBS) | Object (S3) | File (EFS) |
|----------|------------|-------------|-----------|
| Access method | Mounted block device | HTTP REST API | NFS mount |
| Random I/O | Excellent | Not supported | Good |
| Multiple writers | No (one EC2) | Yes | Yes |
| Max size | 64 TB per volume | Unlimited | Unlimited |
| Latency | ~1ms | ~10–100ms | ~5ms |
| Best for | Databases, OS | Images, backups, logs | Shared app data |

---

## 5. EBS Snapshot Internals

EBS Snapshots are not full copies — they are **incremental block-level snapshots** stored in S3.

```mermaid
graph LR
    V1["EBS Volume<br/>Initial state: 100GB used"]
    S1["Snapshot 1<br/>Full: 100GB<br/>(deduped + compressed in S3)"]

    V1 -->|"snap-1"| S1

    V2["EBS Volume<br/>After changes: 10GB different"]
    S2["Snapshot 2<br/>Delta: 10GB changed blocks only<br/>References S1 for unchanged blocks"]
    S1 -->|"base"| S2
    V2 -->|"snap-2"| S2

    Restore["Restore from Snap 2<br/>= S1 blocks + S2 delta blocks<br/>Full 100GB volume"]
    S1 & S2 --> Restore
```

**Why deleting snapshots is safe:** AWS tracks reference counts. If you delete Snap 1 but Snap 2 references its blocks, those blocks are not deleted. Data moves to Snap 2. The EBS snapshot system handles this automatically.

---

## 6. Storage Failure Scenarios

| Scenario | EBS Behavior | S3 Behavior | HDFS Behavior |
|----------|-------------|-------------|---------------|
| Single disk fails | Replica serves I/O, background re-replication | Replica serves, background re-replication | DataNode reported dead to NameNode, NN triggers re-replication |
| AZ power outage | Volume unavailable (if single-AZ). io2 Multi-Attach = still available | No impact (cross-AZ replicated) | Block replicas in surviving AZs still accessible |
| Data corruption | Checksums detected, replica promoted | Background scrubbing detects, re-replicates | Checksums on every block, re-replication on mismatch |
| NameNode failure (HDFS) | N/A | N/A | Cluster down until NN restored (without HA) |
