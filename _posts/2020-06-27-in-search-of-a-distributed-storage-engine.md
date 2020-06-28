---
layout: post
title: In Search of a Distributed Storage Engine
---

This post introduces the architecture of a *distributed storage engine* that satisfies various storage requirements.

## Local Storage Engine

Let's start with the architecture of a local storage engine.

A classic LSM-Tree storage engine (e.g. LevelDB, RocksDB) looks like this:

![LSM-Tree Storage Engine](/images/lsm-tree-storage-engine.png)

All writes go to a *Write Ahead Log* (WAL) first and then applied to a memtable.
When the size of a memtable reaches a threshold, the memtable is flushed to the drive as an SST file.
SST files are organized into multiple levels with an increasing size threshold.
When the size of a level reaches its threshold, the level is compacted with the next level.
During compaction, the input files are merged, sorted, and compressed into new output files, which needs a lot of computation.

As for reads, hot data are cached in memory to speed up performance.
We can say that a memtable is also a kind of cache, which holds the newly written data.

The components described above have very different characteristics.
A WAL is an append-only file, which is mostly written and occasionally read (on restart).
All SST files are static and read-only.
Memtable and cache only stay in memory.
Flush and compaction consume a lot of CPU.

A local storage engine binds all these components together.
This is fine because a local storage engine can only utilize the resource on a single machine.
However, in a distributed environment (cloud), this way is not good for resource distribution, because we can't scale an individual component on demand.
For example, if I want to speed up read performance, I can't just add more memory for the cache.
I also have to add more CPU and IO for other components so that I can deploy more instances of the storage engine.
Similarly, I can't just add more CPU to speed up compaction.

This is why we should decompose these components in a distributed storage engine.

## Distributed Storage Engine

People usually expect more from a distributed storage engine than a local one.
Some common requirements of a distributed system include:

- High Availability. Data should be replicated to prevent a single point of failure.
- High Scalability. Data should be scheduled to different nodes for load balancing.
- Data Consistency. Some applications are fine with eventual consistency, but some require strong consistency.
- Data Analysis. It should provide a convenient and efficient way to run large scale data analysis jobs.

Moreover, a distributed storage engine should not limit the ability of domain-specific optimizations.
For example, a time-series database has some very different characteristics, which provide a lot of opportunities to optimize the data format, compression algorithms, and compaction policies, etc.

Now let's see how we can decompose the components of a local storage engine to form a distributed storage engine.
We assume that data are partitioned into a lot of shards according to some application strategy, mostly hash-based or range-based.
From the perspective of a local storage engine, each shard consists of a WAL, a memtable, a cache, and some SST files.
What we need to do here is to distribute these components of all shards in a way that satisfies our requirements.

### Distributed Journal System

First is the WAL.

We need a distributed journal system to handle writes from all shards.
The journal system provides a WAL for each shard and guarantees the necessary availability and consistency.
For example, we can use a consensus algorithm like Raft or Paxos to guarantee high availability and strong consistency. Or we can use a simple asynchronous replication strategy for eventual consistency.

![Distributed Journal System](/images/distributed-journal-system.png)

### Distributed Cache System

The distributed cache system provides various APIs for upper-level applications.

![Distributed Cache System](/images/distributed-cache-system.png)

Many applications love to use Redis-like APIs because those APIs provide the data structures that match their scenarios best.
This is why there are so many Redis API compatible storage systems, either open-sourced or homebrewed.

However, most of these systems store data in some other general-purpose database, which doesn't give them control to the underlying storage engine and limits the possibility for further optimizations.
Of course, you can build the whole system from the ground up so that you can do whatever you want.
But that requires a lot of effort (if not too much), especially when you only want to customize some data format or file layout.

With these considerations, the distributed cache system needs to provide flexibility for customization.

In the cache system, each shard controls its API, cache, and memtable.
Shards are stateless, they can be distributed to different nodes easily.
You can scale the cache system by adding more memory nodes without any data migration.
You don't need to worry about cache consistency either, because all writes of a shard go to the same node in the cache system.
A shard first writes to the distributed journal system and then updates its memtable.
When the size of a memtable reaches the threshold, the memtable is flushed to the distributed file system (we will talk about it later).
The beauty here is that a shard has full control of the data format and the layout of its files.
A shard can use any data structure for its memtable, flush the memtable into any file format and compact the files using any strategy.
You can also flush a memtable into multiple files with different formats for different workloads.
For example, a record-oriented file format for OLTP and a column-oriented file format for OLAP.
In this case, we can achieve real-time OLAP. The newest written data are accessible in the memtable and all other data are accessible in the distributed file system, no other synchronization mechanisms are needed.

### Distributed File System

The distributed file system stores all files flushed from the distributed cache system.

![Distributed File System](/images/distributed-file-system.png)

The distributed file system guarantees its availability and scalability.
Since files in the file system are static, they can be replicated by simply copying them to different places.
It doesn't require a complex consensus algorithm to ensure data consistency.
And it can just move files to different places for load balancing.
For example, if a file is too hot, the file system can create more replicas of that file in other nodes to split the traffic.

Besides, the distributed file system can also provide some other nice features.
For example, it is reasonable to separate files for online processing and files for offline analysis into different sets of nodes so that they will not interfere with each other.
And most data analysis jobs can benefit from a lightweight snapshot mechanism.
The distributed file system can create a snapshot of some files by adding some kind of references without copying the files.

On the other side, the problem of a distributed file system versus a local file system is that you need to access your data through the network.
That adds some latency and potential bottleneck in the bandwidth.
But I don't very worry about it because the network is becoming more and more powerful, 100Gbps bandwidth is not uncommon.
That said, it is also a good idea to add some kind of coprocessor mechanism to the distributed file system. Coprocessors let us push down the expressions to the data instead of transferring large amounts of data through the network.

### Distributed Job System

Finally, we need a distributed job system to schedule different kinds of background jobs, like compaction and compression.

![Distributed Job System](/images/distributed-job-system.png)

A shard can compact its files in the distributed file system by submitting a compaction job to the distributed job system.
The job system needs to find a proper node to run the job.
These background jobs are mostly CPU-bound. We can utilize some hardware acceleration by adding some GPU and FPGA nodes to the system.

A dedicated job system also empowers some heavy-weight compression algorithms, which may be too expensive for a local storage engine, especially during peak hours.

## Summary

Let's put these systems together to review the architecture of the distributed storage engine:

![Distributed Storage Engine](/images/distributed-storage-engine.png)

The core idea here is to decompose the components with different characteristics to a standalone system so that they can be reused and scale independently.

The distributed cache system requires a lot of CPU and memory. It can be scaled by adding more CPU and memory with no data migration.
The distributed journal system and file system are mostly IO-bound (Well, if we don't consider the coprocessor).
But the journal system is more latency-sensitive than the file system because it handles frontend write requests.
So it is a cost-efficient manner to use high-end drives (e.g. Optane SSD) in the journal system to reduce latency while using ordinary drives (e.g. SATA SSD) in the file system.
The distributed job system can schedule background jobs to any place with enough CPU resources.

Another important thing is that the distributed cache system fully controls its data, which makes all domain-specific optimizations possible.
We can optimize an application with very different assumptions by designing a new set of APIs, file formats, and maybe background jobs, without worrying about the other parts of the whole system.
This is way simpler than building a new storage system again.
