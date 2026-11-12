# Non-Functional Requirements — URL Shortener

> Status: **Pending review**. All sections written; lab environment and estimates E-5 to E-10 still open.
> Inputs: [Functional requirements](01-functional-requirements.md). Scenario form: *Software Architecture in Practice*, ch. 3. Estimates: *System Design Interview*, Vol. 1, "Design a URL Shortener".

## 1. How to read this document

- Every requirement is a **quality attribute scenario** in the SAiP six-part form: source, stimulus, artifact, environment, response, response measure.
- Scenario IDs (`PERF-1`, `AVAIL-2`, ...) are stable. ADRs, experiments and dashboards refer to them, so an ID is never reused or renumbered. A dropped scenario is marked **Withdrawn**, not deleted.
- Load levels in the *environment* row ("normal load", "peak load") refer to the estimates in [§7](#7-back-of-envelope-estimates). Response measures are checked at **lab scale**; book scale is there to reason about the design, not to test.
- **Durability** is deferred for now; it may come back as its own section later.
- **Security is a lab non-goal** (no users, no authentication; see FR §5). Abuse controls such as rate limiting only appear if a scenario here needs them.

### Measurement conventions

These apply to every scenario unless the scenario says otherwise.

| Term | Meaning |
|---|---|
| Latency | Time from the load generator sending the request to receiving the last byte of the response, measured on the same host or network as the system. Internet round trip is excluded, because the lab cannot control it. The service also exports a server-side histogram so the two can be compared. |
| Percentiles | Computed over the steady-state window of a run. The first 60 s (warm-up) are discarded. |
| Error | Any `5xx`, timeout or connection failure. `404` and `410` are correct answers (FR-6, FR-8), not errors. |
| Redirect request | `GET /{code}` for any code: existing, unknown or expired. Unless stated otherwise, the code mix is 95 % existing, 4 % unknown, 1 % expired. |

## 2. Utility tree

Ranked by **business value** (how much the scenario matters to a user of the service) and **difficulty** (how hard it is to meet with this architecture). Both are H/M/L. The (H, H) scenarios drive the architecture first.

| Quality attribute | Refinement | Scenario | Value | Difficulty |
|---|---|---|---|---|
| Performance | Redirect latency | [PERF-1](#perf-1-redirect-latency-under-normal-load) Redirect under normal load, p99 < 50 ms | H | M |
| Performance | Throughput and latency under peak | [PERF-3](#perf-3-redirect-latency-under-peak-load) Redirect sustains 2× normal load, p99 < 100 ms, errors < 0.1 % | H | H |
| Performance | Cold reads | [PERF-4](#perf-4-redirect-latency-for-cold-links) Redirect for links not in cache, p99 < 100 ms | M | M |
| Performance | Create latency | [PERF-2](#perf-2-shorten-latency-under-normal-load) Shorten under normal load, p99 < 100 ms | M | L |
| Availability | Cache failure | [AVAIL-2](#avail-2-cache-unavailable) Redirects keep working with the cache down, errors < 0.1 % | H | H |
| Availability | Storage node failure | [AVAIL-3](#avail-3-storage-node-fails) Redirects unaffected; shorten back within 30 s | H | H |
| Availability | Instance failure | [AVAIL-1](#avail-1-a-redirect-instance-crashes) One redirect instance crashes, recovery < 10 s, no errors after | H | M |
| Availability | Fault isolation | [AVAIL-4](#avail-4-write-overload-does-not-affect-redirects) A write flood doesn't degrade redirects | M | M |
| Scalability | Write growth | [SCAL-3](#scal-3-write-traffic-grows) Shorten load grows G×, PERF-2 holds by adding instances, zero duplicate codes | H | H |
| Scalability | Read growth | [SCAL-1](#scal-1-redirect-traffic-grows) Redirect load grows G×, PERF-3 holds by adding instances only | H | M |
| Scalability | Data growth | [SCAL-2](#scal-2-dataset-grows-10) Dataset grows 10×, cold-read p99 degrades ≤ 20 % | M | H |
| Scalability | Scale out on utilization | [SCAL-4](#scal-4-scale-out-when-utilization-crosses-the-threshold) CPU or memory > 70 % triggers adding an instance, zero errors, used within 30 s | M | M |
| Observability | Measurability | [OBS-1](#obs-1-every-scenario-is-measurable-from-telemetry) Every scenario's response measure readable from a dashboard | H | L |
| Observability | Diagnosability | [OBS-2](#obs-2-diagnose-a-fault-from-telemetry-alone) Unknown injected fault located from telemetry alone in < 10 min | H | M |
| Observability | Reproducibility | [OBS-4](#obs-4-experiments-are-reproducible) Same experiment rerun 3×, p99 within ±10 % | H | M |
| Observability | Request tracing | [OBS-3](#obs-3-trace-any-slow-or-failed-request) Any slow or failed request traceable hop by hop | M | M |
| Observability | Overhead | [OBS-5](#obs-5-instrumentation-overhead-is-small) Telemetry costs ≤ 5 % of capacity | M | L |

## 3. Performance

Redirects are the product: a visitor waits on every one, and they outnumber writes 10 to 1 (FR §1). Because redirects use `302` (D-1), none are absorbed by browser caches, so every redirect reaches the system. The redirect path therefore gets the tightest budget and the peak-load scenario. Shorten is called by API clients that are less sensitive to latency, so it gets a looser budget.

### PERF-1 Redirect latency under normal load

| Part | Value |
|---|---|
| Source | Visitor (browser, bot or HTTP client) |
| Stimulus | Redirect request `GET /{code}` |
| Artifact | Redirect path (redirect service and the storage behind it) |
| Environment | Normal load (E-3, lab scale), all components healthy, cache warm |
| Response | Returns `302 Found` with the long URL in `Location`; `404` for unknown codes; `410` for expired codes |
| Response measure | p99 latency < **50 ms**, p50 < 10 ms, error rate < 0.01 % |

**Why these numbers:** 50 ms keeps the shortener well below what a person notices when following a link (around 100 ms), leaving room for the internet round trip the lab doesn't measure. The p50 target makes sure the common case is a cache hit, not a database read.

### PERF-2 Shorten latency under normal load

| Part | Value |
|---|---|
| Source | Client (API caller) |
| Stimulus | Shorten request `POST` with a valid long URL and optional expiration |
| Artifact | Shorten path (API service, code generation, storage) |
| Environment | Normal load (E-1, lab scale), all components healthy |
| Response | Generates a unique code, stores the mapping, returns the short URL (FR-1, FR-2) |
| Response measure | p99 latency < **100 ms**, error rate < 0.01 % |

**Why these numbers:** a write is committed to storage before it is acknowledged, so it pays for a database commit and code generation that a cached read skips. Double the redirect budget leaves room for that without letting the write path become sloppy.

### PERF-3 Redirect latency under peak load

| Part | Value |
|---|---|
| Source | Visitors |
| Stimulus | Sustained burst of redirect requests at **2× normal load** for 10 minutes |
| Artifact | Redirect path |
| Environment | Peak load (E-4, lab scale), all components healthy, cache warm, no scaling action taken during the run |
| Response | Serves all requests with the same correct responses as PERF-1. No request is dropped or rejected |
| Response measure | Sustains ≥ E-4 reads/s with no dropped requests; p99 latency < **100 ms**; error rate < 0.1 %; latency returns to PERF-1 levels within 60 s of load returning to normal |

**Why these numbers:** peak traffic is where a read-heavy service usually breaks first (connection pools, cache hot keys, database fallback). Allowing p99 to double but not errors to rise forces the system to have headroom, not just to queue. "No scaling action" keeps this a capacity test; elastic scaling is a Scalability scenario.

### PERF-4 Redirect latency for cold links

| Part | Value |
|---|---|
| Source | Visitors |
| Stimulus | Redirect requests for codes that are not in the cache (long-tail or old links) |
| Artifact | Redirect path, specifically the cache-miss path to storage |
| Environment | Normal load (E-3, lab scale), cache cold for the requested codes, full lab dataset loaded (E-6) |
| Response | Same correct responses as PERF-1; the mapping is loaded into the cache after the first miss |
| Response measure | p99 latency < **100 ms** on misses; a second request for the same code meets PERF-1 |

**Why these numbers:** PERF-1 can be met by a cache alone, which hides a slow database. This scenario makes the storage read path (index, partitioning, connection handling) carry its own budget and checks that misses populate the cache. Unknown codes are the worst case, since they always miss; whether to cache negative lookups is an architecture decision.

## 4. Availability

A broken redirect breaks every page, email and message that embeds the link, and the visitor has no way to retry from somewhere else. A failed shorten only affects one API client, who can retry. So the two paths get different targets, and **redirect must not depend on the write path being healthy**.

**Service level objectives** (book scale, for reasoning about the design):

| Path | SLO | Allowed downtime per 30 days |
|---|---|---|
| Redirect | 99.9 % of requests succeed | ≈ 43 min |
| Shorten | 99 % of requests succeed | ≈ 7 h |

A month-long measurement isn't possible in the lab, so the scenarios below inject one fault each and measure what the SLO depends on: how many requests fail, for how long, and whether the system recovers on its own. Unless stated otherwise, every scenario runs at normal load (E-1 + E-3), and the fault is injected after warm-up. "Recovery" means automatic recovery with no operator action.

### AVAIL-1 A redirect instance crashes

| Part | Value |
|---|---|
| Source | Infrastructure (process crash or container kill) |
| Stimulus | One redirect service instance stops abruptly (`SIGKILL`) |
| Artifact | Redirect service (one of at least two instances) |
| Environment | Normal load, all other components healthy |
| Response | Traffic moves to the remaining instances. In-flight requests on the dead instance may fail; no new requests are routed to it |
| Response measure | Requests fail for ≤ **10 s** after the crash, < 1 % of requests fail in that window, zero errors afterwards. Remaining instances still meet PERF-1 |

**Why these numbers:** an instance crash is the most common fault and the cheapest to tolerate, so its window must be short. 10 s gives health checks time to notice the instance is gone. The response also forces at least two redirect instances, which is the first redundancy tactic in the design.

### AVAIL-2 Cache unavailable

| Part | Value |
|---|---|
| Source | Infrastructure |
| Stimulus | The cache stops responding (container killed or network partitioned) for 5 minutes, then comes back empty |
| Artifact | Redirect path |
| Environment | Normal load, cache previously warm, storage healthy |
| Response | Redirects fall back to storage and still return correct answers (`302`, `404`, `410`). Calls to the dead cache fail fast instead of waiting for timeouts. When the cache returns, it is refilled by normal traffic |
| Response measure | Error rate < **0.1 %** during the outage; p99 < 100 ms (the PERF-4 cold-read budget); PERF-1 met again within **2 min** of the cache returning |

**Why these numbers:** the cache exists for speed, so losing it should cost latency, not availability. That holds only if storage can absorb the full redirect load on its own and the service doesn't hang on cache timeouts, which is what makes this (H, H). If the experiment shows storage can't carry normal load uncached, that's a finding to record in an ADR, not a reason to relax the scenario quietly.

### AVAIL-3 Storage node fails

| Part | Value |
|---|---|
| Source | Infrastructure |
| Stimulus | The storage node that accepts writes stops abruptly |
| Artifact | Storage layer, and both paths that depend on it |
| Environment | Normal load, cache warm |
| Response | Redirects keep working from the cache and any surviving storage node. Shorten fails fast with `503` while writes are unavailable, then resumes on its own. No acknowledged write is lost |
| Response measure | Redirect error rate < **0.1 %** throughout. Shorten unavailable for ≤ **30 s**, and its errors during that time are `503`, not timeouts |

**Why these numbers:** this is the scenario that forces a decision about storage replication and failover. It accepts a short write outage (the shorten SLO is looser) but not a redirect outage. 30 s is long enough for automatic leader failover and short enough that a retrying client barely notices.

### AVAIL-4 Write overload does not affect redirects

| Part | Value |
|---|---|
| Source | Client (a misbehaving or bursty API caller) |
| Stimulus | Shorten requests arrive at **10× peak write rate** (10 × E-2) for 5 minutes |
| Artifact | Shorten path and the resources it shares with the redirect path |
| Environment | Normal redirect load (E-3), all components healthy |
| Response | Shorten requests beyond capacity are rejected quickly (`429` or `503`) rather than queued. Redirects are unaffected |
| Response measure | Redirects still meet PERF-1. Rejected shorten requests get a response in < 50 ms. Accepted shorten requests still meet PERF-2 |

**Why these numbers:** without users or authentication, nothing stops one client from flooding the write path. Since the two paths share storage and possibly processes, a write flood is the most likely way for the less important path to take down the more important one. This scenario forces some form of isolation and load shedding; which one is an architecture decision.

## 5. Scalability

Performance asks how much load the system handles with a fixed set of resources. Scalability asks what happens when the load or the data grows and resources are added. At book scale the gap is huge (≈ 23,000 reads/s and 365 billion records, see §7), so the design must grow by adding instances, not by buying a bigger machine.

Each scenario states the **load the business needs to support** and requires that it's reached by adding resources, not by changing code. How many instances that takes (*n*) is not a requirement: it's an output of the experiment, recorded with the results and used for capacity planning in the architecture.

**Growth target (G):** book scale G = 10 (traffic grows 10× over the 10-year horizon; an assumption, the book doesn't give one). Lab scale G = 2, because one machine can't host 10× the baseline.

**Lab caveat:** on one machine, adding instances only adds capacity while there are free cores within the system-under-test budget. If the target is missed because the machine ran out of cores, that's a lab limitation to record, not a design failure.

### SCAL-1 Redirect traffic grows

| Part | Value |
|---|---|
| Source | Visitors, as the service becomes more popular |
| Stimulus | Sustained redirect load grows to **G × E-4** (G × peak) |
| Artifact | Redirect path |
| Environment | Full lab dataset loaded (E-6); operator may add redirect instances and cache capacity before the run |
| Response | The system serves the new load correctly after resources are added, with no code change and no redesign |
| Response measure | PERF-3 holds at G × E-4. The instance count *n* needed is recorded. Resources per 1,000 reads/s grow by ≤ **10 %** compared with the baseline |

**Why these numbers:** the business target is the load, not the instance count. The cost measure is what keeps the scenario honest: if serving 2× the traffic takes 3× the instances, something shared (a lock, a saturated cache, a connection pool) is limiting growth, and the design won't reach book scale affordably. A redirect needs no coordination between instances, so near-constant cost per request is a fair expectation.

### SCAL-2 Dataset grows 10×

| Part | Value |
|---|---|
| Source | Clients, over time |
| Stimulus | The number of stored links grows to 10× the experiment dataset (10 × E-6) |
| Artifact | Storage layer and the cache-miss read path |
| Environment | Normal load (E-1 + E-3), same resources as before the growth |
| Response | Redirects and shortens keep returning correct answers with no manual data migration |
| Response measure | Cold-read p99 (PERF-4) grows by ≤ **20 %**; PERF-1 and PERF-2 still hold |

**Why these numbers:** links are never deleted (FR §5) and expired codes are never reused (FR-2), so the dataset only grows. A lookup by primary key should grow roughly with the log of the dataset size, so a 10× growth costing more than 20 % means the index or data layout isn't working. At book scale this scenario is what forces a decision on partitioning, even if the lab dataset fits on one node.

### SCAL-3 Write traffic grows

| Part | Value |
|---|---|
| Source | Clients, as usage grows |
| Stimulus | Sustained shorten load grows to **G × E-2** (G × peak write rate) |
| Artifact | Shorten service and code generation |
| Environment | Operator may add shorten instances before the run; storage not saturated |
| Response | Every instance generates codes independently; the system serves the new load and no two links ever get the same code (FR-2) |
| Response measure | PERF-2 holds at G × E-2. **Zero** duplicate codes across the run, checked by comparing every code returned against the store. The instance count *n* is recorded. Resources per 100 writes/s grow by ≤ **10 %** compared with the baseline |

**Why these numbers:** generating unique codes is the classic bottleneck of this problem. A single counter or a check-then-insert against storage is correct on one instance, but it either serializes writes (cost per write climbs as instances are added) or collides on two. Asking for both constant cost and zero collisions rules out the easy answers and makes code generation a real design decision with its own ADR.

### SCAL-4 Scale out when utilization crosses the threshold

| Part | Value |
|---|---|
| Source | Operator, watching the utilization panel (OBS-1) |
| Stimulus | Redirect instances stay above **70 %** of their CPU **or** memory limit for 2 minutes as load rises toward peak |
| Artifact | Redirect service and load balancing |
| Environment | Load rising from normal (E-3) to peak (E-4), all components healthy |
| Response | Utilization above the threshold is visible on the dashboard; the operator adds one instance. It starts receiving traffic once ready, without restarting or reconfiguring the other instances |
| Response measure | The dashboard shows the threshold crossing within **30 s** (one scrape interval plus rendering). **Zero** errors caused by the change. The new instance serves its share of traffic within **30 s** of starting, and the utilization that crossed the threshold falls back below 70 %. PERF-1 holds throughout |

**Why these numbers:** utilization is the signal that capacity is running out *before* users notice it in latency. 70 % leaves room for the 2× peak burst (PERF-3) while the new instance starts. Memory is included because the redirect service may hold per-request buffers, connection pools or a local cache that grow with load. If memory stays high after scaling out, it isn't load-driven (a leak, or a cache sized per instance), which is a finding in itself. Both are proxies, though: if the bottleneck is the cache or storage, adding redirect instances won't bring utilization or latency down, and this scenario will show that. Alerting and automatic scaling are out of scope for now (see §6); the scenario only requires that the signal is visible and that acting on it is safe.

## 6. Observability

In this lab, observability serves the experiments: every claim in §3–§5 has to be **measured** (monitorability) and the measurement has to be **repeatable** (testability). It also has to be possible to explain a failure from telemetry alone, without reading the code, as the root README requires.

**Out of scope: alerting.** No alert rules, notification routing or on-call. Thresholds (such as SCAL-4's 70 %) are shown on dashboards and acted on by a person watching them. Alerting can come back later through an ADR. Note that the root README lists alerts among what every case ships; this case is a deliberate exception.

Telemetry signals are metrics, structured logs and distributed traces, viewed in Grafana. Which tools collect them is an architecture decision.

### OBS-1 Every scenario is measurable from telemetry

| Part | Value |
|---|---|
| Source | Experimenter |
| Stimulus | Runs any scenario in §3–§5 |
| Artifact | Metrics pipeline and the case's Grafana dashboard |
| Environment | Any load level, including during injected faults |
| Response | The scenario's response measure can be read from a dashboard panel: latency percentiles, error rate and throughput per path and status code, cache hit ratio, and CPU and memory per container |
| Response measure | **100 %** of scenarios have a panel. Server-side p99 agrees with the load generator's p99 within **10 %** at normal load. Metrics resolution ≤ **15 s** |

**Why these numbers:** a scenario that can't be read off a dashboard can't be checked, which breaks *Every claim needs evidence*. Comparing server-side with client-side p99 catches time hidden in queues or the network, which the server alone can't see.

### OBS-2 Diagnose a fault from telemetry alone

| Part | Value |
|---|---|
| Source | Experimenter, not told which fault was injected |
| Stimulus | One of the AVAIL-1..4 faults is injected at random by a script |
| Artifact | Metrics, logs and traces of the whole system |
| Environment | Normal load |
| Response | Using only the dashboard, logs and traces (not the code, the deploy config or the fault script), the experimenter names the failing component and the symptom |
| Response measure | Correct component identified in **< 10 min**, for every AVAIL fault |

**Why these numbers:** this is the goal from the root README turned into a test. If a fault can't be located from telemetry, a signal is missing, and the fix is to add it, not to look at the code.

### OBS-3 Trace any slow or failed request

| Part | Value |
|---|---|
| Source | Experimenter |
| Stimulus | Picks a request that failed or was slower than p99 |
| Artifact | Tracing and logging |
| Environment | Any load level |
| Response | The request has a trace ID, returned in a response header and written in every log line for that request. The trace shows time spent in each hop (load balancer, service, cache, storage) |
| Response measure | **100 %** of failed and slower-than-p99 requests have a complete trace (every hop present). Finding a trace from a log line takes one lookup by trace ID |

**Why these numbers:** p99 problems live in individual requests, and averages hide them. Tracing every slow and failed request (rather than a random sample) is what lets PERF-4 tell a cache miss from a slow database. How to keep those traces without storing all of them is a sampling decision for the architecture.

### OBS-4 Experiments are reproducible

| Part | Value |
|---|---|
| Source | Experimenter |
| Stimulus | Reruns an experiment from its folder in `experiments/` |
| Artifact | Experiment scripts, dataset seeding, deployment |
| Environment | Same lab environment (§7) and resource limits |
| Response | One command deploys the system, seeds the dataset, runs the load and any fault script, and saves raw results plus a dashboard snapshot. Faults are injected by script, never by hand |
| Response measure | Across **3** reruns, p99 and capacity vary by ≤ **±10 %** |

**Why these numbers:** a result that can't be reproduced isn't evidence. If reruns vary by more than 10 %, the noise (warm-up, other containers, CPU throttling) is larger than many of the effects being measured, and that has to be fixed before any scenario can be judged.

### OBS-5 Instrumentation overhead is small

| Part | Value |
|---|---|
| Source | Experimenter |
| Stimulus | Runs the baseline capacity experiment with tracing and detailed metrics turned on, then off |
| Artifact | Instrumentation in the services |
| Environment | Same lab environment and dataset |
| Response | The system meets its scenarios with full instrumentation on |
| Response measure | Capacity with instrumentation on is ≥ **95 %** of capacity with it off; p99 at normal load grows by ≤ **5 %** |

**Why these numbers:** every other scenario is measured with instrumentation on, so its cost has to be known and small. Otherwise the lab ends up measuring the observer, not the system.

## 7. Back-of-envelope estimates

Book scale follows the assumptions in *System Design Interview*. Lab scale is what the experiments actually generate on a single machine (Docker Compose). It keeps the book's ratios (read : write, peak : average) so that the design pressures stay the same, just smaller.

**Lab-scale load figures are placeholders until the baseline experiment calibrates them.** That experiment ramps redirect load on the system under test until p99 or the error rate breaks. Peak (E-4) is then set a little below that point, and normal (E-3) at half of peak. Writes follow from the 10 : 1 ratio.

### Lab environment

Every response measure in this document only means something on this hardware and budget. Detailed per-container limits live in `deploy/`.

| Item | Value |
|---|---|
| Host | Ubuntu, Docker Compose (native cgroups, no VM). Cores / RAM: _TBD_ |
| OS + Docker headroom | _TBD_ (e.g. 1 core, ~2 GB, no limit) |
| Load generator | _TBD_ (e.g. 2 cores pinned with `cpuset`, 1 GB) |
| Observability stack | _TBD_ (e.g. 1 core, 2–3 GB) |
| System under test | _TBD_ (e.g. 4 cores, ~8 GB, capped as a group; split per component after the baseline) |

### Assumptions

| ID | Assumption | Book scale | Lab scale |
|---|---|---|---|
| A-1 | New URLs per day | 100 million | _TBD_ |
| A-2 | Read : write ratio | 10 : 1 | 10 : 1 |
| A-3 | Peak / average ratio | 2× (assumed; the book doesn't state one) | 2× |
| A-4 | Retention | 10 years | _TBD_ |
| A-5 | Average long URL size | 100 bytes | 100 bytes |

### Traffic

| ID | Estimate | Book scale | Lab scale |
|---|---|---|---|
| E-1 | Shorten, normal (writes/s) | 100M / 86,400 ≈ **1,160** | **100** (to calibrate) |
| E-2 | Shorten, peak (writes/s) | ≈ 2,300 | 200 |
| E-3 | Redirect, normal (reads/s) | ≈ **11,600** | **1,000** (to calibrate) |
| E-4 | Redirect, peak (reads/s) | ≈ 23,200 | 2,000 |

### Storage, cache and bandwidth

| ID | Estimate | Book scale | Lab scale |
|---|---|---|---|
| E-5 | Records after retention period | 100M × 365 × 10 = 365 billion | _TBD_ |
| E-6 | Dataset loaded for experiments | n/a | _TBD_ (e.g. 10M links) |
| E-7 | Storage | _TBD_ | _TBD_ |
| E-8 | Cache size (hot set) | _TBD_ | _TBD_ |
| E-9 | Bandwidth | _TBD_ | _TBD_ |
| E-10 | Short code length | _TBD_ (feeds FR-3) | _TBD_ |
