# Discord Search Architecture — Engineering Case Study

## Phase 1 — Problem & Requirements

### 1. Problem

Discord wanted to provide **message search** across its servers using Elasticsearch.

### 2. Search Freshness Requirement

An important requirement was that search did **not** need to be immediately up to date.

A Discord server could potentially go hours without executing a search query. Therefore, newly indexed messages did not need to become searchable immediately.

### 3. Engineering Trade-off

This requirement allowed Discord to make an important trade-off:

> **Sacrifice search freshness in exchange for more efficient indexing.**

Instead of spending resources making every newly indexed message immediately searchable, the system could prioritize efficient indexing.

### 4. Central Engineering Question

This creates the main question explored in this case study:

> **How can a large-scale message search system efficiently index messages without spending unnecessary resources making newly indexed data searchable immediately?**

### 5. Why This Requirement Matters

This requirement becomes important later in the architecture.

Discord initially encountered unexpectedly high:

- CPU usage
- Disk usage

during large-scale indexing.

The rest of the case study examines how Discord investigated this behaviour, identified the role of Elasticsearch's refresh mechanism, and redesigned the refresh process around the actual search-freshness requirement.

---

## Source

This case study is based on Discord's engineering article about building its search infrastructure. - [https://discord.com/blog/how-discord-indexes-billions-of-messages?trk=public_post_comment-text]


## Phase 2 — Architecture

### 2.1 Queue

Messages were placed into a queue instead of being indexed individually. This allowed Discord to collect messages and use Elasticsearch's bulk indexing operation, which is more efficient.

**Key idea:** The queue is a temporary buffer, not an Elasticsearch index.

### 2.2 Index Worker

Index workers consumed batches from the queue and performed indexing. The worker also determined the destination based on `guild_id`.

### 2.3 Redis — Routing Cache

Redis stored the routing mapping:

`guild_id → Elasticsearch destination`

Redis stored routing metadata, not message data.

### 2.4 Cassandra — Persistent Mapping

Cassandra acted as the persistent source of truth for routing mappings, while Redis provided faster cached lookups.

`Cassandra → persistent mapping`  
`Redis → cached mapping`

### 2.5 etcd — Service Discovery

etcd was used for Elasticsearch service discovery.

`Redis → Which Elasticsearch destination?`  
`etcd → Where are the Elasticsearch nodes?`

### 2.6 Elasticsearch

Elasticsearch indexes stored searchable message documents. An Elasticsearch index can contain native Elasticsearch shards and replicas; these are different from Discord's application-level **Shard**.

Discord configured each index with one native shard and relied on Elasticsearch for replication and balancing.

### 2.7 Complete Architecture

```text
                         DISCORD
                            │
                       New messages
                            │
                            ▼
                      ┌───────────┐
                      │   Queue   │
                      └─────┬─────┘
                            │
                       N messages
                            │
                            ▼
                    ┌───────────────┐
                    │ Index Worker  │
                    └───────┬───────┘
                            │
                    Determine destination
                            │
             ┌──────────────┴──────────────┐
             │                             │
             ▼                             ▼
       ┌───────────┐                  ┌─────────┐
       │   Redis   │                  │  etcd   │
       │ guild →   │                  │ ES node │
       │ destination│                 │ discovery│
       └─────┬─────┘                  └────┬────┘
             │                             │
             └──────────────┬──────────────┘
                            ▼
                    ┌───────────────┐
                    │ Elasticsearch │
                    │    Cluster    │
                    └───────┬───────┘
                            │
                     Searchable data
                            │
                            ▼
                         Search
```

Cassandra sits behind Redis as the persistent source of truth for the routing mapping.

**End-to-end:** Message → Queue → Worker takes batch → use `guild_id` → Redis routing → etcd node discovery → bulk index → Elasticsearch.

## Phase 3 — Production Failure

Discord tested the system by indexing 1,000 of the largest servers on a 3-node Elasticsearch cluster. CPU usage was unexpectedly high, while disk usage grew much faster than expected.

They stopped the indexing jobs. By the next morning, disk usage had dropped significantly, but searches still worked and the data had not been lost.

Discord hypothesized that Elasticsearch's 1-second refresh interval was causing frequent refreshes across many indexes, producing many small Lucene segments and increasing CPU/disk usage.

While the cluster was idle, Elasticsearch merged these small segments into larger, more space-efficient segments, explaining the drop in disk usage.

This led Discord to test whether increasing the refresh interval could reduce the resource overhead.

## Phase 4 — Experiment & Engineering Decision

Discord dropped the existing indexes, increased the Elasticsearch refresh interval to an arbitrarily large value, and re-indexed the same servers.

### Results

- CPU usage dropped to almost nothing during ingestion.
- Disk usage no longer grew at an alarming rate.

This supported their hypothesis that frequent refreshing was responsible for much of the unnecessary resource usage.

### Engineering Decision

Discord concluded that near-real-time search freshness was unnecessary for their use case. They could therefore trade search freshness for significantly more efficient indexing.

> **Frequent refresh → higher CPU and disk pressure**
>
> **Long refresh interval → much lower resource usage**

### Engineering Pattern

**Observation → Hypothesis → Controlled experiment → Measured result → Engineering decision**

## Phase 5 — Application-Level Refresh Control

The long Elasticsearch refresh interval reduced indexing overhead, but it introduced a new problem: newly indexed messages could remain temporarily unsearchable.

Discord solved this by moving refresh control to the application layer.

### 5.1 Track Updates

After indexing a batch of messages, Discord recorded in Redis which Shards had received new messages.

A Shard with newly indexed messages was considered **dirty**.

### 5.2 Refresh Only When Required

When a search request arrived:

1. Determine the Shard containing the requested guild.
2. Check Redis to determine whether that Shard is dirty.
3. If dirty, refresh the Elasticsearch index.
4. Mark the Shard as clean.
5. Execute the search.

This avoided refreshing Elasticsearch after every indexing operation while still making recently indexed messages searchable when required.

### 5.3 Safety Mechanism

The Redis refresh state expired after one hour.

Elasticsearch also automatically refreshed after one hour, providing a fallback if the Redis state was lost.

### Engineering Principle

> **Do not continuously pay the cost of freshness. Refresh when freshness is actually required.**

## Phase 6 — Scaling the System

Once the architecture was working efficiently, Discord needed to support much larger search workloads.

At the scale described in the article, the system ran across multiple Elasticsearch clusters and nodes. New clusters could be added as more servers needed to be indexed, with Shards distributed across the available clusters.

Discord monitored four important capacity metrics:

- `heap_free` — low free heap could lead to heavy garbage collection and potentially stop-the-world pauses.
- `disk_free` — low disk space required adding nodes or increasing storage.
- `cpu_usage` — sustained high CPU indicated the need for more capacity.
- `io_wait` — high values indicated storage I/O was becoming a bottleneck.

The key scaling principle was to add capacity based on the actual resource becoming constrained rather than treating Elasticsearch as a single fixed cluster.

