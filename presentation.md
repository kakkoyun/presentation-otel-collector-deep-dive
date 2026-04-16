---
marp: true
theme: otel
paginate: true
size: 16:9
html: true
header: 'OpenTelemetry Collector Deep Dive'
style: |
    .columns {
        display: grid;
        grid-template-columns: repeat(2, minmax(0, 1fr));
        gap: 1rem;
    }
    .small {
        font-size: 0.75em;
    }
    .tiny {
        font-size: 0.6em;
    }
---

<!-- _class: lead -->
<!-- _paginate: skip -->
<!-- _header: '' -->

# The OpenTelemetry Collector

## Architecture, Code & Contribution

Kemal Akkoyun · OpenTelemetry Maintainer · Datadog

---

# What Is the Collector?

**The Collector** decouples instrumentation from destination.

```
  ┌─────────────┐
  │ App + SDK   │──┐
  └─────────────┘  │    ┌──────────────┐
                   ├──▶│  Collector   │
  ┌─────────────┐  │    └──┬──┬──┬────┘
  │ App + SDK   │──┘       │  │  │
  └─────────────┘          ▼  ▼  ▼
                   ┌───┐┌───┐┌────────┐
                   │ A ││ B ││ Vendor │
                   └───┘└───┘└────────┘
```

One binary: **receives**, **processes**, **exports** telemetry data

---

# Four Signals

The Collector handles all OpenTelemetry signal types:

- **Traces** — distributed request flows
- **Metrics** — measurements and aggregations
- **Logs** — structured and unstructured log records
- **Profiles** — continuous profiling data (in development)

<span class="small">

[opentelemetry.io/docs/collector](https://opentelemetry.io/docs/collector/)

</span>

---

# Deployment: Agent

Runs **locally** — as a sidecar or DaemonSet

```
┌────────────────────────┐
│  Node / Pod            │
│  ┌─────┐  ┌─────────┐ │
│  │ App │─▶│  Agent   │─┼──▶ Backend
│  └─────┘  └─────────┘ │
│  ┌─────┐       ▲      │
│  │ App │───────┘      │
│  └─────┘              │
└────────────────────────┘
```

Low latency, local enrichment, buffer & retry

---

# Deployment: Gateway

Runs **centrally** — load-balanced service

```
  Agent 1 ──┐
             ├──▶ ┌─────────┐ ──▶ Backend A
  Agent 2 ──┤    │ Gateway │
             ├──▶ └─────────┘ ──▶ Backend B
  Agent 3 ──┘
```

Cross-cutting processing: tail sampling, routing, aggregation

Can combine both in a **tiered architecture** (agent → gateway → backends)

<span class="small">

[opentelemetry.io/docs/collector/deployment](https://opentelemetry.io/docs/collector/deployment/)

</span>

---

# The Ecosystem

| Component | Count | Examples |
|-----------|------:|---------|
| **Receivers** | 109 | OTLP, Prometheus, Kafka, k8s |
| **Processors** | 31 | batch, filter, transform |
| **Exporters** | 43 | Datadog, Prometheus, Elasticsearch |
| **Connectors** | 14 | spanmetrics, routing, count |
| **Extensions** | 27 | health check, pprof, OAuth |

---

# Three Repositories

- [`opentelemetry-collector`](https://github.com/open-telemetry/opentelemetry-collector) — core framework & interfaces
- [`opentelemetry-collector-contrib`](https://github.com/open-telemetry/opentelemetry-collector-contrib) — 224+ community components
- [`opentelemetry-collector-releases`](https://github.com/open-telemetry/opentelemetry-collector-releases) — 5 official distributions

---

<!-- _class: lead -->

# Five Component Types

---

# The Building Blocks

| | Receiver | Processor | Exporter |
|---|----------|-----------|----------|
| **Role** | Ingests data | Transforms data | Sends data |
| **Data flow** | Entry point | Middle | Exit point |
| **Examples** | OTLP, Prometheus | batch, filter | OTLP, Datadog |

---

# Connectors & Extensions

| | Connector | Extension |
|---|-----------|-----------|
| **Role** | Bridges pipelines | Auxiliary services |
| **Data flow** | Exit → Entry | Out-of-band |
| **Examples** | traces→metrics | health, auth, zpages |

All five types implement the same interface: **`Start()`** + **`Shutdown()`**

<span class="small">

[opentelemetry.io/docs/collector/components](https://opentelemetry.io/docs/collector/components/)

</span>

---

# Pipeline Config: Components

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: localhost:4317

processors:
  batch:
    send_batch_size: 1024
  memory_limiter:
    limit_mib: 512

exporters:
  debug:
    verbosity: detailed
  otlp:
    endpoint: backend:4317
```

---

# Pipeline Config: Wiring

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp, debug]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp]
```

`service.pipelines` is where the **wiring** happens — signal-typed, declarative

<span class="small">

[opentelemetry.io/docs/collector/configuration](https://opentelemetry.io/docs/collector/configuration/)

</span>

---

# Inside a Pipeline: The DAG

```
  Receiver 1 ─┐                                       ┌── Exporter 1
  Receiver 2 ─┼──▶ Processor 1 ──▶ Processor N ──▶ fan-out ──┼── Exporter 2
  Receiver N ─┘                                       └── Exporter N
```

Internally built as a **Directed Acyclic Graph** using [`gonum`](https://pkg.go.dev/gonum.org/v1/gonum/graph)

- **Start:** reverse topological — exporters first
- **Shutdown:** forward topological — receivers first

<span class="small">

[`service/internal/graph/graph.go`](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/service/internal/graph/graph.go#L75-L94)

</span>

---

# Connectors: Bridging Pipelines

```
  ┌─ Traces Pipeline ──────────────────────┐    ┌─ Metrics Pipeline ──────────────────┐
  │ otlp receiver → batch → spanmetrics ───┼───▶│ spanmetrics → filter → prometheus  │
  └────────────────────────────────────────┘    └─────────────────────────────────────┘
```

- Acts as **Exporter** from one pipeline and **Receiver** into another
- Can cross signals: Traces → Metrics, Logs → Metrics

<span class="small">

[`connector/connector.go`](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/connector/connector.go)

</span>

---

# Connectors in Config

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [spanmetrics, otlp]    # connector as exporter
    metrics:
      receivers: [spanmetrics]           # same connector as receiver
      exporters: [prometheus]
```

Same `spanmetrics` in **exporters** of traces AND **receivers** of metrics.

<span class="small">

[Connector docs](https://opentelemetry.io/docs/collector/components/connector/)

</span>

---

<!-- _class: lead -->

# Code Deep Dive

---

# The Component Interface

```go
// component/component.go

type Component interface {
    Start(ctx context.Context, host Host) error
    Shutdown(ctx context.Context) error
}
```

4-phase lifecycle: **Creation** → **Start** → **Running** → **Shutdown**

<span class="small">

[`component/component.go#L25-L62`](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/component/component.go#L25-L62)

</span>

---

# The Factory Interface

```go
type Factory interface {
    Type() Type                    // e.g. "otlp", "batch", "debug"
    CreateDefaultConfig() Config   // returns zero-value configuration
}
```

Each component type extends this with signal-specific `Create*` methods.

---

# Receiver Factory

```go
// receiver/receiver.go

type Factory interface {
    component.Factory

    CreateTraces(ctx context.Context, set Settings,
        cfg component.Config,
        next consumer.Traces,         // ← push model
    ) (Traces, error)

    CreateMetrics(ctx context.Context, set Settings,
        cfg component.Config,
        next consumer.Metrics,
    ) (Metrics, error)

    CreateLogs(ctx context.Context, set Settings,
        cfg component.Config,
        next consumer.Logs,
    ) (Logs, error)
}
```

<span class="small">

[`receiver/receiver.go#L60-L91`](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/receiver/receiver.go#L60-L91)

</span>

---

# Processor Factory

```go
// processor/processor.go — has next consumer

CreateTraces(ctx context.Context, set Settings,
    cfg component.Config,
    next consumer.Traces,         // ← forwards data
) (Traces, error)
```

Processor **has** `next` — it receives data AND forwards it.

<span class="small">

[`processor/processor.go#L51-L82`](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/processor/processor.go#L51-L82)

</span>

---

# Exporter Factory

```go
// exporter/exporter.go — NO next consumer

CreateTraces(ctx context.Context, set Settings,
    cfg component.Config,
                                  // ← no next parameter
) (Traces, error)
```

Exporter has **no** `next` — it IS the terminal consumer.

<span class="small">

[`exporter/exporter.go#L51-L79`](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/exporter/exporter.go#L51-L79)

</span>

---

# The Consumer Chain

- **Receiver:** gets `next` consumer to push **INTO**
- **Processor:** gets `next` consumer to forward **TO**
- **Exporter:** no `next` — it **IS** the endpoint
- **Connector:** gets `next` on receiver side, sits at exporter position

> **The consumer chain IS the pipeline wiring.**

---

# Graph.Build

```go
// service/internal/graph/graph.go

func Build(ctx context.Context, set Settings) (*Graph, error) {
    pipelines := &Graph{
        componentGraph: simple.NewDirectedGraph(),
        pipelines:      make(map[pipeline.ID]*pipelineNodes),
    }
    pipelines.createNodes(set)          // 1. Parse config → graph nodes
    pipelines.createEdges()             // 2. Wire edges from pipeline defs
    pipelines.buildComponents(ctx, set) // 3. Topo sort → instantiate
    return pipelines, nil
}
```

<span class="small">

[`graph.go#L75-L94`](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/service/internal/graph/graph.go#L75-L94)

</span>

---

# Topological Sort

```go
nodes, _ := topo.Sort(g.componentGraph)
for _, node := range slices.Backward(nodes) {
    // exporters built first → consumers exist before producers
}
```

**Start:** reverse topo order (exporters → processors → receivers)
**Shutdown:** forward topo order (receivers → processors → exporters)

---

# Service.Start

```go
// service/service.go

func (srv *Service) Start(ctx context.Context) error {
    srv.host.ServiceExtensions.Start(ctx, srv.host)
    srv.host.ServiceExtensions.NotifyConfig(ctx, srv.collectorConf)
    srv.host.Pipelines.StartAll(ctx, srv.host)
    srv.host.ServiceExtensions.NotifyPipelineReady()

    srv.telemetrySettings.Logger.Info(
        "Everything is ready. Begin running and processing data.")
    return nil
}
```

Shutdown is the **exact reverse**.

<span class="small">

[`service/service.go#L240-L266`](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/service/service.go#L240-L266)

</span>

---

# Generated: main.go

```go
// Code generated by "go.opentelemetry.io/collector/cmd/builder". DO NOT EDIT.

func main() {
    info := component.BuildInfo{
        Command:     "otelcorecol",
        Description: "Local OpenTelemetry Collector binary, testing only.",
        Version:     "0.150.0-dev",
    }
    set := otelcol.CollectorSettings{
        BuildInfo: info,
        Factories: components,
    }
    if err := run(set); err != nil {
        os.Exit(1)
    }
}
```

<span class="small">

[`cmd/otelcorecol/main.go`](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/cmd/otelcorecol/main.go)

</span>

---

# Generated: components.go

```go
func components() (otelcol.Factories, error) {
    factories.Receivers, _ = otelcol.MakeFactoryMap[receiver.Factory](
        nopreceiver.NewFactory(),
        otlpreceiver.NewFactory(),
    )
    factories.Exporters, _ = otelcol.MakeFactoryMap[exporter.Factory](
        debugexporter.NewFactory(),
        otlpexporter.NewFactory(),
        otlphttpexporter.NewFactory(),
    )
    // ... processors, connectors, extensions
}
```

> Your custom collector is just a **`manifest.yaml`** away.

<span class="small">

[`cmd/otelcorecol/components.go#L41-L110`](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/cmd/otelcorecol/components.go#L41-L110)

</span>

---

<!-- _class: lead -->

# How It's Built

---

# The Builder (OCB)

```
  manifest.yaml ──▶ OCB ──▶ go build ──▶ Binary
                     │
                     ├── main.go        (generated)
                     ├── components.go  (generated)
                     └── go.mod         (generated)
```

**Install:** `go install go.opentelemetry.io/collector/cmd/builder@latest`
**Usage:** `builder --config=manifest.yaml`

<span class="small">

[`cmd/builder/`](https://github.com/open-telemetry/opentelemetry-collector/tree/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/cmd/builder) · [OCB docs](https://opentelemetry.io/docs/collector/extend/ocb/)

</span>

---

# Core Manifest

```yaml
dist:
  name: otelcorecol
  version: 0.150.0-dev

receivers:
  - gomod: .../receiver/otlpreceiver v0.150.0
exporters:
  - gomod: .../exporter/debugexporter v0.150.0
  - gomod: .../exporter/otlpexporter v0.150.0
processors:
  - gomod: .../processor/batchprocessor v0.150.0
connectors:
  - gomod: .../connector/forwardconnector v0.150.0
```

<span class="small">

[`cmd/otelcorecol/builder-config.yaml`](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/cmd/otelcorecol/builder-config.yaml)

</span>

---

# Contrib Manifest

```yaml
dist:
  name: otelcol-contrib
  version: 0.150.1

receivers:
  - gomod: .../receiver/otlpreceiver v0.150.0
  - gomod: .../receiver/kafkareceiver v0.150.0
  - gomod: .../receiver/prometheusreceiver v0.150.0
  # ... 107+ more
exporters:
  - gomod: .../exporter/datadogexporter v0.150.0
  # ... 47+ more
```

<span class="small">

[`otelcol-contrib/manifest.yaml`](https://github.com/open-telemetry/opentelemetry-collector-releases/blob/dffb4896203af6708e8e436a4d39bb2dd438aeb4/distributions/otelcol-contrib/manifest.yaml)

</span>

---

# Five Official Distributions

| Distribution | Purpose |
|---|---|
| **otelcol** | Core components only (frozen) |
| **otelcol-contrib** | All 224+ community components |
| **otelcol-k8s** | Kubernetes-optimized subset |
| **otelcol-otlp** | OTLP-only, ultra-minimal |
| **otelcol-ebpf-profiler** | eBPF profiling (Linux) |

<span class="small">

[Distribution criteria](https://github.com/open-telemetry/opentelemetry-collector-releases/blob/dffb4896203af6708e8e436a4d39bb2dd438aeb4/distributions/README.md)

</span>

---

# Repository Map

```
opentelemetry-collector (core)        ← Stable v1.56.0 / Beta v0.150.0
├── component/          # interfaces
├── receiver/ processor/ exporter/ connector/
├── service/internal/graph/   # DAG construction
├── pdata/              # protocol data model
├── confmap/            # config system
├── cmd/builder/        # OCB tool
└── versions.yaml

opentelemetry-collector-contrib       ← 224+ components
├── receiver/ (109) · processor/ (31) · exporter/ (43)
└── connector/ (14) · extension/ (27)

opentelemetry-collector-releases      ← 5 distributions
└── distributions/ → manifest.yaml → OCB → goreleaser
```

<span class="small">

[`versions.yaml`](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/versions.yaml)

</span>

---

<!-- _class: lead -->

# Contributing

---

# Start Here: Tier 1

**Documentation & Testing**

- Fix READMEs, improve error messages
- Add missing test coverage
- Labels: [`good first issue`](https://github.com/open-telemetry/opentelemetry-collector/labels/good%20first%20issue) · [`help wanted`](https://github.com/open-telemetry/opentelemetry-collector-contrib/labels/help%20wanted)

<span class="small">

[`CONTRIBUTING.md` (core)](https://github.com/open-telemetry/opentelemetry-collector/blob/307e3abdbae90f3b9f5ccec7d4d50087be7f9f67/CONTRIBUTING.md) · [`CONTRIBUTING.md` (contrib)](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/8a0c8b89540ab85a9cb923f581fcde1b75eec348/CONTRIBUTING.md)

</span>

---

# Going Deeper: Tier 2 & 3

**Tier 2 — Existing components**
- Add a config option, fix a bug, improve observability
- PRs should be < 500 lines

**Tier 3 — New components**
- Build externally as a Go module
- Test with OCB + `replaces` directive
- Publish → register → propose to contrib

---

# Building a Component

```go
// 1. Config struct
type Config struct {
    Endpoint string        `mapstructure:"endpoint"`
    Timeout  time.Duration `mapstructure:"timeout"`
}
```

```go
// 2. Factory
func NewFactory() receiver.Factory {
    return receiver.NewFactory(
        component.MustNewType("myreceiver"),
        createDefaultConfig,
        receiver.WithMetrics(createMetrics,
            component.StabilityLevelDevelopment),
    )
}
```

---

# Component Implementation

```go
// 3. Start + Shutdown
func (r *myReceiver) Start(ctx context.Context,
    host component.Host) error {
    // start listening, scraping, or polling...
    return nil
}

func (r *myReceiver) Shutdown(ctx context.Context) error {
    // clean up resources, flush buffers
    return nil
}
```

Tip: `cmd/mdatagen` + `metadata.yaml` generates boilerplate

<span class="small">

[Build a custom receiver](https://opentelemetry.io/docs/collector/extend/custom-component/receiver/)

</span>

---

# Community

- **Slack:** [`#otel-collector`](https://cloud-native.slack.com/archives/C01N6P7KR6W) on [CNCF Slack](https://slack.cncf.io/)
- **SIG Collector:** Tue 17:00 PT / Wed 09:00 PT / Wed 05:00 PT
- [Subscribe to calendar](https://groups.google.com/a/opentelemetry.io/g/calendar-collector)
- [Meeting notes](https://docs.google.com/document/d/1r2JC5MB7GupCE7N32EwGEXs9V_YIsPgoFiLP4VWVMkE)
- **Role progression:** Member → Triager → Approver → Maintainer

<span class="small">

[`community/guides/contributor/membership.md`](https://github.com/open-telemetry/community/blob/main/guides/contributor/membership.md)

</span>

---

# Your Next Steps

1. Clone the three repos
2. Run `make otelcorecol` locally
3. Pick a [`good first issue`](https://github.com/open-telemetry/opentelemetry-collector-contrib/labels/good%20first%20issue)
4. Join `#otel-collector` on CNCF Slack
5. Attend a SIG Collector meeting

[Sustaining OpenTelemetry](https://www.bloomberg.com/company/stories/sustaining-opentelemetry-cncf-moving-from-dependency-management-to-stewardship/) · [CNCF blog](https://www.cncf.io/blog/2026/03/31/sustaining-opentelemetry-moving-from-dependency-management-to-stewardship/)
