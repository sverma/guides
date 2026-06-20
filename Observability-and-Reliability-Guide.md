# Observability & Reliability for Cloud-Native Web Applications

### A practitioner's field guide for web applications running on Kubernetes in AWS / GCP, instrumented with Prometheus, Grafana, OpenTelemetry, and the modern LGTM stack

> **Scope of this document.** This guide is written around a concrete, common reference system: a customer-facing web application (HTTP/gRPC microservices + a single-page front end) packaged as containers, deployed to a managed Kubernetes service (Amazon EKS or Google GKE), fronted by a cloud load balancer, backed by managed databases, caches, and queues, and observed with **Prometheus + Grafana** for metrics, **OpenTelemetry** for instrumentation, **Tempo/Jaeger** for traces, and **Loki** for logs. Reliability is engineered with **SLOs, error budgets, and multi-AZ / multi-region** topologies. Everything here reflects the state of the practice as of 2026 (Prometheus 3.x, Grafana 12, OpenTelemetry GA signals, Loki 3.x).

---

## Table of contents

**Part I — Foundations**
1. Why observability and reliability are two halves of one discipline
2. The reference architecture we will observe
3. Monitoring vs. observability vs. reliability
4. The signals: metrics, logs, traces (and the emerging fourth pillar)
5. OpenTelemetry: the unifying standard

**Part II — Metrics with Prometheus & Grafana**
6. How Prometheus actually works
7. Metric types and native histograms
8. The three measurement methodologies: RED, USE, Four Golden Signals
9. Labels, dimensionality, and the cardinality problem
10. The Kubernetes metrics supply chain
11. Recording rules and query performance
12. Scaling Prometheus: Thanos, Mimir, VictoriaMetrics
13. Grafana: dashboards, drilldown, and alerting

**Part III — Distributed Tracing**
14. The mental model: traces, spans, and context propagation
15. Instrumentation strategy
16. Sampling: head, tail, and adaptive
17. Backends and the collector topology

**Part IV — Logging**
18. Structured logging as a contract
19. Loki's index-free architecture
20. Collection patterns on Kubernetes
21. Retention, cost, and tiering

**Part V — Correlation: making the signals one system**
22. Exemplars and trace-to-logs navigation
23. The OpenTelemetry Collector pipeline
24. The LGTM reference stack

**Part VI — Reliability Engineering**
25. What "reliability" actually means
26. SLI, SLO, SLA, and the error budget
27. Designing good SLIs for a web application
28. Burn-rate alerting (multi-window, multi-burn-rate)
29. The error budget policy

**Part VII — Reliability across AZs and Regions**
30. Failure domains: zones, regions, and what each protects against
31. The disaster-recovery strategy spectrum
32. Data replication and the consistency tax
33. Traffic management and global routing
34. Region-aware observability and SLOs

**Part VIII — Analyzing and improving reliability**
35. The arithmetic of availability
36. MTTD, MTTR, MTBF and incident analytics
37. Verifying reliability: chaos engineering and game days
38. Incident response and blameless postmortems

**Part IX — A reference implementation blueprint**
**Part X — Best-practice checklists and anti-patterns**
**Appendix — Glossary and further reading**

---

# Part I — Foundations

## 1. Why observability and reliability are two halves of one discipline

It is tempting to treat observability as "the dashboards and alerts" and reliability as "keeping the site up." In a modern cloud-native organization they are tightly coupled feedback loops:

- **Reliability is the goal** — the property that the service does what users need, when they need it, at acceptable speed.
- **Observability is the sense organ** — the property of a system that lets you understand its internal state from the outside, well enough to ask new questions about it *without shipping new code*.

You cannot manage what you cannot measure. Reliability targets (SLOs) are defined in terms of signals that observability produces (SLIs). When an SLO is at risk, observability data is what you use to diagnose and recover. After an incident, observability data feeds the postmortem, which feeds reliability improvements. The loop looks like this:

```
   define SLOs ─▶ instrument (OTel) ─▶ collect signals (Prom/Loki/Tempo)
        ▲                                          │
        │                                          ▼
   postmortems ◀── incident response ◀── alert on SLO burn / golden signals
```

The rest of this document walks the loop: first the signals and the tooling that collect them, then the reliability practices that consume them, then how all of it changes when you spread a system across availability zones and regions.

## 2. The reference architecture we will observe

Keep this picture in mind throughout; every recommendation maps onto it.

```
                         ┌─────────────────────────────────────────┐
        Users  ─────────▶│  Global edge: CDN + Anycast / DNS        │
                         │  (CloudFront / Cloud CDN, Route 53 /     │
                         │   Cloud DNS, Global Accelerator / GCLB)  │
                         └───────────────┬─────────────────────────┘
                                         │  latency / geo routing
                  ┌──────────────────────┴───────────────────────┐
                  ▼                                               ▼
        ┌───────────────────┐                          ┌───────────────────┐
        │  Region A (us-east)│                          │ Region B (eu-west)│
        │  ┌──────────────┐  │                          │  ┌──────────────┐ │
        │  │ Ingress / ALB│  │                          │  │ Ingress / ALB│ │
        │  └──────┬───────┘  │                          │  └──────┬───────┘ │
        │   ┌─────┴─────┐    │                          │   ┌─────┴─────┐   │
        │   │ K8s (EKS/ │    │   async replication      │   │ K8s (EKS/ │   │
        │   │   GKE)    │    │◀────────────────────────▶│   │   GKE)    │   │
        │   │ AZ1 AZ2 AZ3│   │                          │   │ AZ1 AZ2 AZ3│  │
        │   └─────┬─────┘    │                          │   └─────┬─────┘   │
        │   managed DB /     │                          │   managed DB /    │
        │   cache / queue    │                          │   cache / queue   │
        └───────────────────┘                          └───────────────────┘
```

Within each Kubernetes cluster:

- **Workloads** are stateless web/API pods (Deployments), plus stateful adjuncts run as managed cloud services where possible (RDS/Aurora, Cloud SQL/Spanner, ElastiCache/Memorystore, SQS/Pub-Sub).
- Pods spread across **3 availability zones** via topology-spread constraints and pod anti-affinity.
- A **service mesh or ingress controller** terminates TLS and provides L7 routing.
- An **observability data plane** (OpenTelemetry Collector / Grafana Alloy as a DaemonSet + Deployment, Prometheus or its agent, Loki, Tempo) runs alongside the workloads, ideally in a dedicated namespace and node pool.

This is the system whose health we will measure and whose reliability we will engineer.

## 3. Monitoring vs. observability vs. reliability

These words are used loosely; precise definitions prevent muddled design.

| Concept | Question it answers | Nature |
|---|---|---|
| **Monitoring** | "Is the thing I already know about broken?" | Watching *known* failure modes via predefined dashboards and threshold alerts. |
| **Observability** | "Why is the system behaving this way, including in ways I never predicted?" | The ability to explore *unknown-unknowns* by slicing high-cardinality telemetry after the fact. |
| **Reliability** | "Does the system meet users' expectations over time?" | An outcome, expressed as SLOs and measured continuously. |

Monitoring is a subset of observability. A dashboard of CPU usage is monitoring. The ability to ask "show me p99 latency for checkout requests, from EU users, on app version 4.12, that touched the new payments service" — without deploying new instrumentation — is observability. The defining property is **high-cardinality, high-dimensionality data that you can pivot on arbitrarily**.

Reliability sits above both: it is *why* you invest in observability at all.

## 4. The signals: metrics, logs, traces (and the emerging fourth pillar)

The classic "three pillars" are **metrics, logs, and traces**. The modern framing (championed by OpenTelemetry and the major vendors) is that pillars in *isolation* are a failure mode — value comes from **correlating** them. Each answers a different question:

- **Metrics** — numeric measurements aggregated over time (request rate, error rate, latency percentiles, CPU, queue depth). Cheap to store, ideal for dashboards, trend analysis, and alerting. They tell you **that** something is wrong and roughly where, but carry little context. Metrics are your first line of defense and the basis of SLOs.

- **Traces** — the end-to-end record of a single request as it crosses service boundaries, decomposed into **spans** (one span per operation: an HTTP handler, a DB query, a cache lookup). Traces tell you **where** time was spent and which dependency failed in a distributed call graph. Indispensable for microservices, serverless, and anything where one user action fans out to many backends.

- **Logs** — timestamped records of discrete events, ideally **structured** (JSON with consistent fields). Logs tell you **why** — the exact error message, the parameter values, the branch taken. They are the highest-detail, highest-volume, and (historically) most expensive signal.

The mnemonic: *metrics say something broke, traces say where it broke, logs say why it broke.*

**The fourth pillar — continuous profiling.** A growing consensus adds **profiling** (CPU/memory flame graphs sampled continuously in production, e.g. Grafana Pyroscope / Parca / eBPF-based profilers) as a fourth signal. Where a trace shows that a span took 800 ms, a profile shows *which functions inside that span burned the CPU*. Some practitioners also count **events** (discrete, high-cardinality structured records, the "wide events" school popularized by Honeycomb) as a distinct signal. The unifying acronym you will increasingly see is **MELT** — Metrics, Events, Logs, Traces.

## 5. OpenTelemetry: the unifying standard

If there is one decision that defines a modern observability stack, it is: **instrument with OpenTelemetry (OTel), and keep your storage backends swappable.**

OpenTelemetry is a CNCF project (the second most active after Kubernetes itself) that provides:

- **A single specification and data model** for all three signals, so a trace ID emitted by a Go service means the same thing to a Java service, the collector, and the backend.
- **Vendor-neutral SDKs** for every major language, with **auto-instrumentation** for common frameworks (HTTP servers, gRPC, database drivers, message clients) so you get traces and metrics with little or no code change.
- **The OTLP wire protocol** (OpenTelemetry Protocol) — the lingua franca that collectors and backends speak.
- **The OpenTelemetry Collector** — a standalone pipeline that receives, processes (batching, filtering, redaction, tail sampling, enrichment), and exports telemetry to one or more backends.
- **Semantic Conventions** — a standardized dictionary of attribute names (`http.request.method`, `service.name`, `k8s.pod.name`, `db.system`) so signals are *correlatable by construction*.

Why this matters strategically:

1. **No vendor lock-in.** You instrument once; you can route the same data to Prometheus, Grafana Cloud, Datadog, or change your mind next year, by editing collector config — not re-instrumenting code.
2. **Correlation for free.** Because the SDK propagates a shared context (via W3C Trace Context headers), a trace ID is automatically stamped onto logs and can be attached to metrics as *exemplars*. This is what lets you jump from a latency spike on a graph to the exact slow trace to the exact error log.
3. **It is converging with Prometheus.** As of **Prometheus 3.0**, Prometheus can act as a native **OTLP receiver** (ingesting OTLP metrics at `/api/v1/otlp/v1/metrics`), supports UTF-8 metric names, and ships Remote-Write 2.0 with native support for exemplars, metadata, and native histograms. The two ecosystems are deliberately merging.

**Practical guidance:** Run OTel SDKs (or zero-code agents) in your applications. Run the **OpenTelemetry Collector** (or **Grafana Alloy**, Grafana's OTel-native distribution) as the in-cluster data plane. Treat Prometheus, Loki, and Tempo as interchangeable storage that happens to be the current best-of-breed open-source choice.

---

# Part II — Metrics with Prometheus & Grafana

Prometheus has effectively *won* metrics for Kubernetes and cloud-native systems. Its exposition format is a de facto standard that virtually every tool speaks; its query language, PromQL, is powerful; and its ecosystem (exporters, the Operator, Alertmanager, Grafana) is enormous. Unless you have a specific requirement it cannot meet, Prometheus is the safe default, and Grafana is its near-inseparable visualization partner.

## 6. How Prometheus actually works

Prometheus is a **pull-based** time-series database and monitoring system. Understanding the data flow is essential to operating it well.

1. **Service discovery.** Prometheus continuously discovers what to monitor. On Kubernetes it queries the API server to find pods, services, and endpoints carrying the right annotations/labels.
2. **Scraping.** On a fixed interval (commonly 15–30 s), Prometheus issues an HTTP GET to each target's `/metrics` endpoint and reads a plain-text exposition of the current metric values. Crucially, **targets do not push**; Prometheus pulls. This makes targets simple (just expose an endpoint) and makes "is the target up?" a first-class signal (`up == 0`).
3. **Storage (TSDB).** Samples are appended to a local, on-disk time-series database optimized for append-heavy workloads, organized into time blocks.
4. **Querying (PromQL).** Dashboards (Grafana) and alerting rules evaluate PromQL expressions against the TSDB.
5. **Alerting.** Prometheus evaluates alerting rules and, when one fires, forwards it to **Alertmanager**, which handles grouping, deduplication, silencing, inhibition, and routing to PagerDuty/Slack/email.

A single time series is uniquely identified by its **metric name plus its set of key/value labels**, e.g.:

```
http_requests_total{method="POST", route="/checkout", status="500", pod="web-7c9", region="us-east-1"}
```

Each unique combination of labels is a distinct series. This is the source of both Prometheus's power (slice and dice by any dimension) and its biggest operational hazard (cardinality — see §9).

**Operational hardening notes:**
- Never expose the Prometheus port publicly; put it behind an authenticating reverse proxy.
- **Monitor your monitoring.** Prometheus exposes its own metrics; alert on scrape failures, config-reload failures (`prometheus_config_last_reload_successful == 0`), and TSDB errors. A blind monitoring system is worse than none.

## 7. Metric types and native histograms

Prometheus has four core metric types. Choosing the right one is the most common instrumentation mistake.

| Type | Use it for | Example |
|---|---|---|
| **Counter** | A value that only ever increases (reset to 0 on restart). Query with `rate()`. | `http_requests_total`, `errors_total` |
| **Gauge** | A value that can go up or down — a snapshot of "now." | `queue_depth`, `memory_bytes`, `active_connections` |
| **Histogram** | The distribution of observations (latency, payload size) into buckets, enabling percentile calculation. | `http_request_duration_seconds` |
| **Summary** | Client-side computed quantiles. **Avoid** unless you specifically need client-side quantiles — they cannot be aggregated across instances. | (rarely) |

Rule of thumb: **counters for things that only go up, gauges for current values, histograms for latencies and sizes, and avoid summaries.**

**Histograms and percentiles.** To compute "p99 latency," you instrument a histogram that counts how many requests fell under each latency bucket (≤100 ms, ≤500 ms, ≤1 s, …). Then `histogram_quantile(0.99, ...)` estimates the 99th percentile from the bucket counts. Percentiles, not averages, are what matter for user experience — an average hides the tail where real users suffer.

**Native histograms (Prometheus 3.x).** Classic histograms force you to *predefine* bucket boundaries; pick wrong and your percentiles are inaccurate, pick many and you pay in cardinality (one series per bucket). **Native histograms**, stabilizing in Prometheus 3.x (`--enable-feature=native-histograms`), use automatically-sized, exponentially-growing buckets with far higher resolution at a fraction of the storage and cardinality cost. They are the direction of travel for latency metrics; adopt them as your client libraries support them, especially for high-traffic SLI histograms.

## 8. The three measurement methodologies: RED, USE, Four Golden Signals

Don't instrument randomly. Use these battle-tested frameworks to decide *what* to measure for each component.

### RED — for request-driven services (your web/API tier)
For every service, expose:
- **Rate** — requests per second.
- **Errors** — failed requests per second (and as a ratio).
- **Duration** — the latency distribution (p50/p90/p99).

RED gives a consistent, comparable view of every microservice. It scales operationally: an on-call engineer can reason about a service they didn't write because every service has the same three signals. **Every service dashboard should prominently show RED.**

### USE — for resources (nodes, databases, queues, caches, disks)
For every resource, measure:
- **Utilization** — the fraction of time the resource was busy (e.g., CPU %).
- **Saturation** — the amount of queued/pending work the resource cannot service yet (e.g., run-queue length, queue depth).
- **Errors** — error events for that resource.

USE reveals capacity problems *before* they become outages. Saturation in particular is the leading indicator: a database at 70% utilization but with a growing query queue is about to fall over.

### The Four Golden Signals — from Google's SRE book (great for SLOs)
- **Latency** — how long requests take (and distinguish the latency of *successful* vs *failed* requests).
- **Traffic** — demand on the system (req/s, transactions/s).
- **Errors** — rate of failing requests.
- **Saturation** — how "full" the service is, relative to a limit (CPU vs quota, memory vs limit). On Kubernetes you can derive saturation by comparing a pod's CPU usage to its quota using kube-state-metrics.

The practical mapping most teams adopt: **RED for application microservices, USE for infrastructure and middleware, and the Four Golden Signals as the framework for defining SLOs and the on-call dashboard.** They overlap heavily — RED is essentially Golden Signals minus saturation.

## 9. Labels, dimensionality, and the cardinality problem

This is the single most important operational topic in Prometheus, and the cause of most self-inflicted outages of the monitoring system itself.

**Cardinality** is the number of unique time series, which equals the product of the number of unique values across all labels. Because every unique label-value combination creates a new series stored in memory and on disk, **unbounded labels cause "cardinality explosion."**

The cardinal sin: putting **unbounded or high-churn values in labels** —
- user IDs, session IDs, request IDs, trace IDs
- email addresses, full URLs with query strings
- `pod_name` used carelessly (pods churn constantly, each new name is a new series)
- timestamps or anything monotonically unique

A single such label can multiply your series count by millions, OOM-killing Prometheus.

**Best practices:**
- **Keep labels low-cardinality and bounded.** Good labels describe *categories*: `method`, `route` (the templated path `/users/:id`, **not** `/users/12345`), `status_code`, `service`, `region`, `environment`.
- **Templatize routes** before they become labels.
- **Relabel and drop** unwanted metrics/labels at scrape time (`metric_relabel_configs`) so junk never reaches storage.
- **Control cardinality from day one.** Retrofitting cardinality control means rewriting every service's instrumentation — vastly more expensive than getting it right initially.
- A useful capacity rule of thumb: **~100,000 active series per 4 vCPU** of a single Prometheus shard. Beyond that, shard or move to a scalable backend (§12).
- High-cardinality identifiers belong in **traces and logs (or exemplars)**, not metric labels. This is precisely the division of labor between the signals.

## 10. The Kubernetes metrics supply chain

On Kubernetes, metrics come from several complementary sources. You need all of them for a complete picture, and the **Prometheus Operator** (via the `kube-prometheus-stack` Helm chart) wires them together declaratively.

| Source | What it provides |
|---|---|
| **Application `/metrics`** | Your RED metrics and business metrics, via the OTel SDK or a Prometheus client library. |
| **node_exporter** | Per-node OS/hardware metrics: CPU, memory, disk, filesystem, network (USE for nodes). |
| **kube-state-metrics (KSM)** | The *desired vs actual* state of Kubernetes objects: deployment replicas, pod phase, restarts, resource requests/limits/quotas. Essential for saturation and for "is the cluster doing what I declared?" |
| **cAdvisor / kubelet** | Per-container resource usage (CPU throttling, memory working set) — built into the kubelet. |
| **Control-plane & cloud** | API server, etcd, scheduler metrics; managed-cloud control planes (EKS/GKE) expose a subset plus cloud-provider metrics (CloudWatch / Cloud Monitoring). |

**The Prometheus Operator** turns Prometheus configuration into Kubernetes Custom Resources:
- `ServiceMonitor` / `PodMonitor` — declaratively say "scrape pods/services matching this selector." Teams add monitoring by shipping a CR alongside their app, no central Prometheus config edits.
- `PrometheusRule` — recording and alerting rules as code.
- `Alertmanager` and `Prometheus` CRs — manage the components themselves.

This GitOps-friendly model is the standard way to run Prometheus on Kubernetes in 2026.

Categorize what you collect into three tiers, and build dashboards per tier:
1. **Cluster/infrastructure** — node health, pod counts, resource pressure, restarts.
2. **Application** — RED + business KPIs per service.
3. **Dependencies** — DB latency/connections, queue depth, cache hit rate.

## 11. Recording rules and query performance

A complex PromQL query that powers a dashboard viewed by 50 people, recomputed on every refresh, will make Grafana sluggish and Prometheus hot. **Recording rules** precompute expensive expressions on a schedule and store the result as a new series.

- Real-world impact: teams routinely cut dashboard load from ~15 s to ~2 s by converting heavy dashboard queries into recording rules.
- **Naming convention:** `level:metric:operations`, e.g. `service:http_requests:rate5m` or `sli:error_ratio:rate5m`. The colons encode the aggregation level, the underlying metric, and the operation applied.
- Recording rules are also the foundation of **SLO/burn-rate alerting** (§28): you precompute error ratios over multiple windows once, then alert and dashboard off those.

Pair this with Grafana dashboard design (§13): standardized RED dashboards per service, USE dashboards per infrastructure component, and template variables (environment, cluster, service) so one dashboard serves many contexts.

## 12. Scaling Prometheus: Thanos, Mimir, VictoriaMetrics

A single Prometheus is a single node with local storage and limited retention (typically 24–48 h hot on disk is the comfortable zone for large clusters). You hit limits in three dimensions: **retention** (you want months/years), **scale** (too many series for one node), and **global view** (many clusters/regions, one query surface). Don't reach for these prematurely — *use single-node Prometheus while you can* — but know the options:

- **Remote write.** Prometheus (or a lightweight agent / `vmagent` / Alloy) ships samples to a remote, horizontally-scalable backend. Configure disk-backed buffering carefully: if the remote backend blips and your buffer is in-memory only, Prometheus memory can balloon and OOM.
- **Thanos.** Adds a sidecar that uploads TSDB blocks to object storage (S3/GCS), a Querier that fans out across many Prometheis + the object store for a global view, plus compaction and downsampling for long-term retention. Keep only hours hot locally; everything else lives cheaply in object storage.
- **Grafana Mimir.** A horizontally-scalable, multi-tenant, object-storage-backed Prometheus-compatible TSDB (the "M" in LGTM). Designed for very high series counts and long retention with a single query endpoint.
- **VictoriaMetrics.** A high-performance, resource-efficient Prometheus-compatible TSDB and remote-write target, popular for its compression and lower memory footprint.

For multi-region (Part VII), this layer is what gives you a **single pane of glass** across regions while each region keeps a local Prometheus for fast, blast-radius-contained scraping.

## 13. Grafana: dashboards, drilldown, and alerting

Grafana turns Prometheus (and Loki/Tempo) data into dashboards, alerts, and exploratory tools. The 2026 generation (Grafana 12) adds capabilities that change how teams work:

- **Observability as Code / Git Sync.** Dashboards can be versioned in a Git repository and changed via pull request, so dashboards are reviewed and tracked like any other code. There is a Terraform provider and CLI for the same purpose. This kills the "someone edited the prod dashboard at 2 a.m. and nobody knows what changed" problem.
- **Dynamic dashboards.** Tabs, conditional rendering (show/hide panels based on variables), and new layouts let one dashboard adapt to context (per environment, per team).
- **Grafana Drilldown** (formerly Explore Metrics/Logs/Traces, now GA) — point-and-click exploration of metrics, logs, and traces without writing PromQL/LogQL/TraceQL, plus **Investigations** for correlating signals across the Drilldown apps.
- **Unified alerting.** Grafana-managed alert rules and recording rules, with the ability to import existing Prometheus/Loki/Mimir rules.

**Dashboard design principles that hold up in production:**
- One **standardized RED dashboard per service**, one **USE dashboard per critical infrastructure component**, and a **top-level SLO / Golden-Signals dashboard** for on-call.
- Drive panels from **recording rules**, not raw queries, for speed.
- Use **template variables** (environment, cluster, region, service) so dashboards are reusable and not copy-pasted per service.
- Put the **user-facing symptom** at the top (latency, error rate), causes below (saturation, dependencies). On-call reads top-down.

**Alerting philosophy (this is where most teams go wrong):**
- **Alert on symptoms, not causes.** Page on "users are seeing errors / high latency" (SLO burn), not on "CPU is 80%." High CPU that isn't hurting users is not an emergency.
- **Keep paging alerts few — roughly 5–10 total**, every one mapped to a user-facing symptom and tied to a runbook. Everything else goes to a ticket queue or a business-hours Slack channel.
- **Silence flapping alerts ruthlessly.** Alert fatigue is a reliability risk: ignored alerts mean the real one gets ignored too.
- Every paging alert annotation should carry a **`runbook_url`** and a **`dashboard_url`**.

---

# Part III — Distributed Tracing

Metrics tell you the checkout endpoint's p99 latency jumped to 3 seconds. They cannot tell you *which* of the eight downstream services in the checkout call graph is responsible. That is the job of distributed tracing, and it is non-negotiable for microservice and serverless architectures.

## 14. The mental model: traces, spans, and context propagation

- A **trace** represents one request's full journey through the system. It has a unique **trace ID**.
- A trace is a tree of **spans**. Each span is a single unit of work — an HTTP handler, a database query, a cache call, a queue publish — and records its **start time, duration, status, and attributes** (key/value metadata).
- Spans carry a **parent span ID**, so the collection of spans reconstructs the causal tree: which operation called which, and how long each took. The waterfall view in Jaeger/Tempo is this tree laid out on a timeline.

**Context propagation** is the magic that stitches spans across process boundaries. When service A calls service B, it injects the trace context into the request — using the **W3C Trace Context** standard headers (`traceparent`, `tracestate`). Service B extracts that context, so its spans attach to the same trace. OpenTelemetry SDKs do this automatically for instrumented clients/servers. The same active context is what stamps `trace_id`/`span_id` onto logs and metrics exemplars — this is the foundation of cross-signal correlation.

A trace answers questions metrics and logs cannot:
- Where exactly did this request spend its time?
- Which dependency in the chain failed or slowed down?
- What is the actual (not assumed) call graph and dependency map of my system?

## 15. Instrumentation strategy

Three layers, from least to most effort:

1. **Zero-code / auto-instrumentation.** OTel agents (e.g., the Java agent, Node/Python auto-instrumentation, or eBPF-based tools like **Grafana Beyla — now donated to OpenTelemetry as "OBI" / OpenTelemetry eBPF Instrumentation**) produce spans for HTTP, gRPC, and DB calls without touching application code. Best starting point; instant coverage.
2. **Library instrumentation.** Pull in OTel instrumentation libraries for your framework and database drivers — richer than eBPF, with semantic-convention attributes.
3. **Manual spans.** Add spans around business-critical operations the auto layer can't see (e.g., a complex pricing calculation), and enrich spans with domain attributes (`order.id`, `customer.tier`) — *as span attributes, not metric labels*.

Always follow **semantic conventions** for attribute names so traces correlate with metrics and logs and so backends can auto-build service maps.

## 16. Sampling: head, tail, and adaptive

Storing 100% of traces at scale is prohibitively expensive and mostly useless (99% of traces are boring successes). Sampling decides which traces to keep. This is one of the most important — and most misconfigured — parts of a tracing pipeline.

**Head sampling** — the keep/drop decision is made at the *start* of the trace (e.g., "keep 10% at random"). Pros: simple, cheap, low overhead, stateless. Con: it's blind to what happens later, so it **drops ~90% of your error and slow traces** — exactly the ones you want. Fine for low-volume or simple environments.

**Tail sampling** — the decision is made *after* the trace completes, by inspecting the whole trace. This lets you express intelligent policies:
- keep **100% of traces that contain an error**,
- keep **100% of slow traces** (latency above a threshold),
- keep a small **probabilistic sample (e.g., 1%) of normal traffic**,
- keep more of new or critical endpoints.

Tail sampling is the **recommended strategy for large, high-volume systems**, because it retains the interesting traces while discarding the noise. Costs: the sampler must be **stateful** — it buffers all spans of in-flight traces in memory until it decides — and it requires careful capacity planning.

**Adaptive sampling** — dynamically adjusts rates based on traffic (sample low-volume endpoints heavily, high-volume endpoints lightly); offered automatically by some vendors.

**Implementing tail sampling correctly (the gotchas):**
- It runs in the **OpenTelemetry Collector / Grafana Alloy**, *not* in the application and *not* in Tempo itself.
- **All spans of a given trace ID must reach the same collector instance** for the decision to be correct. The standard pattern is a **two-tier collector topology**: tier-1 collectors receive spans and use a **load-balancing exporter keyed on trace ID** to route all spans of a trace to the same tier-2 collector, which runs the `tail_sampling` processor.
- **Generate span metrics (RED-from-traces) *before* the sampler**, so your metrics are computed on 100% of traffic, not the sampled subset. Sampling should never distort your metrics.
- Set `decision_wait` ≥ the p99 of your trace duration, so you don't decide on incomplete traces.
- Always include a **catch-all probabilistic policy last**, to keep a representative baseline of normal traffic.
- Prefer **exact-match over regex** policies (regex is CPU-intensive); size `num_traces` for your in-flight volume; **always keep error traces** — the storage cost is trivial next to the debugging value.
- Test sampling policies in staging — a bad policy silently drops the traces you most need.

## 17. Backends and the collector topology

Two dominant open-source trace backends:

- **Grafana Tempo** (the "T" in LGTM) — stores traces in cheap **object storage (S3/GCS)**, indexed only by trace ID (with **TraceQL** for richer search). Extremely cost-effective at scale, tightly integrated with Grafana, and able to **generate metrics from spans** (service graphs, RED). The common modern choice.
- **Jaeger** (CNCF, originally Uber) — mature, full-featured trace UI and storage; can use OTLP ingestion and various storage backends. Still widely deployed.

A typical end-to-end trace pipeline for the reference architecture:

```
 App (OTel SDK / Beyla) ──OTLP──▶ Collector tier-1 (receive, batch,
                                   span-metrics → Prometheus,
                                   load-balance by trace ID)
                                          │
                                          ▼
                                  Collector tier-2 (tail_sampling)
                                          │ OTLP (sampled)
                                          ▼
                                   Tempo / Jaeger  ◀── Grafana (TraceQL,
                                                          trace↔logs↔metrics)
```

---

# Part IV — Logging

Logs are the highest-detail, highest-volume, and historically most expensive signal. The modern approach makes them cheap and queryable by (a) making them **structured** and (b) storing them in an **index-free, object-storage-backed** system.

## 18. Structured logging as a contract

Stop emitting free-text log lines. Emit **structured logs** — JSON (or logfmt) with consistent field names:

```json
{"ts":"2026-06-20T09:31:02.114Z","level":"error","service":"checkout",
 "trace_id":"4bf92f3577b34da6","span_id":"00f067aa0ba902b7",
 "http_route":"/checkout","http_status":500,"customer_tier":"gold",
 "error":"payment gateway timeout","duration_ms":3021}
```

Why this matters:
- **Queryable without full-text indexing** — you filter by field (`level="error" and http_status=500`) instead of grepping prose.
- **Correlation** — embedding the active **`trace_id` and `span_id`** in every log line is the single highest-leverage logging practice. It lets you pivot from a trace's slow span directly to the exact logs emitted during it, and vice versa. OTel-aware logging libraries do this automatically.
- **Consistency across services** — agreeing on a shared schema (level, service, trace_id, and a small set of common fields) cluster-wide is far better than letting every service invent its own format.

Discipline around **log levels** matters for both cost and signal: `ERROR`/`WARN` for actionable problems, `INFO` for significant business events, `DEBUG` off in production (toggle dynamically when investigating). Don't log everything "just in case" — it's expensive and buries the signal.

## 19. Loki's index-free architecture

**Grafana Loki** (the "L" in LGTM) is "Prometheus, but for logs." Its defining design choice: **it indexes only a small set of labels (metadata), not the log content.**

- Logs are grouped into **streams**, where a stream is the unique combination of its labels.
- Only the **labels** are indexed; the **log bodies are compressed and stored in cheap object storage** (S3/GCS/Azure Blob) in chunks.
- At query time, Loki uses labels to find the relevant streams, then does a fast grep-style scan within them (LogQL).

The cost consequences are dramatic. A representative published comparison: a 100-node cluster generating ~200 GB/day with 90-day retention costs on the order of **~$10,000/month on an EFK/Elasticsearch stack** (full-text indexing → 3–5× storage overhead on expensive SSD, plus heavy compute) versus **~$500/month on Loki** (label-only index, compressed chunks on object storage, small compute) — roughly a **90–95% cost reduction** for the bulk of use cases. Storage math: object storage runs ~$0.023/GB/month vs. ~$0.08–$0.125/GB/month for SSD-backed block storage, before replication multipliers.

**The cardinality rule applies here too — even more strictly.** Loki was built for **low-cardinality, long-lived labels** and explicitly *not* for high cardinality (default limit ~15 index labels). So:
- **Good labels** (low-cardinality, descriptive of origin): `cluster`, `namespace`, `app`, `container`, `environment`, `region`.
- **Never** use as labels: `trace_id`, `request_id`, `order_id`, `user_id`, timestamps, or anything unbounded — these create millions of tiny streams and a huge index, destroying performance.
- High-cardinality fields you still want to search go in **structured metadata** (a Loki 3.x feature) — stored and queryable *without* being indexed as labels. When you ingest via **OTLP**, Loki automatically maps low-cardinality OTel **resource attributes** to stream labels and routes high-cardinality **log attributes** (trace IDs, user IDs) to structured metadata — so you largely get the right behavior for free.

## 20. Collection patterns on Kubernetes

The 2026 standard is to collect with an **OpenTelemetry Collector / Grafana Alloy DaemonSet** (one agent per node) that tails container stdout/stderr, enriches with Kubernetes metadata (pod, namespace, labels), and ships via OTLP to Loki.

| Pattern | How it works | When |
|---|---|---|
| **DaemonSet (node agent)** | One collector per node tails all container logs. Zero app changes; covers everything. | The baseline default for all clusters. |
| **Direct push (SDK/OTLP)** | App pushes logs via OTel SDK with rich application context. | Layer on top for services needing richer context. |
| **Sidecar** | A logging container per pod. | Legacy/special cases; higher overhead. |

Older stacks used **Promtail** (Loki's purpose-built agent, now in maintenance in favor of Alloy) or **Fluent Bit / Fluentd** (the "F" in EFK). The migration trend is decisively toward **OTLP ingestion + Alloy/OTel Collector** so logs, metrics, and traces flow through one pipeline with shared enrichment and redaction.

## 21. Retention, cost, and tiering

- **Set retention deliberately.** In Loki, the **Compactor** enforces retention (`retention_enabled: true`, `retention_period: ...`); ensure `max_query_lookback ≤ retention_period`. Support **multiple retention policies per stream** — keep audit/compliance logs long, discard chatty debug logs quickly.
- **Don't store everything.** Storing every line from an entire cluster is costly and slows queries. Drop low-value logs at the collector before they hit storage.
- **Tier storage.** Hot (recent, fast object storage) → cold/archive (Glacier/Coldline) for compliance. Use lifecycle policies to age data out automatically.
- **Redact at the collector.** PII/secrets should be scrubbed in the collector pipeline *before* leaving the cluster — one central place to enforce data-governance rules across all signals.

---

# Part V — Correlation: making the signals one system

Collecting three signals in three silos is the failure mode of "three pillars" thinking. The value multiplies when you can move *seamlessly* between them. This is the defining characteristic of modern observability.

## 22. Exemplars and trace-to-logs navigation

The connective tissue is the **shared context** propagated by OpenTelemetry — specifically the `trace_id` and `span_id` — appearing across all three signals:

- **Metric → Trace, via exemplars.** An **exemplar** is a sample trace ID attached to a specific metric observation (e.g., the trace ID of one request that landed in the ">1s" latency bucket). In Grafana, you see a latency spike on a Prometheus graph, click the exemplar dot, and jump straight to a representative slow trace. Prometheus 3.x + Remote-Write 2.0 carry exemplars natively.
- **Trace → Logs.** Because every log line carries the `trace_id`, from a slow span you pivot to exactly the logs emitted during it.
- **Logs → Trace.** From an error log, jump to the full trace to see where in the request flow the error occurred.
- **Trace → Profile.** With continuous profiling (Pyroscope), jump from a CPU-heavy span to the flame graph of the functions that consumed the CPU.

The investigative flow becomes: **alert (metric/SLO) → dashboard (RED) → exemplar → trace (where) → logs (why) → profile (which code).** No manual stitching, no copy-pasting timestamps between tools.

## 23. The OpenTelemetry Collector pipeline

The Collector (or Grafana Alloy) is the central nervous system of the data plane. A pipeline is **receivers → processors → exporters**:

- **Receivers** — OTLP (from your SDKs), Prometheus scrape, host metrics, filelog (container logs), Kubernetes events.
- **Processors** — batching; `k8sattributes` enrichment (stamp every signal with pod/namespace/node); memory limiting; **redaction/PII scrubbing**; `tail_sampling`; span-to-metrics generation; filtering/dropping.
- **Exporters** — to Prometheus/Mimir (metrics), Loki (logs), Tempo (traces), and/or any vendor — *fan-out to several at once*.

Running this as the single ingress for all telemetry gives you one place to enforce sampling, redaction, attribute normalization, and routing — and the freedom to change backends without touching apps.

## 24. The LGTM reference stack

The de facto open-source observability stack, often called **LGTM** (or the **PLG** stack for the metrics-logs subset):

| Letter | Component | Signal | Storage |
|---|---|---|---|
| **L** | **Loki** | Logs | Object storage (index-free) |
| **G** | **Grafana** | Visualization, alerting, correlation | — |
| **T** | **Tempo** | Traces | Object storage |
| **M** | **Mimir** (or Prometheus/Thanos/VictoriaMetrics) | Metrics | Object storage |

…fed by **Grafana Alloy / OpenTelemetry Collector** and instrumented with **OpenTelemetry**. Every component is open source, object-storage-backed (cheap, durable, scalable), and natively correlated through Grafana. Managed equivalents (Grafana Cloud, AWS Managed Prometheus/Grafana, Google Cloud Managed Service for Prometheus) trade operational toil for cost and are increasingly the default for teams that don't want to run the data plane themselves.

---

# Part VI — Reliability Engineering

We now cross from "how to see the system" to "how to make it dependable." This is the domain of **Site Reliability Engineering (SRE)**, and its central insight is that reliability must be **defined, measured, and budgeted** — not left as a vague aspiration to "100% uptime."

## 25. What "reliability" actually means

**Reliability is the probability that a system performs its intended function, at acceptable quality, over a defined period, under expected conditions, as perceived by its users.**

Several words in that sentence are load-bearing:

- **"As perceived by users."** Reliability is measured at the user's experience, not at the component. A server can be "up" while users get errors (bad load balancer, broken dependency, expired cert). Conversely, a crashed pod that Kubernetes instantly replaced may cause *zero* user-visible impact. **Measure the symptom users feel, not the internal mechanism.**
- **"Acceptable quality."** Reliability is not just up/down. Slow is a form of broken. A 30-second page load is an outage from the user's perspective even though every HTTP request technically returns 200.
- **"100% is the wrong target."** Perfect reliability is impossible (the network, hardware, dependencies, and humans all fail) and economically irrational — each additional "nine" costs exponentially more. The right target is **"reliable enough that unreliability is not the reason users leave,"** which is almost always *below* 100%. The gap between 100% and your target is a resource to *spend*, which leads directly to error budgets.

Reliability is distinct from, but built upon, related properties: **availability** (is it reachable?), **durability** (is data safe?), **latency/performance** (is it fast enough?), **correctness** (are answers right?), and **resilience** (does it recover from failure gracefully?).

## 26. SLI, SLO, SLA, and the error budget

These four concepts are the operating system of reliability. Get them precise.

**SLI — Service Level Indicator.** A *quantitative measurement* of one aspect of service quality, almost always expressed as **a ratio of good events to total events**, from the user's perspective:

```
SLI = good events / valid events  × 100%
```

Examples:
- **Availability SLI** = (successful requests) / (valid requests). "Successful" = non-5xx (or business-defined success).
- **Latency SLI** = (requests served faster than 300 ms) / (valid requests).
- **Quality/correctness SLI** = (responses without data errors) / (total).

This "good/valid" form is powerful: it ranges cleanly from 0% (everything broken) to 100% (nothing broken), and it maps directly to an error budget. **Exclude health-check and synthetic traffic** from the denominator, and define SLIs from the *user's* viewpoint, not the server's.

**SLO — Service Level Objective.** The *target* for an SLI over a window. "**99.9%** of valid checkout requests succeed over a rolling **28 days**." The SLO is an internal goal you engineer toward. It is the single most important number in reliability work because it converts a fuzzy desire ("be reliable") into a measurable, decidable target.

**SLA — Service Level Agreement.** The *contractual* promise to customers, with financial/legal consequences (refunds, credits) if breached. SLAs are looser than SLOs by design: you set your internal SLO **stricter** than your external SLA, so you detect and fix problems before you breach the contract. (A free internal service may have SLOs but no SLA.)

**Error budget.** The brilliant inversion at the heart of SRE:

```
Error budget = 100% − SLO
```

If your SLO is 99.9%, then **0.1% of requests are *allowed* to fail** — that 0.1% is a *budget you may spend*. Over a 28-day window with 3,000,000 requests, a 99.9% SLO gives a budget of **3,000 failed requests**. Expressed as time, 99.9% availability permits ~**43 minutes** of downtime per 30 days.

The error budget reframes the eternal dev-vs-ops conflict:
- As long as you're **within budget**, the team is free to ship features fast, take risks, run experiments — unreliability has *room*.
- When the budget is **exhausted**, the policy kicks in (§29): typically a **feature freeze** until reliability is restored. Reliability work becomes self-prioritizing and *data-driven* rather than political.

This is the practical reason to compute SLOs continuously from your Prometheus metrics: the error budget is a live account balance that governs engineering decisions.

| Term | What it is | Audience | Example |
|---|---|---|---|
| SLI | A measurement | Engineers | 99.95% of requests succeeded |
| SLO | An internal target | Engineering org | 99.9% over 28 days |
| SLA | An external contract | Customers/legal | 99.5% or we credit you |
| Error budget | 100% − SLO; permissible failure | Both dev & ops | 0.1% ≈ 43 min/month |

## 27. Designing good SLIs for a web application

For the reference web app, a strong starter SLO set:

1. **Availability (request-based):**
   `sum(rate(http_requests_total{job="web", code!~"5.."}[28d])) / sum(rate(http_requests_total{job="web"}[28d]))` → target **99.9%**.
2. **Latency:** fraction of requests under a threshold derived from a latency histogram (e.g., 99% under 300 ms). Use **two thresholds** if useful (e.g., 95% < 200 ms *and* 99% < 1 s).
3. **Per-critical-journey SLOs.** Don't average the whole site into one number. The checkout/login/search paths each deserve their own SLO; an outage of checkout matters far more than the help page.

Guidelines:
- **User-journey, not endpoint.** Tie SLOs to what users are trying to *do*.
- **Symptom-based.** Built from the Golden Signals (latency, errors), measured as close to the user as possible (at the load balancer or front-end RUM, not deep in a backend).
- **Realistic.** Set the initial SLO near the *current* achieved level, then tighten deliberately. An aspirational 99.99% you can't meet just keeps you in permanent feature freeze.
- **Windowed.** Use a **rolling window** (e.g., 28 days) rather than calendar months — it avoids "budget resets on the 1st" gaming and reflects a continuous user experience.

## 28. Burn-rate alerting (multi-window, multi-burn-rate)

How do you *alert* on an SLO? Naïve threshold alerts fail in two opposite ways: a 5-minute spike pages you for nothing, while a quiet sustained 0.5% error rate silently drains your whole monthly budget without ever crossing a static threshold. The Google SRE answer is **multi-window, multi-burn-rate alerting**, now the industry standard.

**Burn rate** is *how fast you are consuming the error budget*, as a multiple of the sustainable rate:
- Burn rate **1** = you'll spend exactly 100% of the budget over the full SLO window (e.g., 30 days). Sustainable.
- Burn rate **14.4** = you'll exhaust the entire 30-day budget in ~2 days. Emergency.

For a 99.9% SLO (budget = 0.1%), a burn rate of 14.4 means the *current* error rate is 0.1% × 14.4 = **1.44%**.

**Why two windows?** Each alert combines a **long window** (confirms the problem is *sustained*, not a blip) **AND** a **short window** (confirms the problem is *still happening now*, not already recovered). Both conditions must hold, which filters transient spikes while still firing within minutes on real incidents.

A standard two-tier configuration:

| Tier | Long / short window | Burn rate | Meaning | Action |
|---|---|---|---|---|
| **Critical (page)** | 1h / 5m | **14.4×** | ~2% of monthly budget in 1 hour | Page on-call immediately |
| **Warning (ticket)** | 6h / 30m | **6×** | ~5% of budget in 6 hours | File a ticket / business hours |
| (optional slow) | 3d / 6h | **1×** | Steady slow drain | Ticket |

Implemented in Prometheus with **recording rules** for the error ratio over each window, then alerting rules that AND them together:

```yaml
groups:
  - name: slo_burn_rate
    rules:
      # Fast burn → page. Budget for 99.9% SLO ⇒ error-rate threshold 0.001
      - alert: CheckoutSLOBurnRateCritical
        expr: |
          (sli:checkout_error_ratio:rate1h > (14.4 * 0.001))
          and
          (sli:checkout_error_ratio:rate5m > (14.4 * 0.001))
        for: 2m
        labels: { severity: page, slo: checkout-availability }
        annotations:
          summary: "Checkout burning error budget 14.4x — ~2 days to exhaustion"
          runbook_url: "https://runbooks.example.com/checkout-slo"
          dashboard_url: "https://grafana.example.com/d/checkout-slo"
      # Slow burn → ticket
      - alert: CheckoutSLOBurnRateWarning
        expr: |
          (sli:checkout_error_ratio:rate6h > (6 * 0.001))
          and
          (sli:checkout_error_ratio:rate30m > (6 * 0.001))
        for: 5m
        labels: { severity: ticket, slo: checkout-availability }
```

Lessons baked into this design: a team that alerted only on a 5-minute window drowned in spike-noise and started ignoring pages; switching to multi-window and separating *page* from *ticket* fixed it. Always include `runbook_url` and `dashboard_url`, and keep a recording rule for **budget remaining** so dashboards show the live balance.

## 29. The error budget policy

An error budget is toothless without a **written, pre-agreed policy** specifying what happens when it's exhausted. Decide this *before* an incident, when everyone is calm. A typical policy:

- **Budget healthy (>25% remaining):** normal feature velocity; experiments and risky launches allowed.
- **Budget low (<25%):** heightened caution; require extra review for risky changes; start prioritizing reliability bugs.
- **Budget exhausted (≤0):** **feature freeze** — all non-essential deploys halt; the team redirects engineering effort to reliability (fixing the bugs that burned the budget, adding resilience) until the SLO recovers. Only reliability/security fixes ship.
- **Repeated exhaustion:** escalate to leadership; revisit whether the SLO is realistic or whether the system needs architectural investment.

The policy must be **agreed by both engineering and product/leadership** in advance, or it will be overridden the first time a launch is at stake — and then the budget means nothing.

---

# Part VII — Reliability across AZs and Regions

This is where reliability meets architecture. Spreading a system across failure domains is the primary technical lever for high availability — and it introduces the deepest trade-offs in the whole discipline.

## 30. Failure domains: zones, regions, and what each protects against

Cloud providers structure infrastructure into nested **failure domains**. Understanding exactly what each isolates is the foundation of every resilience decision.

- **Availability Zone (AZ)** — one or more discrete data centers within a region, with independent power, cooling, and networking, but connected to sibling AZs by **low-latency (sub-millisecond to ~2 ms), high-bandwidth** links. AZs are physically separated by a meaningful distance but close enough for **synchronous replication**.
- **Region** — a geographic area (e.g., `us-east-1`, `europe-west1`) containing multiple AZs. Cross-region links carry **50–150 ms** of latency depending on geography, which makes synchronous replication impractical and forces **eventual consistency** for cross-region data.

| You deploy across… | You survive… | You do **not** survive… |
|---|---|---|
| Single AZ | (nothing — single point of failure) | A rack/data-center failure |
| **Multi-AZ (one region)** | Loss of a data center / AZ: power, cooling, network, hardware | A whole-**region** outage (control-plane bug, regional network, natural disaster) |
| **Multi-region** | Loss of an entire region | Global dependencies (a buggy global config push, DNS provider outage) |

**The crucial, often-misunderstood point:** *Multi-AZ protects against data-center failures; multi-region protects against region failures.* These are different risks. A multi-AZ deployment with multiple AZs is enough to reach **>99.99%** availability for the large majority of workloads — and AWS itself states resilience requirements can *almost always* be met within a single region using multiple AZs. **Multi-region is expensive, complex, and should be reserved for workloads with extreme resilience needs** (financial infrastructure, health systems, critical platforms) or hard data-residency/latency requirements. Don't pay for it reflexively.

**Multi-AZ on Kubernetes, concretely:**
- Run worker nodes in **≥3 AZs** (3 is the sweet spot: tolerates one AZ loss while keeping quorum for stateful systems like etcd/databases).
- Use **`topologySpreadConstraints`** and **pod anti-affinity** to force replicas of a Deployment to spread across AZs, so one AZ loss never takes all replicas of a service.
- Managed control planes (EKS/GKE) already run the Kubernetes control plane multi-AZ for you.
- Use **regional** (not zonal) managed services: a Multi-AZ RDS/Aurora, a regional Cloud SQL, a regional load balancer.
- Beware **cross-AZ data-transfer charges** — chatty cross-AZ traffic costs money; balance resilience against egress cost (e.g., topology-aware routing to prefer same-AZ endpoints when safe).

## 31. The disaster-recovery strategy spectrum

For multi-region, the choice is governed by two numbers:

- **RTO (Recovery Time Objective)** — how long you can be down before recovery. *Time.*
- **RPO (Recovery Point Objective)** — how much data you can afford to lose, measured as a time window. *Data.*

AWS's four canonical DR strategies, from cheapest/slowest to most expensive/fastest:

| Strategy | How it works | RTO | RPO | Relative cost |
|---|---|---|---|---|
| **Backup & Restore** | Back up to the DR region; rebuild infra on disaster. | Hours | Hours | $ |
| **Pilot Light** | Core data replicated and minimal services kept running (DB replica + scaled-down core); scale up on failover. | 10s of min – hours | Minutes | $$ |
| **Warm Standby** | A scaled-down but fully functional copy runs continuously; scale up on failover. | Minutes | Seconds–minutes | $$$ |
| **Active-Active (Multi-site)** | Full production runs in **both** regions simultaneously; users routed to nearest healthy region; a region loss is absorbed with no promotion needed. | Near-zero | Near-zero | $$$$ |

Most organizations land on **pilot light or warm standby** as the right balance of cost vs. recovery. **Active-active** delivers the best user experience and near-zero RTO but is the most expensive and the hardest to build (it forces multi-region writes and conflict resolution). Cost-saving tactics: run smaller instances in the DR region and scale on failover; replicate *only critical* data; use intelligent storage tiering in DR.

A blunt cost reality: a fully redundant second region typically adds **~1.6×–2×** to infrastructure cost, plus **cross-region data-transfer egress** (e.g., ~$0.02/GB inter-region on AWS — negligible at 1 TB/month, ~$2,000/month at 100 TB/month). Budget for egress explicitly; it surprises people.

## 32. Data replication and the consistency tax

Stateless compute is easy to spread; **data is the hard part.** Physics (cross-region latency) forces a choice, formalized by the **CAP theorem**: during a network partition you must choose **consistency** or **availability**. Synchronous cross-region replication would add 50–150 ms to every write — usually unacceptable — so most multi-region designs choose **asynchronous replication and eventual consistency**, accepting some replication lag (and thus a non-zero RPO).

Managed building blocks:

- **AWS:** **Aurora Global Database** (cross-region async replication, typically **<1 s** lag, fast promotion of a secondary), **DynamoDB Global Tables** (multi-region, multi-active with last-writer-wins conflict resolution), S3 Cross-Region Replication.
- **GCP:** **Cloud Spanner** (globally-distributed, externally consistent *synchronous* replication via TrueTime — the rare system that gives strong consistency across regions, at a price), Cloud SQL cross-region replicas, multi-region Cloud Storage.
- **Cross-cloud / self-managed:** CockroachDB, YugabyteDB (distributed SQL with tunable consistency).

Architectural patterns to manage the consistency tax:
- **Active-passive (single-writer):** one region owns writes, others are read replicas; failover promotes a replica. Simple, no write conflicts, but failover has an RPO equal to replication lag.
- **Active-active (multi-writer):** both regions accept writes; requires a **conflict-resolution strategy** (last-writer-wins, CRDTs, or app-level merging). Powerful, complex.
- **Geo-sharding:** partition data by geography (EU users' data lives in the EU region). Minimizes cross-region replication and neatly satisfies data-residency rules (GDPR). Each shard is the authoritative home for its users.
- **Global data plane + regional caches:** centralized durable store with local read caches for latency.
- **Externalize session state** (to a replicated store like DynamoDB Global Tables or a global cache) so compute stays stateless and any region can serve any user.

## 33. Traffic management and global routing

To route users to the right (healthy, nearest) region you need a **global traffic manager** above the regional load balancers:

- **AWS:** **Route 53** (DNS-based: latency-based, geolocation, weighted, and *failover* routing with health checks) and **Global Accelerator** (Anycast IPs over the AWS backbone, faster failover than DNS since it isn't subject to DNS TTL caching). Within a region, **Application Load Balancers** spread across AZs and route only to healthy targets.
- **GCP:** **Global External HTTP(S) Load Balancer** (a single Anycast IP, automatic routing to the closest healthy backend), **Cloud DNS** with routing policies.
- **CDN/edge:** CloudFront / Cloud CDN absorb and cache traffic at the edge, shrinking origin load and improving latency.

Routing modes map to DR strategy: **failover routing** for active-passive; **latency-based or geolocation routing** for active-active. **Mind DNS TTL and client caching** — they directly inflate your real-world failover RTO, often far beyond the theoretical value; this is why Anycast-based options (Global Accelerator / GCLB) give faster, more predictable failover.

## 34. Region-aware observability and SLOs

Multi-region quietly breaks naïve observability. The fixes:

- **Per-region SLIs *and* a global SLI.** A single global availability number **masks regional failures** — if region A is 100% and region B is 95%, a traffic-weighted global SLO might still look "green" while half your EU users suffer. Compute **per-region SLOs** for fault detection and a **traffic-weighted global SLO** for the overall user experience. The weighting must match real traffic distribution.
- **Region as a first-class label.** Every metric, log, and trace must carry a `region` (and `az`) label/attribute, so you can pivot per failure domain. (Keep it low-cardinality — region/AZ are perfect labels.)
- **Federated, single-pane observability.** Each region runs its **own local Prometheus/collector** (fast scrapes, blast-radius containment, survives cross-region link loss). A global layer (**Thanos/Mimir Querier**, or a managed Grafana) aggregates across regions for one query surface. *Never* make region B's monitoring depend on region A being alive.
- **Reliability-specific SLIs for the multi-region machinery itself:**
  - **Replication lag** (max and p99) — your live RPO indicator; alert when it exceeds your RPO budget.
  - **Failover RTO** — measured time from failure detection to traffic on a healthy region (target often <5 min for HA); dominated by DNS TTL and cache delays.
  - **Cross-region error rate** — failures attributable to remote calls.
  - **Per-region cost** — egress and duplicate-infra spend; watch for anomalies.
- **Runbooks must include region failover and rollback playbooks**, and the on-call rotation must be trained to execute them under pressure. Automation is essential — manual multi-region failover is slow and error-prone exactly when stress is highest.

---

# Part VIII — Analyzing and improving reliability

Measuring reliability is one thing; *analyzing* it to know where to invest is another. This part covers the math and the practices.

## 35. The arithmetic of availability

Availability is usually quoted in **"nines."** Know what they cost in real downtime, because the jump between each tier is roughly an order of magnitude harder and more expensive:

| Availability | Downtime / year | Downtime / month | Downtime / day | Typical context |
|---|---|---|---|---|
| 99% ("two nines") | 3.65 days | 7.2 h | 14.4 min | Internal/dev tools |
| 99.9% ("three nines") | 8.77 h | 43.8 min | 1.44 min | Standard web apps |
| 99.95% | 4.38 h | 21.9 min | 43 s | Important services |
| 99.99% ("four nines") | 52.6 min | 4.38 min | 8.6 s | Business-critical; needs multi-AZ |
| 99.999% ("five nines") | 5.26 min | 26 s | 0.86 s | Telco/critical; needs multi-region + heavy automation |

**Composing availability — the dependency trap.** When a request depends on several components **in series** (each must work), availabilities **multiply**:

```
A_total = A1 × A2 × A3 × …
```

Five serial dependencies each at 99.9% yield only 0.999⁵ ≈ **99.5%** — worse than any single component. This is why deep dependency chains erode reliability and why you fight back with **redundancy in parallel** (which *adds* nines: two independent 99% replicas give 1 − 0.01² = 99.99% for that tier), graceful degradation, caching, and circuit breakers. Reliability is an architectural property, not just an operational one.

A subtlety: provider-published AZ/region SLAs are not independent in the way the math assumes (correlated failures, shared control planes, the global dependencies you added yourself). Treat composed numbers as upper bounds and verify empirically.

## 36. MTTD, MTTR, MTBF and incident analytics

Beyond a single availability percentage, these time-based metrics tell you *where* reliability is won or lost — and they're directly computable from your incident data and observability signals:

- **MTBF — Mean Time Between Failures.** How often incidents happen. Improved by *prevention*: better testing, canary releases, removing fragile dependencies, capacity headroom.
- **MTTD — Mean Time To Detect.** How long from failure to *knowing*. Improved by *observability*: good SLO/burn-rate alerts, golden-signal dashboards. High MTTD means you're learning about outages from Twitter — fix your alerting.
- **MTTR — Mean Time To Recover/Resolve.** How long from detection to restoration. Often the highest-leverage metric, because failures are inevitable but *recovery speed* is controllable. Improved by good runbooks, fast rollbacks, automated failover, feature flags, and practice.
- **MTTA — Mean Time To Acknowledge.** On-call responsiveness; surfaces paging/escalation problems.

A useful framing: **Availability ≈ MTBF / (MTBF + MTTR).** You raise availability either by failing less often (MTBF↑) or recovering faster (MTTR↓). For most internet-scale systems, **driving MTTR down is cheaper and more effective than chasing MTBF up** — you can't prevent every failure, but you can make recovery near-instant. This is the philosophical core of SRE: *design for fast recovery, not the fantasy of no failure.*

Track these per incident in a consistent system and review trends: is MTTD rising (alerting gaps)? Is MTTR concentrated in one service (missing runbook/automation)? Are the same root causes recurring (an MTBF/architecture problem)?

## 37. Verifying reliability: chaos engineering and game days

Reliability that has never been tested under failure is a hypothesis, not a fact. Two complementary practices validate it:

- **Chaos engineering.** Deliberately inject controlled failures in production (or production-like) systems to verify the system — *and your observability and alerting* — behave as designed. Pioneered by Netflix's Chaos Monkey; modern tooling includes **LitmusChaos**, **Chaos Mesh** (both CNCF, Kubernetes-native), **Gremlin**, and **AWS Fault Injection Service**. Typical experiments: kill random pods, drain an AZ, inject network latency/packet loss, throttle a dependency, fill a disk, blackhole a region. The scientific method applies: form a hypothesis ("if AZ-2 dies, traffic shifts with no SLO breach"), define a small blast radius, run it, observe, and either gain confidence or file a reliability bug. Crucially, chaos experiments also **validate that your alerts fire and your dashboards show the truth** — catching observability gaps before a real incident does.
- **Game days / failover drills.** Scheduled, often higher-level exercises where the team practices responding to a simulated major incident — including **executing the multi-region failover runbook end-to-end** and measuring the *actual* RTO/RPO against the targets. This is the only honest way to know your DR strategy works; a warm standby you've never failed over to is Schrödinger's DR. Regular drills also keep runbooks current and train on-call muscle memory.

Run these regularly, start small, always in a controlled manner with a rollback plan, and graduate from staging to production as confidence grows.

## 38. Incident response and blameless postmortems

When the SLO burn alert fires, a good process turns chaos into fast recovery and lasting learning:

1. **Detect & alert** — SLO burn-rate alert pages on-call with runbook + dashboard links.
2. **Triage & roles** — declare an incident, assign an **Incident Commander** (coordinates, not necessarily the one fixing), a comms lead, and ops responders. Clear roles prevent the "everyone debugging, no one communicating" failure.
3. **Mitigate first, diagnose second** — restore service before finding root cause. Roll back the recent deploy, fail over the region, scale up, shed load, flip the feature flag. Use observability (RED dashboard → exemplar → trace → logs) to localize fast.
4. **Communicate** — keep a timeline and status updates flowing to stakeholders.
5. **Blameless postmortem** — after resolution, write up *what happened, impact (in SLO/error-budget terms), timeline, root cause(s), and action items* — **without blaming individuals**. The premise: people act reasonably given the information and incentives they had; failures are *system* failures. Blame drives information underground and destroys the learning that prevents recurrence. Every postmortem yields tracked action items (with owners) that feed back into MTBF (prevent), MTTD (detect faster), or MTTR (recover faster) — closing the loop we opened in §1.

Track the postmortem action items to completion. An incident you don't learn from, you'll have again.

---

# Part IX — A reference implementation blueprint

Tying it together for the reference web app on EKS/GKE across 3 AZs (with an optional second region):

**Instrumentation layer**
- OpenTelemetry SDKs / zero-code agents (or eBPF via OBI/Beyla) in every service; W3C Trace Context propagation; semantic conventions enforced.
- Structured JSON logs with `trace_id`/`span_id`; RED metrics + native-histogram latency; business KPIs as metrics; high-cardinality IDs go to traces/logs/structured-metadata, never metric or Loki labels.

**Data plane (per cluster)**
- **Grafana Alloy / OTel Collector** as DaemonSet (logs, host metrics) + Deployment (OTLP gateway, span-metrics, tail sampling, k8s enrichment, PII redaction).
- **kube-prometheus-stack** (Prometheus Operator) with `ServiceMonitor`/`PodMonitor`/`PrometheusRule` CRs in Git; node_exporter, kube-state-metrics, cAdvisor.
- Tail sampling in a two-tier collector (load-balance by trace ID → sampler), keeping 100% errors + slow, ~1% baseline.

**Storage & query**
- Metrics: local Prometheus (short retention) → remote-write to **Mimir/Thanos/VictoriaMetrics** for long retention + global view.
- Logs: **Loki** on object storage, label-only index + structured metadata, compactor-enforced retention.
- Traces: **Tempo** on object storage, TraceQL.
- Single **Grafana** (Git-synced dashboards): per-service RED, per-component USE, top-level SLO/Golden-Signals + multi-region view.

**Reliability layer**
- Per-journey SLOs (availability + latency), per-region and global, on rolling 28-day windows; recording rules for error ratios and **budget remaining**.
- Multi-window multi-burn-rate alerts (14.4× page / 6× ticket) with runbook + dashboard links; ≤10 paging alerts total, all symptom-based; routed via Alertmanager.
- Written, leadership-agreed **error-budget policy** (freeze on exhaustion).

**Topology & DR**
- 3-AZ spread via topology-spread/anti-affinity; regional managed DB (Multi-AZ Aurora / regional Cloud SQL or Spanner), externalized session state.
- DR tier chosen by RTO/RPO (default: warm standby or pilot light; active-active only if justified); global routing via Route 53/Global Accelerator or GCLB; cross-region async replication with monitored lag.
- Quarterly **game days** + continuous **chaos experiments** (Chaos Mesh/Litmus) validating failover, alerting, and dashboards.

---

# Part X — Best-practice checklists and anti-patterns

**Observability do's**
- Instrument once with OpenTelemetry; keep backends swappable.
- Correlate signals via shared `trace_id` (exemplars, structured logs).
- RED for services, USE for resources, Golden Signals for SLOs.
- Control cardinality from day one; templatize routes; drop junk at the collector.
- Recording rules for heavy/SLO queries; Git-versioned dashboards.
- Alert on user-facing symptoms; ≤10 pages; runbook + dashboard on every alert.
- Monitor your monitoring (scrape health, config reloads, TSDB).
- Redact PII centrally in the collector.

**Observability anti-patterns (avoid)**
- High-cardinality labels (user/request/trace IDs) in metrics or Loki streams.
- Averages instead of percentiles for latency.
- 100% trace storage, or head-sampling that discards your error traces.
- Alerting on causes (CPU%) instead of symptoms; alert sprawl → fatigue.
- Storing every log line forever; unstructured free-text logs.
- One global SLO that masks regional/per-journey failures.
- A monitoring stack that dies with the region it's monitoring.

**Reliability do's**
- Define SLIs from the user's perspective (good/valid events); set SLOs below 100%.
- Run an error budget with a pre-agreed, leadership-backed policy.
- Multi-AZ (≥3) by default; multi-region only when justified by RTO/RPO/residency.
- Choose DR tier by explicit RTO/RPO; monitor replication lag and failover RTO.
- Optimize MTTR (fast rollback, automated failover, feature flags) over chasing perfect MTBF.
- Test reliability: chaos experiments + regular game days that actually fail over.
- Blameless postmortems with tracked action items.

**Reliability anti-patterns (avoid)**
- Targeting "100% uptime"; treating any failure as unacceptable.
- Reflexive multi-region spend without an RTO/RPO justification.
- Assuming serial dependencies are reliable (availabilities multiply *down*).
- Synchronous cross-region writes on the hot path.
- A DR plan never exercised; runbooks that don't match reality.
- DNS-TTL-bound failover with no Anycast option, then surprise at real RTO.
- Blameful incident reviews that suppress learning.

---

# Appendix — Glossary and further reading

**Glossary**
- **AZ / Region** — nested cloud failure domains (data-center-level / geographic).
- **Cardinality** — number of unique time series; product of label-value combinations.
- **Error budget** — 100% − SLO; the permissible amount of failure.
- **Exemplar** — a trace ID attached to a metric sample, linking metrics→traces.
- **LGTM** — Loki, Grafana, Tempo, Mimir (the open-source stack).
- **MELT** — Metrics, Events, Logs, Traces.
- **MTBF / MTTD / MTTR / MTTA** — mean time between failures / to detect / to recover / to acknowledge.
- **OTLP** — OpenTelemetry Protocol; the wire format for telemetry.
- **PromQL / LogQL / TraceQL** — query languages for Prometheus / Loki / Tempo.
- **RED / USE** — Rate-Errors-Duration (services) / Utilization-Saturation-Errors (resources).
- **RPO / RTO** — Recovery Point Objective (data loss tolerance) / Recovery Time Objective (downtime tolerance).
- **SLI / SLO / SLA** — indicator (measurement) / objective (internal target) / agreement (external contract).
- **Span / Trace** — a single operation / the tree of spans for one request.
- **W3C Trace Context** — standard headers (`traceparent`) for propagating trace context.

**Authoritative sources & further reading**
- *Site Reliability Engineering* and *The SRE Workbook* — Google ([sre.google/books](https://sre.google/books/), [Implementing SLOs](https://sre.google/workbook/implementing-slos/)).
- OpenTelemetry docs — [opentelemetry.io](https://opentelemetry.io/docs/) (concepts, sampling, semantic conventions).
- Prometheus docs & the 3.0 release notes — [prometheus.io](https://prometheus.io/docs/), [Prometheus 3.0 announcement](https://prometheus.io/blog/2024/11/14/prometheus-3-0/).
- Grafana docs — Loki, Tempo, Mimir, Alloy, and [What's new in Grafana 12](https://grafana.com/docs/grafana/latest/whatsnew/whats-new-in-v12-0/).
- *The RED Method* — [Grafana Labs](https://grafana.com/blog/the-red-method-how-to-instrument-your-services/); *The USE Method* — Brendan Gregg.
- **AWS Well-Architected Framework — Reliability Pillar** ([REL10: fault isolation / multi-location](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/)) and AWS DR strategies whitepaper.
- **Google Cloud Architecture Framework — Reliability** and DR planning guides.
- CNCF chaos tooling — [LitmusChaos](https://litmuschaos.io/), [Chaos Mesh](https://chaos-mesh.org/).

*This document reflects the state of the practice as of mid-2026. The tooling moves fast — the principles (correlate signals, measure from the user's perspective, budget for failure, test your recovery) move slowly. Anchor on the principles; let the tools be swappable.*






