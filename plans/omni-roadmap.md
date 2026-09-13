# Omni Roadmap
### From a TinyKV-style store to a distributed memory layer for AI inference

**Philosophy:** each phase should teach one hard concept in isolation, be demoable/writable-about on its own, and build toward the final claim — "Omni is a distributed KV store whose memory model doubles as an inference-serving cache layer."

---

## Phase 0 — Foundations (read alongside Phase 1, if not already)

You've likely absorbed some of this already, but treat these as reference material to return to throughout the project, not a one-time read.

- **Database Internals** by Alex Petrov — the single best book for exactly this project. Part 1 covers B-trees, LSM-trees, WAL, and recovery; Part 2 covers distributed systems (replication, consensus). Read Part 1 now, Part 2 before Phase 4.
- **Designing Data-Intensive Applications** by Martin Kleppmann — broader systems context (replication, partitioning, consistency models). Chapters 3 (storage engines) and 5 (replication) are most relevant.
- **CMU 15-445/645 Database Systems** (Andy Pavlo, free on YouTube + Carnegie Mellon site) — lectures on buffer pool management, B+trees, LSM-trees, concurrency control. This is the closest thing to a formal course for what you're building.
- **"Let's Build a Simple Database"** by cstack (GitHub tutorial, C) — a small SQLite clone walkthrough. Good for seeing WAL/B-tree basics in C before you write your own in C++.

---

## Phase 1 — Core KV Store 

Basic get/set/delete, some persistence. Before moving on, sanity-check it has:
- Durability on write (fsync behavior understood, even if basic)
- Basic tests for correctness under normal operation

Don't gold-plate this — move on.

---

## Phase 2 — Block-Based Memory Management (Go prototype)

**Goal:** replace naive storage with a block/page allocator, reference counting, and an eviction policy. This is the conceptual bridge to vLLM-style KV-cache paging.

**Build, in order:**
1. Fixed-size block allocator (allocate/free blocks from a pre-reserved arena)
2. Block header metadata (id, ref count, size, in-use flag)
3. Reference counting so multiple logical owners can share a block
4. Eviction policy (start with LRU) triggered when the arena is full

**Resources:**
- **Handmade Hero, Day 160** — "Basic General Purpose Allocation" ([link](https://hero.handmade.network/episode/code/day160)) — block splitting/freeing, the mechanics you're building here.
- **Handmade Hero, Day 341** — "Dynamically Growing Arenas" ([link](https://hero.handmade.network/episode/code/day341/)) — reserve/commit pattern, relevant once you outgrow a fixed arena.
- **vLLM paper** — *"Efficient Memory Management for Large Language Model Serving with PagedAttention"* (arXiv, 2023) — read this now, not later. It's short, and Phase 2 is deliberately a simplified version of what it describes. Understanding it early will shape your block design correctly from the start.
- **Go's `sync.Pool` source/docs** — a production example of reuse-based allocation, useful pattern reference even though your use case differs.

**Deliverable:** README section + short blog post: "I built a paging allocator for my KV store and it turned out to be the same problem vLLM solves for GPU memory."

---

## Phase 3 — Concurrency & Request Scheduling

**Goal:** a request queue in front of the store with batching and backpressure — the scheduling half of an inference server.

**Build, in order:**
1. Concurrent-safe request queue (readers/writers)
2. Batching: group requests arriving within a small time window
3. Backpressure: reject/delay when overloaded rather than falling over
4. Basic priority scheduling (some requests preempt others)

**Resources:**
- **CMU 15-445, lecture on concurrency control** — MVCC vs locking, the two dominant approaches; pick one deliberately rather than by accident.
- **vLLM / SGLang GitHub source** — read the scheduler code specifically (not the whole repo). Look for how they implement continuous batching — this is the concept you're reproducing at smaller scale.
- Your own **Craw** project — you already built worker pools, backpressure, and rate limiting there. Re-read your own code before starting this phase; you're extending a pattern you've already proven you understand.

**Deliverable:** benchmark write-up showing throughput/latency before and after batching.

---

## Phase 4 — Distributed Layer

**Goal:** replication and sharding — the part that turns Omni into a legitimate distributed system, not just a fast local store.

**Build, in order:**
1. Leader-follower replication (start simple, async replication)
2. Raft-based consensus (upgrade once async replication works)
3. Sharding/partitioning across nodes

**Resources:**
- **"In Search of an Understandable Consensus Algorithm"** (the Raft paper, Ongaro & Ousterhout) — read this fully before implementing; it's written to be implementable, unlike Paxos papers.
- **MIT 6.824 Distributed Systems** (free course, YouTube + MIT site) — labs literally have you build Raft from scratch. Doing the labs alongside this phase is close to ideal preparation.
- **etcd's raft implementation** (`etcd-io/raft` on GitHub) — read as a production reference once you've implemented your own; don't start by copying it.
- **Database Internals, Part 2** (Petrov) — covers replication and consistency models in the same conceptual language as Part 1.

**Deliverable:** this is your strongest open-source visibility material — a working Raft-backed distributed KV store, correctly attributed as a learning project inspired by etcd/TinyKV, is genuinely impressive for a fresher.

---

## Phase 5 — AI-Infra Bridge Layer

**Goal:** prove the memory model transfers — add a vector/semantic access mode and wire Omni as a KV-cache backend for a small inference server.

**Build, in order:**
1. Vector storage mode alongside key-value mode (same underlying engine, nearest-neighbor access pattern)
2. Thin adapter exposing Omni's block allocator as a pluggable KV-cache backend
3. Integration test against a small open-source inference server (even a toy/simplified one)

**Resources:**
- **vLLM source, `kv_cache_manager` / block manager modules** — study the actual interface a KV-cache backend needs to expose; design your adapter to match its shape.
- **Mem0 and Zep (GitHub + blog posts)** — both are commercial "memory layer for AI" products; reading their architecture write-ups tells you what a real product in this space looks like, which sharpens your own design decisions.
- **SGLang paper** ("Efficient Execution of Structured Language Model Programs") — a second, slightly different take on serving-system design; useful for contrast against vLLM's approach.

**Deliverable:** this is your headline project claim — write it up properly, this is the piece that gets shared and gets you noticed.

---

## Phase 6 — Port Storage Engine to C++

**Goal:** once Phase 2's design is proven in Go, rewrite the storage core in C++ for real manual memory control (custom allocators, possibly mmap-backed blocks).

**Resources:**
- **"Writing a WAL"** — study Badger's own design doc / the WiscKey paper it's based on (*"WiscKey: Separating Keys from Values in SSD-conscious Storage"*) since you're already using Badger and it explains the design choices you'll be replacing.
- **RocksDB Wiki** (GitHub) — production reference for compaction, WAL, and crash recovery in C++; read specific wiki pages (WAL, Compaction) rather than trying to absorb the whole project.
- **CGo documentation** (Go's official blog post "C? Go? Cgo!") — read before deciding your Go↔C++ boundary approach; understand the overhead tradeoffs first.
- **Handmade Hero, Days 14 and 305** — platform-layer separation and arena usage in a real C codebase, directly relevant to structuring the C++ rewrite cleanly.

---

## Cadence notes

- Each phase gets its own README section, and ideally a short write-up (blog + LinkedIn) — visible incremental progress matters more than one big reveal at the end, given your OSS/LinkedIn strategy.
- Don't skip to Phase 5 early. The credibility of "Omni serves as a memory layer for inference" depends entirely on having built the paging/eviction/scheduling logic yourself in Phases 2–3 first.
- Time-box Phase 2's Go implementation — it's a working prototype/spec for Phase 6, not the final product. Don't over-polish it.
