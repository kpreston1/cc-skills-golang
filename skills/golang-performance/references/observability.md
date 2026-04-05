# Production Observability for Performance

Third-party monitoring complements local profiling (pprof, benchmarks) by providing continuous monitoring, historical trends, and regression detection in production.

## Datadog Runtime Metrics

Datadog automatically collects Go runtime metrics when the APM tracer is active. Enable runtime metrics by setting `DD_RUNTIME_METRICS_ENABLED=true` or via code:

```go
import "gopkg.in/DataDog/dd-trace-go.v2/profiler"

// Runtime metrics are collected by the Datadog Agent
// Set DD_RUNTIME_METRICS_ENABLED=true in the environment
```

The Datadog Agent scrapes Go runtime metrics and makes them available in Datadog as `go.*` metrics:

| Metric | What to look for |
| --- | --- |
| `go.goroutines` | Should correlate with load; growing independently of traffic = goroutine leak |
| `go.memstats.alloc` | Should be roughly stable under constant load; continuous increase = memory leak |
| `go.memstats.alloc_bytes_per_sec` | Allocation rate; high value drives GC frequency |
| `go.gc.pause_quantile_99` | Worst-case GC pause — spikes cause tail latency |
| `go.gc.runs_per_sec` | GC cycles/s — >2/s sustained suggests excessive allocation rate |
| `go.threads` | OS threads; growing above GOMAXPROCS = excessive blocking syscalls |

### Datadog Monitors (Alert Examples)

Use these as starting points — adjust thresholds to your service's baseline:

- **Goroutine leak**: `avg(last_10m):avg:go.goroutines{service:myapp} > 1000`
- **High GC pressure**: `avg(last_5m):avg:go.gc.runs_per_sec{service:myapp} > 2`
- **Memory growth**: `avg(last_15m):derivative:go.memstats.alloc{service:myapp} > 0` (positive derivative = leak)
- **P99 GC pause**: `avg(last_5m):avg:go.gc.pause_quantile_99{service:myapp} > 0.1` (>100ms)

## Continuous Profiling with Datadog

Enable the Datadog Continuous Profiler alongside APM. It collects always-on CPU and heap profiles with ~2-5% overhead and links them to traces automatically.

→ See `samber/cc-skills-golang@golang-observability` skill (profiling.md) for setup, profile types, and cost guidance.

### Using Profiles for PGO

Continuous profiling data can feed Profile-Guided Optimization (PGO). Capture a production CPU profile and compile with it:

```bash
# Download a production CPU profile
curl -o cpu.pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Build with PGO — compiler inlines and optimizes based on actual call frequency
go build -pgo=cpu.pprof ./cmd/myapp/
```

See [Runtime Tuning](./runtime.md#profile-guided-optimization-pgo) for more details.

## Regression Detection After Deploy

Use Datadog APM to compare service performance before and after a deploy:

1. Open the **Service page** in Datadog APM
2. Enable **Deployment Tracking** — tag spans with `DD_VERSION`
3. Compare P50/P95/P99 latency and error rate between versions in the **Deployments** tab

For allocation regressions, compare Datadog Continuous Profiler flamegraphs between two deploy versions using the **Compare** feature in the Profiling UI.

## Real-Time Visualization (Development)

| Tool | What it does |
| --- | --- |
| **statsviz** (`github.com/arl/statsviz`) | Real-time browser dashboard at `/debug/statsviz` — heap, GC pauses, goroutines, scheduler. Register with `statsviz.Register(mux)`. Great for local development |
| **expvar** (stdlib `expvar`) | JSON metrics at `/debug/vars` — lightweight, no dependencies |
