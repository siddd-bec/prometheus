# PromQL Hands-On Guide
## For API Traffic Monitoring with Thanos
*A Practical, Exercise-Driven Guide for SRE Teams*

---

### What You Will Learn
- Understand how Prometheus stores and names API metrics
- Write PromQL queries from scratch — instant vectors, range vectors, aggregations
- Build real dashboards for latency, errors, throughput (RED method)
- Query across clusters using Thanos global view
- Debug slow queries and avoid common pitfalls
- 50+ hands-on exercises you can run today

---

## Table of Contents
1. [How Prometheus Stores API Metrics](#chapter-1-how-prometheus-stores-api-metrics)
2. [Your First PromQL Queries](#chapter-2-your-first-promql-queries)
3. [Label Matchers — Filtering Your API Traffic](#chapter-3-label-matchers--filtering-your-api-traffic)
4. [Rate, Increase & irate](#chapter-4-rate-increase--irate)
5. [Aggregation Operators](#chapter-5-aggregation-operators--slicing-data-your-way)
6. [The RED Method — Rate, Errors, Duration](#chapter-6-the-red-method--rate-errors-duration)
7. [Histogram & Summary — Latency Percentiles](#chapter-7-histogram--summary--latency-percentiles)
8. [Binary Operators & Joins](#chapter-8-binary-operators--joins)
9. [Thanos — Querying Across Clusters](#chapter-9-thanos--querying-across-clusters)
10. [Recording Rules & Alerts for APIs](#chapter-10-recording-rules--alerts-for-apis)
11. [Performance Tuning & Debugging Slow Queries](#chapter-11-performance-tuning--debugging-slow-queries)
12. [50 Hands-On Exercises](#chapter-12-50-hands-on-exercises)

---

## Chapter 1: How Prometheus Stores API Metrics

### 1.1 The Anatomy of a Metric

Before you write a single query, you need to understand what Prometheus is actually storing. Every metric in Prometheus has three parts:

| Part | Example | What It Means |
|------|---------|---------------|
| Metric name | `http_requests_total` | What is being measured (total HTTP requests) |
| Labels | `{method="GET", path="/api/v1/cards"}` | Key-value pairs that add dimensions |
| Value + Timestamp | `14523 @ 1707840000` | The actual number at a point in time |

Think of it like a spreadsheet row: the metric name is the sheet name, labels are column headers, and the value is the cell.

### 1.2 The Four Metric Types

Prometheus has exactly four metric types. Every API metric you will ever see falls into one of these:

| Type | Behavior | API Example |
|------|----------|-------------|
| **Counter** | Only goes UP (resets to 0 on restart) | `http_requests_total`, `http_errors_total` |
| **Gauge** | Goes UP or DOWN | `http_active_connections`, `queue_depth` |
| **Histogram** | Counts values falling into buckets | `http_request_duration_seconds_bucket` |
| **Summary** | Calculates percentiles on the client | `http_request_duration_seconds{quantile=...}` |

> 🟢 **WHY THIS MATTERS:** If you use a counter directly on a graph, you will see an ever-increasing line. You MUST use `rate()` to make it useful. This is the #1 beginner mistake.

### 1.3 Common API Metric Labels

In a typical API monitoring setup, your metrics will carry labels like these. Thanos may add extra external labels:

| Label | Example Values | Purpose |
|-------|---------------|---------|
| `method` | GET, POST, PUT, DELETE, PATCH | HTTP method |
| `handler` / `path` / `route` | /api/v1/cards, /api/v2/transfers | API endpoint |
| `status_code` / `code` | 200, 201, 400, 404, 500, 503 | HTTP response code |
| `service` / `job` | visa-direct-cards, payment-gateway | Which microservice |
| `instance` | 10.0.1.5:8080 | Individual pod/host |
| `cluster` (Thanos) | us-east-1, eu-west-1 | Which Kubernetes cluster |
| `environment` | prod, staging, canary | Deployment environment |
| `api_version` | v1, v2, v3 | API version |
| `client_id` | partner-abc, internal-app | Who is calling your API |
| `region` (Thanos) | us, eu, apac | Geographic region |

> 🟠 **SRE TIP:** Run this in Prometheus to see ALL labels on a metric: `http_requests_total` (just type the metric name and hit Execute). This shows you every time series.

---

## Chapter 2: Your First PromQL Queries

### 2.1 Instant Vector — A Snapshot Right Now

An instant vector gives you the most recent value for each time series. Think of it as: "What is the current value?"

**Query 1: See all HTTP request counts**
```promql
http_requests_total
```
This returns one row per unique combination of labels. If you have 5 endpoints × 4 methods × 3 status codes, that could be 60+ rows.

**Query 2: Filter to just POST requests**
```promql
http_requests_total{method="POST"}
```
Now you only see rows where method is POST. This is called a label matcher, and we will cover them deeply in Chapter 3.

### 2.2 Range Vector — A Window of History

A range vector gives you all the data points within a time window. You cannot graph a range vector directly — you must wrap it in a function like `rate()` or `avg_over_time()`.

**Query 3: Get the last 5 minutes of data**
```promql
http_requests_total{method="POST"}[5m]
```

The `[5m]` suffix means "give me all samples from the last 5 minutes". Common suffixes:

| Suffix | Duration | When to Use |
|--------|----------|-------------|
| `[1m]` | 1 minute | Very short-term, high-resolution debugging |
| `[5m]` | 5 minutes | Standard rate() window (most common) |
| `[15m]` | 15 minutes | Smoother graphs, fewer spikes |
| `[1h]` | 1 hour | Trend analysis, SLO calculations |
| `[1d]` | 1 day | Daily aggregations, capacity planning |

> 🔴 **COMMON MISTAKE:** Trying to graph a range vector directly. Prometheus will say "cannot plot range vector". Always wrap it: `rate(metric[5m])`, not `metric[5m]`.

### 2.3 The Prometheus UI Walkthrough

Open your Prometheus UI (usually at `:9090` or your Thanos Query endpoint). Here is a step-by-step exercise:

**Step 1:** In the Expression box, type: `up` and click Execute. This shows which targets are healthy (1) or down (0).

**Step 2:** Click the Graph tab. Change the time range to 1h. You should see flat lines at 1 (healthy).

**Step 3:** Now type: `http_requests_total` in Table view. Look at the labels on each row — this is your metric landscape.

**Step 4:** Type: `rate(http_requests_total[5m])` and switch to Graph. Now you see requests per second. Congratulations, this is your first real query!

---

## Chapter 3: Label Matchers — Filtering Your API Traffic

### 3.1 The Four Matcher Types

Label matchers are how you filter down to exactly the data you want. There are exactly four:

| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | Exact match | `{method="GET"}` |
| `!=` | Not equal | `{status_code!="200"}` |
| `=~` | Regex match | `{path=~"/api/v[12]/.*"}` |
| `!~` | Regex NOT match | `{handler!~"/health\|/ready"}` |

### 3.2 Combining Matchers

You can chain multiple matchers with commas. They work as AND logic (all must be true):

**Query: POST requests to cards API that returned errors**
```promql
http_requests_total{
  method="POST",
  path=~"/api/v.*/cards.*",
  status_code=~"5.."
}
```
This matches POST requests to any cards endpoint that returned a 5xx error code.

### 3.3 Regex Patterns for API Paths

API paths are where regex matchers really shine. Here are patterns you will use constantly:

| Pattern | Matches | Use Case |
|---------|---------|----------|
| `/api/v.*/cards.*` | /api/v1/cards, /api/v2/cards/123 | All cards endpoints, any version |
| `/api/v1/(cards\|transfers)` | /api/v1/cards, /api/v1/transfers | Specific endpoints only |
| `status_code=~"5.."` | 500, 502, 503, 504 | All server errors |
| `status_code=~"4.."` | 400, 401, 403, 404 | All client errors |
| `method=~"POST\|PUT\|PATCH"` | POST, PUT, PATCH | All write operations |
| `handler!~"/health\|/ready\|/metrics"` | Anything not health checks | Exclude noise endpoints |

> 🟢 **WHY THIS MATTERS:** If you do not exclude health check endpoints, your traffic dashboards will be dominated by Kubernetes liveness probes calling /health every 10 seconds. Always filter them out.

### 3.4 The \_\_name\_\_ Meta-Label

Every metric has a hidden label called `__name__` which holds the metric name. This lets you query across multiple metric names:

```promql
# Find all HTTP-related metrics in your system
{__name__=~"http_.*"}

# Combine request and error metrics
{__name__=~"http_(requests|errors)_total", service="visa-direct-cards"}
```

> 🟠 **SRE TIP:** This is extremely useful for discovery. When you join a new team, run `{__name__=~".*"}` (careful — returns everything!) or better: `{job="your-service-name"}` to see all metrics your service exposes.

---

## Chapter 4: Rate, Increase & irate

### 4.1 Why You Need rate()

Counters in Prometheus only go up. If your `http_requests_total` is at 1,000,000 — that tells you nothing useful. What you want to know is: how many requests per second are we getting RIGHT NOW?

That is exactly what `rate()` does. It takes a counter and turns it into a per-second rate:

```promql
# Raw counter (useless on a graph - just goes up)
http_requests_total{service="visa-direct-cards"}

# Per-second request rate (THIS is what you want)
rate(http_requests_total{service="visa-direct-cards"}[5m])
```

### 4.2 rate() vs irate() vs increase()

| Function | Calculates | Best For | Window |
|----------|-----------|----------|--------|
| `rate(m[5m])` | Avg per-second rate over window | Dashboards, alerting, SLOs | Use 5m for most cases |
| `irate(m[5m])` | Per-second rate using last 2 points | Spotting sudden spikes | Use for volatile metrics |
| `increase(m[1h])` | Total increase over window | "How many requests in last hour?" | Use for total counts |

**Visual Example: Imagine These Data Points**
```
Time:   :00   :15   :30   :45   :00
Value:  100   110   115   180   200

rate() over 1m = (200-100)/60 = 1.67/sec  (smooth average)
irate()        = (200-180)/15 = 1.33/sec  (last 2 points only)
increase()     = 200-100 = 100 total       (raw count increase)
```

> 🔴 **COMMON MISTAKE:** Using `irate()` for alerting. Because irate() only uses the last two data points, it is extremely volatile and will cause alert flapping. Always use `rate()` for alerts.

### 4.3 Choosing the Right Window

The window size in `rate(metric[WINDOW])` affects smoothness:

| Window | Effect | Use When |
|--------|--------|----------|
| `[1m]` | Very spiky, shows micro-bursts | Debugging a specific incident |
| `[5m]` | Balanced — smooth but responsive | Default for dashboards and alerts |
| `[15m]` | Very smooth, hides short spikes | Executive dashboards, trend lines |
| `[1h]` | Ultra smooth, shows trends only | Capacity planning, weekly reviews |

> 🟢 **WHY THIS MATTERS:** Your window must be at least 4x your scrape interval. If Prometheus scrapes every 15s, your minimum useful window is `[1m]`. Using `[30s]` would give you unreliable results.

### 4.4 Hands-On Practice

**Exercise 4A: Request Rate by Endpoint**
```promql
# Total request rate for your service
sum(rate(http_requests_total{service="visa-direct-cards"}[5m]))

# Break it down by endpoint
sum by (path) (rate(http_requests_total{service="visa-direct-cards"}[5m]))

# Break it down by method AND endpoint
sum by (method, path) (rate(http_requests_total{service="visa-direct-cards"}[5m]))
```

**Exercise 4B: Error Rate as a Percentage**
```promql
# Error rate percentage (5xx errors / total requests * 100)
sum(rate(http_requests_total{service="visa-direct-cards", status_code=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="visa-direct-cards"}[5m]))
* 100
```

---

## Chapter 5: Aggregation Operators — Slicing Data Your Way

### 5.1 The Aggregation Operators

Aggregation operators combine multiple time series into fewer series. Think of them as GROUP BY in SQL:

| Operator | What It Does | SQL Equivalent |
|----------|-------------|----------------|
| `sum` | Add values together | SUM() |
| `avg` | Calculate average | AVG() |
| `min` / `max` | Find smallest / largest | MIN() / MAX() |
| `count` | Count number of time series | COUNT() |
| `topk(n, ...)` | Top N series by value | ORDER BY ... LIMIT N |
| `bottomk(n, ...)` | Bottom N series | ORDER BY ... ASC LIMIT N |
| `count_values` | Count series with same value | GROUP BY value, COUNT(*) |
| `quantile(q, ...)` | Calculate quantile across series | PERCENTILE_CONT |
| `stddev` / `stdvar` | Standard deviation / variance | STDDEV() |

### 5.2 by() vs without() — Two Ways to Group

These control which labels survive after aggregation:

**by() = KEEP only these labels (drop everything else)**
```promql
# "Give me request rate, grouped by endpoint"
sum by (path) (rate(http_requests_total[5m]))
# Result: one row per unique 'path' value
```

**without() = DROP these labels (keep everything else)**
```promql
# "Give me request rate, but collapse the instance label"
sum without (instance) (rate(http_requests_total[5m]))
# Result: removes 'instance' label, keeps all other labels
```

> 🟠 **SRE TIP:** Use `by()` when you know exactly what you want to see. Use `without()` when you want everything EXCEPT noisy labels like `instance` or `pod`.

### 5.3 Practical Aggregation Patterns for APIs

**Pattern 1: Top 5 busiest endpoints**
```promql
topk(5, sum by (path) (rate(http_requests_total[5m])))
```

**Pattern 2: Error count by service and status code**
```promql
sum by (service, status_code) (
  rate(http_requests_total{status_code=~"[45].."}[5m])
)
```

**Pattern 3: Average latency per endpoint**
```promql
avg by (path) (
  rate(http_request_duration_seconds_sum[5m])
  /
  rate(http_request_duration_seconds_count[5m])
)
```

**Pattern 4: How many pods are serving each endpoint?**
```promql
count by (path) (rate(http_requests_total[5m]) > 0)
```

---

## Chapter 6: The RED Method — Rate, Errors, Duration

The RED method is the gold standard for monitoring APIs. Every API dashboard should answer three questions:

| Letter | Question | Metric |
|--------|----------|--------|
| **R — Rate** | How many requests per second? | `rate(http_requests_total[5m])` |
| **E — Errors** | How many of those are failing? | `rate(http_requests_total{status=~"5.."}[5m])` |
| **D — Duration** | How long do successful requests take? | `histogram_quantile(0.99, ...)` |

### 6.1 Rate — Total Request Throughput

```promql
# Total requests per second across all endpoints
sum(rate(http_requests_total{service="visa-direct-cards"}[5m]))

# Broken down by endpoint (for your Grafana dashboard)
sum by (path) (rate(http_requests_total{service="visa-direct-cards"}[5m]))
```

### 6.2 Errors — Error Rate & Error Ratio

**Error Rate (errors per second)**
```promql
sum(rate(http_requests_total{
  service="visa-direct-cards",
  status_code=~"5.."
}[5m]))
```

**Error Ratio (percentage of requests that are errors)**
```promql
# This is the query you will use for SLO alerts
(
  sum(rate(http_requests_total{service="visa-direct-cards", status_code=~"5.."}[5m]))
  /
  sum(rate(http_requests_total{service="visa-direct-cards"}[5m]))
) * 100
```

### 6.3 Duration — Latency Percentiles

```promql
# P99 latency (99th percentile) - "worst 1% of requests"
histogram_quantile(
  0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{service="visa-direct-cards"}[5m])
  )
)

# P50 latency (median) - "typical request"
histogram_quantile(
  0.50,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{service="visa-direct-cards"}[5m])
  )
)
```

> 🟢 **WHY THIS MATTERS:** P99 catches the slow tail that averages hide. If your average latency is 50ms but P99 is 2s, that means 1% of your users (potentially thousands) are having a terrible experience.

### 6.4 Complete RED Dashboard Queries

Copy these directly into Grafana panels for a complete API health dashboard:

| Panel Name | PromQL |
|-----------|--------|
| Request Rate | `sum by(path)(rate(http_requests_total{service="$service"}[5m]))` |
| Error Rate % | `(sum(rate(http_requests_total{status_code=~"5.."}[5m]))/sum(rate(http_requests_total[5m])))*100` |
| P50 Latency | `histogram_quantile(0.50, sum by(le)(rate(http_request_duration_seconds_bucket[5m])))` |
| P99 Latency | `histogram_quantile(0.99, sum by(le)(rate(http_request_duration_seconds_bucket[5m])))` |
| Active Conns | `sum(http_active_connections{service="$service"})` |

---

## Chapter 7: Histogram & Summary — Latency Percentiles

### 7.1 How Histograms Work

A histogram in Prometheus is actually THREE metrics combined:

| Metric | What It Stores | Example |
|--------|---------------|---------|
| `_bucket{le="..."}` | Count of requests faster than le | `http_request_duration_seconds_bucket{le="0.5"} = 4500` |
| `_sum` | Total sum of all observed values | `http_request_duration_seconds_sum = 892.5` |
| `_count` | Total count of observations | `http_request_duration_seconds_count = 5000` |

The `le` label stands for "less than or equal to". A bucket with `le="0.5"` counts all requests that took 0.5 seconds or less.

### 7.2 histogram_quantile() Explained

This function calculates percentiles from histogram buckets. The syntax is:

```promql
histogram_quantile(
  <percentile>,       # 0.50 = P50, 0.95 = P95, 0.99 = P99
  <rate of buckets>   # MUST be rate(), MUST keep the 'le' label
)
```

**Common Percentile Queries**
```promql
# P50 (median) — "how fast is a typical request?"
histogram_quantile(0.50, sum by (le) (
  rate(http_request_duration_seconds_bucket{service="visa-direct-cards"}[5m])
))

# P95 — "how fast is the experience for 95% of users?"
histogram_quantile(0.95, sum by (le) (
  rate(http_request_duration_seconds_bucket{service="visa-direct-cards"}[5m])
))

# P99 per endpoint — keep the 'path' label AND the 'le' label
histogram_quantile(0.99, sum by (le, path) (
  rate(http_request_duration_seconds_bucket{service="visa-direct-cards"}[5m])
))
```

> 🔴 **COMMON MISTAKE:** Forgetting to keep the `le` label in your aggregation. If you write `sum by (path)` without `(le)`, `histogram_quantile` will return NaN because it needs the bucket boundaries.

### 7.3 Average Latency (Without Histograms)

If you just want average latency (not percentiles), divide `_sum` by `_count`:

```promql
rate(http_request_duration_seconds_sum{service="visa-direct-cards"}[5m])
/
rate(http_request_duration_seconds_count{service="visa-direct-cards"}[5m])
```

> 🟢 **WHY THIS MATTERS:** Average latency hides problems. If 99% of requests are 10ms and 1% are 30 seconds, your average is ~300ms which looks fine. But 1% of users are suffering. Always monitor P99 alongside average.

### 7.4 Histogram vs Summary

| Feature | Histogram | Summary |
|---------|-----------|---------|
| Aggregatable across instances? | **YES** (use histogram_quantile) | **NO** (pre-calculated on client) |
| Bucket configuration | Server-side, defined at instrumentation | Client-side quantiles |
| Recommended for APIs? | **YES — always prefer histograms** | Use only if library forces it |
| Thanos compatible? | **YES — works perfectly with global queries** | Limited — cannot merge quantiles |

---

## Chapter 8: Binary Operators & Joins

### 8.1 Arithmetic Operators

You can do math between two queries. Prometheus will automatically match them by labels:

| Operator | Meaning | Example |
|----------|---------|---------|
| `+` | Addition | `metric_a + metric_b` |
| `-` | Subtraction | `total - errors` |
| `*` | Multiplication | `rate * 100` (for percentages) |
| `/` | Division | `errors / total` (error ratio) |
| `%` | Modulo | `counter % 1000` |
| `^` | Exponentiation | `value ^ 2` |

### 8.2 Label Matching with on() and ignoring()

When two sides of an operation have different labels, you need to tell Prometheus how to match them:

**on() = match ONLY on these labels**
```promql
# Error ratio per endpoint
# Left side has: {path, method, status_code}
# Right side has: {path, method, status_code}
# Match only on path:

sum by (path) (rate(http_requests_total{status_code=~"5.."}[5m]))
/
sum by (path) (rate(http_requests_total[5m]))
```

**ignoring() = match on all labels EXCEPT these**
```promql
# When one side has extra labels the other does not
sum by (path, status_code) (rate(http_requests_total{status_code=~"5.."}[5m]))
/ ignoring(status_code)
sum by (path) (rate(http_requests_total[5m]))
```

### 8.3 group_left and group_right (Many-to-One Joins)

When the left side has more labels than the right (many-to-one), you need `group_left`:

```promql
# Attach human-readable service names from a metadata metric
# service_info{service="visa-direct-cards", team="payments", tier="critical"} = 1

rate(http_requests_total[5m])
* on(service) group_left(team, tier)
service_info
```

This "joins" the `team` and `tier` labels from `service_info` onto your request rate. It works like a LEFT JOIN in SQL.

> 🔴 **COMMON MISTAKE:** Forgetting `group_left` when sides have different cardinality. Prometheus will say "many-to-one matching must be explicit". Fix: add `group_left()` or `group_right()`.

### 8.4 Comparison Operators

Filter results to only show series meeting a condition:

```promql
# Show only endpoints with error rate > 1%
(
  sum by (path) (rate(http_requests_total{status_code=~"5.."}[5m]))
  / sum by (path) (rate(http_requests_total[5m]))
) > 0.01

# Show only endpoints doing more than 100 req/sec
sum by (path) (rate(http_requests_total[5m])) > 100

# Use 'bool' modifier to return 0/1 instead of filtering
http_up == bool 1  # Returns 1 if up, 0 if not
```

---

## Chapter 9: Thanos — Querying Across Clusters

### 9.1 Why Thanos Exists

In a real production environment, you do not have one Prometheus server. You typically have one per Kubernetes cluster or data center. This creates a problem: how do you see a global view of your API traffic?

Thanos solves this by adding a query layer that sits on top of multiple Prometheus instances and makes them look like one big Prometheus. It also provides long-term storage (object storage like S3) and downsampling.

### 9.2 Thanos Architecture (What You Need to Know)

| Component | What It Does | Why You Care |
|-----------|-------------|--------------|
| **Thanos Sidecar** | Sits next to each Prometheus, uploads data to S3 | Your data survives pod restarts |
| **Thanos Store** | Reads historical data from S3 | Query data from weeks/months ago |
| **Thanos Query** | Merges data from all sidecars + stores | **THIS IS YOUR QUERY ENDPOINT** |
| **Thanos Compactor** | Downsamples old data (5m, 1h resolution) | Fast queries over long time ranges |
| **Thanos Ruler** | Runs recording rules and alerts globally | Cross-cluster alerting |

> 🟠 **SRE TIP:** When you open Grafana and point it at Thanos Query instead of a single Prometheus, your PromQL queries work exactly the same. The only difference is you now see data from ALL clusters.

### 9.3 External Labels — The Thanos Superpower

Each Prometheus instance is configured with external labels that identify where its data comes from:

```yaml
# In prometheus.yml on Cluster A:
global:
  external_labels:
    cluster: us-east-1
    environment: production
    region: us

# In prometheus.yml on Cluster B:
global:
  external_labels:
    cluster: eu-west-1
    environment: production
    region: eu
```

When Thanos merges these, every metric automatically gets the `cluster`, `environment`, and `region` labels. This is how you query across clusters:

### 9.4 Cross-Cluster PromQL Queries

**Query 1: Global request rate across all clusters**
```promql
sum(rate(http_requests_total{service="visa-direct-cards"}[5m]))
```
Because Thanos merges all Prometheus instances, this single query returns the worldwide total.

**Query 2: Request rate broken down by cluster**
```promql
sum by (cluster) (
  rate(http_requests_total{service="visa-direct-cards"}[5m])
)
```

**Query 3: Compare error rates across regions**
```promql
(
  sum by (region) (rate(http_requests_total{status_code=~"5..", service="visa-direct-cards"}[5m]))
  /
  sum by (region) (rate(http_requests_total{service="visa-direct-cards"}[5m]))
) * 100
```

**Query 4: Find which cluster has the highest P99 latency**
```promql
histogram_quantile(0.99,
  sum by (le, cluster) (
    rate(http_request_duration_seconds_bucket{service="visa-direct-cards"}[5m])
  )
)
```

**Query 5: Filter to a single region**
```promql
sum by (path) (
  rate(http_requests_total{
    service="visa-direct-cards",
    region="us"
  }[5m])
)
```

### 9.5 Thanos-Specific Query Considerations

#### Deduplication

If you have HA Prometheus pairs (two instances scraping the same targets), Thanos Query will return duplicate data. Enable deduplication:

```bash
# Thanos Query startup flag
--query.replica-label="prometheus_replica"

# In Grafana, enable "Dedup" in the Thanos data source settings
```

#### Partial Response

When one Prometheus sidecar is down, Thanos can either return partial results or fail the entire query. For dashboards, you usually want partial results:

```bash
# Enable in Thanos Query:
--query.partial-response

# Or per-query in the UI: check 'Enable Partial Response'
```

#### Step and Resolution

Thanos automatically uses downsampled data for long time ranges:

| Time Range | Resolution Used | Accuracy |
|-----------|----------------|----------|
| < 2 hours | Raw data (full resolution) | Exact values |
| 2h – 2 weeks | 5-minute downsampled | Slight smoothing |
| > 2 weeks | 1-hour downsampled | Good for trends, not incident forensics |

> 🔴 **COMMON MISTAKE:** Zooming into a 30-minute incident from 3 weeks ago and seeing smooth lines. Thanos is using 1h-downsampled data. If you need raw data from the past, you need to increase your Store retention.

### 9.6 Thanos Store API Queries

**Weekly traffic comparison**
```promql
# This week's traffic vs last week
sum(rate(http_requests_total{service="visa-direct-cards"}[1h]))
-
sum(rate(http_requests_total{service="visa-direct-cards"}[1h] offset 7d))
```

**30-day error budget burn rate**
```promql
# How fast are we burning our 99.9% SLO error budget?
1 - (
  sum(rate(http_requests_total{service="visa-direct-cards", status_code!~"5.."}[30d]))
  /
  sum(rate(http_requests_total{service="visa-direct-cards"}[30d]))
) / 0.001
```

---

## Chapter 10: Recording Rules & Alerts for APIs

### 10.1 Why Recording Rules?

Some PromQL queries are expensive — they scan millions of time series. Recording rules pre-compute them and store the result as a new metric. This makes dashboards load instantly and alerts evaluate fast.

### 10.2 Recording Rule Examples

```yaml
groups:
  - name: api_traffic
    interval: 30s
    rules:
      # Pre-compute request rate by service and path
      - record: job:http_requests:rate5m
        expr: sum by (service, path) (rate(http_requests_total[5m]))

      # Pre-compute error ratio by service
      - record: job:http_errors:ratio5m
        expr: |
          sum by (service) (rate(http_requests_total{status_code=~"5.."}[5m]))
          / sum by (service) (rate(http_requests_total[5m]))

      # Pre-compute P99 latency by service
      - record: job:http_latency_p99:5m
        expr: |
          histogram_quantile(0.99,
            sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

> 🟠 **SRE TIP:** Naming convention for recording rules: `level:metric:operations`. Example: `job:http_requests:rate5m` means it is aggregated at the job level, tracks http_requests, and uses rate over 5m.

### 10.3 Alert Rule Examples

**Alert: High Error Rate**
```yaml
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: job:http_errors:ratio5m > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.service }} error rate is {{ $value | humanizePercentage }}"
          description: "Error rate has been above 1% for 5 minutes."
```

**Alert: High P99 Latency**
```yaml
      - alert: HighP99Latency
        expr: job:http_latency_p99:5m > 2.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.service }} P99 latency is {{ $value | humanizeDuration }}"
```

**Alert: Traffic Drop (possible outage)**
```yaml
      - alert: TrafficDrop
        expr: |
          sum by (service) (rate(http_requests_total[5m]))
          < (sum by (service) (rate(http_requests_total[5m] offset 1h)) * 0.5)
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.service }} traffic dropped >50% vs 1 hour ago"
```

### 10.4 Thanos Ruler for Cross-Cluster Alerts

Thanos Ruler evaluates recording rules and alerts using the Thanos Query endpoint, giving you a global view:

```bash
# thanos-ruler deployment config
thanos rule \
  --query="thanos-query.monitoring.svc:9090" \
  --rule-file="/etc/thanos-ruler/*.yml" \
  --alert.query-url="http://thanos-query.monitoring.svc:9090"
```

This means your alerts fire based on GLOBAL data, not just one cluster. A regional outage affecting one cluster will be caught even if other clusters are healthy.

---

## Chapter 11: Performance Tuning & Debugging Slow Queries

### 11.1 Why Queries Get Slow

PromQL queries slow down when they touch too many time series. The number of unique time series is called **cardinality**. High cardinality is the #1 performance killer.

### 11.2 Check Your Cardinality

```promql
# How many time series does this metric have?
count(http_requests_total)

# What labels have the most unique values?
count by (__name__) ({__name__=~"http_.*"})

# Which label creates the most series?
count(http_requests_total) by (path)      # High? Too many unique paths
count(http_requests_total) by (client_id)  # High? Too many unique clients
```

### 11.3 Common Cardinality Problems

| Problem | Example | Fix |
|---------|---------|-----|
| Unbounded path label | path=/api/users/12345 (user IDs in paths) | Normalize paths: /api/users/:id |
| Request ID in labels | request_id=abc-123-def | NEVER put unique IDs in labels |
| Too many client IDs | 1000+ unique client_id values | Group into categories: internal/partner/public |
| Full URL in labels | url=https://full.url.with/params?foo=bar | Use path only, strip query params |

> 🟢 **WHY THIS MATTERS:** A metric with 10 labels that each have 10 possible values creates 10^10 = 10 billion potential time series. In reality not all combinations exist, but even a fraction of that will destroy your Prometheus.

### 11.4 Query Optimization Techniques

**Technique 1: Filter BEFORE aggregation**
```promql
# SLOW — aggregates ALL series, then filters
sum(rate(http_requests_total[5m])) > 100

# FAST — filters first, aggregates fewer series
sum(rate(http_requests_total{service="visa-direct-cards"}[5m])) > 100
```

**Technique 2: Use recording rules for complex queries**
```promql
# Instead of running this expensive query on every dashboard load:
histogram_quantile(0.99, sum by (le, path, cluster) (
  rate(http_request_duration_seconds_bucket[5m])
))

# Pre-compute it as a recording rule and query the result:
job:http_latency_p99_by_path_cluster:5m
```

**Technique 3: Avoid regex when possible**
```promql
# SLOW — regex requires scanning all series
{__name__=~"http_.*"}

# FAST — exact match uses the index
http_requests_total{service="visa-direct-cards"}
```

### 11.5 Thanos-Specific Performance Tips

| Tip | Why | How |
|-----|-----|-----|
| Use deduplication | HA Prometheus doubles query work | Set `--query.replica-label` |
| Enable query caching | Same dashboards hit same queries | Deploy Thanos Query Frontend with response cache |
| Limit time range | Long ranges scan more data | Default dashboards to 6h, not 7d |
| Use downsampled data | 1h resolution is fine for weekly views | Thanos does this automatically |
| Set max concurrent queries | Prevents one user from killing Thanos | `--query.max-concurrent=20` |

---

## Chapter 12: 50 Hands-On Exercises

Try each one in your Prometheus/Thanos UI before looking at the answer. Replace service names with your actual metric names.

### Beginner (1–15): Getting Comfortable

**1. Show all HTTP request metrics**
```promql
http_requests_total
```

**2. Filter to GET requests only**
```promql
http_requests_total{method="GET"}
```

**3. Show only 500 errors**
```promql
http_requests_total{status_code="500"}
```

**4. Show all 5xx errors (regex)**
```promql
http_requests_total{status_code=~"5.."}
```

**5. Exclude health check endpoints**
```promql
http_requests_total{path!~"/health|/ready"}
```

**6. Calculate request rate (5m window)**
```promql
rate(http_requests_total[5m])
```

**7. Total requests per second for one service**
```promql
sum(rate(http_requests_total{service="visa-direct-cards"}[5m]))
```

**8. Request rate broken down by method**
```promql
sum by (method) (rate(http_requests_total[5m]))
```

**9. Request rate broken down by path**
```promql
sum by (path) (rate(http_requests_total[5m]))
```

**10. Count how many time series exist**
```promql
count(http_requests_total)
```

**11. Show the top 5 busiest endpoints**
```promql
topk(5, sum by (path) (rate(http_requests_total[5m])))
```

**12. Show the 3 least busy endpoints**
```promql
bottomk(3, sum by (path) (rate(http_requests_total[5m])))
```

**13. Calculate increase in requests over 1 hour**
```promql
increase(http_requests_total[1h])
```

**14. Show only endpoints with >10 req/sec**
```promql
sum by (path) (rate(http_requests_total[5m])) > 10
```

**15. Check which targets are up/down**
```promql
up{job="visa-direct-cards"}
```

### Intermediate (16–35): Real-World Patterns

**16. Error rate as a percentage**
```promql
(sum(rate(http_requests_total{status_code=~"5.."}[5m]))/sum(rate(http_requests_total[5m])))*100
```

**17. Error rate per endpoint**
```promql
(sum by(path)(rate(http_requests_total{status_code=~"5.."}[5m]))/sum by(path)(rate(http_requests_total[5m])))*100
```

**18. Average request latency**
```promql
rate(http_request_duration_seconds_sum[5m])/rate(http_request_duration_seconds_count[5m])
```

**19. P50 (median) latency**
```promql
histogram_quantile(0.50, sum by(le)(rate(http_request_duration_seconds_bucket[5m])))
```

**20. P99 latency**
```promql
histogram_quantile(0.99, sum by(le)(rate(http_request_duration_seconds_bucket[5m])))
```

**21. P99 latency per endpoint**
```promql
histogram_quantile(0.99, sum by(le,path)(rate(http_request_duration_seconds_bucket[5m])))
```

**22. Request rate per pod/instance**
```promql
sum by(instance)(rate(http_requests_total{service="visa-direct-cards"}[5m]))
```

**23. Requests excluding write operations**
```promql
sum(rate(http_requests_total{method!~"POST|PUT|PATCH|DELETE"}[5m]))
```

**24. 4xx errors vs 5xx errors side by side**
```promql
sum by(status_code)(rate(http_requests_total{status_code=~"[45].."}[5m]))
```

**25. Traffic compared to 1 day ago (offset)**
```promql
sum(rate(http_requests_total[5m])) - sum(rate(http_requests_total[5m] offset 1d))
```

**26. Percentage of traffic that is POST**
```promql
sum(rate(http_requests_total{method="POST"}[5m]))/sum(rate(http_requests_total[5m]))*100
```

**27. Max latency across all pods**
```promql
max(rate(http_request_duration_seconds_sum[5m])/rate(http_request_duration_seconds_count[5m]))
```

**28. Number of pods serving traffic**
```promql
count(sum by(instance)(rate(http_requests_total[5m]) > 0))
```

**29. Request rate grouped by API version**
```promql
sum by(api_version)(rate(http_requests_total[5m]))
```

**30. Endpoints with error rate above 5%**
```promql
(sum by(path)(rate(http_requests_total{status_code=~"5.."}[5m]))/sum by(path)(rate(http_requests_total[5m])))*100 > 5
```

**31. Total 503 errors in the last hour**
```promql
sum(increase(http_requests_total{status_code="503"}[1h]))
```

**32. Rate of change of active connections**
```promql
deriv(http_active_connections[5m])
```

**33. Smoothed request rate (15m window)**
```promql
sum(rate(http_requests_total{service="visa-direct-cards"}[15m]))
```

**34. Multi-metric discovery for your service**
```promql
{__name__=~"http_.*", service="visa-direct-cards"}
```

**35. Requests by client_id, top 10**
```promql
topk(10, sum by(client_id)(rate(http_requests_total[5m])))
```

### Advanced (36–50): Thanos, SLOs & Complex Queries

**36. Global request rate across all clusters (Thanos)**
```promql
sum(rate(http_requests_total{service="visa-direct-cards"}[5m]))
```

**37. Request rate by cluster (Thanos)**
```promql
sum by(cluster)(rate(http_requests_total{service="visa-direct-cards"}[5m]))
```

**38. Compare error rates across regions (Thanos)**
```promql
(sum by(region)(rate(http_requests_total{status_code=~"5.."}[5m]))/sum by(region)(rate(http_requests_total[5m])))*100
```

**39. P99 latency per cluster (Thanos)**
```promql
histogram_quantile(0.99, sum by(le,cluster)(rate(http_request_duration_seconds_bucket[5m])))
```

**40. Traffic for a single region (Thanos)**
```promql
sum by(path)(rate(http_requests_total{service="visa-direct-cards",region="us"}[5m]))
```

**41. SLO: 99.9% availability (30-day)**
```promql
sum(rate(http_requests_total{status_code!~"5.."}[30d]))/sum(rate(http_requests_total[30d]))
```

**42. Error budget remaining (30-day)**
```promql
(1-(sum(rate(http_requests_total{status_code=~"5.."}[30d]))/sum(rate(http_requests_total[30d]))))/0.001*100
```

**43. Multi-window burn rate alert**
```promql
sum(rate(http_requests_total{status_code=~"5.."}[1h]))/sum(rate(http_requests_total[1h])) > 14.4*0.001
```

**44. Apdex score (target 0.5s, tolerable 2s)**
```promql
(sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))+sum(rate(http_request_duration_seconds_bucket{le="2"}[5m])))/2/sum(rate(http_request_duration_seconds_count[5m]))
```

**45. Week-over-week traffic growth**
```promql
(sum(rate(http_requests_total[1h]))-sum(rate(http_requests_total[1h] offset 7d)))/sum(rate(http_requests_total[1h] offset 7d))*100
```

**46. Predict disk usage in 4 hours**
```promql
predict_linear(prometheus_tsdb_storage_bytes[1h], 4*3600)
```

**47. Quantile across instances (not histogram)**
```promql
quantile(0.95, sum by(instance)(rate(http_requests_total[5m])))
```

**48. Absent alert (no traffic = outage)**
```promql
absent(sum(rate(http_requests_total{service="visa-direct-cards"}[5m])))
```

**49. Label replace for dashboard grouping**
```promql
label_replace(rate(http_requests_total[5m]),"method_group","write","method","POST|PUT|PATCH|DELETE")
```

**50. Full RED dashboard query set**
```promql
# R: sum by(path)(rate(http_requests_total[5m]))
# E: (sum(rate(http_requests_total{status_code=~"5.."}[5m]))/sum(rate(http_requests_total[5m])))*100
# D: histogram_quantile(0.99, sum by(le)(rate(http_request_duration_seconds_bucket[5m])))
```

---

> *Remember: The best way to learn PromQL is to type queries and see what happens. **Break things. Get errors. Read the error messages. Try again.***
