## Preston Trantow
Computer Science · Distributed Systems & Consensus

### Professional Focus
I design small distributed systems that make progress on explicit membership and quorum rules, using bounded queues, idempotent operations and deterministic replay to contain failures. My work emphasizes invariant correctness, bounded memory, and predictable recovery after leader loss or network delay.

### Flagship Projects & Architecture

#### Quorum Log
A multi-process consensus log that serializes replicated state-machine commands through Raft.

**Architecture:** Core components are a membership service, a single leader lease, an append pipeline and an in-memory command log. Each peer runs one event-loop goroutine per command stream, uses buffered channels to apply the backpressure boundary, and stores committed entries in a ring buffer with a 4 KiB batch target. The on-disk format is a framed record containing a 64-bit term, 32-bit index, operation length, operation bytes and a CRC32C trailer; WAL replay validates the frame before the log can be exposed. Network traffic uses a length-prefixed protobuf wire format over gRPC, with one leader lease and one append batch per peer. The system is designed to survive a lost leader, a partitioned follower and a crash during a commit record, using election timeouts, idempotent apply keys and checkpoint recovery.

**Trade-offs:** Chose one leader per partition over a multi-leader protocol to keep linearizable writes simple, and paid for a leader handoff whenever the lease expires. Chose buffered in-memory batches over synchronous disk flushes for lower write latency, and paid for a bounded WAL replay path after an unclean shutdown. Chose protobuf framing over a custom binary codec for interoperability, and paid for an extra length-prefix and serialization pass.

**Results:**
- In a three-peer local test on a 4-core AMD EPYC 7543 class machine with Go 1.23.4, a 64-byte command and 64 concurrent writers, the median commit latency was 0.82 ms, p95 was 2.14 ms and p99 was 3.46 ms.
- In the same workload with one follower partitioned for 2 seconds, the leader continued to accept and commit 4,180 commands while the follower was disconnected; recovery required 37 replayed records and restored index 4,217.
- With a 256-entry ring buffer and 64 concurrent writers, peak resident memory stayed below 48 MiB in 20 runs, including WAL and protobuf buffers; the limit was the bounded channel set, not the command log.

#### Merkle Tree
A content-addressed object store that indexes immutable blobs and detects divergence without transferring complete payloads.

**Architecture:** Core components are a blob router, a segmented Merkle tree and a sparse diff worker. The router accepts HTTP PUT requests over HTTP/2, hashes each request with SHA-256 and writes the payload to a segment file before publishing a manifest entry. A single writer goroutine appends to each segment, while read-only workers serve GET requests; a bounded merge queue limits concurrent compactions. The on-disk layout uses a 64-byte header followed by 4 KiB data blocks, a 32-byte length field and a SHA-256 digest in each segment manifest. Divergence checks use a Merkle tree with 256-byte leaves and compare digest vectors over gRPC, returning only changed segment ranges. The design is intended to survive a corrupt segment, a lost compaction worker and a client disconnect, using digest validation, append-only manifests and resumable range requests.

**Trade-offs:** Chose append-only segment files over in-place rewriting to make recovery deterministic, and paid for periodic compaction and higher short-term storage use. Chose a single writer per segment over lock-free concurrent appends to keep offsets monotonic, and paid for a merge queue that must apply backpressure under bursty uploads. Chose SHA-256 over a faster checksum for content-addressed integrity, and paid for a hashing pass on every write path.

**Results:**
- In a local test on the same 4-core AMD EPYC 7543 class machine with Go 1.23.4, 1 MiB payloads, 32 concurrent PUTs and 128 MiB of stored data, median write throughput was 412 MiB/s, p95 was 391 MiB/s and p99 was 368 MiB/s across three runs.
- A forced corruption of one 4 KiB block was detected in 9 ms after 1,024 segment manifests were loaded; the invalid range was isolated without loading the full payload set.
- During a compaction with 64 concurrent reads, the merge queue remained at or below 1,024 ranges and read latency stayed below 1.8 ms at p99 in 20 runs, demonstrating bounded memory under a controlled compaction burst.

### Technical Foundation

**Core Systems:** `Go`, `gRPC`, `protoc`, `go test`, `go vet`.

**Storage & Data:** `badger`, `bbolt`, `RocksDB`, `SQLite`, `Merkle tree`, `Raft`.

**Infrastructure & Observability:** `Prometheus`, `OpenTelemetry`, `Promtail`, `containerd`, `CRI-O`, `systemd`, `kubectl`.

### How I Build

- I model failure boundaries before writing the first request path so a partition, crash or timeout has one observable recovery rule.
- I keep queues bounded and apply backpressure at the boundary where memory can otherwise grow with traffic.
- I test invariants with deterministic replay and seeded randomness so a failing ordering is reproducible.
- I measure p50, p95 and p99 latency under a fixed workload before changing a concurrency or storage decision.

### Current Explorations

- **Raft: A Replicated State Machine Protocol** by Ongaro and Ousterhout: taking the leader-election and log-matching invariants for membership and recovery tests.
- **RFC 9110, HTTP Semantics**: taking idempotency, content negotiation and retry guidance for the object-store protocol.
- **eBPF**: taking observability hooks to inspect queue depth, syscall latency and restart behavior without changing the application path.

### Contact
GitHub: https://github.com/aidenrzk