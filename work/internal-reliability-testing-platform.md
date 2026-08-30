# Internal Reliability Testing Platform — The Feedback System

## 1. Purpose

Build an internal service — the Feedback System — that gives every engineering team at Atomity a single platform for benchmarking, load-testing, scenario-testing, and chaos-testing microservices. The platform exposes a common CLI (`mytest`) and control plane so that any engineer can run reliability tests without building custom infrastructure.

The Feedback System is not an afterthought bolted onto the deployment process. It is a continuous feedback loop: code changes flow in, structured evidence of their impact flows out, and the team acts on that evidence before shipping. If a change degrades performance or breaks a flow, the loop catches it and sends the developer back to fix it. If the change holds up, the loop clears it for deployment.

Core goals:

- Test individual services at internal boundaries (component-level).
- Test complete user-facing flows end-to-end through the gateway.
- Test any intermediate slice of the architecture — a service and its immediate dependencies, a subset of the call chain — not only the extremes.
- Run repeatable, deterministic performance tests in CI and on demand.
- Introduce controlled failure conditions and measure the system's response.
- Correlate test results with metrics, logs, and traces to explain outcomes, not just report them.
- Enforce authorization and blast-radius limits so the testing platform itself cannot cause incidents.
- Store historical results so regressions surface automatically over time.
- Generate shareable reports that teams pass around to build a shared understanding of system behavior.

This platform is a long-term investment. It grows with the product and must itself be engineered for correctness.

---

## 2. The Four Pillars

The Feedback System is organized around four capabilities. Every test, from a quick one-service benchmark to a full resilience scenario, is composed from these building blocks.

### Workflows

End-to-end user journeys that exercise the system the way a real customer would. A workflow describes a sequence of actions — browse, add to cart, check out, receive confirmation — and the platform translates that into real HTTP/gRPC calls through the gateway.

Workflows are the highest-fidelity form of testing the platform supports. They answer the question: does this flow still work correctly from the user's perspective?

### Workloads

Raw traffic patterns aimed at a specific target. A workload defines a rate, duration, protocol, and payload — it is the engine behind every performance measurement. Workloads can target a single service directly (component benchmark) or drive traffic through the gateway (system-level load).

### Scenarios

The reusable unit that ties everything together. A scenario combines a workload (or workflow), an environment, optional chaos injections, and a set of assertions into a single, named, versioned definition. Scenarios are what developers reference in the CLI, what CI pipelines invoke, and what release gates evaluate against.

### Chaos Testing

Controlled fault injection — added latency, forced errors, network partitions, process kills — applied during a test run. Chaos is always layered on top of a workload or workflow; it never runs in isolation. The platform coordinates the timing so that faults are injected at precise points during the traffic window and removed cleanly afterward.

---

## 3. The Development Feedback Loop

The Feedback System sits inside the normal development cycle, not beside it. The loop works like this:

```text
     ┌─────────────────────────────────────────────────┐
     │                                                 │
     ▼                                                 │
  Develop                                              │
     │                                                 │
     ▼                                                 │
  Feedback System                                      │
     │                                                 │
     ├── Is the flow correct end-to-end, no errors?    │
     │      If no ──────────────────────────────────────┘
     │                                                 
     ▼                                                 
  Scenarios, load testing, chaos testing               
     │                                                 
     ├── Found issues?                                 
     │   Degrades the existing deployment?             
     │      If yes ─────────────────────────────────────┐
     │                                                  │
     ▼                                                  │
  No issues — improves or holds steady                  │
     │                                                  │
     ▼                                                  │
  Deploy                           back to Develop ◄────┘
```

The important property is consistency. Every change, on every branch, for every team goes through the same loop. This only works if the feedback loop is fast enough that developers do not skip it and reliable enough that they trust its results. Those two requirements drive most of the design decisions in this document.

> A consistent internal feedback loop should be carefully engineered and not vibed. The platform's own correctness is a prerequisite for trusting the signals it produces about the rest of the system.

---

## 4. Testing Granularity

A common mistake in reliability platforms is offering only two modes: test one service, or test the whole system. Real architectures have important behavior at every layer in between. The Feedback System must support testing at any level of the stack.

### Level 1 — Component Benchmark

Test one service directly, bypassing the gateway and all other services.

```text
Runner → Inventory Service
```

Answers: what is this service's throughput ceiling, CPU efficiency, serialization cost, and how do those change between releases?

### Level 2 — Dependency Chain

Test a service and the services it calls, including their shared infrastructure.

```text
Runner → Order Service → Inventory Service → DB
```

Answers: how does this service behave under load when its dependencies are real, not mocked? Where do connection pools saturate? Where does the database become the bottleneck?

### Level 3 — Partial Flow

Test a meaningful slice of the architecture that does not span the entire system. For example, the payment flow exercises the gateway, order service, and payment provider, but not the recommendation engine or the search index.

```text
Runner → Gateway → Order Service → Payment Service → External Provider
```

Answers: does this subflow meet its latency and error-rate targets independently of unrelated services?

### Level 4 — End-to-End Flow

Generate traffic through the gateway that exercises a complete user journey across all participating services.

```text
Runner → Gateway → All participating services → All dependencies
```

Answers: does the full checkout flow still work? What is the real end-to-end latency a user experiences? How does cross-service interaction behave under load?

### Level 5 — Resilience Test

Combine any of the above levels with controlled failure injection and assert that the system degrades gracefully.

```text
Traffic (any level)
  +
Chaos injection
  +
SLO assertions
```

Answers: when inventory adds 300ms of latency, does the checkout flow still complete within its p99 budget? Does the error rate stay below 1%?

Every scenario specifies which level it operates at. The platform does not assume a default.

---

## 5. User Interfaces

### CLI — `mytest`

The CLI is the primary developer interface. It can be used by anyone within Atomity.

```bash
# Run a named benchmark
mytest run checkout-load

# Benchmark a single service directly
mytest benchmark inventory --rps 1000

# Run an end-to-end scenario
mytest scenario checkout-resilience

# Inject chaos manually
mytest chaos inject inventory --latency 500ms

# View results
mytest results checkout-resilience

# Run in CI mode (blocks until complete)
mytest run checkout-resilience --wait
```

Developers should not need to know whether a test is executed by an HTTP runner, a gRPC runner, or a distributed worker pool. The CLI resolves the scenario definition and delegates to the control plane.

### CI Integration

The same scenario definitions used interactively are used in CI. The `--wait` flag causes the CLI to block until the run completes and exit with a non-zero status if any assertion fails.

```bash
mytest run checkout-resilience --environment staging --wait
```

CI receives structured, machine-readable output. A failed assertion is a failed build.

---

## 6. Scenario Definition

Use declarative scenario definitions rather than putting all test configuration into CLI flags. The scenario file is the source of truth for what a test does, and it lives in version control alongside the code it tests.

```yaml
name: checkout-resilience

environment: staging

traffic:
  - target: gateway
    protocol: http
    method: POST
    path: /checkout
    rate: 500rps
    duration: 5m

chaos:
  - target: inventory
    type: latency
    value: 300ms
    start: 60s
    duration: 120s

assertions:
  - metric: request.p99
    operator: "<"
    value: 1000ms

  - metric: error_rate
    operator: "<"
    value: 1%
```

The same definition is used by developers, CI, scheduled nightly runs, release gates, and reliability engineers. There is one definition, used everywhere.

---

## 7. Architecture

```text
                         ┌──────────────┐
                         │   Developer  │
                         └──────┬───────┘
                                │
                         ┌──────▼───────┐
                         │  mytest CLI  │
                         └──────┬───────┘
                                │
                         HTTP/gRPC API
                                │
                  ┌─────────────▼─────────────┐
                  │      Control Plane        │
                  │                           │
                  │  Scenario Registry         │
                  │  Run Manager               │
                  │  Scheduler                 │
                  │  Auth / RBAC               │
                  │  Environment Policy        │
                  └─────────────┬─────────────┘
                                │
                         Job / Task Queue
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
       ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
       │ HTTP Runner │   │ gRPC Runner  │   │ Chaos Agent │
       └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
              │                 │                 │
              └─────────────────┼─────────────────┘
                                │
                         Target Environment
                                │
                 ┌──────────────┼──────────────┐
                 ▼              ▼              ▼
              Gateway        Services       Dependencies
                 │              │              │
                 └──────────────┼──────────────┘
                                │
                     Metrics / Logs / Traces
                                │
                  ┌─────────────▼─────────────┐
                  │  Result Aggregation &     │
                  │  Per-Service Analysis      │
                  └─────────────┬─────────────┘
                                │
                  ┌─────────────▼─────────────┐
                  │  Report Generation        │
                  └───────────────────────────┘
```

---

## 8. Load Generation and Tooling

Different testing goals require different tools. The platform does not commit to a single load generator; it chooses the right tool for the job and abstracts the choice behind a common runner interface.

### Constant-Rate and Spike Traffic

For straightforward HTTP/gRPC benchmarks — saturating a single service, measuring raw throughput, or simulating traffic spikes — use lightweight, high-performance generators written in Go:

- **bombardier** — constant-rate HTTP load, minimal overhead.
- **vegeta** — constant-rate and spike-pattern HTTP load; can replay traffic from a file; produces structured output.

These tools are appropriate when the traffic pattern is uniform: every request hits the same endpoint with the same payload, and the goal is to measure throughput, latency distribution, or error rate under a controlled load.

### Complex End-to-End Flows

For testing realistic user journeys that span multiple endpoints, require sequential logic (add to cart, then check out, then verify confirmation), or need conditional branching, use a programmable test framework:

- **k6** — scriptable load testing with JavaScript test definitions; supports complex multi-step flows, custom metrics, thresholds, and distributed execution.
- **gateline** — end-to-end flow testing with built-in support for chained requests and assertions.

These tools are appropriate when a flat stream of identical requests does not capture the behavior being tested.

### Load Models

The platform supports both major load models:

**Constant-rate** — a fixed number of requests per second, regardless of server response time. Useful for capacity testing and SLO validation because the offered load is independent of server performance.

```text
1000 requests/sec for 5 minutes
```

**Concurrency-based** — a fixed number of in-flight requests at any time. Useful for simulating client connection pressure and identifying concurrency bottlenecks.

```text
100 concurrent connections for 5 minutes
```

Both models are exposed through the scenario definition. The underlying tool is selected by the control plane based on the scenario's requirements.

### Runner Interface

The control plane cares about target, protocol, load model, duration, rate or concurrency, payload, headers, and assertions. It does not care which binary produces the traffic.

```text
Runner Interface
├── HTTPRunner      (bombardier, vegeta)
├── GRPCRunner      (ghz)
├── FlowRunner      (k6, gateline)
├── ChaosAgent      (fault injection)
└── FutureRunner
```

---

## 9. CI Tiers

Not every test belongs in every pipeline stage. Running a five-minute resilience scenario on every pull request wastes resources and trains developers to ignore the results. Use tiers.

### Pull Request

Small, fast, deterministic tests that catch regressions before review.

Run: single-service benchmark, small load test against the changed service, regression threshold comparison against the last merge result.

Budget: under two minutes.

### Merge to Main

Broader tests that validate the change in context.

Run: service capacity tests, selected end-to-end flows involving the changed service, dependency-chain tests.

Budget: under ten minutes.

### Nightly

Sustained and exploratory tests that would be too expensive to run on every commit.

Run: ramp tests, sustained load, multi-scenario resilience suites, full end-to-end user journeys.

### Pre-Release Gate

Tests that must pass before a release reaches production.

Run: all critical user journeys, capacity validation, resilience scenarios with chaos injection, SLO assertion suites.

A pre-release gate failure blocks the release. The result report is attached to the release ticket.

### Production

Only explicitly approved, tightly constrained experiments.

Run: low-rate traffic, short duration, strict blast radius, explicit RBAC authorization required.

---

## 10. Assertions

Tests declare their expectations. The platform evaluates them and produces a pass/fail verdict without requiring an engineer to interpret raw numbers.

Latency and error assertions:

```yaml
assertions:
  - metric: request.p50
    operator: "<"
    value: 200ms

  - metric: request.p95
    operator: "<"
    value: 500ms

  - metric: request.p99
    operator: "<"
    value: 1000ms

  - metric: error_rate
    operator: "<"
    value: 1%

  - metric: throughput
    operator: ">="
    value: 1000rps
```

Infrastructure assertions (later phases):

```yaml
  - metric: cpu_utilization
    operator: "<"
    value: 80%

  - metric: db_connections
    operator: "<"
    value: 90%
```

Assertions are the contract between the scenario author and the deployment pipeline. If a scenario passes all its assertions, the change is cleared. If not, the feedback loop sends the developer back to investigate.

---

## 11. Result Model and Reports

A run produces a structured result. The result includes not only aggregate metrics and assertion verdicts but also per-service impact analysis — the platform should detect which specific services contributed most to any degradation.

### Run Result Structure

```text
Run
├── ID
├── Scenario
├── Environment
├── Start / End
├── Traffic configuration
├── Chaos configuration
├── Aggregate metrics (p50, p95, p99, error rate)
├── Per-service detected impact
├── Assertion verdicts
├── Overall pass / fail
└── Observability references (trace IDs, dashboard links)
```

### Example Report

```text
┌──────────────────────────────────────────────┐
│  Checkout Resilience Test                    │
├──────────────────────────────────────────────┤
│                                              │
│  Load:          500 RPS                      │
│  Duration:      5m                           │
│                                              │
│  Injected:                                   │
│      inventory latency +300ms                │
│                                              │
│  Results:                                    │
│      p50:       143ms                        │
│      p95:       612ms                        │
│      p99:       941ms                        │
│      errors:    0.42%                        │
│                                              │
│  SLO:                                        │
│      p99 < 1000ms       PASS                 │
│      errors < 1%        PASS                 │
│                                              │
│  Detected:                                   │
│      inventory p99      +327ms               │
│      order p99          +291ms               │
│      DB p99             +12ms                │
│                                              │
│  Result: PASS                                │
└──────────────────────────────────────────────┘
```

The "Detected" section is what makes the report actionable. When a resilience test passes but latency is close to the threshold, per-service attribution tells the team exactly where the time is going: in this example, inventory absorbed most of the injected latency, the order service added its own overhead, and the database was not a factor.

### Report Sharing

Reports are designed to be passed around teams. A reliability test result should be a self-contained artifact that any engineer can read and understand without needing to reproduce the run. Teams share reports during release reviews, incident retrospectives, and capacity planning to build a shared, evidence-based picture of system behavior.

---

## 12. Observability Integration

Every test run is assigned a unique run ID. The run ID propagates through the entire request chain:

```text
CLI → Control Plane → Runner → Requests (as header/metadata)
```

The run ID is used to correlate Prometheus metrics, structured logs, distributed traces, test output, and chaos injection events. This correlation is what allows the result page to answer "why did p99 latency increase?" rather than merely reporting that it increased.

---

## 13. Governance and Blast Radius

Chaos capabilities require stronger controls than ordinary load tests. The platform enforces environment-specific policies that are evaluated before execution, not after.

By environment:

- **Development** — broad experimentation allowed.
- **Staging** — most experiments allowed; rate and duration caps enforced.
- **Production** — explicitly approved experiments only; RBAC authorization required.

Constraints enforced at the control plane:

- Maximum RPS.
- Maximum duration.
- Allowed targets.
- Allowed chaos types.
- Maximum affected replicas.
- Allowed Kubernetes namespaces.

An experiment that violates any constraint is rejected before a single request is sent. The platform should fail closed, not open.

---

## 14. Engineering Rigor

This platform is not a prototype that happens to work; it is infrastructure that the entire organization depends on to make deployment decisions. That distinction demands specific engineering discipline.

### The platform must be tested

The Feedback System validates other services. Nothing validates the Feedback System unless we make it so. The platform needs its own test suite — unit tests for the control plane logic, integration tests for the runner interface, end-to-end tests that run a known scenario against a known service and verify the result matches expectations. If the platform has a bug in its latency measurement or assertion evaluation, every team using it gets wrong answers silently.

### Determinism over convenience

Two runs of the same scenario against the same environment in the same state should produce results within a defined variance band. If they do not, the platform has a measurement problem, and that problem must be solved before the results can be trusted for deployment gating. Sources of non-determinism (shared staging environments, background traffic, noisy neighbors) should be documented, and the platform should provide tools to account for them — baseline comparisons, variance tracking, confidence intervals.

### Failure modes must be designed, not discovered

What happens when the control plane loses contact with a runner mid-test? What happens when a chaos injection fails to clean up? What happens when a runner exhausts its own resources and becomes the bottleneck instead of the target service? Each of these failure modes needs a designed response: timeout, cleanup, invalidation of results. They should not be left for production to discover.

### Versioned, auditable configuration

Every scenario definition, every policy, every assertion threshold lives in version control. When a test result says "PASS," there must be a way to reconstruct exactly what was tested, what the thresholds were, and what version of the platform evaluated them. This is not optional for a system that gates deployments.

### Incremental correctness over feature velocity

It is better to have a platform that reliably benchmarks one service with accurate numbers than a platform that nominally supports chaos testing, distributed execution, and multi-cluster scheduling but produces results no one trusts. Each phase must work correctly before the next phase begins. The phases described in this document are ordered by this principle.

---

## 15. Implementation Phases

### Phase 1 — Load Testing MVP

Build the core loop: CLI → Control Plane → Runner → Results.

Deliver:

- `mytest` CLI with `run`, `benchmark`, and `results` commands.
- Control plane with scenario registry and run manager.
- HTTP runner backed by bombardier or vegeta.
- gRPC runner backed by ghz.
- Basic latency and error-rate results.
- Declarative scenario definitions in YAML.
- CI integration via `--wait` flag and exit codes.
- Structured result output.

Do not implement a custom load generator. Do not implement chaos. Do not implement distributed execution. Get the feedback loop working end-to-end with accurate, trustworthy numbers first.

### Phase 2 — Scenario Engine and Assertions

Add:

- Reusable, versioned scenarios as the primary abstraction.
- Multi-target scenarios (test a service and its dependencies in one run).
- Test phases (ramp-up, steady state, ramp-down).
- Assertion evaluation with pass/fail verdicts.
- Result persistence and historical storage.
- Regression detection against previous runs.

### Phase 3 — End-to-End Flows

Add:

- Flow runner backed by k6 or gateline.
- Multi-step workflow definitions (sequential requests with dependencies).
- Partial-flow testing (subset of the architecture).
- Full end-to-end user journey testing through the gateway.
- Per-service latency attribution in results.

### Phase 4 — Observability

Integrate:

- Run ID propagation through all requests.
- Prometheus metric correlation.
- Log correlation.
- Distributed trace correlation.
- Automated per-service impact detection (the "Detected" section in reports).
- Bottleneck summaries.

### Phase 5 — Chaos

Add controlled fault injection:

- Latency injection.
- Error injection.
- Network partition simulation.
- Process and container termination.
- CPU and memory pressure.
- Chaos scheduling coordinated with traffic timing.
- Automatic fault cleanup with verification.

Use existing infrastructure mechanisms (Kubernetes, service mesh fault injection) where possible rather than implementing low-level fault injection from scratch.

### Phase 6 — Distributed Execution

Move runners into the target environment for generating traffic volumes that a single machine cannot produce:

```text
Control Plane
      │
      ▼
Scheduler
      │
      ├── Runner A → cluster A
      ├── Runner B → cluster B
      └── Runner C → cluster C
```

### Phase 7 — Reliability Platform

Add:

- Scheduled recurring tests.
- Release gates integrated with the deployment pipeline.
- Historical performance trend analysis.
- Service-level reliability profiles.
- Team ownership and notification routing.
- Environment policies enforced via RBAC.
- Dashboards and report sharing.
- Automated recommendations ("inventory p99 has regressed 40% over the last five releases").

---

## 16. Repository Structure

```text
feedback-system/
├── cmd/
│   ├── mytest/              # CLI binary
│   └── controller/          # Control plane binary
│
├── api/
│   └── proto/               # gRPC service definitions
│
├── internal/
│   ├── controlplane/
│   │   ├── registry/        # Scenario registry
│   │   ├── runmanager/      # Run lifecycle
│   │   ├── scheduler/       # Job scheduling
│   │   └── policy/          # Environment policies, RBAC
│   │
│   ├── runners/
│   │   ├── http/            # bombardier, vegeta
│   │   ├── grpc/            # ghz
│   │   ├── flow/            # k6, gateline
│   │   └── chaos/           # fault injection agent
│   │
│   ├── scenarios/           # Scenario parsing and validation
│   ├── assertions/          # Assertion evaluation engine
│   ├── results/
│   │   ├── store/           # Persistence
│   │   ├── analysis/        # Per-service attribution
│   │   └── reports/         # Report generation
│   │
│   └── observability/       # Run ID propagation, correlation
│
├── scenarios/               # Scenario definitions (YAML)
│   ├── checkout-load.yaml
│   ├── checkout-resilience.yaml
│   └── inventory-capacity.yaml
│
└── deploy/
    └── kubernetes/
```

The runner processes run as separate containers, not embedded in the control plane. This allows scaling runners independently and replacing the underlying load generator without touching the control plane.

---

## 17. Design Principles

**Start as an orchestration layer, not a custom benchmark engine.** Reuse mature, battle-tested generators. The platform's value is in the feedback loop, scenario abstraction, and result analysis — not in reimplementing HTTP load generation.

**Make scenarios the stable abstraction.** The underlying tools can be swapped without changing a single scenario definition or developer workflow.

**Treat traffic and chaos as composable.** A scenario is traffic plus faults plus assertions. Any combination is valid at any testing level.

**Support every level of granularity.** Component benchmarks, dependency chains, partial flows, full end-to-end journeys, and resilience tests are all first-class. No level is privileged; no level is an afterthought.

**Make results reproducible.** Store the exact scenario definition, environment state, software versions, and runner configuration for every run so that any result can be understood and compared months later.

**Make the same test executable everywhere.** The same scenario runs from a developer's terminal, in CI, as a nightly job, and as a release gate. One definition, many contexts.

**Keep production experimentation opt-in and constrained.** The reliability platform must itself be safe. Fail closed. Require explicit authorization. Enforce blast-radius limits before execution.

**Engineer it; do not vibe it.** Every component of this platform needs designed failure modes, a test suite, deterministic behavior, and versioned configuration. The Feedback System is infrastructure. Treat it accordingly.

---

## 18. End State

The desired workflow:

```bash
# Developer benchmarks a service they changed
mytest benchmark inventory --rps 1000

# Developer runs the relevant scenario interactively
mytest scenario checkout-resilience

# CI runs a targeted test on PR
mytest run inventory-regression --environment staging --wait

# CI runs broader tests on merge
mytest run checkout-load --environment staging --wait

# Release gate
mytest run critical-path --environment staging --wait

# Chaos experiment by reliability engineer
mytest chaos inject inventory --latency 500ms
mytest run checkout-resilience --environment staging --wait

# Production experiment, with authorization
mytest run checkout-resilience-prod --environment production --wait
```

Behind every command is the same platform — the Feedback System — composed of the same four pillars:

```text
                        Feedback System
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
     Workflows &         Chaos Testing       Observability
     Workloads                │                   │
          │              Fault Injection      Metrics
     HTTP / gRPC              │               Logs
     k6 / gateline            │              Traces
          │                   │                   │
          └───────────────────┼───────────────────┘
                              │
                        Scenarios
                              │
                     SLO Assertions
                              │
                  Pass / Fail / Analysis
                              │
                  Per-Service Attribution
                              │
                    Shareable Reports
```

The feedback loop is the product. Every feature, every phase, every design decision exists to make that loop faster, more accurate, and more trustworthy — so that deploying with confidence becomes the default, not the exception.
