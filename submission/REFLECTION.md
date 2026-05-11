# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Nguyễn Phan Tuấn Anh
**Submission date:** 2026-05-11
**Lab repo URL:** https://github.com/minerals/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.4.3)
Compose v2:    OK  (5.1.3)
RAM available: 7.76 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: /home/minerals/Day23-Track2-Observability-Lab/00-setup/setup-report.json
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/02_06panels.png`.

### Burn-rate panel

Drop `submission/screenshots/02_SLO_burnrate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | submission/screenshots/02_trigger_servicedown.png |
| _T0+90s_ | `ServiceDown` fired   | submission/screenshots/03_Slack_fire_resolve.png |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | submission/screenshots/03_Slack_fire_resolve.png |

### One thing surprised me about Prometheus / Grafana

The most surprising insight was how Prometheus's pull model combined with service discovery makes scaling observability trivial. Unlike push-based systems where each service needs to know about the collector, Prometheus scrapes metrics from configured targets — meaning services don't need any code changes to be monitored. This decoupling meant I could add the inference-api to monitoring with just a single scrape config entry, no restarts, no library changes. The downside I didn't anticipate was the cardinality explosion: adding a high-cardinality label like `user_id` to a heavily-hit endpoint could quickly bloat the TSDB. The prometheus.yml in this lab avoids this by using bounded labels (model, status) on the inference metrics.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/03_jaeger_trace_3_span.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 14, "quality": 0.771, "duration_seconds": 0.1575, "trace_id": "8122c3fcdd8bfef529878d57106f68d5", "langfuse_trace_id": "27c2c792-f3c9-4576-8a50-97e89b480c0b", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T16:19:17.620874Z"}
```

Trace ID: `8122c3fcdd8bfef529878d57106f68d5`

### Tail-sampling math

The otel-collector config (`03-tracing-and-logs/otel-collector/otel-config.yaml`) uses three policies:
- `keep-errors`: keeps ALL traces with ERROR status (100% sampling)
- `keep-slow`: keeps ALL traces with latency > 2000ms (100% sampling)
- `probabilistic-1pct`: keeps 50% of all other traces

If the service produced N traces/sec:
- Let E = error traces/sec, S = slow traces/sec (>2000ms), H = healthy/fast traces/sec
- N = E + S + H
- Traces kept = E + S + 0.5 * H
- Fraction kept = (E + S + 0.5H) / (E + S + H) = 0.5 + 0.5 * (E + S) / N

For example, if 1% of traces are errors and 5% are slow (E=0.01N, S=0.05N, H=0.94N):
- Fraction kept = 0.5 + 0.5 * (0.06) = 53%

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

| Feature | Test | Reason |
|---|---|---|
| `prompt_length` | **KS (Kolmogorov-Smirnov)** | Continuous numeric feature; KS tests for distribution shift in the entire CDF, ideal for detecting shifts in the range/median of prompt lengths |
| `embedding_norm` | **KS or PSI** | Continuous numeric; PSI is sensitive to binning but works well for near-Gaussian data like embedding norms |
| `response_length` | **PSI** | Continuous; PSI excels at detecting population stability changes, good for token counts |
| `response_quality` | **KL (Kullback-Leibler)** | Bounded [0,1] score; KL divergence directly measures how one probability distribution diverges from another, perfect for quality scores that have a known reference distribution |

In production, I'd use **PSI** as the default for all numeric features (it's industry standard for drift detection in finance/ML), with KL as a secondary check for bounded features like quality scores. KS is useful when you care about the distribution shape (e.g., detecting if prompt lengths have shifted from short to long), but it's sensitive to sample size. For categorical features, I'd add a Chi-squared test.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The hardest metric to expose would be **Day 20 (llama.cpp serving)**. Unlike Day 19's Qdrant which exposes a `/metrics` endpoint in Prometheus format natively, llama.cpp's native metrics are often minimal or require parsing stdout logs. The stub in `05-integration/` shows this challenge: you'd need to either run llama.cpp with the `--metrics-port` flag (if the build supports it) or wrap it with a sidecar that scrapes and re-exposes metrics. Additionally, model-serving workloads have unique observability needs: you want to track not just latency and throughput, but also GPU utilization, KV cache hit rates, and token generation speed — none of which are standard Prometheus metrics. This requires custom instrumentation or a specialized exporter.

---

## 6. The single change that mattered most

> **Grader reads this closest.** What one thing about your stack design — a metric you added, a label you dropped, a panel you reorganized, an alert threshold you tuned — made the biggest difference between "works" and "useful"? Write 1-2 paragraphs. Connect it to a concept from the deck.

The single most impactful change was **adding the `gen_ai.*` semantic convention labels to spans** in the `/predict` endpoint. Following OpenTelemetry's LLM-specific semantic conventions (from deck §6), I added `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, and `gen_ai.response.finish_reason` to every trace. This transformed traces from generic "HTTP request took X ms" into actionable LLM observability.

This connects directly to the **RED + USE** framework from deck §2. The token counts are **USE** metrics (Utilization, Saturation, Errors) — they tell us if we're approaching model context limits or if batching would help. The model label enables **per-model SLOs** and cost tracking (deck §8). Without these labels, we'd see a flat latency distribution; with them, we can answer "which model is slowest?" and "how does latency scale with prompt length?". The finish_reason label also catches edge cases like length-truncated responses that would otherwise look like normal requests. This is the difference between monitoring (knowing something is broken) and observability (understanding why).
