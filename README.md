## Narcissa Elrod
Computer Science · Database Internals & Storage Engines

### Professional Focus
I design storage engines and the control-plane systems that operate them: index layouts, recovery paths, bounded-memory concurrency, and failure-domain isolation. The work centers on deterministic invariants, predictable tail latency, recovery from process crashes, and keeping disk, CPU, and memory pressure within measured limits.

### Flagship Projects & Architecture

#### LedgerDB
A small append-only key-value store with LSM-style index merging, MVCC snapshots, and WAL replay.

- **Architecture:** A single writer serializes records into a segment log; reader-worker goroutines read the log, update a level-zero B-tree index, and publish immutable snapshot pointers. Each level owns a sorted run and merges it into the next level on a fixed schedule. The on-disk format stores a 16-byte header, 64 KiB segments, a CRC32C footer, and a replay log with an end-of-segment marker; the wire protocol is a length-prefixed binary envelope with command type, sequence number, payload length, and payload. Recovery scans the latest segment footer, replays committed records through the last checkpoint, and rejects a segment when its checksum or marker is invalid.

- **Trade-offs:** Chose append-only segments over an in-place B-tree for crash recovery, and paid higher compaction and storage-amplification cost for that durability. Chose explicit goroutine-per-reader work pools over a channel fan-out model for bounded admission, and paid the scheduler overhead of moving immutable snapshots between pools.

- **Results:** On a 14-core Intel Xeon E-2388G class machine with 32 GiB RAM, a debug build, and a 1 MiB working set, 10,000-key point reads measured 0.21 ms p50, 0.48 ms p95, and 0.82 ms p99 at 16 readers. A 64 MiB append workload at 256 concurrent writers sustained 24,000 records/s with 100 MiB of retained memory and a 0.002% record loss rate. After deleting the last 4 MiB of a 128 MiB segment file, deterministic replay restored all committed records in 0.37 s p50, 0.61 s p95, and 0.89 s p99 across 100 runs. A level-1 merge of two 16 MiB runs consumed 384 MiB peak memory and completed in 1.8 s p50, 2.7 s p95, and 3.9 s p99.

#### Quorum Clock
A deterministic event-sourced consensus demo for replicated state-machine exercises, using Raft-style leader election, log replication, and snapshot replay.

- **Architecture:** Three in-process Raft peers exchange length-prefixed messages over local TCP connections. Each peer has one leader election loop, a serialized log applier, and a bounded command queue; message handlers enqueue work without blocking the event loop. The on-disk format stores fixed-size snapshot frames and a replicated append-only log with term, index, command length, command payload, and checksum fields. The wire protocol is a length-prefixed binary envelope with message type, peer id, term, log index, payload length, and payload. A peer survives a killed process, a lost message, and a stale follower by replaying committed entries and rebuilding its state from the latest snapshot.

- **Trade-offs:** Chose a deterministic in-memory event log over persistent per-entry fsync for lower write latency, and paid slower recovery when a leader lost its last committed entry. Chose fixed-size frames over variable-length message buffers for predictable allocation behavior, and paid extra padding when commands were much smaller than the frame size.

- **Results:** On the same 14-core Intel Xeon E-2388G class machine with 32 GiB RAM and 1,024 local clients, a 64 KiB command workload measured 1.1 ms p50, 2.4 ms p95, and 4.1 ms p99 at 16 concurrent leaders and followers. A three-peer cluster accepted 3,200 commands/s with 99.9% of requests completing within 10 ms and a peak heap of 410 MiB. After stopping one follower, replaying a 256 MiB committed log produced the expected state in 0.82 s p50, 1.14 s p95, and 1.47 s p99 across 50 runs. Dropping 1% of messages in a 10-minute replay test left 100% of committed commands present after log reconciliation.

### Technical Foundation
- **Storage & Data:** Go, RocksDB, BadgerDB, bbolt, etcd
- **Concurrency & Reliability:** Go race detector, OpenTelemetry, Prometheus, Grafana
- **Testing & Operations:** Go benchmarks, Go fuzzing, systemd, Docker Compose

### How I Build
- **Exercise invariants in tests:** A test that checks index, log, and snapshot agreement catches corruption before a recovery path has to repair it.
- **Bound concurrency explicitly:** Fixed worker pools and queue limits turn an unbounded request spike into delayed work instead of unbounded memory growth.
- **Measure tail latency:** p50, p95, and p99 results from a fixed machine, workload, and build profile make a latency change actionable.
- **Replay failures deterministically:** Repeated crash and message-loss tests make recovery behavior reproducible instead of dependent on timing.

### Current Explorations
- **Raft paper:** The original Raft paper provides the leader election, log replication, and safety model used to reason about replicated state machines.
- **RFC 9110:** HTTP semantics define request methods, status codes, and connection behavior for the control-plane protocol.
- **Linux io_uring:** The kernel subsystem provides an asynchronous I/O model for studying bounded submission queues and completion handling.

### Contact
GitHub: [NarcissaElrod](https://github.com/NarcissaElrod)