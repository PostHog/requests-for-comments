# Request for comments: Fixing Delta Lake OOMs in warehouse sources

Author: Estefania Rabadan

Date: May 28, 2026

> **Status:** Draft. Looking for feedback on which path(s) to pursue.

## Problem

Our data warehouse import pipeline writes incoming data to Parquet on S3 and then merges it into Delta Lake tables.
The merge is where things break: when the partition we're merging into is large, the pod runs out of memory and dies.
Because the loader runs many merges concurrently on one pod, a single oversized merge can take down every other job sharing that pod, which then all retry.

A few facts that frame everything below:

- We use [delta-rs](https://github.com/delta-io/delta-rs) (the Rust/Python implementation).
  The merge OOMs because delta-rs's `MergeBarrier` buffers the **target** side of the join in memory, it holds onto records for each target file until a matching insert/update/delete is seen or the input is exhausted ([delta-rs#2573](https://github.com/delta-io/delta-rs/issues/2573)).
  So peak memory is driven by how much of the existing partition gets scanned, not by the (already-bounded) batch we're writing.
- Partition pruning only helps if the merge predicate names the partition value explicitly, which our partitioned path already does ([delta-rs#3786](https://github.com/delta-io/delta-rs/issues/3786)).
- delta-rs does **not** support deletion vectors or liquid clustering yet ([deletion vectors](https://milescole.dev/data-engineering/2024/11/04/Deletion-Vectors.html), [liquid clustering](https://delta.io/blog/liquid-clustering/)).

**Scope:** All this is built on top of PipelinesV3

---

## 1. Stick with Delta, no infra changes

These all live inside the existing `warehouse-sources-load` consumer. No new deployments, no new services.

### 1.1 Pre-flight partition-size guard

Before merging, read the size of the partition(s) we're about to merge into using `DeltaTable.get_add_actions()`, which reads the transaction log only, no data scan ([docs](https://delta-io.github.io/delta-rs/api/delta_table/)).
Phase one is **observe only**: emit the size distribution as a metric so we understand how bad things actually are.
Phase two uses that signal to route oversized partitions to a safer path (1.4) or flag them for repartitioning (or even repartition automatically, but this is out of scope here).

It changes no behavior on its own, but it's the measurement foundation that we'll need later.

**Pros**

- Nearly free (metadata read), zero behavior change, and gives us the partition-size data we currently don't have.
- It's the routing signal every "heavy path" option needs.

**Cons**

- Doesn't fix anything by itself.
- Size is a proxy: row-group pruning can read less, wide rows can blow up more, so thresholds need calibration.

### 1.2 Tune writer properties on every merge call

We currently call delta-rs `.merge()` with default writer properties.
Passing `write_batch_size` and `max_row_group_size` (e.g. 8k / 128k rows) caps how much the writer holds in memory while building output row groups ([writing docs](https://delta-io.github.io/delta-rs/usage/writing/)).

**Pros**

- Smallest possible change.
- Applies uniformly to every source type and sync.

**Cons**

- Bounds the writer's contribution but not the dominant target-side scan, so this is partial mitigation only (helful for wide rows).
- Smaller row groups mean slightly more metadata and a bit more downstream read/compaction overhead.

### 1.3 Pass source as a `RecordBatchReader`

Instead of handing the merge a fully materialized PyArrow table, stream it in record-batches so the source side is consumed incrementally.

**Pros**

- Also helps the wide-row edge case.
- Same merge semantics.

**Cons**

- Our source batches are already capped (~200 MB), and the OOM is target-side, so the win is marginal for the common case.
- Our code still has to load the whole batch into memory to split it by partition, so streaming the source only trims a small internal copy.

### 1.4 Adaptive "partition overwrite" fallback

When a partition is too big to merge safely, skip `.merge()` and instead do a streaming anti-join: build a key set from the (small, bounded) incoming batch, stream the existing partition while dropping overwritten rows, append the new rows, and atomically swap the partition with `write_deltalake(mode="overwrite", predicate=...)` (replaceWhere — [docs](https://delta-io.github.io/delta-rs/usage/writing/), [blog](https://delta.io/blog/delta-lake-replacewhere/)).
The trick is inverting the join: we build the hash on the *small* side and stream the *big* side, which is the opposite of what delta-rs's merge does.
This is the same "break work into smaller, predictable rewrites" advice from [this MERGE-at-scale write-up](https://cdelmonte.medium.com/delta-lake-merge-is-not-a-simple-upsert-what-actually-happens-at-scale-42ec8737a307) and [this one on fast upsert pipelines](https://medium.com/@harsh11csb/why-merge-is-slower-than-you-think-and-how-to-design-fast-upsert-pipelines-in-delta-lake-979c45491e0c).

**Pros**

- Peak memory becomes roughly constant regardless of partition size, this actually *removes* the OOM rather than pushing the frontier out.
- Uses shipping delta-rs primitives (replaceWhere + iterator source + schema merge); atomic, and works with our existing idempotency tagging.

**Cons**

- Write amplification: it rewrites the whole partition, so we only want it for big partitions (hence pair it with 1.1).
- delta-rs replaceWhere edge cases. There are open issues like #2867 (predicate index-out-of-bounds) and #3744 (nullable property overwrite). we'd need to pin a known-good delta-rs version and cover these in tests

### 1.5 Subprocess-per-merge for full isolation

Run each merge in its own subprocess so that if it OOMs, the kernel kills *that* process (the biggest memory user) instead of the whole consumer pod, which then survives and retries the batch.

**Pros**

- Reduces blast radius from "whole pod / all concurrent jobs" to "the one merge that ballooned," without a second deployment.
- Can enable in-pod admission control, let a flagged-heavy merge run alone with the full heap.

**Cons**

- Containment, not cure: the giant merge still fails, just alone.
- multiprocessing + asyncio + Django + a Rust extension is genuinely fragile, and reliable per-process memory caps are hard (Arrow/Rust allocate off-heap — [delta-rs#3371](https://github.com/delta-io/delta-rs/issues/3371)).

---

## 2. Stick with Delta, with infra changes

Same storage format, but we change how/where the work runs.

### 2.1 Append + compact + view (merge-on-read)

Stop merging on the ingest path entirely: always **append** (bounded memory, can't OOM), resolve "latest row per key wins" at query time, and run a scheduled background compaction to collapse the append log.
Incremental syncs are upsert-only (no deletes), which removes the hardest part of merge-on-read.
Background on the trade-offs: [copy-on-write vs merge-on-read](https://www.dremio.com/blog/row-level-changes-on-the-lakehouse-copy-on-write-vs-merge-on-read-in-apache-iceberg/), [AWS Iceberg write best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/best-practices-write.html).

**Pros**

- The write path becomes fundamentally OOM-proof since appends never touch the target partition.
- Decouples failure domains: ingestion can't be blocked or retried-to-death by a giant partition.

**Cons**

- Read-side change lands in the HogQL query layer (inject dedup on every read) plus every downstream consumer.
- Read amplification and storage growth until compaction catches up; needs a reliable version column for "latest wins."

### 2.2 Kubernetes Job per heavy batch

For flagged-heavy batches, the consumer spins up an ephemeral k8s Job sized to the partition, runs the single merge there, and exits.

**Pros**

- Dynamic right-sizing can land a single huge merge no standing pool could fit, with no idle big pods.
- Maximum isolation, each heavy merge is its own pod/node/cgroup.

**Cons**

- Highest operational + security lift: the app gains RBAC to create pods, plus Job lifecycle, version pinning, log routing.
- Scheduling/cold-start latency tail (image pull + autoscaler + boot), and still has a max-node ceiling.

### 2.3 Two consumer pools

Run the `warehouse-sources-load` consumer as **two** deployments: a light pool (high concurrency, small pods, many replicas) and a heavy pool (low concurrency, big pods, isolated node pool).
Routing is per-schema (a schema is light or heavy), driven by the 1.1 signal, because batch ordering and our advisory lock are per-`(team, schema)`.
Standard mixed-workload pattern ([GKE dedicated node pools](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/isolate-workloads-dedicated-nodes), [Kueue queues](https://medium.com/google-cloud/mastering-workloads-in-kubernetes-with-kueue-part-1-queues-and-priorities-339257d8b4ba)).

**Pros**

- Contains blast radius (one heavy OOM kills one schema's batch, not 16) and right-sizes cost since only heavy schemas pay for big pods.
- Minimal new code: same consumer binary, a `pool` column, a `WHERE` clause, and manifests.

**Cons**

- Bigger pods alone don't fix a single partition larger than 20× even the big pod needs to be paired with 1.4 (or DuckDB below) to truly be safe.
- One-run reclassification lag, and the recovery sweep needs to be pool-aware.

---

## 3. If we don't stick with Delta (for the merge)

### 3.1 DuckDB as the merge engine

This is really **1.4 (partition overwrite) with a better engine doing the join**.
Instead of hand-rolling the streaming anti-join in PyArrow, we express it as SQL and let DuckDB execute it.
DuckDB is **out-of-core**: its hash joins, aggregations and sorts spill to disk when they exceed `memory_limit`, so the merge stays bounded regardless of partition size, we trade RAM pressure for disk ([DuckDB memory management](https://duckdb.org/2024/07/09/memory-management), [out-of-memory guide](https://duckdb.org/docs/current/guides/troubleshooting/oom_errors)).

It depends on / connects to the earlier pieces:

- Same architectural slot as **1.1 + 1.4**: detect heavy partition → bounded-memory partition rewrite → replaceWhere commit via delta-rs.
- delta-rs stays the commit layer; ClickHouse reads the result unchanged. Only the merge *compute* changes.

**Different execution process** : DuckDB is an embedded library, not a server — it runs **in the `warehouse-sources-load` consumer process itself**, in the same thread the merge already uses.
The one genuinely new requirement is **local scratch disk** on those pods for spilling (set `temp_directory`), and a concurrency cap so N concurrent merges × `memory_limit` stays under the pod's memory.
It can run in the current single pool (heavy merges behind a semaphore) or, combined with 2.3, in a heavy pool or 2.2 in a different K8s job.

**Pros**

- Bounded memory by construction (spill-to-disk), removes the OOM, works on every deployment (it's a library, no cloud lock-in).
- Less custom code than 1.4, the merge becomes a SQL query DuckDB runs and spills to disk for us, instead of streaming logic we maintain by hand.

**Cons**

- Needs provisioned scratch disk and a concurrency cap; spill IO makes heavy merges slower than in-RAM.
- Type fidelity (DuckDB ↔ Arrow ↔ Delta on decimals/timestamps/nested).

### 3.2 PySpark + EMR

Offload heavy merges to a Spark job on EMR (likely EMR Serverless: submit, it provisions, runs, scales to zero; [Spark jobs](https://docs.aws.amazon.com/emr/latest/EMR-Serverless-UserGuide/jobs-spark.html), [CLI](https://docs.aws.amazon.com/emr/latest/EMR-Serverless-UserGuide/jobs-cli.html)).
Spark + Delta is the reference implementation. It shuffles and spills across executors, so it doesn't OOM the way single-node delta-rs does, and it writes the *same* Delta table on S3 (no format migration).

**Pros**

- No single node needs to fit the partition, since Spark distributes and spills across executors.
- Same Delta format on the same S3 path, delta-rs and ClickHouse keep reading it unchanged.

**Cons**

- Heavy new ops (JVM/Spark/EMR, a separate artifact, IAM).
- Two writers on one table means pinning Delta protocol features (no deletion vectors) forever so delta-rs and ClickHouse can still read it.

### 3.3 Iceberg (with ClickHouse as the merge engine)

Switching the table format from Delta to Apache Iceberg, but not to merge it from a single-node Python writer.
The Python-native writer (PyIceberg) has the same OOM we're trying to escape.
The interesting part is that ClickHouse, which we already run as the query engine, now has mature Iceberg support: read with positional + equality deletes (merge-on-read) since 24.12, and write support: INSERT, equality deletes, ALTER UPDATE, compaction.
So tables stay as cheap open files on S3, but ClickHouse does the upserts via equality deletes + merge-on-read, with background compaction.

**Pros**

- Native merge-on-read (equality deletes) that delta-rs lacks, with a large-scale engine (ClickHouse, already in our stack) doing the write.
- Keeps data as open files on S3; no expensive ClickHouse-native storage.

**Cons**

- Huge migration: format swap on every table, write path off delta-rs, plus Iceberg needs a catalog (REST/Glue/Nessie)...
- ClickHouse Iceberg write support is very new (25.7–25.9), so we'd be betting on recent features and need to be on a current version; MoR also pushes cost to reads and needs a compaction cadence.


---

## Where my head is at

Needs:

1. **1.2 + 1.3**: these two look like the easy ones to try, even if we end up not keeping delta as the engine.
2. **1.1**: I think we definitely need this one. The observability will help us track growing partitions, and we can filter to only track partitions bigger than X.
3. **2.3**: No matter which route we take, separating inserts/appends/small merges from the bigger ones makes sense.

We need to decide on architecture changes anyway. Since we're not sure what's going to happen with DuckDB and the managed warehouse, we should assume that if we go for DuckDB, we'll have to handle our own provisioning.

**2.1 (merge-on-read)**: I really like this one; it's a nice separation of concerns. I'm just not sure how much it would affect our read layer, as I'm not familiar with it.
**3.2 (Spark/EMR)**: I've used Spark a lot, so maybe I'm a bit biased here. I'm not sure how expensive EMR clusters are these days, but if we think this could be a solution, I can do a full write-up on costs, time to spin up clusters, and everything else.
**3.1 (DuckDB)**: I'm less familiar with this one, and having to do our own provisioning might be a buzzkill, but it could be worth exploring.



## References

- delta-rs: [merge buffers the target side (#2573)](https://github.com/delta-io/delta-rs/issues/2573), [streamed_exec & partition predicates (#3786)](https://github.com/delta-io/delta-rs/issues/3786), [writer memory (#1225)](https://github.com/delta-io/delta-rs/issues/1225), [memory allocation (#3371)](https://github.com/delta-io/delta-rs/issues/3371)
- delta-rs docs: [writing / replaceWhere](https://delta-io.github.io/delta-rs/usage/writing/), [merging tables](https://delta-io.github.io/delta-rs/usage/merging-tables/), [get_add_actions](https://delta-io.github.io/delta-rs/api/delta_table/)
- Delta Lake: [replaceWhere](https://delta.io/blog/delta-lake-replacewhere/), [liquid clustering](https://delta.io/blog/liquid-clustering/), [deletion vectors aren't in delta-rs yet](https://milescole.dev/data-engineering/2024/11/04/Deletion-Vectors.html)
- Merge performance: [Databricks partition pruning](https://kb.databricks.com/delta/delta-merge-into.html), [why MERGE is slower than you think](https://medium.com/@harsh11csb/why-merge-is-slower-than-you-think-and-how-to-design-fast-upsert-pipelines-in-delta-lake-979c45491e0c), [MERGE at scale](https://cdelmonte.medium.com/delta-lake-merge-is-not-a-simple-upsert-what-actually-happens-at-scale-42ec8737a307)
- Merge-on-read: [copy-on-write vs merge-on-read (Dremio)](https://www.dremio.com/blog/row-level-changes-on-the-lakehouse-copy-on-write-vs-merge-on-read-in-apache-iceberg/), [AWS Iceberg write best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/best-practices-write.html)
- DuckDB: [memory management](https://duckdb.org/2024/07/09/memory-management), [out-of-memory guide](https://duckdb.org/docs/current/guides/troubleshooting/oom_errors)
- EMR: [Spark on EMR Serverless](https://docs.aws.amazon.com/emr/latest/EMR-Serverless-UserGuide/jobs-spark.html), [running jobs from the CLI](https://docs.aws.amazon.com/emr/latest/EMR-Serverless-UserGuide/jobs-cli.html)
- Infra patterns: [GKE dedicated node pools](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/isolate-workloads-dedicated-nodes), [Kueue queues & priorities](https://medium.com/google-cloud/mastering-workloads-in-kubernetes-with-kueue-part-1-queues-and-priorities-339257d8b4ba)
