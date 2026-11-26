# Issue Draft: Add Automatic Inference Endpoint Discovery for Lens-Based CI

## Title

Add automatic inference endpoint discovery for Lens-based CI

## Summary

The Lens CLI version currently used by CI does not automatically identify which Service URL is the model's inference endpoint. Its status output exposes a generic list of Service URLs, without classifying or selecting the inference endpoint. As a result, `.github/workflows/reusable-ci-nightly-benchmark.yaml` compensates by selecting `jq -er '.endpoints[0]'` and passing that URL to the benchmark. This index-based choice is not endpoint discovery: the first URL is not guaranteed to serve inference traffic.

Lens should automatically discover and identify the inference endpoint from the rendered stack resources, expose the selected endpoint and any discovery findings in status, and let CI consume that structured result. If no unambiguous inference endpoint can be discovered, CI should stop with actionable diagnostics rather than guess.

This is a follow-up to [llm-d-lens#148](https://github.com/llm-d-incubation/llm-d-lens/issues/148), the broader CI/CD deployment reliability request, and depends on the endpoint discovery contract in [llm-d-lens#143](https://github.com/llm-d-incubation/llm-d-lens/pull/143).

## Current Behavior

The current Lens status contract used by CI does not distinguish inference URLs from other Service URLs. In the Lens branch of the `Standup` step, the workflow therefore:

1. Runs `llmd -o json status` and saves the result.
2. Selects `.endpoints[0]` with `jq`.
3. Appends that URL as `--endpoint-url` for the benchmark run.

The legacy `endpoints` field contains Service URLs, but its order does not express inference priority or endpoint purpose. A namespace may expose multiple Services or multiple ports per Service, including inference, metrics, health, admin or gRPC endpoints. Until Lens provides automatic inference endpoint discovery and CI consumes it, the workflow relies on the incidental first entry.

## Potential Problems

- **Benchmark can target a non-inference port.** If a metrics, health or admin URL appears first, the harness may receive metrics text, an unrelated health response, or an HTTP error instead of model completions.
- **The workflow can benchmark the wrong backend.** With separate router and model Services, selecting the first URL may bypass the intended router or select a service that is not the supported client entry point.
- **Resource ordering can make runs flaky.** Adding a Service, changing rendered manifest order, or changing the guide can change which URL occupies index zero without changing the intended inference endpoint.
- **Failures can be misdiagnosed.** A bad URL may surface as a benchmark or inference failure even though the deployed model is healthy, obscuring the actual endpoint-selection problem.
- **Results may be invalid without an obvious failure.** A reachable non-inference endpoint can accept a connection but return data that does not represent model inference, contaminating benchmark results.

## Proposed Changes

- Lens status automatically discovers inference endpoints from the rendered stack resources and exposes the selected endpoint as `inferenceEndpoint.url`, with structured `endpointFindings` when discovery is missing or ambiguous.
- In Lens mode, CI reads `inferenceEndpoint.url` from `llmd -o json status`; it does not select `.endpoints[0]` or implement its own heuristic search.
- If `inferenceEndpoint` is null, missing or has no usable URL, stop before the benchmark run. Print the Lens `endpointFindings` alongside a clear error; do not guess or fall back to the legacy URL list.
- Keep the existing `llmdbenchmark` lifecycle and endpoint behavior unchanged for non-Lens callers.
- Ensure the Lens revision used by the workflow provides the `inferenceEndpoint` status field. For controlled testing, this can use the endpoint-discovery PR revision; use the supported merged or released revision for normal CI.

## Acceptance Criteria

- Lens status can automatically distinguish and return the inference endpoint for supported rendered stacks, independently of the position of other Service URLs.
- Lens-mode CI passes only a non-empty `inferenceEndpoint.url` to the benchmark and does not search the generic URL list itself.
- Missing or ambiguous endpoint discovery fails before benchmark execution and prints the relevant Lens findings.
- The workflow never falls back to `.endpoints[0]` in Lens mode.
- A test or workflow-level validation covers a status response where a non-inference Service/port precedes the inference endpoint, and confirms the inference URL is selected.
- A test or workflow-level validation covers a null/missing inference endpoint and confirms the benchmark is not started.
- Existing `llmdbenchmark` callers and the default deployment lifecycle remain unchanged.

## Scope

Update `.github/workflows/reusable-ci-nightly-benchmark.yaml` to use Lens's structured inference endpoint and add focused workflow validation if a suitable test surface exists. The Lens CLI revision selected by `lens_ref` must include automatic discovery and the `inferenceEndpoint` status field; that Lens-side capability is tracked by [llm-d-lens#143](https://github.com/llm-d-incubation/llm-d-lens/pull/143) and should not be reimplemented here. Do not change the `llmdbenchmark` implementation or its default lifecycle.

## References

- Parent feature: [llm-d-incubation/llm-d-lens#148](https://github.com/llm-d-incubation/llm-d-lens/issues/148)
- Lens endpoint discovery: [llm-d-incubation/llm-d-lens#143](https://github.com/llm-d-incubation/llm-d-lens/pull/143)
- Workflow: [reusable-ci-nightly-benchmark.yaml](https://github.com/llm-d/llm-d-infra/blob/main/.github/workflows/reusable-ci-nightly-benchmark.yaml#L1238)
