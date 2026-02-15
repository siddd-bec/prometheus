# PromQL Functions Deep Dive
## When to Use What — With Real SRE Scenarios
*"I know the syntax, but WHEN do I use rate vs irate vs increase vs sum?"*

---

## How This Guide Works

Every function is explained with:
- **What it does** (one sentence)
- **The real-world problem it solves** (SRE scenario you've actually faced)
- **Sample data → what the function does to it → result**
- **When to use it / When NOT to use it**
- **Ready-to-paste query**

---

## Table of Contents
1. [rate()](#1-rate)
2. [irate()](#2-irate)
3. [increase()](#3-increase)
4. [sum()](#4-sum)
5. [sum by() / sum without()](#5-sum-by--sum-without)
6. [avg()](#6-avg)
7. [min() / max()](#7-min--max)
8. [count()](#8-count)
9. [topk() / bottomk()](#9-topk--bottomk)
10. [histogram_quantile()](#10-histogram_quantile)
11. [avg_over_time() and friends](#11-_over_time-family)
12. [absent()](#12-absent)
13. [predict_linear()](#13-predict_linear)
14. [deriv()](#14-deriv)
15. [resets()](#15-resets)
16. [changes()](#16-changes)
17. [offset modifier](#17-offset)
18. [label_replace()](#18-label_replace)
19. [clamp_min() / clamp_max()](#19-clamp_min--clamp_max)
20. [Cheat Sheet: Which Function When?](#20-cheat-sheet-which-function-when)

---

## 1. rate()

**What it does:** Takes a counter and calculates the average per-second increase over a time window.

### The Problem

Your Prometheus has this metric:

```
http_requests_total{service="visa-direct-cards"} = 1,482,937
```

That number means: "since this service started, it has handled 1.48 million requests." If you graph it, you see a line going up forever. **Completely useless.** What you actually want to know is: **"how many requests per second are we doing RIGHT NOW?"**

### How rate() Solves It

```
Imagine these raw counter values over 5 minutes:

Time:    12:00     12:01     12:02     12:03     12:04     12:05
Value:   10,000    10,030    10,065    10,100    10,140    10,180

rate(http_requests_total[5m]) calculates:
  (10,180 - 10,000) / 300 seconds = 0.6 requests/second

Result: 0.6
```

The counter went up by 180 in 300 seconds. That's 0.6 per second. Now you have a meaningful number you can graph, alert on, and compare.

### When rate() is Useful — 6 Real Scenarios

**Scenario 1: "How much traffic is hitting my API right now?"**

You're on-call. PagerDuty fires. First question: is traffic normal or did it spike/drop?

```promql
rate(http_requests_total{service="visa-direct-cards"}[5m])
```
This gives you: requests per second, per time series. You'll see one line per unique label combination (per endpoint, per method, per status code, etc.).

---

**Scenario 2: "What percentage of requests are failing?"**

This is the single most important SRE query. You need TWO rates and divide them:

```promql
# The error RATIO (percentage)
(
  rate(http_requests_total{service="visa-direct-cards", status_code=~"5.."}[5m])
  /
  rate(http_requests_total{service="visa-direct-cards"}[5m])
) * 100
```

What's happening step by step:
```
Top:    rate() of 5xx errors    → 0.006 errors/sec
Bottom: rate() of ALL requests  → 0.6 requests/sec
Divide: 0.006 / 0.6            → 0.01
× 100                          → 1% error rate
```

---

**Scenario 3: "How fast is data flowing into ClickHouse?"**

You're managing the Splunk→ClickHouse migration. You want to see ingestion speed:

```promql
rate(clickhouse_inserted_rows_total{table="visa_logs"}[5m])
```
Result: rows being inserted per second. If this drops to zero, your pipeline is broken.

---

**Scenario 4: "Are my Kubernetes pods restarting?"**

```promql
rate(kube_pod_container_status_restarts_total{namespace="visa-direct"}[15m])
```
Restarts is a counter. rate() tells you "restarts per second." If this is > 0, something is crash-looping. Using `[15m]` here because restarts are infrequent — a short window might miss them.

---

**Scenario 5: "How fast is disk filling up?"**

```promql
rate(node_disk_written_bytes_total[5m])
```
Bytes written is a counter. rate() converts it to bytes/second write speed.

---

**Scenario 6: Alerting on error rate**

```yaml
# Alert when error rate exceeds 1% for 5 minutes
- alert: HighErrorRate
  expr: |
    (
      sum(rate(http_requests_total{status_code=~"5.."}[5m]))
      / sum(rate(http_requests_total[5m]))
    ) > 0.01
  for: 5m
```

### When NOT to Use rate()

| Don't use rate() when... | Why | Use this instead |
|--------------------------|-----|-----------------|
| The metric is a **gauge** (goes up AND down) | rate() on a gauge gives you nonsense — it assumes values only go up | `deriv()` for gauges |
| You want **total count** ("how many in last hour?") | rate() gives per-second, not total | `increase()` |
| You want to **catch sudden spikes** | rate() averages over the window, smoothing out spikes | `irate()` |
| The metric is already a rate (like `cpu_usage_percent`) | Applying rate() to something already per-second doubles the math | Just use the metric directly |

### The [5m] Window Explained

```
rate(http_requests_total[5m])
                         ^^^
                         This is the lookback window
```

rate() looks backwards this far and calculates the average rate over that period:

```
[1m]  window:
  12:04────────12:05
  Sees a 30-second spike clearly
  Very noisy graph
  Use for: live debugging during an incident

[5m]  window:
  12:00────────────────12:05
  Smooths out 30-second spikes
  Clean graph, still responsive
  Use for: dashboards, alerts (THIS IS YOUR DEFAULT)

[15m] window:
  11:50──────────────────────────12:05
  Hides anything shorter than ~5 minutes
  Very smooth line
  Use for: executive dashboards, SLO tracking

[1h]  window:
  11:05──────────────────────────────────────────12:05
  Only shows sustained trends
  Use for: capacity planning
```

**Rule:** Window must be ≥ 4× your scrape interval.
Prometheus scrapes every 15s → minimum useful window = `[1m]`

---

## 2. irate()

**What it does:** Calculates per-second rate using ONLY the last two data points in the window.

### How It's Different from rate()

```
Same data points:

Time:    12:00    12:01    12:02    12:03    12:04    12:05
Value:   10,000   10,030   10,065   10,100   10,800   10,810
                                             ^^^^^^
                                             Spike at 12:04!

rate(http_requests_total[5m]):
  (10,810 - 10,000) / 300 = 2.7/sec  ← Spike averaged away

irate(http_requests_total[5m]):
  (10,810 - 10,800) / 60 = 0.17/sec  ← Only sees last 2 points
  
But if you query at 12:04:
  irate = (10,800 - 10,100) / 60 = 11.67/sec  ← SPIKE VISIBLE!
```

irate() is like looking through a magnifying glass at just the latest moment. rate() is like stepping back and looking at the whole picture.

### When irate() is Useful

**Scenario 1: "There was a 10-second burst of errors — show me!"**

During incident investigation, you suspect a brief spike caused a cascade:

```promql
irate(http_requests_total{status_code="503"}[5m])
```

rate() with `[5m]` would smooth this away. irate() shows you the exact moment the burst happened because it only looks at the last two scrape points.

**Scenario 2: CPU spike detection on a specific pod**

```promql
irate(process_cpu_seconds_total{pod="visa-direct-cards-7b4d8f"}[5m])
```

If a pod briefly pegged the CPU and then recovered, irate() catches it.

### When NOT to Use irate()

| Don't use irate() when... | Why | Use this instead |
|---------------------------|-----|-----------------|
| **Writing alert rules** | irate() is so volatile it causes alert flapping (fires/resolves/fires/resolves) | `rate()` always for alerts |
| **Dashboard panels for daily use** | Too noisy, graph looks like an earthquake | `rate()` with `[5m]` |
| **Calculating SLOs/error budgets** | You need stable averages, not momentary spikes | `rate()` with `[5m]` or longer |

> **Simple rule: rate() for 99% of your work. irate() only when you're actively investigating a specific short incident.**

---

## 3. increase()

**What it does:** Calculates the total increase of a counter over a time window. It's like `rate() × seconds`.

### How It's Different from rate()

```
Same data:

Time:    12:00     12:05
Value:   10,000    10,180

rate(http_requests_total[5m]):
  (10,180 - 10,000) / 300 = 0.6 per second

increase(http_requests_total[5m]):
  10,180 - 10,000 = 180 total requests

Same underlying math, different framing:
  rate()     → "0.6 requests per second"    (speed)
  increase() → "180 requests in 5 minutes"  (count)
```

### When increase() is Useful

**Scenario 1: "How many errors happened in the last hour?"**

Your manager asks: "How many 500 errors did we have this morning?"

```promql
# "Give me a COUNT, not a rate"
sum(increase(http_requests_total{status_code="500", service="visa-direct-cards"}[1h]))
```
Result: **42 errors** — that's an answer your manager understands. "0.012 errors per second" is not.

**Scenario 2: "How many Visa Direct transactions did we process today?"**

```promql
sum(increase(visa_api_requests_total{api="visa_direct", http_status="200"}[24h]))
```
Result: **145,023 successful transactions today**.

**Scenario 3: "How many pod restarts this week?"**

```promql
sum(increase(kube_pod_container_status_restarts_total{namespace="visa-direct"}[7d]))
```
Result: **8 restarts this week** — clear number for your weekly SRE report.

**Scenario 4: Grafana stat panel showing "Requests Today"**

In a single-stat Grafana panel showing a big number:
```promql
sum(increase(http_requests_total{service="visa-direct-cards"}[24h]))
```
This shows: **1.2M** in big text. Much more intuitive than a rate.

### When NOT to Use increase()

| Don't use increase() when... | Why | Use this instead |
|------------------------------|-----|-----------------|
| Building a **time-series graph** | increase() gives you a flat number, not a line over time | `rate()` for graphs |
| Writing **alert rules** | Alerts need per-second rates to be comparable across time ranges | `rate()` |
| The window is **very short** (< 1m) | Too few data points for accurate calculation | `rate()` |

> **Simple rule: rate() for graphs and alerts. increase() for "how many total in the last X time."**

---

## 4. sum()

**What it does:** Adds up the values of all matching time series into one number.

### The Problem

When you run:
```promql
rate(http_requests_total{service="visa-direct-cards"}[5m])
```

You might get back **50 separate lines** — one for each combination of labels:

```
{method="GET",  path="/api/v1/cards",     status_code="200", instance="pod-1"} → 0.15
{method="GET",  path="/api/v1/cards",     status_code="200", instance="pod-2"} → 0.12
{method="GET",  path="/api/v1/cards",     status_code="200", instance="pod-3"} → 0.18
{method="POST", path="/api/v1/cards",     status_code="200", instance="pod-1"} → 0.05
{method="POST", path="/api/v1/cards",     status_code="400", instance="pod-1"} → 0.02
{method="GET",  path="/api/v1/transfers", status_code="200", instance="pod-1"} → 0.08
... 44 more rows ...
```

You just want ONE number: **total requests per second for the whole service.**

### How sum() Solves It

```promql
sum(rate(http_requests_total{service="visa-direct-cards"}[5m]))
```

```
Takes all 50 rows and adds them:
  0.15 + 0.12 + 0.18 + 0.05 + 0.02 + 0.08 + ... = 2.4

Result: 2.4 requests/second (one single number)
```

### When sum() is Useful

**Scenario 1: "What is my total traffic?"**

One number for total throughput:

```promql
sum(rate(http_requests_total{service="visa-direct-cards"}[5m]))
```
Result: **2.4 req/sec** — one number, no labels.

**Scenario 2: "Total error rate across all endpoints"**

```promql
# Error ratio for the ENTIRE service
(
  sum(rate(http_requests_total{service="visa-direct-cards", status_code=~"5.."}[5m]))
  /
  sum(rate(http_requests_total{service="visa-direct-cards"}[5m]))
) * 100
```

Without `sum()`, you'd get one error ratio PER label combination. With `sum()`, you get one number: **0.8% error rate for the whole service**.

**Scenario 3: "Total active connections across all pods"**

```promql
sum(http_active_connections{service="visa-direct-cards"})
```
Active connections is a gauge (goes up and down). `sum()` works on gauges too! It just adds up the current values from all pods.

**Scenario 4: "Total CPU usage for my namespace"**

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="visa-direct"}[5m]))
```
Adds up CPU usage across all containers in the namespace. Result: **4.2 CPU cores total**.

### When NOT to Use sum()

| Don't use sum() when... | Why | Use this instead |
|-------------------------|-----|-----------------|
| You want to see **each endpoint separately** | sum() collapses everything into one number | `sum by (path) (...)` |
| You want the **average**, not the total | sum() adds, it doesn't average | `avg()` |
| You want to find **which pod is the problem** | sum() hides individual pods | `topk()` or no aggregation |

---

## 5. sum by() / sum without()

**What it does:** sum by() adds values BUT keeps specified labels. sum without() adds values BUT drops specified labels.

### The Core Concept

Think of it like SQL GROUP BY:

```sql
-- SQL:
SELECT path, SUM(requests) FROM metrics GROUP BY path

-- PromQL equivalent:
sum by (path) (rate(http_requests_total[5m]))
```

### sum by() — "Show me totals grouped by THIS"

```promql
sum by (path) (rate(http_requests_total{service="visa-direct-cards"}[5m]))
```

What happens:
```
BEFORE (50 rows with many labels):
  {method="GET",  path="/api/v1/cards",     status="200", pod="pod-1"} → 0.15
  {method="POST", path="/api/v1/cards",     status="200", pod="pod-1"} → 0.05
  {method="GET",  path="/api/v1/cards",     status="200", pod="pod-2"} → 0.12
  {method="GET",  path="/api/v1/transfers", status="200", pod="pod-1"} → 0.08
  {method="GET",  path="/api/v1/transfers", status="200", pod="pod-2"} → 0.10

AFTER sum by (path):
  {path="/api/v1/cards"}     → 0.15 + 0.05 + 0.12 = 0.32
  {path="/api/v1/transfers"} → 0.08 + 0.10         = 0.18
```

All labels except `path` are dropped. Values with the same `path` are added together.

### When sum by() is Useful

**Scenario 1: "Show me traffic per endpoint"**

```promql
sum by (path) (rate(http_requests_total[5m]))
```
Result: one line per endpoint on your Grafana graph. This is your bread-and-butter dashboard panel.

**Scenario 2: "Show me errors by status code"**

```promql
sum by (status_code) (rate(http_requests_total{status_code=~"[45].."}[5m]))
```
Result: separate lines for 400, 401, 403, 404, 500, 502, 503. Instantly see which error type is dominant.

**Scenario 3: "Show me traffic per cluster" (Thanos)**

```promql
sum by (cluster) (rate(http_requests_total{service="visa-direct-cards"}[5m]))
```
Result: one line per cluster (us-east-1, eu-west-1, etc).

**Scenario 4: "Error rate per endpoint" (two labels)**

```promql
sum by (path, method) (rate(http_requests_total{status_code=~"5.."}[5m]))
```
You can group by multiple labels. This shows: GET /api/v1/cards errors vs POST /api/v1/cards errors separately.

### sum without() — "Sum everything EXCEPT this noisy label"

```promql
sum without (instance, pod) (rate(http_requests_total[5m]))
```

This is the opposite approach: instead of saying "keep only path," you say "drop instance and pod, keep everything else."

**When to use which:**
```
Use sum by()      when you know EXACTLY which labels you want to see
                  "Show me traffic by path"

Use sum without() when you want EVERYTHING except noisy labels
                  "Show me traffic without the per-pod breakdown"
                  Great when metrics have 10+ labels and you want most of them
```

---

## 6. avg()

**What it does:** Calculates the average value across all matching time series.

### How It's Different from sum()

```
Three pods reporting CPU usage:
  pod-1: 0.8 cores
  pod-2: 0.3 cores
  pod-3: 0.6 cores

sum()  = 0.8 + 0.3 + 0.6 = 1.7 (total cores used)
avg()  = (0.8 + 0.3 + 0.6) / 3 = 0.57 (average per pod)
```

### When avg() is Useful

**Scenario 1: "What's the average CPU per pod?"**

```promql
avg(rate(container_cpu_usage_seconds_total{namespace="visa-direct"}[5m]))
```
Result: **0.57 cores average**. If this is close to your pod CPU limit, you need to scale.

**Scenario 2: "What's the average latency per endpoint?"**

```promql
avg by (path) (
  rate(http_request_duration_seconds_sum[5m])
  /
  rate(http_request_duration_seconds_count[5m])
)
```
This gives you the average latency across all pods, for each endpoint.

**Scenario 3: "Average memory usage across all nodes"**

```promql
avg(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```
Result: **65% memory available on average across all nodes**.

### When NOT to Use avg()

| Don't use avg() when... | Why | Use this instead |
|-------------------------|-----|-----------------|
| Monitoring **latency** for SLOs | Averages hide the slow tail (the P99 pain) | `histogram_quantile()` |
| You need **total capacity** | Average doesn't tell you if you have enough total | `sum()` |
| One outlier matters | avg smooths away the one bad pod | `max()` or `topk()` |

> **The famous problem with averages:** If 99 pods have 10ms latency and 1 pod has 10,000ms latency, the average is 109ms. Looks fine! But 1% of your users are waiting 10 seconds. That's why P99 exists.

---

## 7. min() / max()

**What it does:** Returns the smallest (min) or largest (max) value across all matching series.

### When max() is Useful

**Scenario 1: "Which pod has the highest CPU?"**

```promql
max(rate(container_cpu_usage_seconds_total{namespace="visa-direct"}[5m]))
```
Result: **0.95 cores** — the hottest pod. If this is near the limit, that pod might get OOM-killed.

**Scenario 2: "What's the worst-case latency across pods?"**

```promql
max by (path) (
  rate(http_request_duration_seconds_sum[5m])
  /
  rate(http_request_duration_seconds_count[5m])
)
```
Shows you the slowest pod for each endpoint. If one pod is 10× slower than others, it might be on a bad node.

**Scenario 3: "Highest memory usage across any node"**

```promql
max(100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100))
```
Result: **92%** — your most memory-constrained node. Even if average is 65%, THIS node is about to run out.

### When min() is Useful

**Scenario: "What's the minimum free disk space across all nodes?"**

```promql
min(node_filesystem_avail_bytes{mountpoint="/"})
```
The node with the least free disk is the one that will fill up first. Alert on this, not the average.

---

## 8. count()

**What it does:** Counts how many time series match the query (not their values, just how many exist).

### When count() is Useful

**Scenario 1: "How many pods are running?"**

```promql
count(up{job="visa-direct-cards"} == 1)
```
Result: **6** — six pods are healthy. If your deployment expects 6 replicas and this shows 4, you have a problem.

**Scenario 2: "How many unique endpoints exist?"**

```promql
count(count by (path) (http_requests_total))
```
The inner `count by (path)` collapses to one series per path. The outer `count()` counts how many paths exist. Useful for cardinality analysis.

**Scenario 3: "How many pods are actually receiving traffic?"**

```promql
count(sum by (instance) (rate(http_requests_total[5m]) > 0))
```
If you have 6 pods but only 4 receive traffic, your load balancer might be misconfigured.

**Scenario 4: "How many unique error codes did we see today?"**

```promql
count(count by (error_code) (increase(visa_api_requests_total{http_status="400"}[24h]) > 0))
```
Result: **12 different error codes active today**.

---

## 9. topk() / bottomk()

**What it does:** Returns the top N (or bottom N) time series by their current value.

### When topk() is Useful

**Scenario 1: "Which 5 endpoints get the most traffic?"**

```promql
topk(5, sum by (path) (rate(http_requests_total[5m])))
```
Result:
```
{path="/api/v1/cards"}      → 1.2 req/sec
{path="/api/v1/transfers"}  → 0.8 req/sec
{path="/api/v1/query"}      → 0.5 req/sec
{path="/api/v2/cards"}      → 0.3 req/sec
{path="/health"}            → 0.2 req/sec
```
Instant visibility into what your API is actually doing.

**Scenario 2: "Which 3 pods consume the most CPU?"**

```promql
topk(3, sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="visa-direct"}[5m])))
```
During an incident, this immediately tells you which pod is the troublemaker.

**Scenario 3: "Top 10 Visa error codes"**

```promql
topk(10, sum by (error_code) (increase(visa_api_requests_total{http_status="400"}[1h])))
```
Result: ranked list of which error codes are happening most. This is your starting point for debugging.

### When bottomk() is Useful

**Scenario: "Which endpoints get the LEAST traffic?" (maybe decommission them)**

```promql
bottomk(5, sum by (path) (rate(http_requests_total[5m])))
```

---

## 10. histogram_quantile()

**What it does:** Calculates percentiles (P50, P95, P99) from histogram bucket data.

### The Problem It Solves

You want to know: "What's the latency that 99% of requests are faster than?"

average() won't work — it hides the slow tail. You need **percentiles**.

### How It Works

Your app exposes histogram buckets:
```
http_request_duration_seconds_bucket{le="0.05"}  = 4000   ← 4000 requests were ≤ 50ms
http_request_duration_seconds_bucket{le="0.1"}   = 4500   ← 4500 requests were ≤ 100ms
http_request_duration_seconds_bucket{le="0.25"}  = 4800   ← 4800 requests were ≤ 250ms
http_request_duration_seconds_bucket{le="0.5"}   = 4950   ← 4950 requests were ≤ 500ms
http_request_duration_seconds_bucket{le="1.0"}   = 4990   ← 4990 requests were ≤ 1s
http_request_duration_seconds_bucket{le="+Inf"}  = 5000   ← 5000 total requests
```

histogram_quantile(0.99, ...) finds: "at what latency are 99% of requests finished?"

```
99% of 5000 = 4950th request
The 4950th request falls in the le="0.5" bucket
So P99 ≈ 0.5 seconds
```

### When histogram_quantile() is Useful

**Scenario 1: "What's our P99 latency?" (SLO tracking)**

```promql
histogram_quantile(
  0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{service="visa-direct-cards"}[5m])
  )
)
```

**CRITICAL:** You MUST keep `le` in the `sum by()`. Without it, the function can't see the bucket boundaries and returns NaN.

**Scenario 2: "P99 latency broken down by endpoint"**

```promql
histogram_quantile(
  0.99,
  sum by (le, path) (
    rate(http_request_duration_seconds_bucket{service="visa-direct-cards"}[5m])
  )
)
```
Notice: `sum by (le, path)` — keep `le` AND the label you want to group by.

**Scenario 3: "Show P50, P95, P99 on the same graph"**

Three separate queries in one Grafana panel:
```promql
# P50 (median) — "typical request"
histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# P95 — "95% of users experience this or better"
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# P99 — "worst 1% of requests"
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

On your graph you'll see three lines. If P50 is 20ms but P99 is 3s, you know the slow tail is bad.

### The Most Common Mistake

```
❌ WRONG — forgot 'le' in sum by:
histogram_quantile(0.99, sum by (path) (rate(..._bucket[5m])))
                              ^^^^^^^^
                              Missing 'le'! Returns NaN.

✅ CORRECT — always include 'le':
histogram_quantile(0.99, sum by (le, path) (rate(..._bucket[5m])))
                              ^^^^^^^^^
                              le is always required
```

---

## 11. _over_time() Family

**What they do:** Apply a function to a range of values from a single series over time.

These are for **gauges** (values that go up and down, like memory, CPU %, temperature).

| Function | What it calculates | Use case |
|----------|-------------------|----------|
| `avg_over_time(m[1h])` | Average value over the last hour | "What was average CPU this morning?" |
| `max_over_time(m[1h])` | Highest value in the last hour | "What was peak memory usage?" |
| `min_over_time(m[1h])` | Lowest value in the last hour | "What was lowest free disk?" |
| `quantile_over_time(0.95, m[1h])` | 95th percentile value over time | "What was P95 CPU usage?" |
| `count_over_time(m[1h])` | How many data points exist | "Did this metric report consistently?" |
| `last_over_time(m[1h])` | Most recent value in window | "What's the latest reading?" |

### When avg_over_time() is Useful

**Scenario: "What was the average CPU usage of this pod over the last 6 hours?"**

```promql
avg_over_time(container_cpu_usage_percent{pod="visa-direct-7b4d8f"}[6h])
```

This is different from `avg()` which averages across MULTIPLE pods at one moment in time. `avg_over_time()` averages ONE pod across a period of time.

```
avg()              = average across pods AT a single moment
avg_over_time()    = average of ONE pod OVER a period of time
```

### When max_over_time() is Useful

**Scenario: "What was peak memory usage last night?"**

```promql
max_over_time(container_memory_usage_bytes{pod="visa-direct-7b4d8f"}[12h])
```
Even if current memory is 500MB, this might show 1.8GB — meaning there was a spike at 3 AM.

---

## 12. absent()

**What it does:** Returns 1 if the metric doesn't exist or has no data. Returns nothing if the metric exists.

### When absent() is Useful

**Scenario 1: "Alert me if my service stops sending metrics entirely"**

```promql
absent(up{job="visa-direct-cards"})
```
If the `up` metric for your service disappears (Prometheus can't scrape it at all), this returns 1. Use it in an alert.

**Scenario 2: "Alert if Visa API traffic drops to literally zero"**

```promql
absent(sum(rate(visa_api_requests_total{api="visa_direct"}[5m])))
```
If there are zero Visa Direct requests, something is catastrophically wrong — either your app crashed or the Visa integration is completely broken.

```yaml
- alert: NoVisaTraffic
  expr: absent(sum(rate(visa_api_requests_total{api="visa_direct"}[5m])))
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "ZERO Visa Direct API traffic for 5 minutes"
```

---

## 13. predict_linear()

**What it does:** Uses linear regression to predict where a gauge value will be in N seconds.

### When predict_linear() is Useful

**Scenario: "Will my disk fill up in the next 4 hours?"**

```promql
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 4*3600)
```

This looks at the last 6 hours of disk free space trend and predicts what it will be 4 hours from now.

```
Current free:  50GB
Trend:         losing 5GB per hour
Prediction:    50 - (5 × 4) = 30GB in 4 hours
```

Alert version:
```yaml
- alert: DiskWillFillIn4Hours
  expr: predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 4*3600) < 0
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Disk on {{ $labels.instance }} predicted to fill within 4 hours"
```

**Scenario 2: "Will my Prometheus TSDB run out of space?"**

```promql
predict_linear(prometheus_tsdb_storage_bytes[24h], 7*24*3600) 
```
Predicts TSDB size in 7 days based on the last 24 hours of growth.

---

## 14. deriv()

**What it does:** Calculates the per-second derivative (rate of change) of a gauge using linear regression.

### When deriv() is Useful

Think of `deriv()` as `rate()` but for gauges. rate() is for counters (only go up). deriv() is for gauges (go up AND down).

**Scenario: "Is memory usage trending up or down?"**

```promql
deriv(process_resident_memory_bytes{job="visa-direct-cards"}[1h])
```

```
Result > 0  →  Memory is growing (possible leak!)
Result = 0  →  Memory is stable
Result < 0  →  Memory is shrinking (GC working, scaling down)
```

**Scenario 2: "Are active connections increasing or decreasing?"**

```promql
deriv(http_active_connections{service="visa-direct-cards"}[15m])
```
If positive and growing, traffic is ramping up. If negative, traffic is dropping.

---

## 15. resets()

**What it does:** Counts how many times a counter has reset to zero within a time window.

### When resets() is Useful

**Scenario: "How many times has my service restarted?"**

```promql
resets(http_requests_total{service="visa-direct-cards"}[1h])
```

Every time a counter resets to 0, it usually means the process restarted. This tells you: **3 restarts in the last hour** — your service is crash-looping.

```yaml
- alert: ServiceCrashLooping
  expr: resets(http_requests_total[30m]) > 2
  for: 5m
  labels:
    severity: critical
```

---

## 16. changes()

**What it does:** Counts how many times a gauge's value changed within a time window.

### When changes() is Useful

**Scenario: "Is this config reload metric actually flapping?"**

```promql
changes(config_last_reload_successful{job="visa-direct-cards"}[1h])
```
If this shows 20 changes in an hour, the config is being reloaded constantly — something is wrong.

**Scenario 2: "How often did the number of ready pods change?"**

```promql
changes(kube_deployment_status_replicas_ready{deployment="visa-direct-cards"}[1h])
```
Lots of changes = pods are flapping up and down. Investigate HPA settings or resource limits.

---

## 17. offset

**What it does:** Shifts a query back in time by the specified duration.

### When offset is Useful

**Scenario 1: "Compare today's traffic to yesterday"**

```promql
# Current traffic
sum(rate(http_requests_total[5m]))

# Yesterday's traffic at this same time
sum(rate(http_requests_total[5m] offset 1d))

# The difference
sum(rate(http_requests_total[5m])) 
- sum(rate(http_requests_total[5m] offset 1d))
```

**Scenario 2: "Is traffic higher than last week?"**

```promql
# Percentage change vs last week
(
  sum(rate(http_requests_total[1h])) 
  - sum(rate(http_requests_total[1h] offset 7d))
)
/ sum(rate(http_requests_total[1h] offset 7d))
* 100
```
Result: **+15%** — traffic is 15% higher than last week at this time.

**Scenario 3: "Alert if traffic drops more than 50% vs 1 hour ago"**

```yaml
- alert: TrafficDrop
  expr: |
    sum(rate(http_requests_total[5m])) 
    < sum(rate(http_requests_total[5m] offset 1h)) * 0.5
  for: 5m
```

---

## 18. label_replace()

**What it does:** Creates or overwrites a label using a regex on another label's value.

### When label_replace() is Useful

**Scenario: Group all write methods together on a dashboard**

```promql
label_replace(
  sum by (method) (rate(http_requests_total[5m])),
  "method_type",         # new label name
  "write",               # new label value
  "method",              # source label
  "POST|PUT|PATCH|DELETE" # regex to match
)
```
This adds a label `method_type="write"` to all POST/PUT/PATCH/DELETE series. Useful when you want to group methods on a Grafana graph.

---

## 19. clamp_min() / clamp_max()

**What it does:** Sets a floor (min) or ceiling (max) on values.

### When clamp_min() is Useful

**Scenario: Avoid division by zero**

```promql
# Without clamp — crashes if total is 0:
rate(errors[5m]) / rate(total[5m])

# With clamp — if total rate is 0, use 1 instead (returns 0, not NaN):
rate(errors[5m]) / clamp_min(rate(total[5m]), 1)
```

---

## 20. Cheat Sheet: Which Function When?

### "I want to know..." → Use this function

| I want to know... | Function | Example |
|-------------------|----------|---------|
| Requests per second right now | `rate()` | `rate(http_requests_total[5m])` |
| If there was a brief spike | `irate()` | `irate(http_requests_total[5m])` |
| Total requests in the last hour | `increase()` | `increase(http_requests_total[1h])` |
| Total across all pods | `sum()` | `sum(rate(...[5m]))` |
| Broken down by endpoint | `sum by()` | `sum by (path) (rate(...[5m]))` |
| Average per pod | `avg()` | `avg(rate(...[5m]))` |
| The worst pod | `max()` | `max(rate(...[5m]))` |
| How many pods are running | `count()` | `count(up{job="x"} == 1)` |
| Top 5 busiest endpoints | `topk()` | `topk(5, sum by(path)(...))` |
| P99 latency | `histogram_quantile()` | `histogram_quantile(0.99, sum by(le)(...))` |
| Average CPU over last 6h | `avg_over_time()` | `avg_over_time(cpu[6h])` |
| Peak memory today | `max_over_time()` | `max_over_time(mem[24h])` |
| Service is completely dead | `absent()` | `absent(up{job="x"})` |
| Will disk fill in 4 hours | `predict_linear()` | `predict_linear(disk[6h], 4*3600)` |
| Is memory leaking (trending up) | `deriv()` | `deriv(mem[1h])` |
| How many process restarts | `resets()` | `resets(counter[1h])` |
| How often did value change | `changes()` | `changes(gauge[1h])` |
| Compare to yesterday | `offset` | `metric[5m] offset 1d` |
| Rename/group labels | `label_replace()` | `label_replace(metric, ...)` |
| Avoid division by zero | `clamp_min()` | `clamp_min(rate, 1)` |

### The Decision Tree

```
Is your metric a COUNTER (only goes up)?
├── YES: 
│   ├── Want per-second rate?      → rate()
│   ├── Want to catch spikes?      → irate()
│   ├── Want total count?          → increase()
│   └── Want restart count?        → resets()
│
└── NO, it's a GAUGE (goes up and down):
    ├── Want rate of change?        → deriv()
    ├── Want average over time?     → avg_over_time()
    ├── Want peak value?            → max_over_time()
    ├── Want to predict future?     → predict_linear()
    └── Want change frequency?      → changes()

After getting your base result, do you want to AGGREGATE across series?
├── Total across all pods          → sum()
├── Total per endpoint             → sum by (path) ()
├── Average per pod                → avg()
├── Find the worst one             → max()
├── Count how many                 → count()
└── Top/bottom N                   → topk() / bottomk()
```

### The Golden Rules

```
1. rate()     → for counters, for graphs, for alerts. YOUR DEFAULT.
2. increase() → for counters, when you want a total count number.
3. irate()    → ONLY for incident investigation. Never for alerts.
4. sum()      → always wrap rate() in sum() for service-wide views.
5. sum by()   → to break down by a specific dimension (endpoint, cluster, etc).
6. histogram_quantile() → ALWAYS include 'le' in sum by(). P99 > average.
7. absent()   → for "is anything alive?" dead-man-switch alerts.
8. offset     → for comparing now vs past (yesterday, last week).
```

---

> *The best way to internalize these: open your Prometheus UI, pick a metric, and try each function on it. See what changes. rate() and sum by() alone will handle 80% of your daily work.*
