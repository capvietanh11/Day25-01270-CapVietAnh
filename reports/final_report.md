# Day 10 Reliability Report

## 1. Architecture summary

`ReliabilityGateway.complete()` routes every request through three layers in order:
cache → circuit-breaker-guarded provider chain → static degraded response.

```
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached (route=cache_hit:<score>)
    |                                 |
    v                                 v MISS
[Circuit Breaker: primary] -------> Provider "primary"
    |  (OPEN? skip, don't call provider)
    v  success -> cache.set() + route="primary"
    |  failure/open -> continue
[Circuit Breaker: backup] --------> Provider "backup"
    |  (OPEN? skip)
    v  success -> cache.set() + route="fallback"
    |  failure/open -> continue
[Static fallback message]  (route="static_fallback", error=last_error)
```

Each provider has its own `CircuitBreaker` instance keyed by provider name, so a
failing primary does not affect the backup's circuit state. The cache sits in
front of the whole chain — a hit costs 0 latency and 0 estimated cost, and a
miss populates the cache after a successful provider call so the next similar
query is served from cache.

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Tolerates isolated blips (network jitter, one bad request) without tripping the breaker, but reacts within 3 consecutive failures instead of waiting for a long streak. |
| reset_timeout_seconds | 2 | Short enough that chaos scenarios (100 requests/scenario at ~200-300ms latency each) actually exercise HALF_OPEN → CLOSED recovery within the run instead of staying OPEN the whole time. |
| success_threshold | 1 | A single successful probe is enough evidence the provider recovered for this simulation; a stricter production setting (e.g. 2-3) would reduce flapping risk at the cost of slower recovery. |
| cache TTL | 300s | Long enough to get reuse across a 100-request burst without serving very stale answers. |
| similarity_threshold | 0.92 | Tested 0.85 first and got false hits on queries that only differed by a year/date (e.g. "refund policy for 2024" vs "...2026") since n-gram cosine still scores those very high on shared words. 0.92 plus the explicit `_looks_like_false_hit()` 4-digit-number guard closed that gap. |
| load_test requests | 100 per scenario (300 total across 3 scenarios) | Large enough for stable P95/P99 percentiles and multiple circuit-breaker open/close cycles, small enough to run in a few seconds. |

## 3. SLO definitions

SLOs evaluated against the `all_healthy`-inclusive combined run in
[`reports/metrics.json`](metrics.json) (300 requests across all 3 scenarios,
memory cache backend):

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 99.33% | ✅ Yes |
| Latency P95 | < 2500 ms | 315.47 ms | ✅ Yes |
| Fallback success rate | >= 95% | 96.72% | ✅ Yes |
| Cache hit rate | >= 10% | 64.0% | ✅ Yes |
| Recovery time | < 5000 ms | 2281.5 ms | ✅ Yes |

All five SLOs are met in this run. Note that `run_scenario()` samples a
random query and provider jitter per request (no fixed RNG seed), so
individual metrics move a few points between runs — a prior run of the same
config measured 98.33% availability / 93.51% fallback success rate (both
just under target) purely from that randomness, since two of the three
scenarios deliberately fail the primary provider 50-100% of the time. See
[Chaos scenarios](#7-chaos-scenarios) for the per-scenario breakdown, and
[Failure analysis](#8-failure-analysis) for what drives the occasional miss.

## 4. Metrics

From `reports/metrics.json` (memory cache backend, 300 requests,
reproduced via `make run-chaos`):

| Metric | Value |
|---|---:|
| availability | 0.9933 |
| error_rate | 0.0067 |
| latency_p50_ms | 265.06 |
| latency_p95_ms | 315.47 |
| latency_p99_ms | 319.32 |
| fallback_success_rate | 0.9672 |
| cache_hit_rate | 0.64 |
| estimated_cost_saved | 0.192 |
| circuit_open_count | 7 |
| recovery_time_ms | 2281.46 |

## 5. Cache comparison

Ran the same 3 scenarios (300 requests total) twice with
[`configs/default.yaml`](../configs/default.yaml) (cache enabled) vs
[`configs/no_cache.yaml`](../configs/no_cache.yaml) (`cache.enabled: false`):

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 277.46 | 273.18 | -4.28 ms |
| latency_p95_ms | 317.44 | 316.47 | -0.97 ms |
| estimated_cost | 0.121692 | 0.051358 | -0.070334 (-57.8%) |
| cache_hit_rate | 0.0 | 0.59 | +0.59 |
| availability | 0.95 | 0.9833 | +0.0333 |

Latency percentiles barely move because `FakeLLMProvider` sleeps
synchronously for its full simulated latency regardless of cache state — a
cache hit only saves time/cost on the ~59% of requests it actually serves, so
it doesn't shift the tail of the *provider-call* latency distribution (misses
still pay full latency). The clearest win is **cost**: with cache enabled,
59% of requests never call a paid provider at all, cutting estimated cost by
57.8%. Availability also improves because cache hits bypass the (partially
failing) provider chain entirely — a query that previously required a
fallback trip now returns instantly from cache once seen once.

## 6. Redis shared cache

- **Why in-memory cache is insufficient for multi-instance deployments**: `ResponseCache` stores entries in a Python list local to one process (`self._entries`). If the gateway runs behind a load balancer with multiple replicas, each instance has its own cache — a query cached by instance A is a cold miss on instance B, so the effective hit rate (and cost savings) collapses as replica count grows.
- **How `SharedRedisCache` solves this**: entries are stored as Redis hashes (`{prefix}{md5(query)}` → `{query, response}`) with `EXPIRE` for TTL, so any process pointed at the same Redis instance sees writes from any other process immediately. Similarity search falls back to `SCAN` + local `ResponseCache.similarity()` scoring since Redis has no built-in cosine similarity.

### Evidence of shared state

Output of a script creating two independent `SharedRedisCache` instances
(same Redis, same key prefix) and reading across them:

```
c1.set("What is the refund policy?", "Refunds within 30 days.")
c2.get(same query) -> value='Refunds within 30 days.' score=1.0
SHARED STATE CONFIRMED: c2 (separate instance) read what c1 wrote.
```

This also matches `test_shared_state_across_instances` in
[`tests/test_redis_cache.py`](../tests/test_redis_cache.py), which passes.

### Redis CLI output

After running `make run-chaos` with `cache.backend: redis`
([`configs/redis_backend.yaml`](../configs/redis_backend.yaml)):

```bash
$ docker compose exec redis redis-cli KEYS "rl:cache:*"
rl:cache:fff10da1c72c
rl:cache:3936614ac4c2
rl:cache:d354658dc020
rl:cache:3dab98c0e49e
rl:cache:9e413fd814eb
rl:cache:0bc3b1acf73d
rl:cache:844ef0143a5c
rl:cache:8baa2cfa11fa
rl:cache:dacb2b833659
rl:cache:095946136fea
rl:cache:734852f3cf4a
rl:cache:da61fb49b4f6
rl:cache:98332d0d1c9c
rl:cache:4fc3c69b9376

$ docker compose exec redis redis-cli DBSIZE
14
```

### In-memory vs Redis latency comparison

| Metric | In-memory cache | Redis cache | Notes |
|---|---:|---:|---|
| latency_p50_ms | 273.18 | 273.33 | Both dominated by simulated provider latency; Redis round-trip (local Docker container) is negligible in comparison. |
| latency_p95_ms | 316.47 | 317.54 | Same as above. |
| cache_hit_rate | 0.59 | 0.6867 | Redis run had a slightly higher hit rate in this trial (random query sampling), not a structural difference. |
| estimated_cost | 0.051358 | 0.039634 | Both benefit similarly from caching; the exact number moves with which queries randomly repeated. |

## 7. Chaos scenarios

All 3 scenarios from [`configs/default.yaml`](../configs/default.yaml), 100
requests each:

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Primary fails every call; circuit opens almost immediately; all served traffic routes through backup fallback | Circuit opened after 3 consecutive failures as designed; nearly all successful responses had `route="fallback"` via backup; `fallback_success_rate > 0.9` in this scenario alone | ✅ Pass |
| primary_flaky_50 | Primary fails ~50% of calls; circuit should oscillate between CLOSED/OPEN/HALF_OPEN as it trips, recovers, and re-trips; mix of `primary` and `fallback` routes | `circuit_open_count` and non-zero `recovery_time_ms` confirm multiple open→half_open→closed cycles; both `primary` and `fallback` routes observed; availability stayed above 80% | ✅ Pass |
| all_healthy | Both providers healthy; most traffic served by primary; circuit stays CLOSED | No unexpected circuit opens beyond baseline `fail_rate` noise (primary=0.25, backup=0.05 from base config); availability > 95% | ✅ Pass |

Combined `reports/metrics.json` `"scenarios"` block confirms all three:
`{"primary_timeout_100": "pass", "primary_flaky_50": "pass", "all_healthy": "pass"}`.

## 8. Failure analysis

**Remaining weakness**: fallback success rate meets the 95% SLO in this run
(96.72%) but is not guaranteed to — a prior run of the identical config
measured 93.51%, just under target, because `static_fallbacks` are counted
whenever *both* providers fail or are circuit-open for the same request.
During `primary_flaky_50`, there's a window right after the primary breaker
opens (and before backup absorbs full traffic) where a backup call can
itself fail (backup's own 5% baseline fail rate) with no third provider to
fall back to — that request becomes a static fallback and drags down the
rate. With only two providers, any correlated or simultaneous failure has no
further fallback layer, so the SLO is met on average but not on every run.

**Proposed fix**: add a third, cheaper/lower-quality provider (or a
cache-only "best-effort" response using a relaxed similarity threshold) as a
last resort before the static message, and/or increase `success_threshold`
so backup's breaker doesn't itself get destabilized by a couple of unlucky
failures while under elevated load from absorbing primary's traffic.

## 9. Next steps

1. Add a third fallback provider (or relaxed-threshold cache-only fallback) so a backup failure doesn't fall straight through to the static message — should close the fallback-success-rate SLO gap.
2. Implement the "Redis circuit state" stretch goal (INCR/EXPIRE-based breaker counters in Redis) so circuit state is shared across gateway instances the same way the cache already is — right now each replica's breaker trips independently.
3. Add cost-aware routing: once cumulative `estimated_cost` crosses a budget threshold, route to the cheaper provider or cache-only responses instead of always trying primary first.
