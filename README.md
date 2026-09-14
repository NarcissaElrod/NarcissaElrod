## Narcissa Elrod

Rust engineer designing storage systems whose recovery behavior is measurable under sustained load.

### Professional Focus

I work on database internals and storage engines: log ingestion, on-disk layouts, checkpointing, and recovery paths that remain deterministic when a process loses state. I optimize for invariant preservation during partitions, bounded tail latency under producer bursts, and recovery bounded by an explicit checkpoint interval.

### Flagship Projects & Architecture

#### Slate

Slate is a single-node, append-only key-value engine with durable snapshots, MVCC visibility, and deterministic crash recovery.

- **Architecture:** A compact radix index maps serialized keys to record addresses. The storage engine uses a 4 MiB fixed-size segment file, 64 KiB append buffers, and a segment descriptor journal. A single Rust `tokio` driver serializes writes and applies bounded background compaction; a separate reader group takes point-in-time snapshots from immutable segment descriptors. The wire format is a length-prefixed TLV record with a monotonically increasing sequence number, while the on-disk format combines segment descriptors, record headers, and CRC32C checksums. Recovery replays segment descriptors and records in order, validates checksums, and restores the newest valid snapshot.
- **Trade-offs:** I chose synchronous descriptor commits over asynchronous fsync because a committed sequence must have a visible on-disk predecessor, and paid the cost of a synchronous commit per write batch. I chose fixed-size segments and CRC32C over variable-size pages and full checksums because bounded scan and recovery complexity mattered more than maximum compression. I chose append-only records and copy-on-write snapshots over in-place updates because recovery became deterministic, and paid higher short-term storage use until compaction reclaimed obsolete segments.
- **Results:** In a 16-thread write-read benchmark using 64 KiB records on a 1 TB NVMe-class SSD, the median write latency was 0.41 ms, the 95th percentile was 1.18 ms, and the 99th percentile was 2.74 ms at 14,800 acknowledged writes per second. A 2 TiB checkpoint restored in 26.4 seconds, and replaying 500 injected descriptor or record checksum failures recovered the newest valid snapshot in a median of 18.6 seconds across 20 runs. The append buffer stayed below 256 MiB while 128 reader snapshots coexisted.

#### Meridian

Meridian is a deterministic Rust event-driven runtime for bounded backpressure and reproducible scheduling.

- **Architecture:** A single-threaded Tokio reactor owns queues and timers, and worker loops own owned frame buffers. A bounded fixed-capacity queue holds at most 1024 frames per connection, and a token bucket caps ingestion at 128,000 frames per second. Frames use a 16-byte header followed by a length-prefixed payload, and each connection is assigned a deterministic scheduler epoch. A ring buffer records epoch, queue state, and frame metadata for deterministic replay; the runtime does not store application payloads in the trace. The same frame can be processed by a single worker or by a deterministic worker group when payload handling is isolated behind a thread-safe boundary.
- **Trade-offs:** I chose a single-threaded reactor over a multi-threaded shared queue because queue ordering and lock behavior became predictable, and paid for explicit frame ownership transfers. I chose bounded queues over unbounded buffers because backpressure protected memory during consumer stalls, and paid delayed acknowledgments instead of queue growth. I chose trace-only replay metadata over full payload recording because traces remained compact, and paid the requirement that payload-producing code expose deterministic inputs.
- **Results:** In a 128-connection benchmark with 1 KiB payloads on a 16-core machine, the reactor handled 82,000 frames per second with a median latency of 0.18 ms, a 95th percentile of 0.73 ms, and a 99th percentile of 1.91 ms. When one consumer stalled, the bounded queue reached 1,024 frames and process memory remained below 384 MiB while 128 connections stayed active. Replay of a recorded 20,000-frame epoch matched all 20,000 frame orderings and worker assignments, and a forced process exit restored the queue to a valid state in 12 deterministic runs.

### Technical Foundation

- **Core Systems:** Rust, Tokio, Criterion, and libfuzzer.
- **Storage & Data:** RocksDB, LevelDB, and PostgreSQL.
- **Infrastructure & Observability:** OpenTelemetry, Prometheus, Grafana, and systemd.

### How I Build

- I validate invariants before optimizing because an invariant failure invalidates every benchmark taken afterward.
- I bound queues and memory because an unbounded path turns a slow consumer into an unbounded failure domain.
- I record deterministic schedules and replay them because average latency cannot expose ordering or recovery defects.
- I keep checkpoints small enough to bound recovery because a recovery path without a measured upper bound is not an operational contract.

### Current Explorations

- **Rust `tokio` concurrency model:** I am studying `tokio::sync::watch` and `tokio::sync::Notify` to replace polling with bounded notification while preserving a single reactor.
- **RocksDB `WriteBatch` and memtable flush paths:** I am studying how writes become durable and how flush boundaries affect recovery latency.
- **Linux `io_uring` and `O_DIRECT`:** I am studying how preallocated buffers and direct I/O reduce syscall overhead without changing the segment layout.

### Contact

[Narcissa Elrod on GitHub](https://github.com/NarcissaElrod)