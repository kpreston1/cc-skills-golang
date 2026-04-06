# Profiling and Continuous Profiling

→ See `samber/cc-skills-golang@golang-troubleshooting` skill (pprof.md) for on-demand debugging.

## What Profiling Is

Profiling analyzes the runtime behavior of your program — where CPU time is spent, how memory is allocated, which goroutines are blocked, and where lock contention occurs. While traces tell you "this endpoint is slow," profiling tells you "this specific function on line 42 is the bottleneck."

## On-Demand Profiling with `pprof`

pprof endpoints MUST be protected — NEVER expose them publicly. They leak sensitive runtime information and can be abused for DoS.

→ See `samber/cc-skills-golang@golang-troubleshooting` pprof.md for the full pprof CLI reference (profile types, capturing, analyzing, commands).

## Continuous Profiling with Datadog

On-demand profiling requires you to be there when the problem happens. Datadog Continuous Profiler runs always-on in the background with low overhead (~2-5% CPU), so you can look back at profiles after an incident. Enable it alongside the APM tracer — it reuses the same agent connection.

```go
import (
    "gopkg.in/DataDog/dd-trace-go.v2/profiler"
)

func setupContinuousProfiling() {
    if os.Getenv("DD_PROFILING_ENABLED") != "true" {
        return
    }

    err := profiler.Start(
        profiler.WithService(os.Getenv("DD_SERVICE")),
        profiler.WithEnv(os.Getenv("DD_ENV")),
        profiler.WithVersion(os.Getenv("DD_VERSION")),
        profiler.WithProfileTypes(
            profiler.CPUProfile,
            profiler.HeapProfile,
            profiler.GoroutineProfile,
            profiler.MutexProfile,
            profiler.BlockProfile,
        ),
    )
    if err != nil {
        slog.Error("failed to start Datadog profiler", "error", err)
    } else {
        slog.Info("continuous profiling enabled")
    }
}

func shutdownContinuousProfiling() {
    profiler.Stop()
}
```

Call `setupContinuousProfiling()` after `StartObservability()` from the utils package. Call `shutdownContinuousProfiling()` in your shutdown sequence alongside `ShutdownObservability()`.

## Profile Types

| Type | What it measures | When to enable |
| --- | --- | --- |
| `CPUProfile` | Where CPU time is spent | Always — low overhead |
| `HeapProfile` | Live heap objects and allocations | Always — catches memory leaks |
| `GoroutineProfile` | Goroutine count and stack traces | When investigating leaks or hangs |
| `MutexProfile` | Mutex contention | When investigating lock-related latency |
| `BlockProfile` | Channel/sync blocking | When investigating goroutine stalls |

Start with CPU + Heap. Add the others when investigating specific issues.

## Cost of Continuous Profiling

- **CPU overhead** ~2-5% per instance — negligible for most services
- **Network** — profiles are shipped to Datadog Agent, then forwarded to Datadog. High-replica deployments multiply this
- **Toggle via `DD_PROFILING_ENABLED`** — enable only when needed, or on a fraction of replicas (e.g., 1 in 10) for large deployments

## Correlating Profiles with Traces

Datadog automatically links profiles to traces when both are active. In the Datadog APM UI, a slow trace shows a "Profile" link that opens the CPU flame graph for the exact time window of that trace. No extra instrumentation needed.

## When to Profile

1. Datadog APM shows a slow span → CPU profile to find the hot function
2. Memory usage grows steadily → Heap profile to find the allocating code path
3. Goroutine count increasing → Goroutine profile to find leaks
4. Mutex contention suspected → Mutex profile to quantify lock pressure
5. Before and after an optimization → compare Datadog profile snapshots to verify improvement
