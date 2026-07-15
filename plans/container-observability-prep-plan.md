# 4-Week Prep Plan: Container/Runtime Engineering + Observability Internals

_For: fresher, targeting 25-30 offers at **container/runtime companies** (Docker, containerd/CNCF ecosystem, gVisor, Kata, Sysbox, etc.) and **observability companies** (Datadog, Grafana Labs, Honeycomb, Chronosphere, Elastic, New Relic, SigNoz, etc.) — not general DevOps/SRE-at-a-random-company roles._

**Framing shift:** these companies build the tools other engineers use. You're not being hired to _run_ Prometheus or Docker — you're being hired to reason about how a scheduler, a runtime, a TSDB, or a tracing pipeline is _engineered_: data structures, protocols, failure modes, performance tradeoffs. Every day below leans toward that "how is this built" depth rather than "how do I operate this" usage.

Each "Day" below is broken into **micro-sessions**, not one long block — built for a 5-10 minute attention span, not a 25-minute Pomodoro. Do NOT try to do a whole day in one sitting. Spread each day's micro-sessions across the day if needed, with real breaks (stand up, walk, no phone) between them.

### The Struggle-Before-Lookup Rule (read this before Day 1)

Before searching or asking AI anything, give yourself a **mandatory 10-15 min attempt first** — write down what you think is happening, what you'd try, where you're stuck, even if it's wrong. Only after that attempt is written down are you allowed to look something up, and even then ask for a _hint_, not the answer. This is how the hands-on parts actually build skill instead of just producing a working command you don't understand.

### Micro-session template (repeat this structure for every "Day")

1. **[5 min] Orient** — read just the concept name + 1-2 sentence definition from the resource. Stop there. Don't read further yet.
2. **[Break — 2-3 min]** Stand up, look away from the screen.
3. **[10-15 min] Struggle** — attempt the hands-on part yourself first, per the rule above. Write down your attempt even if wrong/incomplete.
4. **[Break — 2-3 min]**
5. **[5-10 min] Resolve** — now read the rest of the resource / look up a hint to unstick yourself. Fix your attempt.
6. **[Break — 2-3 min]**
7. **[5 min] Notes** — 3 bullet points in your notes doc, in your own words.

That's ~30-40 min total per "Day" topic, split into 5 short chunks with breaks — not one sitting. If even 5 minutes feels too long some days, cut every step in half and do two rounds instead of one. Consistency across days matters far more than session length.

### The Curiosity Parking List (for when an article's own links distract you)

Good technical blogs — iximiuz's especially — are written with tons of internal links to other fascinating posts. That's great content, and exactly why it derails you: every link is a tiny immediate-gratification pull, same mechanism as reaching for AI too early. Don't suppress the curiosity — redirect it:

1. Keep a running **"Parking List"** in your notes doc — just a bulleted list of titles/links.
2. When a link in an article grabs your attention but isn't the one assigned for today's Day, **do not click it.** Write its title in the Parking List and keep reading only the assigned material.
3. Only open the exact resource listed for the current Day — treat every other link on the page as invisible until you've finished today's micro-session.
4. Once a week (e.g., Saturday), give yourself a real **20-30 min "curiosity hour"** to freely browse your Parking List, no rules, no notes required. This turns the distraction into a scheduled reward instead of something you're white-knuckling against every day — and most of what felt urgent in the moment won't even feel that interesting a week later, which is useful information on its own.

---

## WEEK 1 — Container/Runtime Engineering: Linux Primitives & OCI Spec

- [x] **Day 1** — Namespaces overview (pid, net, mnt, uts, ipc, user). Read `man namespaces`. **Reading:** iximiuz "How to on starting processes (mostly in Linux)" (fork/exec basics before namespaces make sense). Hands-on: `unshare --pid --fork --mount-proc bash`, inspect `/proc/[pid]/ns/`.
- [x] **Day 2** — cgroups v1 vs v2. Explore `/sys/fs/cgroup/`. **Reading:** official kernel cgroup docs (no dedicated iximiuz post for this one — stick to primary source). Hands-on: manually limit memory/CPU for a process by writing to cgroup files (no Docker).
- [ ] **Day 3** — Union filesystems (overlayfs). **Reading:** `man overlayfs` (primary source). Hands-on: manually mount an overlayfs with lower/upper/work dirs, understand copy-on-write.
- [ ] **Day 4** — Build a "container from scratch": combine `unshare` + `chroot` + cgroup limits into one script. This is your #1 talking point for internals questions.
- [ ] **Day 5** — (github.com/opencontainers/runtime-spec) — skim `config.json` structure, lifecycle (create/start/kill/delete). **Reading:** the `spec.md` file directly in that repo (primary source, ~20-30 min, no interpretation layer) + iximiuz "Containers Aren't Linux Processes" (how the OCI Runtime Spec defines a standard container) alongside it — explains _why_ the spec looks the way it does.
- [ ] **Day 6** — Read OCI Image Spec — manifest, layers, config, how an image is just tarballs + JSON metadata. **Reading:** `spec.md` in `github.com/opencontainers/image-spec` (primary source) **then, in this order:** iximiuz "You Don't Need an Image To Run a Container" → "You Need Containers To Build an Image" → "Not every container has an operating system inside" (scratch/distroless images). These three are written specifically to build on each other for exactly this concept.
- [ ] **Day 7** — Hands-on: install `runc`, hand-build an OCI bundle (rootfs + config.json), run it directly with `runc run`. **Reading:** iximiuz "Implementing Container Runtime Shim: runc" (in-depth analysis of how runc actually starts a containerized process). **Hands-on alternative/supplement:** labs.iximiuz.com/challenges filtered by `container-runtime` — auto-graded, so you know immediately if you actually understood it instead of assuming. Review week 1 notes.

## WEEK 2 — Container/Runtime Engineering: Runtime Architecture & Internals

- [ ] **Day 8** — Draw and memorize the call chain: Docker CLI → dockerd → containerd → containerd-shim → runc → kernel. Explain _why_ each layer exists (what problem it solves) — not just the names. **Reading:** you already have the core diagram from the iximiuz learning-path article's "containerd vs. docker" section — re-derive it from memory before re-reading.
- [ ] **Day 9** — Install containerd standalone (no Docker). Pull an image and run it with `ctr`. **Reading:** iximiuz "Why and How to Use containerd From Command Line" — this is written as exactly this hands-on exercise. Read how containerd's snapshotter and content store work (this is closer to "how it's built" than "how it's used").
- [ ] **Day 10** — CRI (Container Runtime Interface) as a gRPC API — read the actual proto definitions (`cri-api`). **Reading:** `github.com/kubernetes/cri-api` — just the `.proto` definitions, ~15 min, primary source. Understand it as a protocol design problem: why Kubernetes needed a stable interface instead of coupling to Docker directly.
- [ ] **Day 11** — Compare runtimes as engineering tradeoffs: runc (namespaces/cgroups, minimal overhead) vs gVisor (userspace kernel intercepting syscalls — sentry/gofer architecture) vs Kata Containers (lightweight VM, hardware-level isolation). **Reading:** gvisor.dev/docs/architecture_guide/ (official, written by the team that built it — best source for the sentry/gofer model) + katacontainers.io → Docs → Architecture (official, same logic — go to source, skip third-party summaries). Know the performance/isolation tradeoff curve, not just names.
- [ ] **Day 12** — Deep-dive one runtime's source at a shallow level: skim `runc`'s Go source for how it sets up namespaces/cgroups via syscalls. **Optional deeper exercise:** iximiuz "Implementing a Container Manager From Scratch (conman)" — build a minimal container manager in Go; this is the container-manager-level equivalent of your Day 4 "container from scratch" exercise, do it only if Day 4 felt solid and you want more.
- [ ] **Day 13** — Security internals: seccomp-bpf (how it filters syscalls at the kernel level), Linux capabilities, rootless containers (user namespaces + newuidmap/newgidmap), AppArmor/SELinux basics. **Reading:** `man capabilities`, `man seccomp` (ground-truth primary source) + Datadog Security Labs' "Container Security Fundamentals" series (hosted on labs.iximiuz.com tutorials — fills the "why" gap concisely) + iximiuz "The Need For Slimmer Containers" (thick container vulnerabilities) — ties attack surface directly to what you're isolating.
- [ ] **Day 14** — Consolidate: write a one-page "container runtime architecture" summary from memory covering call chain, CRI, and 3-runtime comparison, then check against notes.

## WEEK 3 — Observability Internals: How the Tools Are Built

- [ ] **Day 15** — Three pillars + golden signals as a foundation, then go deeper: read how Prometheus's TSDB actually stores data — blocks, chunks, WAL (write-ahead log), compaction. **Reading:** Fabian Reinartz's "Writing a Time Series Database from Scratch" (Prometheus blog, written by a Prometheus core author — closest thing to a primary source on TSDB internals) + official docs prometheus.io/docs/prometheus/latest/storage/. This is core "how is it built" material for observability companies.
- [ ] **Day 16** — Cardinality: what it means, why it's the #1 operational/engineering problem in metrics systems (label combinations exploding storage/query cost), how systems mitigate it (relabeling, cardinality limits, recording rules). **Reading:** Robust Perception / Grafana Labs blog posts on cardinality (search "cardinality" on grafana.com/blog) — Brian Brazil (Prometheus core dev) and Grafana's engineers write the most technically precise explanations of this exact problem.
- [ ] **Day 17** — PromQL internals: how `rate()`/`histogram_quantile()` actually compute over time-series data, not just syntax. **Reading:** iximiuz's "Learning Prometheus and PromQL" series — matches your "efficient learning" need better than the official docs' reference-style writing. Install Prometheus, query real data, reason about what's happening under the query.
- [ ] **Day 18** — OpenTelemetry Collector architecture: receivers → processors → exporters pipeline. **Reading:** opentelemetry.io/docs/collector/architecture/ (official — don't substitute a blog for this, it's short and exactly what you need). Read the OTLP protocol spec. This is a very common deep-dive at observability companies — know it structurally, not just "OTel is a standard."
- [ ] **Day 19** — Sampling strategies for tracing: head-based vs tail-based sampling, why tail-based is architecturally hard (you need to buffer/decide after the fact), context propagation via W3C Trace Context headers. **Reading:** OpenTelemetry docs → "Sampling" page + Honeycomb's blog post on tail-based sampling (search "tail-based sampling Honeycomb" — their engineers literally built production tail-sampling, unusually concrete writeups).
- [ ] **Day 20** — Log pipeline internals: contrast Loki's index-light "just index labels, grep the rest" design vs Elasticsearch's full-text inverted index — cost, latency, and query-flexibility tradeoffs. **Reading:** grafana.com/docs/loki → "Architecture" page (direct from source, explains the index-light design decision itself). Also worth 30 min: **ebpf.io** → "What is eBPF?" (the free short guide, not the full book) for eBPF-based observability (how tools like Pixie/Cilium Hubble get metrics/traces with zero app instrumentation).
- [ ] **Day 21** — SRE practice in depth: error budgets and how they gate releases, multi-window multi-burn-rate alerting (why naive threshold alerts create pager noise), on-call/postmortem culture. **Reading:** sre.google/sre-book — read only "Embracing Risk" (error budgets) + "Practical Alerting" chapters, skip the rest of the book. Consolidate week 3 notes.

## WEEK 4 — Integration, One Deep Product, & Interview Polish

- [ ] **Day 22** — Pick ONE company/product from each track (e.g., containerd or Sysbox for runtimes; Grafana LGTM stack or an OTel Collector deployment for observability) and read its architecture docs deeply. **Reading:** opentelemetry.io/docs/demo/ (official OTel Demo app) — look at how it's wired for a real reference architecture in ~30 min, don't necessarily reuse all of it. Going deep on one real product signals genuine interest far more than broad shallow coverage.
- [ ] **Day 23** — Build your demo project: small app instrumented with OpenTelemetry (metrics + traces), run in a container you understand the runtime layer of. **Sandbox option:** labs.iximiuz.com and Killercoda — skips VM/local-install overhead entirely so you spend your time on the concept, not `apt-get` troubleshooting.
- [ ] **Day 24** — Wire up Prometheus scraping + a Grafana dashboard; deliberately introduce a high-cardinality label so you can explain the problem live in an interview.
- [ ] **Day 25** — Add tail-based-sampling-style reasoning to your traces (even conceptually, if not fully implemented) and ship logs via Loki alongside metrics/traces.
- [ ] **Day 26** — Push project to GitHub with a README + architecture diagram covering _both_ the runtime layer (what's isolating your container) and the observability layer (how you're instrumenting it). This dual angle is your differentiator.
- [ ] **Day 27** — Revise weak spots flagged in your running notes doc across all 3 weeks.
- [ ] **Day 28** — Do 2-3 self-recorded mock interviews using the question bank below. Time yourself — aim for crisp 60-90 second answers, and practice the "why is X engineered this way" framing, not just "what is X."

---

## Interview Question Bank (by stage)

### Linux Primitives / "Container from Scratch"

- What actually _is_ a container? Walk me through it without saying "Docker."
- Explain the difference between a container and a VM at the kernel level.
- What are Linux namespaces? Name all 6-7 types and what each isolates.
- What's the difference between cgroups v1 and v2?
- How does overlayfs enable image layering and copy-on-write?
- If I run the same image twice, what's shared vs. isolated between the two containers?

### OCI Spec

- What is the OCI, and why does it exist? (Hint: standardization after early Docker-only ecosystem)
- What's the difference between the OCI Runtime Spec and the OCI Image Spec?
- What's inside `config.json` in an OCI bundle?
- What is `runc`, and how does it relate to the OCI Runtime Spec?
- Can you run an OCI-compliant image without Docker at all? How?

### Container Runtimes (engineering depth)

- Walk me through what happens end-to-end when you run `docker run nginx`, at every layer including syscalls.
- What is containerd, and what does it do that Docker CLI doesn't? What's the role of a snapshotter?
- What is the CRI, and why did Kubernetes move away from dockershim? What protocol does CRI actually use?
- Compare runc, gVisor, and Kata Containers architecturally — how does gVisor's sentry/gofer model intercept syscalls? How does Kata get VM-level isolation without VM-level boot overhead?
- What's a "shim" and why does containerd use one per container? What happens to the shim if containerd itself restarts?
- How would you run a container as non-root, and why does it matter? How do user namespaces make rootless containers possible?
- What Linux capabilities would you drop for a hardened container, and why? What's the difference between a capability and a seccomp filter?
- If you were designing a _new_ container runtime, what tradeoffs would you have to make between isolation strength and startup latency?

### Observability (engineering depth)

- What are the three pillars of observability, and when do you reach for each?
- How does Prometheus's TSDB actually store data on disk? What's a block, a chunk, and the WAL for?
- What is cardinality, and why is it the central engineering problem in metrics systems? How would you design a system to protect against a cardinality explosion?
- Explain the OpenTelemetry Collector's receiver/processor/exporter pipeline. Why is this pluggable architecture useful?
- Head-based vs tail-based sampling — why is tail-based sampling harder to build? How would you architect a tail-based sampler?
- Compare Loki's and Elasticsearch's indexing strategies. What query patterns does each optimize for, and what does each sacrifice?
- What is eBPF, and how does it let tools like Cilium Hubble get network-level observability without code instrumentation?
- Define SLI, SLO, SLA, and error budget — and describe multi-window multi-burn-rate alerting and why it reduces pager noise vs a naive threshold.
- How would you debug a slow API call across 5 microservices in production, using traces specifically?

### Integration / Design Questions (asked even to freshers at these companies)

- If a container is OOMKilled, what would you check first — and how does that connect to what cgroups are reporting?
- How would you design an observability pipeline for a system you didn't build, keeping cardinality and cost under control?
- Why might a container/runtime company also care deeply about observability internals (hint: they need to observe their own runtime's behavior)?
- Walk me through your demo project's architecture — both the runtime/isolation layer and the instrumentation layer.
- What's one design decision in Prometheus, containerd, or runc that you disagree with or would do differently, and why?

---

## How to Actually Get In the Door (Fresher, Tier-3 College, Targeting 25-30 Companies)

**Core reality check:** Big tech campus drives (AWS/Google/Microsoft new-grad programs) are gated by college tier before any technical evaluation happens — an ATS or placement-cell agreement filters by institute, not skill. No amount of technical depth fixes that specific channel. So drop "campus drive" as a strategy and go off-campus + OSS-first, where college tier barely matters because proof of work in public repos is the actual currency.

### The fast route (do these in parallel with the study plan above)

1. **Contribute to the actual project tied to your target company** — not a random OSS repo. Prometheus/OTel Collector/Loki/Tempo → observability companies; containerd/runc/CRI-O → runtime companies. Start with "good first issue" labels, move to real bugs.
2. **Apply to CNCF's paid mentorship programs** — LFX Mentorship (term-based, next term opens ~July 2026) and GSoC 2026 are both active right now and explicitly list Prometheus, Jaeger, and containerd-ecosystem projects. Since 2020, 25 LFX mentee alumni have gone on to become CNCF project maintainers — this is a legitimate, college-blind fast track.
3. **Be visible where maintainers already are**: CNCF Slack (#prometheus, #opentelemetry, #containerd, #cri-o), project GitHub Discussions. Answer others' questions before asking your own.
4. **Publish your deep-dive learning** (TSDB internals, cardinality, OTel Collector architecture, runtime internals from weeks 1-3) as blog posts. Observability/runtime engineers actively read and share this kind of writing.
5. **Ship the demo project as a real public repo**, not a throwaway — link it everywhere.
6. **Attend/volunteer at CNCF India meetups** (Bangalore, Hyderabad, Pune, Delhi-NCR) — engineers from target companies actually show up and speak there.
7. **Apply direct via careers pages, never through placement cells.** Search for "Associate Software Engineer," "SDE-1," "Site Reliability Engineer I" — not "Campus Hire" or "New Grad Program." Off-campus reqs usually don't carry the same college-tier filter.
8. **Get referrals once you're active in a project's community.** A referral from a maintainer or engineer you've interacted with routes around the ATS/college-tier screen almost entirely — this is your single biggest lever.
9. **Direct outreach on LinkedIn/Twitter once you have proof of work** — 2-3 merged PRs or a published deep-dive is enough to message engineers directly and ask for advice or a referral.

### Company targets, re-weighted for your situation

**Priority tier — realistic, worth the most effort:**

|Company|Track|Why realistic for you|
|---|---|---|
|SigNoz|Observability|India-based (Bangalore), founders/early engineers often personally review strong GitHub profiles; no hiring volume to justify tier-filtering|
|Last9|Observability|India-based, YC-backed, small team = high visibility for contributors|
|Middleware.io|Observability|India-origin, same logic as above|
|Red Hat (Pune)|Runtime|Real India engineering office, apply off-campus directly — owns CRI-O/Podman|
|Cisco / Splunk / AppDynamics (Bangalore, Hyderabad)|Observability|Large India R&D centers, off-campus reqs available|
|Grafana Labs|Observability|100% remote, no central office, provides health coverage for India-based team members — confirms real India hiring; no fresher pipeline, so OSS contribution to Loki/Tempo/Mimir is your entry|
|Elastic|Observability|Global remote hiring, OSS contribution weighs heavily|

**Secondary tier — worth applying, lower odds without a strong OSS profile first:**

|Company|Track|Note|
|---|---|---|
|AWS (Firecracker) / Google (gVisor) / Microsoft (Kata/WSL2)|Runtime|Deprioritize their _campus_ channel entirely; off-campus + referral only|
|Palo Alto Networks (Prisma Cloud) / Isovalent-Cilium (Cisco)|Runtime/security|Large India R&D presence, apply off-campus|
|Docker, Inc. / SUSE-Rancher / Chainguard|Runtime|Small, senior-biased — OSS contribution to Moby/containerd/K3s is basically the only door|
|Datadog / Honeycomb / Chronosphere / New Relic|Observability|Concentrated in US/EU offices, very competitive as a fresher — apply broadly but don't anchor your plan here|
|VictoriaMetrics / InfluxData / Sentry|Observability|Small remote teams — OSS contribution is close to the only path in|

### Bottom line

Weight your month like this: **study plan (technical depth) + OSS contribution (weeks 2-4 onward, running in parallel) + priority-tier applications sent off-campus, not through placement cells.** College tier stops being a factor the moment a maintainer or engineer is looking at your commit history instead of your resume.

---

## Proof of Work Portfolio (build this in parallel, timed against the weeks above)

Your proof of work should let anyone verify your claims in under two minutes, without trusting your resume's word for it. Build it as a connected stack, not one artifact.

- [ ] **Start now (Week 1):** Publish your first blog post from Week 1 notes (e.g., "container from scratch" walkthrough). Free, requires nothing to be "ready," and it's the earliest piece of proof you can produce.
- [ ] **Week 1-2:** Open your first "good first issue" PR on a target project (containerd/runc/CRI-O for runtime, Prometheus/Loki/OTel Collector for observability). Even a docs fix counts — pin the repo on your GitHub profile once merged.
- [ ] **Week 2-3:** Aim for 3-5 total merged PRs across target projects by end of Week 3. Quality/merge-worthiness over volume.
- [ ] **Week 2-3:** Publish 2-3 more blog posts from your deep-dive notes (TSDB internals, cardinality, runtime architecture). 3-5 solid posts total is the target.
- [ ] **Week 3:** Once you have 1-2 merged PRs and feel steadier, start browsing Algora bounties in under-contested infra/Go repos. Link your Algora profile publicly once you have any history there.
- [ ] **Week 4:** Build the flagship demo project (runtime layer + observability layer). Document it like a product: README with architecture diagram, "why I built this" section tying it to real concepts, screenshots/GIFs or a live demo link if possible.
- [ ] **Week 4:** Assemble the single link that ties it together — a personal site or GitHub profile README linking: flagship project, best 2-3 blog posts, Algora profile, one-line focus statement. This becomes the link you put at the top of every application and outreach message.
- [ ] **Ongoing, background effort the whole time:** Answer questions in CNCF Slack / GitHub Discussions / Stack Overflow for your target projects. Low effort per instance, compounds into maintainer recognition over time.
- [ ] **Stretch goals, pursue if the timing lines up:** LFX Mentorship Term 3 (applications Aug 3-18) — outweighs most of the above combined for credibility if you land it. A 5-10 min recorded lightning talk at a CNCF India meetup — rare enough among freshers to be disproportionately persuasive.

---

## Notes Doc Template (fill in daily)

```
Date:
Topic:
3 things I learned:
1.
2.
3.
1 thing I'm still unclear on:
```