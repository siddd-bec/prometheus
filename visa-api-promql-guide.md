# Visa API Error Codes — PromQL Monitoring & Alerting Guide
## Visa Direct | VDA | VDW | Cards APIs
*Ready-to-Use Queries for Prometheus & Thanos*

---

This guide maps every Visa Developer Platform error code to actionable PromQL queries you can paste into Grafana, Thanos, or Prometheus. Each error category includes monitoring patterns, alert rules, and SRE playbook guidance.

**Sources:**
- developer.visa.com/pages/visa-developer-error-codes
- developer.visa.com/capabilities/visa_direct/docs-error-codes
- developer.visa.com/capabilities/visa-direct-account-and-wallet/docs-error-codes

---

## Table of Contents
1. [Visa API Error Code Categories & What They Mean](#part-1-visa-api-error-code-categories--what-they-mean)
2. [Label Strategy — How to Tag Visa Error Codes in Prometheus](#part-2-label-strategy--how-to-tag-visa-error-codes-in-prometheus)
3. [VDP Platform Error Codes (9xxx)](#part-3-vdp-platform-error-codes-9xxx--queries--alerts)
4. [Visa Direct Error Codes](#part-4-visa-direct-error-codes--queries--alerts)
5. [VDA/VDW Error Codes](#part-5-vdavdw-error-codes--queries--alerts)
6. [Reject/Return Codes (RE-series)](#part-6-rejectreturn-codes-re-series--queries--alerts)
7. [Complete RED Dashboard for Visa APIs](#part-7-complete-red-dashboard-for-visa-apis)
8. [Thanos Cross-Cluster Visa API Monitoring](#part-8-thanos-cross-cluster-visa-api-monitoring)
9. [Recording Rules & Alert Playbook](#part-9-recording-rules--alert-playbook)
10. [30 Ready-to-Paste Exercises](#part-10-30-ready-to-paste-exercises)

---

## Part 1: Visa API Error Code Categories & What They Mean

Visa APIs return errors in a layered system. Understanding which layer an error comes from is the first step to building effective monitoring. There are four distinct layers:

### 1.1 The Four Error Layers

| Layer | HTTP Codes | Error Code Range | Who Owns It |
|-------|-----------|-----------------|-------------|
| **VDP Platform (Gateway)** | 400, 401, 403, 404, 405, 415, 429 | 9xxx series (9000-9615) | Visa Developer Platform infrastructure |
| **Visa Direct (Cards)** | 200, 202, 303, 400, 401, 403, 404, 500, 503, 504 | 1001, 2001, 3001, 8001 | Visa Direct card processing |
| **VDA/VDW (Account & Wallet)** | 200, 202, 400, 401, 403, 404, 429, 500, 503, 504 | 4xxx-53xxx, 100xxx, 200xxx | Visa Direct Account & Wallet |
| **Reject/Return (RE-series)** | Within 200 callback or query | RE302-RE835 | Recipient bank/partner rejection |

> 🟢 **WHY THIS MATTERS:** A 400 from the VDP gateway (code 9000) means your request never reached Visa Direct. A 400 from Visa Direct (code 3001) means it reached the processing engine but had bad data. Your runbook response is completely different.

### 1.2 HTTP Status Code Quick Reference

These HTTP status codes are shared across all Visa APIs:

| HTTP Code | Meaning | Retry? | SRE Action |
|-----------|---------|--------|------------|
| **200** | Success — check actionCode in body for approve/decline | No | Monitor for declines in actionCode |
| **202** | Accepted — still processing (timeout reached) | Use GET with statusIdentifier | Alert if 202 rate spikes (indicates slowness) |
| **303** | Duplicate detected — original still processing | No — use GET URL | Monitor duplicate rate for client issues |
| **400** | Bad request — check error code for specific reason | No — fix payload first | Alert on spike; check error code breakdown |
| **401** | Authentication failed | No — fix credentials | CRITICAL alert — certs may have expired |
| **403** | Forbidden — no access to this API | No | Check project permissions |
| **404** | URL/resource not found | No — fix URL | Alert if new deployment changed routes |
| **429** | Rate limited | Yes — with backoff | Alert; check if traffic spike or config issue |
| **500** | Visa internal server error | Contact Visa support | CRITICAL alert; escalate to Visa |
| **503** | Service unavailable (overload/maintenance) | Yes — with backoff | CRITICAL alert; check Visa status page |
| **504** | Gateway timeout (network issue) | **DO NOT re-POST** — check settlement | CRITICAL alert; DO NOT retry transactions |

> 🔴 **CRITICAL SRE RULE:** For 503 and 504 on Visa Direct: **NEVER re-post the transaction.** It may have been processed. Check the settlement report first. This is in Visa's official documentation.

---

## Part 2: Label Strategy — How to Tag Visa Error Codes in Prometheus

Before you can query Visa errors in PromQL, your application needs to expose them as metrics with the right labels.

### 2.1 Recommended Metric Names

| Metric Name | Type | Purpose |
|------------|------|---------|
| `visa_api_requests_total` | Counter | Total API calls to Visa (all statuses) |
| `visa_api_errors_total` | Counter | Failed API calls (non-2xx or business errors) |
| `visa_api_request_duration_seconds` | Histogram | End-to-end latency to Visa |
| `visa_api_timeouts_total` | Counter | Calls that hit 202/504 timeout |
| `visa_api_retries_total` | Counter | Retry attempts to Visa |
| `visa_api_rejects_total` | Counter | RE-series reject/return from recipient banks |

### 2.2 Recommended Labels

| Label | Values | Why |
|-------|--------|-----|
| `api` | visa_direct, vda, vdw, vces | Which Visa API product |
| `operation` | aft, oct, query, send_payout, validate_payout, cancel_payout | Which API operation (endpoint) |
| `http_status` | 200, 202, 400, 401, 403, 500, 503, 504 | HTTP response code from Visa |
| `error_code` | 3001, 8001, 9125, 11000, 44060, etc. | Visa-specific error code |
| `error_category` | validation, auth, rate_limit, duplicate, timeout, server_error | Grouped category for dashboards |
| `environment` | sandbox, production | Which Visa environment |
| `cluster` | us-east-1, eu-west-1 (via Thanos external labels) | Which cluster made the call |
| `reject_code` | RE302, RE358, RE411, etc. | Reject/Return code from recipient bank |

### 2.3 Example Metric Exposition

Here is what your application should expose at `/metrics`:

```
# HELP visa_api_requests_total Total Visa API requests
# TYPE visa_api_requests_total counter
visa_api_requests_total{api="visa_direct",operation="aft",http_status="200",error_code="",error_category="success"} 145023
visa_api_requests_total{api="visa_direct",operation="aft",http_status="400",error_code="3001",error_category="validation"} 42
visa_api_requests_total{api="vda",operation="send_payout",http_status="400",error_code="11000",error_category="validation"} 15
visa_api_requests_total{api="vda",operation="send_payout",http_status="400",error_code="44060",error_category="velocity"} 3
visa_api_requests_total{api="visa_direct",operation="oct",http_status="401",error_code="9125",error_category="auth"} 1

# HELP visa_api_rejects_total Recipient bank rejections
# TYPE visa_api_rejects_total counter
visa_api_rejects_total{api="vda",operation="send_payout",reject_code="RE358",reject_type="reject"} 8
visa_api_rejects_total{api="vda",operation="send_payout",reject_code="RE411",reject_type="reject"} 12
```

> 🟢 **WHY THIS MATTERS:** Without the `error_code` label, you can only see that you got 400 errors. With it, you can instantly see that 80% of your 400s are code 11000 (missing mandatory field) vs code 3001 (invalid PAN). This changes your debugging from hours to minutes.

---

## Part 3: VDP Platform Error Codes (9xxx) — Queries & Alerts

These errors come from the Visa Developer Platform gateway **before** your request reaches any Visa product. They indicate infrastructure-level problems.

### 3.1 VDP Platform Error Code Reference

| Code | HTTP | Meaning | Category |
|------|------|---------|----------|
| 9000 | 400 | Invalid URI | routing |
| 9001 | 400 | Unsupported Media Type | payload |
| 9002 | 500 | Internal Server Error (VDP) | server_error |
| 9004 | 405 | Method Not Allowed | routing |
| 9005 | 404 | Route Not Found | routing |
| 9006 | 401 | Invalid API Key | auth |
| 9100 | 401 | AuthN Error (general) | auth |
| 9101 | 401 | Token Validation Failed | auth |
| 9123 | 401 | Wrong user credentials | auth |
| 9124 | 401 | Wrong user credentials | auth |
| 9125 | 401 | Client certificate mismatch | auth |
| 9611 | 403 | URL access not permitted | authz |
| 9615 | 400 | URL Not Readable | routing |

### 3.2 PromQL Queries for VDP Platform Errors

**Total VDP platform errors per minute**
```promql
sum(rate(visa_api_requests_total{error_code=~"9.*"}[5m])) * 60
```

**Auth errors breakdown (certificate vs key vs token)**
```promql
sum by (error_code) (
  rate(visa_api_requests_total{error_code=~"9006|9100|9101|9123|9124|9125"}[5m])
)
```

**Routing errors (wrong URL/method)**
```promql
sum by (error_code, operation) (
  rate(visa_api_requests_total{error_code=~"9000|9004|9005|9615"}[5m])
)
```

**Alert: Certificate expiry/mismatch (9125 spike)**
```yaml
# CRITICAL: Code 9125 means your TLS client certificate does not match
# This usually means a cert expired or was rotated incorrectly
- alert: VisaCertificateMismatch
  expr: sum(rate(visa_api_requests_total{error_code="9125"}[5m])) > 0
  for: 1m
  labels:
    severity: critical
    team: sre
  annotations:
    summary: "Visa API client certificate mismatch (code 9125)"
    runbook: "Check TLS cert expiry. Rotate cert via Visa Developer Portal."
```

**Alert: API key invalid (9006)**
```yaml
- alert: VisaAPIKeyInvalid
  expr: sum(rate(visa_api_requests_total{error_code="9006"}[5m])) > 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Visa API key is invalid or deactivated (code 9006)"
    runbook: "Verify API key in Vault. Check if key was rotated in VDP portal."
```

> 🟣 **ALERT RULE:** Any 9125 error should be an immediate page. It means ALL requests to Visa are failing because your certificate is wrong. This is a total outage for Visa-dependent services.

---

## Part 4: Visa Direct Error Codes — Queries & Alerts

These errors come from the Visa Direct card processing engine (AFT, OCT, Query operations). They indicate transaction-level issues.

### 4.1 Visa Direct Error Code Reference

| Code | HTTP | Meaning | Category |
|------|------|---------|----------|
| 3001 (400) | 400 | Message validation error / invalid PAN | validation |
| 8001 | 400 | Velocity limit exceeded | rate_limit |
| 1001 | 500 | Visa internal server error | server_error |
| 2001 | 500 | Visa internal server error | server_error |
| 597 | 400 | Consistency: multiple duplicate requests | duplicate |
| 600 | 400 | Consistency: account number mismatch in duplicate | duplicate |
| 601 | 400 | Consistency: amount mismatch in duplicate | duplicate |
| — | 202 | POST timed out; use GET with statusIdentifier | timeout |
| — | 303 | Duplicate detected; original still processing | duplicate |
| — | 504 | Gateway timeout; DO NOT re-POST | timeout |

### 4.2 PromQL Queries for Visa Direct

**Visa Direct success vs error ratio**
```promql
# Success ratio (should be > 99%)
sum(rate(visa_api_requests_total{api="visa_direct",http_status="200"}[5m]))
/
sum(rate(visa_api_requests_total{api="visa_direct"}[5m]))
* 100
```

**Validation errors by operation (AFT vs OCT)**
```promql
sum by (operation) (
  rate(visa_api_requests_total{api="visa_direct",error_code="3001"}[5m])
)
```

**Velocity limit hits (code 8001)**
```promql
# Track velocity limit breaches over time
sum(increase(visa_api_requests_total{api="visa_direct",error_code="8001"}[1h]))
```

**Timeout rate (202 + 504)**
```promql
# Timeouts indicate Visa processing delays or network issues
sum(rate(visa_api_requests_total{api="visa_direct",http_status=~"202|504"}[5m]))
/
sum(rate(visa_api_requests_total{api="visa_direct"}[5m]))
* 100
```

**Duplicate transaction rate (303 + consistency errors)**
```promql
sum(rate(visa_api_requests_total{
  api="visa_direct",
  error_code=~"597|600|601",
}[5m]))
+
sum(rate(visa_api_requests_total{api="visa_direct",http_status="303"}[5m]))
```

**Alert: Visa Direct server errors (500s from Visa)**
```yaml
- alert: VisaDirectServerError
  expr: |
    sum(rate(visa_api_requests_total{
      api="visa_direct",http_status="500"
    }[5m])) > 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Visa Direct returning 500 errors (codes 1001/2001)"
    runbook: "Contact Visa production support. DO NOT retry."
```

**Alert: Timeout spike (potential Visa slowdown)**
```yaml
- alert: VisaDirectTimeoutSpike
  expr: |
    (sum(rate(visa_api_requests_total{api="visa_direct",http_status="202"}[5m]))
    / sum(rate(visa_api_requests_total{api="visa_direct"}[5m])))
    > 0.05
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Visa Direct timeout rate >5%"
    runbook: "Check X-Transaction-Timeout-MS header. Use GET with statusIdentifier."
```

> 🔴 **CRITICAL SRE RULE:** When Visa Direct returns 504: **NEVER re-post the transaction.** The original may have been processed. Use GET with the statusIdentifier or check the settlement report. Re-posting can cause DUPLICATE PAYMENTS.

---

## Part 5: VDA/VDW Error Codes — Queries & Alerts

The Visa Direct Account and Wallet APIs have the most extensive error code system. These are grouped into functional categories to make monitoring practical.

### 5.1 Error Code Categories for VDA/VDW

| Category | Code Range / Pattern | Count | Example Codes |
|----------|---------------------|-------|---------------|
| Validation (mandatory/format) | 11000-11055 | ~5 | 11000, 11052, 11053, 11054, 11055 |
| Route/Corridor | 4003-4111, 44067, 100117-100118 | ~8 | 4003, 4004, 4111, 44067 |
| Recipient Bank Info | 11001-11510 | ~50 | 11002, 11057, 11162, 11364, 11507 |
| Recipient Account | 11300-11376, 100348-100534 | ~30 | 11300, 11301, 11360, 11368 |
| Recipient Identity | 4203-4227, 12003-12506 | ~20 | 4203, 4221, 12004, 12506 |
| Sender Validation | 41001-42851 | ~20 | 41001, 41005, 42620 |
| Velocity/Limits | 8001, 44060-44063, 53016-53022 | ~10 | 44060, 44061, 53016, 53022 |
| Duplicate/Consistency | 4700-4902, 5000-5300 | ~15 | 4900, 5102, 5103, 5300 |
| Transaction State | 4407-4604, 5201-5301 | ~10 | 4407, 4600, 4603, 5301 |
| Wallet Specific | 44029-44038, 200008-200012 | ~10 | 44029, 44031, 200008 |
| Compliance | 44064, 53015 | ~3 | 44064, 53015 |
| Configuration | 44070, 100888, 100906 | ~3 | 44070, 100888, 100906 |
| Partner/MFS Errors | 52010-53023 | ~15 | 52010, 53008, 53011, 53014 |

### 5.2 PromQL Queries for VDA/VDW

**Top 10 error codes in the last hour**
```promql
topk(10, 
  sum by (error_code) (
    increase(visa_api_requests_total{api="vda",http_status="400"}[1h])
  )
)
```

**Validation errors rate (missing/invalid fields)**
```promql
# These are the most common errors — usually client-side bugs
sum by (operation, error_code) (
  rate(visa_api_requests_total{
    api="vda",
    error_code=~"11000|11052|11053|11054|11055"
  }[5m])
)
```

**Recipient bank account errors (account closed/blocked/not found)**
```promql
# These indicate issues with the recipient's bank account
sum by (error_code) (
  rate(visa_api_requests_total{
    api="vda",
    error_code=~"11300|11301|11302|4102|4404"
  }[5m])
)
# 11300 = account not found
# 11301 = account closed/dormant/blocked
# 11302 = bank code/BIC incorrect
# 4102  = bank account not supported
# 4404  = bank account inactive
```

**Velocity limit breaches by direction**
```promql
# Sender velocity limits
sum(rate(visa_api_requests_total{
  api="vda",
  error_code=~"44060|44062|53016|53018|53020"
}[5m])) 

# Recipient velocity limits
sum(rate(visa_api_requests_total{
  api="vda",
  error_code=~"44061|44063|53017|53019|53021"
}[5m]))
```

**Duplicate transaction detection**
```promql
sum by (error_code) (
  rate(visa_api_requests_total{
    api="vda",
    error_code=~"4900|4901|4902|5102|5103|5300|100892"
  }[5m])
)
# 4900 = duplicate, original in ERROR
# 4901 = duplicate, original in undefined state
# 5102 = duplicate transaction
# 5103 = same clientRefID, different data
# 100892 = duplicate transaction
```

**Route/corridor errors (unsupported destinations)**
```promql
sum by (error_code) (
  rate(visa_api_requests_total{
    api="vda",
    error_code=~"4003|4004|4111|44067|100117|100118"
  }[5m])
)
```

**Wallet-specific failures**
```promql
sum by (error_code) (
  rate(visa_api_requests_total{
    api="vdw",
    error_code=~"44029|44030|44031|44037|44038|200008|200009|200012"
  }[5m])
)
# 44029 = wallet account not found
# 44030 = details mismatch at wallet operator
# 44031 = wallet account unavailable
```

### 5.3 Key VDA/VDW Alerts

**Alert: High validation error rate (client-side bugs)**
```yaml
- alert: VDAHighValidationErrors
  expr: |
    (sum(rate(visa_api_requests_total{api="vda",error_category="validation"}[5m]))
    / sum(rate(visa_api_requests_total{api="vda"}[5m])))
    > 0.10
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "VDA validation error rate >10%"
    runbook: "Check top error_codes. Likely 11000 (missing field) or 11054 (bad format)."
```

**Alert: Duplicate transaction spike**
```yaml
- alert: VDADuplicateSpike
  expr: |
    sum(rate(visa_api_requests_total{
      api="vda",error_code=~"4900|4901|5102|5103|100892"
    }[5m])) > 1
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "VDA duplicate transaction rate elevated"
    runbook: "Check idempotency. Verify clientReferenceId uniqueness. Review retry logic."
```

**Alert: Compliance rejection (immediate investigation)**
```yaml
- alert: VDAComplianceRejection
  expr: sum(rate(visa_api_requests_total{api="vda",error_code="44064"}[5m])) > 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Visa compliance rejection (code 44064)"
    runbook: "Contact Visa representative immediately. Transaction blocked for compliance."
```

---

## Part 6: Reject/Return Codes (RE-series) — Queries & Alerts

RE-series codes come from the **recipient's bank or partner**, not from Visa. A Reject means the bank refused the transaction before processing; a Return means the bank processed it but is reversing it. Both require different handling.

### 6.1 Most Common Reject/Return Codes

| Code | Type | Meaning | SRE Category |
|------|------|---------|-------------|
| RE302 | Reject | Additional KYC info required | kyc |
| RE358 | Reject | Recipient bank account is closed | account_issue |
| RE361 | Reject | Account name doesn't match account number | account_issue |
| RE365 | Reject | Recipient bank cannot process (general) | bank_issue |
| RE380 | Reject | Bank doesn't support this transaction type | bank_issue |
| RE381 | Reject | Account blocked | account_issue |
| RE386 | Reject | Regulatory reasons | compliance |
| RE411 | Reject | Account cannot be located / doesn't exist | account_issue |
| RE414 | Reject | Compliance rejection | compliance |
| RE429 | Reject | Account exceeded daily limit | velocity |
| RE525 | Reject | Processing interruption/timeout at bank | bank_issue |
| RE327 | Return | Account is closed | account_issue |
| RE412 | Return | Account cannot be located / doesn't exist | account_issue |
| RE478 | Return | Compliance return | compliance |

### 6.2 PromQL Queries for RE-series

**Reject vs Return ratio**
```promql
# Reject rate
sum(rate(visa_api_rejects_total{reject_type="reject"}[5m]))

# Return rate
sum(rate(visa_api_rejects_total{reject_type="return"}[5m]))

# Total rejection rate as % of all payouts
(sum(rate(visa_api_rejects_total[5m]))
/ sum(rate(visa_api_requests_total{api="vda",operation="send_payout",http_status="200"}[5m])))
* 100
```

**Top rejection reasons**
```promql
topk(10, sum by (reject_code) (increase(visa_api_rejects_total[1h])))
```

**Account-related rejections (closed, blocked, not found)**
```promql
sum by (reject_code) (
  rate(visa_api_rejects_total{
    reject_code=~"RE358|RE381|RE411|RE327|RE412|RE470|RE471"
  }[5m])
)
```

**Compliance rejections (need immediate investigation)**
```promql
sum(rate(visa_api_rejects_total{
  reject_code=~"RE386|RE414|RE452|RE473|RE478|RE602"
}[5m]))
```

**Alert: Rejection rate spike**
```yaml
- alert: VDARejectRateHigh
  expr: |
    (sum(rate(visa_api_rejects_total[5m]))
    / sum(rate(visa_api_requests_total{api="vda",operation="send_payout"}[5m])))
    > 0.10
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "VDA payout rejection rate >10%"
    runbook: "Check top reject_codes. If RE411 dominant, check recipient validation flow."
```

> 🟢 **WHY THIS MATTERS:** A spike in RE358 (account closed) is normal business. But a spike in RE525 (bank timeout) across multiple recipient banks suggests a network issue at the partner level. Your alert thresholds should be different for each category.

---

## Part 7: Complete RED Dashboard for Visa APIs

Copy these queries directly into Grafana panels. Replace `$api` and `$operation` with Grafana template variables.

### 7.1 Rate Panels

**Total Visa API Throughput**
```promql
sum by (api) (rate(visa_api_requests_total[5m]))
```

**Throughput by Operation**
```promql
sum by (api, operation) (rate(visa_api_requests_total{api="$api"}[5m]))
```

**Success vs Failure Rate**
```promql
# Success (green line)
sum(rate(visa_api_requests_total{api="$api",http_status="200"}[5m]))

# Failure (red line)
sum(rate(visa_api_requests_total{api="$api",http_status!="200"}[5m]))
```

### 7.2 Error Panels

**Error Rate Percentage**
```promql
(
  sum(rate(visa_api_requests_total{api="$api",http_status!~"200|202"}[5m]))
  / sum(rate(visa_api_requests_total{api="$api"}[5m]))
) * 100
```

**Error Breakdown by Category**
```promql
sum by (error_category) (
  rate(visa_api_requests_total{api="$api",http_status!="200"}[5m])
)
```

**Top 10 Error Codes (table)**
```promql
topk(10, sum by (error_code) (
  increase(visa_api_requests_total{api="$api",http_status="400"}[$__range])
))
```

**Visa Server Errors (500/503/504)**
```promql
sum by (http_status) (
  rate(visa_api_requests_total{api="$api",http_status=~"5.."}[5m])
)
```

### 7.3 Duration Panels

**P50 / P95 / P99 Latency to Visa**
```promql
# P50
histogram_quantile(0.50, sum by (le) (
  rate(visa_api_request_duration_seconds_bucket{api="$api"}[5m])
))

# P95
histogram_quantile(0.95, sum by (le) (
  rate(visa_api_request_duration_seconds_bucket{api="$api"}[5m])
))

# P99
histogram_quantile(0.99, sum by (le) (
  rate(visa_api_request_duration_seconds_bucket{api="$api"}[5m])
))
```

**Latency by Operation**
```promql
histogram_quantile(0.99, sum by (le, operation) (
  rate(visa_api_request_duration_seconds_bucket{api="$api"}[5m])
))
```

**Timeout Rate (approaching 30s limit)**
```promql
# X-Transaction-Timeout-MS defaults to 30000ms
# Track requests approaching that threshold
sum(rate(visa_api_request_duration_seconds_bucket{api="$api",le="25"}[5m]))
/ sum(rate(visa_api_request_duration_seconds_count{api="$api"}[5m]))
```

### 7.4 Visa-Specific Panels

**Rejection Funnel**
```promql
# Total Sent -> Accepted -> Rejected -> Returned

# Sent
sum(rate(visa_api_requests_total{api="vda",operation="send_payout"}[5m]))

# Successfully accepted
sum(rate(visa_api_requests_total{api="vda",operation="send_payout",http_status="200"}[5m]))

# Rejected by recipient bank
sum(rate(visa_api_rejects_total{reject_type="reject"}[5m]))

# Returned by recipient bank
sum(rate(visa_api_rejects_total{reject_type="return"}[5m]))
```

---

## Part 8: Thanos Cross-Cluster Visa API Monitoring

When your Visa Direct integrations run across multiple clusters (us-east, eu-west, apac), Thanos gives you the global view.

### 8.1 Global View Queries

**Global Visa API throughput**
```promql
sum(rate(visa_api_requests_total[5m]))
```

**Visa API throughput by cluster**
```promql
sum by (cluster) (rate(visa_api_requests_total[5m]))
```

**Error rate comparison across regions**
```promql
(
  sum by (region) (rate(visa_api_requests_total{http_status!~"200|202"}[5m]))
  / sum by (region) (rate(visa_api_requests_total[5m]))
) * 100
```

**Which cluster has highest 400 error rate?**
```promql
topk(3, 
  sum by (cluster) (rate(visa_api_requests_total{http_status="400"}[5m]))
  / sum by (cluster) (rate(visa_api_requests_total[5m]))
  * 100
)
```

### 8.2 Cross-Cluster Error Analysis

**Auth failures by cluster (certificate issues per region)**
```promql
sum by (cluster, error_code) (
  rate(visa_api_requests_total{error_code=~"9125|9006|9100"}[5m])
)
```
If 9125 appears in only one cluster, that cluster's TLS certificate was likely rotated incorrectly.

**P99 Latency to Visa by cluster**
```promql
histogram_quantile(0.99, sum by (le, cluster) (
  rate(visa_api_request_duration_seconds_bucket[5m])
))
```
Higher latency in eu-west vs us-east is expected due to geographic distance to Visa endpoints. Set different thresholds per region.

**Global rejection rate trend (30-day via Thanos Store)**
```promql
# Uses Thanos long-term storage for trend analysis
(sum(rate(visa_api_rejects_total[1d]))
/ sum(rate(visa_api_requests_total{api="vda",operation="send_payout",http_status="200"}[1d])))
* 100
```

### 8.3 Thanos Alert for Regional Visa Outage

```yaml
- alert: VisaRegionalOutage
  expr: |
    sum by (region) (rate(visa_api_requests_total{http_status=~"5.."}[5m]))
    / sum by (region) (rate(visa_api_requests_total[5m]))
    > 0.50
  for: 3m
  labels:
    severity: critical
  annotations:
    summary: "Visa API >50% server errors in {{ $labels.region }}"
    runbook: "Check if Visa has a regional outage. Failover traffic if possible."
```

---

## Part 9: Recording Rules & Alert Playbook

### 9.1 Recording Rules for Visa API Metrics

```yaml
groups:
  - name: visa_api_recording_rules
    interval: 30s
    rules:
      # Pre-compute total request rate by API and operation
      - record: job:visa_api_requests:rate5m
        expr: sum by (api, operation) (rate(visa_api_requests_total[5m]))

      # Pre-compute error ratio by API
      - record: job:visa_api_error_ratio:5m
        expr: |
          sum by (api) (rate(visa_api_requests_total{http_status!~"200|202"}[5m]))
          / sum by (api) (rate(visa_api_requests_total[5m]))

      # Pre-compute top error codes (avoids expensive topk on dashboards)
      - record: job:visa_api_errors_by_code:rate5m
        expr: sum by (api, error_code) (rate(visa_api_requests_total{http_status="400"}[5m]))

      # Pre-compute P99 latency by API
      - record: job:visa_api_latency_p99:5m
        expr: |
          histogram_quantile(0.99,
            sum by (le, api) (rate(visa_api_request_duration_seconds_bucket[5m]))
          )

      # Pre-compute rejection rate
      - record: job:visa_api_reject_ratio:5m
        expr: |
          sum(rate(visa_api_rejects_total[5m]))
          / sum(rate(visa_api_requests_total{api="vda",operation="send_payout",http_status="200"}[5m]))
```

### 9.2 Complete Alert Playbook

| Alert | Condition | Severity | First Response |
|-------|-----------|----------|---------------|
| **VisaCertMismatch** | error_code 9125 > 0 | P1 Critical | Check TLS cert expiry in Vault. Rotate via VDP portal. |
| **VisaAPIKeyInvalid** | error_code 9006 > 0 | P1 Critical | Verify API key. Check if deactivated in VDP portal. |
| **VisaServerError** | http_status 500 > 0 for 2m | P1 Critical | Contact Visa support. DO NOT retry transactions. |
| **VisaTimeoutSpike** | 202 rate > 5% for 5m | P2 Warning | Check Visa status. Adjust X-Transaction-Timeout-MS. |
| **VisaHighErrorRate** | error ratio > 5% for 5m | P2 Warning | Check topk error_codes. Usually validation (11000). |
| **VisaDuplicateSpike** | duplicate codes > 1/sec | P2 Warning | Review idempotency. Check clientReferenceId generation. |
| **VisaVelocityLimit** | error_code 8001 spike | P3 Info | Review rate limits. Contact Visa to increase if needed. |
| **VisaRejectionHigh** | reject ratio > 10% for 10m | P2 Warning | Check top reject_codes. If RE411: validation issue. |
| **VisaComplianceBlock** | error_code 44064 > 0 | P1 Critical | Contact Visa representative immediately. |
| **VisaRegionalOutage** | 5xx > 50% in one region | P1 Critical | Failover traffic. Check Visa status page. |

---

## Part 10: 30 Ready-to-Paste Exercises

Try each query in your Prometheus/Thanos UI. Replace `visa_api_requests_total` with your actual metric name if different.

### Beginner (1–10)

**1. Show all Visa API metrics**
```promql
visa_api_requests_total
```

**2. Filter to VDA send_payout only**
```promql
visa_api_requests_total{api="vda",operation="send_payout"}
```

**3. Show only 400 errors**
```promql
visa_api_requests_total{http_status="400"}
```

**4. Show all 5xx errors from Visa**
```promql
visa_api_requests_total{http_status=~"5.."}
```

**5. Total Visa API request rate**
```promql
sum(rate(visa_api_requests_total[5m]))
```

**6. Request rate by API product**
```promql
sum by (api) (rate(visa_api_requests_total[5m]))
```

**7. Request rate by operation**
```promql
sum by (api,operation) (rate(visa_api_requests_total[5m]))
```

**8. Count total 401 errors in last hour**
```promql
sum(increase(visa_api_requests_total{http_status="401"}[1h]))
```

**9. Check for any 9125 (cert mismatch) errors**
```promql
visa_api_requests_total{error_code="9125"}
```

**10. Top 5 error codes**
```promql
topk(5, sum by(error_code)(increase(visa_api_requests_total{http_status="400"}[1h])))
```

### Intermediate (11–20)

**11. Error rate % for Visa Direct**
```promql
(sum(rate(visa_api_requests_total{api="visa_direct",http_status!="200"}[5m]))/sum(rate(visa_api_requests_total{api="visa_direct"}[5m])))*100
```

**12. Validation error rate (11000-series)**
```promql
sum(rate(visa_api_requests_total{error_code=~"11.*"}[5m]))
```

**13. Timeout rate (202s) for Visa Direct**
```promql
(sum(rate(visa_api_requests_total{api="visa_direct",http_status="202"}[5m]))/sum(rate(visa_api_requests_total{api="visa_direct"}[5m])))*100
```

**14. Duplicate transaction count**
```promql
sum(increase(visa_api_requests_total{error_code=~"5102|5103|100892|4900"}[1h]))
```

**15. P99 latency to Visa by API**
```promql
histogram_quantile(0.99, sum by(le,api)(rate(visa_api_request_duration_seconds_bucket[5m])))
```

**16. Velocity limit hits per hour**
```promql
sum(increase(visa_api_requests_total{error_code=~"8001|44060|44061|53016|53017"}[1h]))
```

**17. Auth errors breakdown**
```promql
sum by(error_code)(rate(visa_api_requests_total{error_code=~"9006|9100|9101|9125"}[5m]))
```

**18. Route/corridor errors**
```promql
sum by(error_code)(rate(visa_api_requests_total{error_code=~"4003|4004|4111|44067"}[5m]))
```

**19. Recipient bank account issues**
```promql
sum by(error_code)(rate(visa_api_requests_total{error_code=~"11300|11301|11302|4102|4404"}[5m]))
```

**20. Compare error rates: VDA vs Visa Direct**
```promql
sum by(api)((sum by(api)(rate(visa_api_requests_total{http_status="400"}[5m]))/sum by(api)(rate(visa_api_requests_total[5m])))*100)
```

### Advanced (21–30): Thanos, SLOs & Complex Queries

**21. Global Visa throughput across all clusters**
```promql
sum(rate(visa_api_requests_total[5m]))
```

**22. Error rate by cluster**
```promql
(sum by(cluster)(rate(visa_api_requests_total{http_status!="200"}[5m]))/sum by(cluster)(rate(visa_api_requests_total[5m])))*100
```

**23. Which cluster has cert errors?**
```promql
sum by(cluster)(rate(visa_api_requests_total{error_code="9125"}[5m]))
```

**24. Top rejection codes globally**
```promql
topk(10, sum by(reject_code)(increase(visa_api_rejects_total[1h])))
```

**25. Reject vs Return ratio**
```promql
sum by(reject_type)(rate(visa_api_rejects_total[5m]))
```

**26. Compliance rejections across regions**
```promql
sum by(region)(rate(visa_api_rejects_total{reject_code=~"RE386|RE414|RE452|RE478"}[5m]))
```

**27. Week-over-week Visa error trend**
```promql
sum(rate(visa_api_requests_total{http_status="400"}[1h]))-sum(rate(visa_api_requests_total{http_status="400"}[1h] offset 7d))
```

**28. SLO: 99.9% Visa API availability (30d)**
```promql
sum(rate(visa_api_requests_total{http_status=~"200|202"}[30d]))/sum(rate(visa_api_requests_total[30d]))
```

**29. P99 latency by cluster (Thanos)**
```promql
histogram_quantile(0.99, sum by(le,cluster)(rate(visa_api_request_duration_seconds_bucket[5m])))
```

**30. Absent check: no Visa traffic = outage**
```promql
absent(sum(rate(visa_api_requests_total{api="visa_direct"}[5m])))
```

---

> *Every error code is a signal. Build dashboards that turn signals into stories. **The best SRE is the one who knows what broke before the PagerDuty fires.***
