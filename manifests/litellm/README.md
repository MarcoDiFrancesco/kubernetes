# LiteLLM vs GCP Billing Discrepancy

There is a noticeable difference between the spend reported by LiteLLM Prometheus metrics (`litellm_spend_metric_total`) and actual GCP Billing (`gcp_billing_export_v1_*`).

### Token Counts Match Closely

The tokens processed through LiteLLM and recorded by Google Cloud match almost identically:

| Metric | Google Cloud Actual | LiteLLM Recorded |
| :--- | :--- | :--- |
| **Input Tokens (Flash)** | ~20.35M (17.16M uncached + 3.20M cached) | ~19.90M |
| **Output Tokens (Flash)** | 111,052 | 110,760 |

## Root Causes

1. **Model Pricing Discounts**: LiteLLM's built-in `model_cost_map` accounts for discounts not applied in GCP Vertex AI.
2. **Untracked Vertex AI Services**:
   - **Context Caching Storage**: Hourly fees for cache retention.
   - **Multimodal Inputs**: Image/video inputs billed under separate SKUs.

---

## Example Breakdown (2026-10-22)

| Category | Billed by GCP (USD) | Calculated by LiteLLM (USD) |
| :--- | :--- | :--- |
| **Gemini 3.8 Flash** (LiteLLM rate is 50% lower) | $26.57 | $13.41 |
| **Context Caching Storage** (not tracked by LiteLLM) | $1.32 | $0.00 |
| **Multimodal Inputs / Images** (not tracked by LiteLLM) | $1.27 | $0.00 |
| **Other Services** (Cloud Run, etc.) | $0.01 | $0.00 |
| **Total** | **$31.47 USD** | **~$15.00 USD** |
