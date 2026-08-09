
Personal repo, hand-coded, hand-tested. PingCAP's `talent-plan/tinykv` and `talent-plan/tinysql` repos are used as reference only — no copied code, no copied tests.

Pace assumption: **5–8 hrs/week**, comfortable with Go, new to systems/concurrency. Dates start **Aug 10, 2026** — shift everything if you start later. Raft (Phase 2) is the highest-risk phase for slippage; if it runs long, nothing else needs to move except the weeks after it.

Progress legend: `[ ]` not started · `[~]` in progress · `[x]` done

---

## Phase 0 — Foundations + big-picture reading

**Aug 10 – Aug 21, 2026 (weeks 1–2)**

- [~]Read protobuf language guide (proto3) — https://protobuf.dev/programming-guides/proto3/
- [ ] Read Protocol Buffers in Go tutorial — https://protobuf.dev/getting-started/gotutorial/
- [ ] Read gRPC Go quickstart — https://grpc.io/docs/languages/go/quickstart/
- [ ] Read gRPC Go basics tutorial (RouteGuide) — https://grpc.io/docs/languages/go/basics/
- [ ] Build a disposable `Echo` gRPC service end-to-end (own `.proto`, own client+server), then discard it
- [ ] Read TiKV design overview (data storage)
- [ ] Read PD design overview (scheduling)
- [ ] Skim full TinyKV reading list, triage which sections map to which project — https://github.com/talent-plan/tinykv/blob/course/doc/reading_list.md
- [ ] Read RocksDB Column Families wiki page for what a "real" CF implementation looks like
- [ ] Scaffold blank repo structure (`cmd/`, `internal/storage/`, `internal/server/`, `internal/engineutil/`, `proto/`)

---

## Phase 1 — Project1: StandaloneKV

**Aug 22 – Sep 11, 2026 (weeks 3–5)**

- [ ] Write your own `.proto` for RawGet/RawPut/RawDelete/RawScan (design your own fields, don't copy)
- [ ] Generate Go stubs, get bare gRPC server returning hardcoded responses
- [ ] CF-prefix helper (`KeyWithCF` / reverse) + table-driven tests, including collision edge cases
- [ ] Naive `Storage` interface (direct Get/Put/Delete/Scan)
- [ ] `BadgerStorage`: Put, Delete, Get
- [ ] `BadgerStorage`: Scan with CF isolation + limit
- [ ] Hand-written tests: missing key, overwrite, same key different CFs, scan ordering, scan limit
- [ ] Refactor to `Write(batch)` + snapshot `Reader`
- [ ] Snapshot isolation test (open reader, mutate DB, confirm reader sees old value)
- [ ] Wire gRPC handlers to `Storage`
- [ ] Txn/iterator cleanup test (loop opening many txns, confirm no leak/hang)
- [ ] **Milestone:** working standalone KV server, own tests passing, own CLI or manual gRPC calls verified

---

## Phase 2 — Project2: RaftKV

**Sep 12 – Nov 6, 2026 (weeks 6–12)** — expect slippage here, that's normal

- [ ] Read the Raft paper (In Search of an Understandable Consensus Algorithm)
- [ ] Read reading list's Consensus section (Quorum, Raft references, MIT 6.824 Raft lab writeup)
- [ ] Use https://thesecretlivesofdata.com/raft/ as a visual reference while implementing
- [ ] Leader election + heartbeats
- [ ] Own test harness: kill/restart nodes, force re-elections
- [ ] Log replication: AppendEntries, commit index advancement
- [ ] Follower log conflict resolution
- [ ] Failure-injection tests: partition a node, restart it stale, verify convergence
- [ ] `RaftStorage` implementing the same `Storage` interface from Project1
- [ ] Wire Raft-backed KV server end-to-end
- [ ] Log compaction / snapshotting so log doesn't grow unbounded
- [ ] Run everything under `go test -race`, fix races
- [ ] **Milestone:** fault-tolerant single-Raft-group KV store — this alone is a complete, portfolio-worthy project

---

## Phase 3 — Project3: MultiRaftKV

**Nov 7 – Dec 4, 2026 (weeks 13–16)**

- [ ] Read reading list's Scale & Balance section (Multi-Raft, Split & Merge, range vs hash partitioning)
- [ ] Multiple Raft groups (regions), each covering a key range
- [ ] Region-aware request routing
- [ ] Conf changes (adding/removing nodes)
- [ ] Region splitting
- [ ] Basic scheduler: balance regions/leaders across nodes
- [ ] **Milestone:** multi-region cluster that can split and rebalance

---

## Phase 4 — Project4: Transactions

**Dec 5, 2026 – Jan 1, 2027 (weeks 17–19)**

- [ ] Read reading list's Replication & Consistency section (clocks, consistency models)
- [ ] Read the Percolator paper (TiKV's transaction model is based on it)
- [ ] MVCC: multi-version keys, read timestamps
- [ ] `KvGet` with version awareness
- [ ] `KvPrewrite` (lock + provisional write, conflict detection)
- [ ] `KvCommit`
- [ ] Concurrent-client tests: overlapping transactions, lock contention
- [ ] **Milestone:** TinyKV-equivalent complete — distributed, fault-tolerant, transactional KV store, fully hand-built

---

## Phase 5 — TinySQL (optional push)

**Jan 2 – Feb 19, 2027 (weeks 20–27)** — looser estimate, less granular than TinyKV phases above

- [ ] Read TinySQL's own reference list + relational algebra / life-of-a-query intro material
- [ ] Get a bare SQL-shell-style binary talking to your TinyKV + TinyScheduler
- [ ] SQL parsing → simple read-only query execution against your KV store
- [ ] Query optimization basics (plan construction)
- [ ] DML/DDL: statements that mutate state, interaction with reads
- [ ] Wire everything together end-to-end against your own TinyKV instance
- [ ] **Milestone:** full TiDB-style SQL layer running on your own hand-built storage engine

---

## If you need to cut scope

Highest value stopping points, in order:

1. Project1 + Project2 (working fault-tolerant single-Raft KV store) — smallest complete deliverable
2. - Project4 (skip multi-region, add transactions) — if distributed transactions matter more to you than sharding
3. - Project3 — full TinyKV
4. - TinySQL — full stack, biggest time investment, most optional