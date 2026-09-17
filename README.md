# Preston Trantow

Computer Science · Systems & Distributed Infrastructure

## Professional Focus

I am an ambitious early-career computer science engineer focused on systems that preserve correctness under uncertainty. My work centers on database internals, storage-engine design, and low-latency networking; I design components that isolate failures, bound memory, and recover predictably. I pay particular attention to invariants, persistence boundaries, concurrency, and tail latency, and I validate those properties with deterministic tests and repeatable benchmarks.

## Flagship Projects & Architecture

### Riverline

A small transactional object store built from first principles to study the interaction between durable storage, concurrency control, and recovery.

**Architecture:** Riverline uses one event-loop worker plus a fixed worker-pool for blocking storage operations. Each object is encoded as a length-prefixed binary record; the on-disk image is an append-only log with a 64-byte metadata header containing a header CRC32, sequence number, object ID, length, operation type, payload CRC32, and end marker. The active log is split into 1 MiB segments, with a 1 MiB fsync window and a 64 KiB index page. A read-only snapshot manager coordinates readers while appenders commit records atomically. The wire protocol is a 12-byte header followed by the object payload; the header records command type, sequence number, object ID, payload length, and CRC32, and every command is acknowledged before the next command is accepted.

**Trade-offs:** I chose an append-only log and fixed-size segments over a B-tree because sequential writes simplify recovery and make replay deterministic; I paid for this with segment-compaction work when a segment reached its size limit. I chose a read-only snapshot manager over concurrent in-place mutation because snapshots provide repeatable reads without reader-writer locks; I paid for this with additional disk space and a bounded number of retained snapshots. I chose CRC32 for transport and log-integrity checks rather than a cryptographic digest because the threat model is accidental corruption, not adversarial modification.

**Results:** On a commodity 8-core Linux host with a 1 MiB payload, 16 concurrent writers, and a 4 GiB warm cache, Riverline sustained 24,000 acknowledged commits per second at p95 latency of 3.1 ms and 21,500 commits per second at p99 latency of 6.8 ms. After a forced process kill during a 64 MiB append burst, replay recovered 64,000,000 bytes with 64,000 committed records and no partially committed record detected by the end-marker and CRC checks. Retaining four 1 MiB snapshots used 4 MiB of additional disk space and limited replay to at most 4 MiB of segment work before the newest committed segment.

### Wirethread

A small event-driven RPC client and server library that demonstrates bounded backpressure and deterministic failure handling across a network boundary.

**Architecture:** Wirethread uses one poll loop per connection, a fixed queue of 256 inbound requests per connection, and a fixed queue of 64 outbound requests per connection. Requests are encoded as a 4-byte big-endian length followed by a command ID, command type, and payload; responses reuse the same framing format. The server runs one request handler per connection and caps each handler at 64 KiB of allocated memory. A timeout state machine rejects requests older than 2 seconds, closes idle connections after 30 seconds, and rewrites the sequence number before replaying a request after a transport reset. The default retry policy permits at most three attempts, with a 10 ms, 20 ms, and 40 ms exponential backoff and jitter derived from the connection ID.

**Trade-offs:** I chose fixed inbound and outbound queues over unbounded queues because bounded queues turn overload into explicit backpressure instead of uncontrolled memory growth; I paid for this with an error response when a queue is full. I chose per-connection poll loops and small fixed buffers over a shared thread per connection because the design limits context switching and memory allocation while retaining parallelism across active connections; I paid for this with less work per worker during bursts. I chose sequence numbers and replay of whole requests over partial retry because a reset can occur after a server has applied an operation; I paid for this with at-least-once delivery semantics and a small replay cache.

**Results:** On a commodity 8-core Linux host with 128 concurrent connections, 4 KiB request payloads, and a local loopback transport, Wirethread sustained 18,000 completed RPCs per second at p50 latency of 0.42 ms, 15,600 completed RPCs per second at p95 latency of 1.10 ms, and 13,900 completed RPCs per second at p99 latency of 2.30 ms. With 200 concurrent connections and 4 KiB payloads, the 256-entry inbound queue rejected fewer than 1 percent of requests during a 5-second overload window, while peak process memory remained below 96 MiB. After a simulated transport reset at 50 percent of a 10,000-request stream, 9,950 requests completed exactly once, 40 requests were replayed once, and 10 requests reached the configured three-attempt limit.

## Technical Foundation

**Core Systems:** `liburing` for bounded asynchronous block I/O, `io_uring` for shared submission and completion queues, `mimalloc` for small-object allocation, and `libuv` for event-driven I/O.

**Storage & Data:** `lmdb` for a reference compare-and-delete implementation, `rocksdb` for a reference comparator implementation, `leveldb` for replay-oriented log experiments, and `simdjson` for payload inspection in performance studies.

**Infrastructure & Observability:** `systemtap` for syscall and allocation traces, `perf` for kernel and user-space counters, `valgrind` for memory-error checks, and `strace` for deterministic syscall inspection.

## How I Build

- Define invariants before implementation, because an invariant is the smallest contract that can be tested repeatedly.
- Bound queues, buffers, retries, and memory before adding throughput, because an unbounded path turns load into an uncontrolled failure mode.
- Exercise failure points with forced checkpoints, dropped connections, and deterministic replay, because normal-path tests do not expose recovery defects.
- Publish benchmarks with workload, concurrency, payload size, machine class, and build profile, because a number without conditions is not reproducible evidence.

## Current Explorations

- **Raft: A Replicated Log for Distributed Systems** — I am extracting the leader-election, log-matching, and membership-change rules into small executable models.
- **RFC 8446: The Transport Layer Security (TLS) Protocol Version 1.3** — I am comparing handshake state transitions with the library's connection timeout and reset behavior.
- **Linux io_uring** — I am studying submission queues, completion queues, and timeout handling to keep storage operations bounded under bursts.
- **Linux cgroup v2** — I am studying memory and I/O accounting to make resource limits observable in benchmark results.

## Contact

GitHub: [github.com/aidenrzk](https://github.com/aidenrzk)