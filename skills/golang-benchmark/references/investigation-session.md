# Investigation Session Setup

Tools and techniques for **temporary deep-dive performance investigation** — not everyday monitoring. These are things you enable for hours or days while debugging a specific issue, then disable.

## Setting Up a Session

Before diving into profiles, set up the environment to collect high-resolution data:

1. **Enable pprof** via environment variable — no recompile needed:

   ```bash
   kubectl set env deployment/my-service PPROF_ENABLED=true
   kubectl rollout restart deployment/my-service
   ```

2. **Enable Datadog Continuous Profiler** on the target instance only — not fleet-wide. Fleet-wide continuous profiling has cost and overhead implications; a single instance is enough for investigation.

   ```bash
   kubectl set env deployment/my-service DD_PROFILING_ENABLED=true
   kubectl rollout restart deployment/my-service
   ```

3. **Enable debug logging** via env var if needed — but only on the target instance. Debug logging has significant throughput impact:

   ```bash
   kubectl set env deployment/my-service LOG_LEVEL=debug
   kubectl rollout restart deployment/my-service
   ```

**Key principle:** all costly debug features (pprof HTTP, continuous profiling, debug log level, trace collection) SHOULD be configurable via environment variables. This allows instant toggle without recompile. Design your application to support this from day one.

## Datadog Go Runtime Metrics

Datadog automatically collects Go runtime metrics when APM is enabled. These are invaluable during investigation sessions — they provide a time-series view of memory, GC, goroutines, and CPU that complements point-in-time profiles.

Runtime metrics are available in Datadog under the `runtime.go.*` namespace. Enable with:

```go
import "gopkg.in/DataDog/dd-trace-go.v1/profiler"

// In your observability setup:
profiler.Start(
    profiler.WithRuntimeMetrics(),
    profiler.WithProfileTypes(
        profiler.CPUProfile,
        profiler.HeapProfile,
        profiler.GoroutineProfile,
        profiler.MutexProfile,
    ),
)
```

Or via environment variable:

```bash
DD_RUNTIME_METRICS_ENABLED=true
```

### Key Metrics

**Memory:**

| Metric | What to look for |
| --- | --- |
| `runtime.go.mem.heap_alloc` | Current heap allocation. Continuously increasing = memory leak. |
| `runtime.go.mem.heap_inuse` | Heap in active use. Compare with `heap_alloc` — large gap = GC can reclaim. |
| `runtime.go.mem.heap_sys` | Total heap requested from OS. Should grow slowly, not continuously. |
| `runtime.go.mem.live_objects` | Count of live heap objects. High count = many small allocations, GC-heavy. |

**GC pressure:**

| Metric | What to look for |
| --- | --- |
| `runtime.go.gc.count` | GC cycles per second. >2/s sustained = high allocation rate. Reduce allocations per request. |
| `runtime.go.gc.pause_ns` | GC pause duration in nanoseconds. Spikes here cause tail latency (P99). |
| `runtime.go.gc.cpu_fraction` | Fraction of CPU used by GC. >0.25 = GC overhead significant. |

**Goroutines:**

| Metric | What to look for |
| --- | --- |
| `runtime.go.num_goroutine` | Should correlate with load. Growing independently of traffic = goroutine leak. |
| `runtime.go.num_cgo_call` | C calls from Go. High counts = CGO overhead in hot path. |

**CPU:**

| Metric | What to look for |
| --- | --- |
| `runtime.go.cpu.user_time` | User CPU time consumed by the Go program. |
| `runtime.go.num_cpu` | GOMAXPROCS value. Unexpected changes affect concurrency. |

## Datadog APM Deep-Dive Queries

Use the Datadog Metrics Explorer or Dashboards during investigation sessions. Each metric below maps to a root cause.

### GC pressure

Query `runtime.go.gc.pause_ns` with `max:runtime.go.gc.pause_ns{service:my-service}`. Correlate GC spikes with latency increases on the same timeline.

### Memory leak detection

Graph `runtime.go.mem.heap_alloc` over a 24h window under constant load. A linear upward trend = leak. A sawtooth pattern = healthy GC collection.

### Goroutine leak detection

Graph `runtime.go.num_goroutine` correlated with traffic (`trace.http.request` count). If goroutines grow while traffic stays flat = leak.

### Post-deploy regression detection

Use Datadog deployment tracking (set `DD_VERSION` env var) — Datadog automatically marks deploy events on all metric charts. Compare `runtime.go.mem.heap_alloc` and GC metrics before/after the deploy marker.

```bash
kubectl set env deployment/my-service DD_VERSION=v1.2.3
```

### Example Datadog Monitor alerts

```yaml
# GC CPU overhead too high
name: "Go GC CPU fraction high"
query: "avg(last_10m):avg:runtime.go.gc.cpu_fraction{env:production} > 0.25"
message: "GC consuming >25% CPU — reduce allocations or tune GOGC"

# Goroutine leak
name: "Goroutine count growing"
query: "avg(last_30m):anomaly(avg:runtime.go.num_goroutine{env:production}, 'basic', 3) >= 1"
message: "Goroutine count anomaly — check for goroutine leaks"

# Memory near container limit
name: "Heap approaching limit"
query: "avg(last_5m):avg:runtime.go.mem.heap_sys{env:production} > <limit_bytes>"
message: "Heap sys approaching container memory limit"
```

## Datadog Continuous Profiler

When `DD_PROFILING_ENABLED=true`, Datadog collects CPU, heap, goroutine, and mutex profiles continuously and uploads them to the Datadog UI.

Profiles are viewable at **APM → Profiling** in the Datadog UI. Key profile types:

| Profile type | What it shows | Use when |
| --- | --- | --- |
| **CPU** | Which functions consume CPU | High CPU usage, slow responses |
| **Heap (allocs)** | Where memory is allocated | GC pressure, high allocation rate |
| **Heap (inuse)** | What is currently alive in memory | Memory leak investigation |
| **Goroutine** | Goroutine count and stack traces | Goroutine leak investigation |
| **Mutex** | Lock contention hotspots | Throughput bottleneck from synchronization |

The Datadog Continuous Profiler also supports **flame graph diffs** — compare a before/after deploy to see exactly which functions changed CPU or memory usage.

## Cost Warnings

**Profiles and traces are expensive to collect.** Keep them short-term and localized:

- **pprof CPU profiling** — CPU-intensive during the capture window. Don't run 30s profiles back-to-back in production. Space them out.
- **Datadog Continuous Profiler** — ~2-5% CPU overhead per instance. At scale (hundreds of instances), this adds up in compute cost. Enable on a subset of instances or on-demand via `DD_PROFILING_ENABLED` env var.
- **Execution traces** — generate large files quickly (MB/s). Capture 5-10s max. Longer traces are unwieldy and slow to analyze.
- **Debug log level** — significant throughput impact due to allocation and I/O overhead. Never leave on permanently.
- **All costly features** SHOULD be toggleable via environment variables for instant on/off without recompile. Design for this from day one.
