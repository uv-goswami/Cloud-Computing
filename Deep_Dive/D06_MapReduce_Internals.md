# D06 — MapReduce Internals
**Track: Deep Dive | Scheduler, shuffle, sort, and why Spark replaced it**

---

## 1. MapReduce Execution Phases — Detailed

```mermaid
flowchart TD
    subgraph Input["Input Phase"]
        HDFS["HDFS 1TB file<br/>Divided into 128MB splits"]
        Split1["Split 1: bytes 0-128MB"]
        Split2["Split 2: bytes 128-256MB"]
        SplitN["Split N: last bytes"]
    end

    subgraph Map_Phase["Map Phase (Runs in parallel on DataNodes)"]
        M1["Mapper 1<br/>Input: Split 1 (key=offset, val=line)<br/>Output: (word, 1) pairs<br/>Written to LOCAL disk (not HDFS)"]
        M2["Mapper 2<br/>Input: Split 2<br/>Output: (word, 1) pairs"]
        MN["Mapper N"]
    end

    subgraph Shuffle["Shuffle Phase (Most Expensive)"]
        Part["Partitioner<br/>hash(word) % num_reducers<br/>Determines which reducer gets which words<br/>All 'apple' keys go to Reducer 1"]
        Sort["Sort<br/>Within each partition: keys sorted<br/>So reducer gets sorted stream"]
        Transfer["Network Transfer<br/>Mapper outputs sent to correct Reducer nodes<br/>This is the network bottleneck"]
    end

    subgraph Reduce_Phase["Reduce Phase"]
        R1["Reducer 1<br/>Input: sorted (apple,[1,1,1]), (ant,[1,1])<br/>Output: apple=3, ant=2<br/>Written to HDFS"]
        R2["Reducer 2<br/>Input: sorted (banana,[1,1])<br/>Output: banana=2"]
    end

    Input --> Map_Phase
    Map_Phase --> Shuffle
    Shuffle --> Reduce_Phase
```

---

## 2. The Shuffle — Why It's the Bottleneck

The shuffle is the most expensive phase in MapReduce:

1. Every Mapper writes output to local disk (not HDFS — local disk is faster)
2. Partitioner decides: for each (key, value) pair, which Reducer should handle this?
3. The framework moves data from every Mapper node to every Reducer node over the network
4. At scale (1000 mappers × 100 reducers = 100,000 network connections)

```mermaid
graph LR
    subgraph Mappers["100 Mapper Nodes"]
        M1["M1 output:<br/>apple: partition 1<br/>banana: partition 2<br/>cherry: partition 3"]
        M2["M2 output:<br/>apple: partition 1<br/>ant: partition 1<br/>banana: partition 2"]
        M100["M100 output:<br/>..."]
    end

    subgraph Reducers["10 Reducer Nodes"]
        R1["R1: receives all 'a*' words<br/>from ALL 100 mappers"]
        R2["R2: receives all 'b*' words<br/>from ALL 100 mappers"]
        R10["R10: receives all 'z*' words"]
    end

    M1 & M2 & M100 -->|"100 × 10 = 1000 network connections"| R1 & R2 & R10
```

**Why shuffle bottlenecks:** Network I/O at scale. 100 mappers × 10GB each = 1TB transferred. 10 reducers each receive 100GB from 100 sources. This requires careful bandwidth management.

---

## 3. MapReduce vs Spark — The Internal Difference

```mermaid
graph LR
    subgraph MapReduce_Flow["MapReduce: Disk-Based"]
        MR_Input["HDFS Input"] --> MR_Map["Map (disk output)"] --> MR_HDFS1["HDFS write"] --> MR_Shuffle["Shuffle (disk I/O)"] --> MR_HDFS2["HDFS write"] --> MR_Reduce["Reduce (disk output)"] --> MR_Out["HDFS Output"]
    end

    subgraph Spark_Flow["Spark: Memory-Based (RDDs)"]
        SP_Input["HDFS Input"] --> SP_RDD["RDD in Memory"] --> SP_Transform["transformations (lazy)"] --> SP_Action["Action (collect/save)"] --> SP_Out["HDFS Output"]
        SP_RDD -.->|"Spill to disk only if OOM"| SP_Disk["Disk (fallback)"]
    end
```

**For iterative algorithms (ML training):** MapReduce reads from HDFS every iteration. Spark caches the dataset in memory — only one HDFS read for all iterations. 10 iterations = 10x disk I/O for MR vs 1x for Spark.

**Spark fault tolerance:** If an RDD partition is lost (node crash), Spark recomputes it from its lineage (knows all transformations applied to input). No need to store every intermediate result.

---

## 4. YARN — Resource Management

YARN (Yet Another Resource Negotiator) separates resource management from computation.

```mermaid
graph TB
    subgraph YARN_Architecture["YARN Architecture"]
        Client["Job Client"]
        RM["Resource Manager<br/>Cluster-level resource tracking<br/>Application lifecycle"]
        NM1["Node Manager (Host 1)<br/>Container launch/monitor<br/>Heartbeat to RM"]
        NM2["Node Manager (Host 2)"]
        NM3["Node Manager (Host 3)"]
        AM["Application Master<br/>Per-job, runs IN cluster<br/>Negotiates containers with RM<br/>Assigns tasks to containers"]
    end

    Client --> RM
    RM --> AM
    AM --> NM1 & NM2 & NM3
    NM1 & NM2 & NM3 -->|heartbeat| RM
```

**Why YARN matters:** Before YARN, Hadoop's JobTracker did resource management AND job scheduling — a bottleneck. YARN separates these. A YARN cluster can run MapReduce, Spark, Flink, and other frameworks simultaneously.