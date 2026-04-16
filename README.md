# OpenTelemetry Collector Deep Dive

A 20-minute code deep dive into the OpenTelemetry Collector architecture, internals, and contribution paths.

Presented at the [Bloomberg "Sustaining OpenTelemetry" Mentorship Program](https://www.bloomberg.com/company/stories/sustaining-opentelemetry-cncf-moving-from-dependency-management-to-stewardship/) (April 2026).

## View

```bash
make install   # first time only
make watch     # opens live preview in browser
```

Or build and serve:

```bash
make html
make serve     # http://localhost:8000/presentation.html
```

## Export

```bash
make pdf       # presentation.pdf
```

## Topics Covered

- What the Collector does and how it fits into the OTel architecture
- Five component types: receivers, processors, exporters, connectors, extensions
- Pipeline configuration and the internal DAG
- Code walkthrough: Component interface, Factory pattern, consumer chain, graph builder
- How distributions are built with OCB (OpenTelemetry Collector Builder)
- Three repos: core, contrib, releases
- Contribution entry points and community resources

## Links

- [opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector)
- [opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)
- [opentelemetry-collector-releases](https://github.com/open-telemetry/opentelemetry-collector-releases)
- [Collector docs](https://opentelemetry.io/docs/collector/)
