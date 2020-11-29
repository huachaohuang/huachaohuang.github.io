---
layout: post
title: "Anna: KVS for Serverless Computing"
---

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

## About

**Anna** is a distributed key-value storage (KVS) system.
It aims to be a high-performance and cost-efficient storage system for serverless computing.
Anna is a key component in the [Hydro](https://hydro-project.github.io/) project, which is a framework for data-centric cloud programming research from the [RISE Lab](https://rise.cs.berkeley.edu/) at [UC Berkeley](https://berkeley.edu/).

This post shares the design of Anna based on the following papers:
- [Anna: A KVS for Any Scale](https://dsf.berkeley.edu/jmh/papers/anna_ieee18.pdf) (ICDE, 2018)
- [Autoscaling Tiered Cloud Storage in Anna](https://dsf.berkeley.edu/jmh/papers/anna_vldb_19.pdf) (VLDB, 2019)
- [Transactional Causal Consistency for Serverless Computing](https://dl.acm.org/doi/pdf/10.1145/3318464.3389710) (SIGMOD, 2020)

These papers revealed the development of Anna:
- Anna 2018 solved the scalability problem, it designed a KVS that runs well at any scale.
- Anna 2019 overcame some cost-performance limitations, it extended Anna into an autoscaling, multi-tier service for the cloud.
- Anna 2020 developed a cache system providing *Transactional Causal Consistency* for serverless computing (or FaaS) based on Anna.

The following sections extract more design details from these papers in order.

## Anna 2018: A KVS for Any Scale

### Introduction

> Anna is a partitioned, multi-mastered system that achieves high performance and elasticity via *wait-free execution* and *coordination-free consistency*.

The goal here is to design a KVS that can effectively scale from a single core to multicore to the globe.

### Related Work

Existing KVS optimized for different scales employ different programming models.

For single-server KVS, the shared-memory model is the architecture of choice for most of them.
Shared-memory architectures use synchronization mechanisms such as latches or atomic instructions to protect the integrity of shared data-structures, which can significantly inhibit multi-core scalability under contention.

Some systems employ a partitioned architecture, assigning non-overlapping shards of key-value pairs to each system thread.
However, threads in partitioned systems are prone to under-utilization if a subset of shards receive a disproportionate fraction of requests due to workload skew.

Redis uses a single-threaded model to avoid shared-memory synchronization overheads, but it cannot take any advantage of multi-core parallelism.
These systems provide only a single form of consistency, typically either linearizability or serializability, at the expense of performance and scalability.

For distributed KVS, the majority of them are not designed to run on a single multi-core machine and only support a single, relaxed consistency level.
Distributed replicated systems use state machine replication to provide strong consistency.
They enforce that replicas deterministically process requests according to a total order.
Totally ordered request processing requires waiting for global consensus at each step, and thus fundamentally limits the throughput of each replica-set.

### Design

Anna addresses the scalability and consistency problems with a simple architecture of *coordination-free actors* that perform state update via merge of *lattice-based composite data structures*.

#### Coordination-Free Actors

In contrast to using shared memory, a message-passing architecture consists of a collection of actors, each running on a separate CPU core.
Each actor maintains a private state that is inaccessible to other actors, and runs a tight loop in which it continuously processes client requests and inter-core messages from an input queue.

A message-passing system has two alternatives for managing each key: single-master and multi-master replication.

In single-master replication, each key is assigned to a single actor.
This prevents concurrent modifications of the key’s value, which in turn guarantees that it will always remain consistent.
However, this limits the rate at which the key can be updated to the maximum update rate of a single actor.

In multi-master replication, a key is replicated on multiple actors, each of which can read and update its own local copy.
To update a key’s value, actors can either engage in coordination to control the global order of updates, or can leave updates uncoordinated.
Coordination occurs on the critical path of every request, and achieves the effect of totally-ordered broadcast.
Although multiple actors can process updates, totally ordered broadcast ensures that every actor processes the same set of updates in the same order, which is semantically equivalent to single-master replication.

Anna combines asynchronous multi-master replication with lattice-based state management to remain scalable across both low and high conflict workloads while still guaranteeing consistency.

#### Lattice-Powered, Coordination-Free Consistency

The lattice mentioned in the paper employs a merge operator that satisfied three properties: **Commutativity**, **Associativity**, and **Idempotence** (ACI).

I am going to explain lattices with two simple examples here and leaves the formal definition to related papers.

For brevity, we consider only two actors *A* and *B* in the following examples.
Each actor maintains a logical clock.
A value version is represented in the form `(actor, clock, value)`.
Initially, A and B are both empty, that is, `(A, 0, nil)` and `(B, 0, nil)`.

The first example explains how to maintain causal consistency.
Supposes that two clients write to A and B concurrently. A writes `(A, 1, "a")` and B writes `(B, 1, "b")`.
A and B broadcast their changes to each other. So both of them see the two versions but in a different order.
Since `(A, 1, "a")` and `(B, 1, "b")` happened concurrently with no causality, we rely on the upper application to decide the final result.
Let's say that we choose `(B, 1, "b")` here.
Then another client writes a new value "c" to A.
A updates its state to `[(A, 2, "c"), (B, 1, "b")]` and then broadcasts to B.
Now B knows that A must has seen `(B, 1, "b")` before writing `(A, 2, "c")`. In other word, `(A, 2, "c")` happened after `(B, 1, "b")`.
So B can resolve the final result as `(A, 2, "c")` automatically.

The second example reveals the power of associativity.
Supposes that clients updates the same counter from A and B concurrently.
A and B broadcast updates to each other periodically and resolve the counter value independently.
Different versions from the same actor are resolved to the last write according to the logical clock (aka *last write wins*).
Versions from different actors are summed to get the final value of the counter. This is correct because addition satisfies associativity.

In the above two examples, both A and B resolve to the same final result without any coordination.
This is how Anna employs lattice-based composite data structures to provide different levels of consistency, while keeping the actor execution coordination-free.

#### Architecture

![Anna 2018 Architecture](/images/anna-2018-architecture.png)

This figure illustrates Anna’s architecture on a single server.

Each Anna server consists of a collection of independent threads, each of which runs the coordination-free actor model.
Each thread is pinned to a unique CPU core to avoid the overhead of preemption due to oversubscription of CPU cores.

Anna actors share no key-value state.
They employ consistent hashing to partition the key-space, and multi-master replication with a tunable replication factor to replicate data partitions across actors.
Anna actors engage in epoch-based key exchange to propagate key updates at a given actor to other masters in the key’s replication group.
Each actor’s private state is maintained in a lattice-based data-structure, which guarantees that an actor’s state remains consistent despite message delays, re-ordering, and duplication.

### Evaluation

The experiments justified Anna’s design decisions to run well at any scale.

#### Anna vs Redis

![Anna 2018 Redis](/images/anna-2018-redis-comparison.png)

Under low contention, Anna ran as well as Redis with a single thread and Redis Cluster with multiple threads.

Under high contention, Anna outperformed Redis with multi-master replication while Redis was subject to skewed utilization.

#### Anna vs Cassandra

![Anna 2018 Cassandra](/images/anna-2018-cassandra-comparison.png)

In a distributed setting, Anna ran as well as Cassandra with a single thread per node.

As the number of threads per node increases, Anna outperformed Cassandra significantly even though Cassandra used multi-threading too.

#### Scale Across Multiple Servers

![Anna 2018 Multiple Servers](/images/anna-2018-multiple-servers-scale.png)

This experiment showed that Anna can scale near-linearly as the number of threads increases in a single node and across multiple servers.

## Anna 2019: Autoscaling Tiered Cloud Storage

### Introduction

> Three key aspects of Anna’s new design: multi-master selective replication of hot keys, a vertical tiering of storage layers with different cost-performance tradeoffs, and horizontal elasticity of each tier to add and remove nodes in response to load dynamics.

Cloud providers offer various storage services that tuned to a unique point in that design space, making it well-suited to certain performance goals.
For example, AWS offers DynamoDB, Elastic Cache, Elastic Block Store (EBS), Elastic File System (EFS), and Simple Storage Service (S3).

However, application developers typically deal with a non-uniform distribution of performance requirements.
For example, many applications generate a skewed access distribution, in which some data is "hot" while other data is "cold".

Developers are inhibited by two key types of barriers when building applications with non-uniform workload distributions:
- **Cost-Performance Barriers**. Each of the services discussed above offers a different, fixed tradeoff of cost, capacity, latency, and bandwidth.
- **Static Deployment Barriers**. Cloud providers offer very few truly autoscaling storage services. Elastic Cache requires system administrators to allocate and deallocate instances manually. S3 auto scales to match data volume but ignores workload while DynamoDB offers workload-based autoscaling but is prohibitively expensive to scale to a memory-speed service.

This paper extends Anna to remove its cost-performance and static deployment barriers to span the cost-performance design space more flexibly, enabling it to adapt dynamically to workload variation in a cloud-native setting.

### Design

#### Architecture

![Anna 2019 Architecture](/images/anna-2019-architecture.png)

This figure shows Anna's architecture built on AWS components.

In the initial implementation and evaluation, Anna’s memory tier stores data in RAM attached to AWS EC2 nodes.
The flash tier leverages the Elastic Block Store (EBS), a fault-tolerant block storage service that masquerades as a mounted disk volume on an EC2 node.

The storage kernel (labeled as Anna v0) is extended to support multiple storage media and three new subsystems are designed:
- The routing service is a stateless client-facing API that provides a stable abstraction above the internal dynamics of the system.
- The monitoring system and policy engine are the internal services responsible for responding to workload dynamics and meeting SLOs.
- The cluster management system is another stateless service that modifies resource allocation based on decisions reached by the policy engine.

#### Storage System

![Anna 2019 Storage Kernel](/images/anna-2019-storage-kernel.png)

Anna requires maintaining certain metadata to help the policy engine adapt to changing workloads.
Anna manages three distinct kinds of metadata.

First, every storage tier has two hash rings.
A global hash ring, G, determines which nodes in a tier are responsible for storing each key.
A local hash ring, L, determines the set of worker threads within a single node that is responsible for a key.

Second, each individual key K has a replication vector of the form $$[<R_1, ..., R_n>, <T_1, ..., T_n>]$$.
$$R_i$$ represents the number of nodes in tier i storing key K, and $$T_i$$ represents the number of threads per node in tier i storing key K.
In the current implementation, i is either M (memory tier) or E (EBS tier).
Each key has a default replication vector of the form $$[<1, k>, <1, 1>]$$, meaning that it has one memory tier replica and k EBS-tier replicas.

Anna also tracks monitoring statistics, such as the access frequency of each key and the storage consumption of each node.
This information is analyzed by the policy engine to trigger actions in response to variations in workload.

#### Policy Engine

![Anna 2019 Policy Engine](/images/anna-2019-policy-engine.png)

Anna supports three kinds of SLOs: an average request latency in milliseconds, a cost budget (B) in dollars/hour, and a fault tolerance (k) in the number of replicas.

Anna ensures there will never be fewer than k + 1 replicas of each key to achieve the fault tolerance goal.
If a latency SLO is specified, Anna minimizes cost while meeting the latency goal.
If the budget is specified, Anna uses no more than $B per hour while maximizing performance.

##### Cross-Tier Data Movement

Anna’s policy engine uses its monitoring statistics to calculate how frequently each key was accessed in the past T seconds.

If a key’s access frequency exceeds a configurable threshold, P, and all replicas currently reside in the EBS tier, Anna promotes a single replica to the memory tier.
If the key’s access frequency falls below a separate internal threshold, D, and the key has one or more memory replicas, all replicas are demoted to the EBS tier.

If the aggregate storage capacity of a tier is full, Anna adds nodes to increase capacity before performing data movement.
If the budget does not allow for more nodes, Anna employs a least-recently-used caching policy to demote keys.

##### Hot-Key Replication

When the access frequency of a key stored in the memory tier increases, hot-key replication increases the number of memory-tier replicas of that key.

The policy engine classifies a key as "hot" if its access frequency exceeds an internal threshold, H, which is s standard deviations above the mean access frequency.

### Evaluation

Let's discuss some interesting experimental results here.

#### Cost-effectiveness Comparison

![Anna 2019 Cost-effectiveness Comparison](/images/anna-2019-cost-effectiveness-comparison.png)

Under low contention, Anna consistently outperforms both Masstree and ElastiCache because Anna’s thread-per-core coordination-free execution model.

Under high contention, Anna’s throughput increases linearly with cost, while both ElastiCache and Masstree plateau.

Anna selectively replicates hot keys across nodes and threads to spread the load, enabling this linear scaling.

#### Changing Workload

![Anna 2019 Changing Workload](/images/anna-2019-changing-workload.png)

This experiment shows Anna's response to changing workload.

At minute 3, Anna replicates the highly contended keys and meets the latency SLO (the dashed red line).

At minute 13, Anna finds that all nodes are occupied with client requests and triggers the addition of four new nodes to the cluster.
It takes 5 minutes for the new nodes to join the cluster.

At minute 18, the new nodes come online and trigger a round of data repartitioning, seen by the brief latency spike and throughput dip.

At minute 28, the load is reduced and Anna removes nodes to save cost.

#### Cost Budget and Latency Objective

![Anna 2019 Cost and Latency](/images/anna-2019-cost-and-latency.png)

This figure shows the cost-performance tradeoffs.
As we increase the budget, latency improves; as we increase the latency objective, the required cost reduces.

One thing that catches my attention here is that the reduction of latency and the increase of cost budget are not proportional.

If we reduce the latency objective from 100ms to 50ms, the cost doesn't increase very much.
However, if we want to reduce the latency objective to 5ms, the cost will increase significantly.

I think most customers is willing to spend a bit more money to reduce the latency to 50ms.
But they may think over the necessity to further reduce the latency since it costs a lot more money.
This is good because customers can make very intuitive tradeoffs according to their requirements.

## Anna 2020: Transactional Causal Consistency

### Introduction

This paper presents *HydroCache*, a distributed caching layer attached to each node in a FaaS system.
HydroCache provides causal consistency for all I/Os within a given transaction even if it runs across multiple physical sites.

FaaS platforms achieve flexible autoscaling by *disaggregating* the compute and storage layers, so they can scale independently.
For example, AWS Lambda typically uses AWS S3 or DynamoDB as the autoscaling storage layer.
This design comes at the cost of high-latency I/O — often orders of magnitude higher than attached storage.

A natural solution is to attach caches to FaaS compute nodes to eliminate the I/O latency for data that is frequently accessed from remote storage.
However, this raises challenges around maintaining consistency of the cached data — particularly in the context of multi-I/O applications that may run across different physical machines with different caches.

HydroCache simultaneously provides low-latency data access and introduces *Multisite Transactional Causal Consistency (MTCC)* protocols to guarantee data consistency for requests that execute on multiple nodes.

### Design

![Anna 2020 Cloudburst Architecture](/images/anna-2020-cloudburst-architecture.png)

This figure shows an overview of the system architecture, which consists of a high-performance key-value store (KVS) Anna and a function execution layer Cloudburst.

In Cloudburst, all requests are received by a scheduler and routed to worker threads based on compute utilization and data locality heuristics.
Cloudburst deploys KVS-aware caches on the same nodes as the compute workers, allowing for low-latency data access.
The compute threads on a single machine interact with one HydroCache instance, which retrieves data for the function executors as necessary.

The key challenge of HydroCache is to maintain transactional causal consistency for functions executed in multiple nodes.

This paper presents the definitions and protocols quite formally and technically, which are not easy to extract and refine in this post.
So I will try to introduce the core ideas informally here so that readers can have a basic understanding of the design without tangling with the mathematics.

#### Transactional Causal Consistency (TCC)

TCC guarantees the causal consistency of reads and writes for a set of keys.

Specifically, given a read set $$R$$, TCC requires that $$R$$ forms a causal snapshot.
That is, for any pair of keys $$a_i$$, $$b_i$$ in $$R$$, if $$a_k$$ is a dependency of $$b_i$$, then $$a_i$$ must be equal to $$a_k$$ or happend after $$a_k$$ (aka $$a_i$$ supersedes $$a_k$$).

For example, in a social network, Alice updates her photo permission and then posts a photo.
TCC ensures that no one can view the new photo before revealing the new permission, since the new permission is a dependency of the new photo.

TCC also ensures atomic visibility of written keys; either all writes from a transaction are seen or none are.

Now let's see how HydroCache achieves TCC for individual functions executed at a single node.

HydroCache initially contains no data. When a function requests a key $$b$$ that is missing, the cache fetches a version $$b_j$$ from Anna.
Before exposing it to the functions, the cache checks to see if all dependencies of $$b_j$$ are superseded.
If a dependency $$a_i$$ is not superseded, the cache fetches versions of $$a$$, $$a_m$$, from Anna until $$a_i$$ is superseded by $$a_m$$.
HydroCache then recursively ensures that all dependencies of $$a_m$$ are superseded.
This process repeats until the dependencies of all new keys are superseded.
Moreover, HydroCache guarantees atomic visibility by making keys in the same transaction mutually dependent.

#### Multisite Transactional Causal Consistency (MTCC)

Although the design above guarantees TCC for individual functions executed at a single node, this is insufficient for serverless applications.

A DAG in Cloudburst consists of multiple functions, each of which can be executed at a different node.
To achieve TCC for the DAG, we must ensure that a read set spanning multiple physical sites forms a distributed snapshot.

The goal of MTCC protocols is to ensure that each DAG observes a snapshot as computation moves across nodes.
This requires care, as reading arbitrary versions from the various local cache may not correctly create a snapshot.

In a linear function execution flow, we have an "upstream" function in the DAG that has completed with its version set $$R_u$$, and an about-to-be-executed "downstream" function whose version set $$R_d$$ is still unbound.
MTCC need to ensure that the union of $$R_u$$ and $$R_d$$ forms a causal snapshot.
This requires two properties: $$R_d$$ supersedes the $$R_u$$'s dependencies $$R_u.deps$$, and $$R_u$$ supersedes $$R_d$$'s dependencies $$R_d.deps$$.

The paper discusses a set of protocols to address this challenge while minimizing coordination and data shipping overheads across caches.

The **Optimistic (OPT)** protocol eagerly start running the functions in a DAG and check for violations of the snapshot property at the time of each function execution.
When violations are found at some node, it potentially adjusts the version set being read at that node to re-establish the snapshot property for the DAG.

The **Conservative (CON)** protocol is the opposite of OPT.
Instead of lazily validating read sets as the DAG progresses, the scheduler coordinates with all caches involved in the DAG request to construct a distributed snapshot before execution.

The **Hybrid (HB)** protocol combines the benefits of the OPT and CON protocols.
HB (run by the scheduler) starts the OPT subroutine and simultaneously performs a simulation of OPT.
The CON subroutine is activated only when the simulation aborts.
HB includes some key optimizations that enable the two subroutines to cooperate to improve performance: pre-fetching, early abort, and function result caching.

## Conclusion

I am very interested in serverless database recently. That's why these Anna series papers draw my attention.

As a database engineer, the very first impression Anna gives me is that the provided consistency is not strong enough as a general database for many applications.
Maybe because I've gotten used to stronger consistency or transaction isolations at work.

However, after thinking about the big picture of the Hydro project, I realize that Anna is built for serverless computing.
Specifically, Anna's mission is to fill the gap between FaaS and existing storage services on the cloud.

From this perspective, I think Anna does a good job for the serverless ecosystem.
Anna accelerates data exchanges between functions by orders of magnitude, while providing relatively strong consistency guarantees.
