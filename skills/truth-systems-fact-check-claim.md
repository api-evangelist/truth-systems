---
name: Fact-check a claim against source documents with Gateway
description: >-
  Use the Truth Systems Gateway API to detect hallucinations in LLM output by
  judging a claim against supplied source documents and interpreting the
  verdict and evidence.
api: openapi/truth-systems-gateway-openapi-original.json
operations:
  - POST /api/get_context_output
generated: '2026-07-21'
method: generated
---

# Fact-check a claim with Gateway

Gateway is deployed inside the customer's own cloud account (AWS or Azure).
There is no hosted public endpoint: resolve the deployment-specific base
address first (AWS Step Function ARN, or Azure Function App URL + function
key). See `authentication/truth-systems-authentication.yml`.

## Steps

1. **Resolve the deployment endpoint.**
   - AWS: get the Step Function ARN (`state_machine_arn` Terraform output) and
     ensure the caller has `states:StartExecution` + `states:DescribeExecution`.
   - Azure: get the Function App URL and key (`terraform output function_app_default_key`).
2. **Build the request.** The single operation is `POST /api/get_context_output`
   (no operationId in the published spec; the overlay proposes
   `getContextOutput`). Body shape (`gatewayInput`):
   ```json
   {"params": {"claim": "<text to verify>",
               "sources": [{"id": "1", "text": "<source text>"}]}}
   ```
   Every source needs both `id` and `text` (only plain-text sources are
   supported). Optionally set an `X-Request-ID` header to correlate logs.
3. **Prefer the SDK for polling.** Execution is asynchronous in the deployment
   (Step Functions / Durable Functions); the Python SDK's `client.judge(claim,
   sources)` submits and polls to completion, returning a `Ruling`.
4. **Interpret the result** (`gatewayOutput` / SDK `Ruling`):
   - Overall verdict is conservative: `SUPPORTS` only if every statement is
     supported; `REFUTES` if any statement is refuted; else `NOT_ENOUGH_INFO`.
   - Per-statement: `sentences[].verdict` with `evidence.ids` pointing back at
     your source ids and `source_spans` giving character offsets of the
     evidence inside each source.
5. **Handle errors.** Failures raise `gateway.errors.APIError` in the SDK; the
   wire error envelope is `{error: <int>, message: <string>}` (see
   `errors/truth-systems-problem-types.yml`). There is no idempotency-key or
   rate-limit contract - throughput is bounded by the customer's own model
   deployment.
