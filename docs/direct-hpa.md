# Direct HPA Integration

`llm-d-benchmark` supports a **Direct HPA** autoscaling mode as a baseline
alternative to the [Workload Variant Autoscaler (WVA)](workload-variant-autoscaler.md).
In this mode, a plain Kubernetes `HorizontalPodAutoscaler` drives replica count
directly from EPP (Endpoint Policy Plugin) Prometheus metrics, with no custom
controller in the loop.

The primary use case is **comparative benchmarking**: running the same workload
profile under Direct HPA and WVA side-by-side to quantify how much value the
WVA's saturation-aware logic adds over vanilla metric-driven scaling.

This design is based on the benchmarking framework described in
[asm582/inferno-autoscaler/hack/benchmark](https://github.com/asm582/inferno-autoscaler/tree/main/hack/benchmark),
which introduced the `-d` / `--direct-hpa` flag for this purpose.

---

## How it works

In WVA mode, a custom controller reads vLLM metrics (KV cache utilization,
queue depth) and emits a `wva_desired_replicas` metric. An HPA follows that
metric via prometheus-adapter. The WVA controller is the intelligent
intermediary.

In Direct HPA mode, the controller is removed from the path entirely:

```
EPP pod
  └─ emits llm_d_router_epp_flow_control_queue_size{inference_pool=...}
       └─ prometheus-adapter exposes as external metric
            └─ HPA reads metric directly and scales decode Deployment
```

There is no WVA controller. The HPA reacts to raw EPP queue pressure and
running request counts without any saturation-aware smoothing.

---

## Metrics used

Direct HPA scales on two EPP metrics emitted by the
[llm-d-inference-scheduler](https://github.com/llm-d/llm-d-inference-scheduler)
(the EPP / Endpoint Policy Plugin).

### `llm_d_router_epp_flow_control_queue_size`

| Property | Value |
|---|---|
| Type | Gauge |
| Subsystem | `llm_d_router_epp` |
| Labels | `inference_pool`, `model_name`, `target_model_name`, `fairness_id`, `priority` |
| Meaning | Number of requests currently held in the Flow Control queue |

The `inference_pool` label matches the InferencePool resource name (which in
llm-d stacks equals `{model_id_label}`). This is the label used to scope the
HPA metric selector to a specific deployment.

> **Deprecated name:** The inferno-autoscaler benchmark guide references
> `inference_extension_flow_control_queue_size` (subsystem `inference_extension`).
> This name is deprecated. Use `llm_d_router_epp_flow_control_queue_size` for
> new deployments.

### `llm_d_router_epp_running_requests`

| Property | Value |
|---|---|
| Type | Gauge |
| Subsystem | `llm_d_router_epp` |
| Labels | `model_name` only |
| Meaning | Number of inference requests currently running |

> **Deprecated name:** `inference_objective_running_requests`
> (subsystem `inference_objective`).

> **Multi-pool caveat:** This metric carries only `model_name` — no
> `inference_pool` label. If multiple InferencePools in the cluster serve the
> same model name, this metric is aggregated across all of them and cannot be
> scoped to a single deployment via HPA `selector.matchLabels`. For multi-model
> or multi-pool clusters, prefer `llm_d_router_epp_average_running_requests`,
> which includes a pool `name` label.

---

## Label propagation

Kubernetes HPA `External` metrics require the metric values to be reachable
via the [custom metrics API](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-metrics-apis).
prometheus-adapter bridges Prometheus series into this API and controls which
labels are exposed for HPA `selector.matchLabels` filtering.

### Flow for `llm_d_router_epp_flow_control_queue_size`

```
Prometheus series labels:
  inference_pool="qwen-qwen3-32b"
  model_name="Qwen/Qwen3-32B"
  ...

prometheus-adapter rule:
  resources.overrides:
    exported_namespace: {resource: "namespace"}   # injected from scrape context
    inference_pool:     {resource: "deployment"}  # pool name → deployment name

HPA selector:
  matchLabels:
    inference_pool: qwen-qwen3-32b   # resolves to the decode Deployment
```

The EPP pod emits metrics without a Kubernetes `namespace` label. The adapter
infers the namespace from the pod's scrape context (`exported_namespace` or the
pod's own namespace label as relabeled by Prometheus). This must be explicitly
mapped in the adapter rule — it is **not** automatic.

### Why this matters for multi-model clusters

Without the `inference_pool` → `deployment` override, an HPA in namespace A
could react to queue pressure from a different pool in namespace B. The
`matchLabels` filter is the only guard, and it only works if prometheus-adapter
surfaces `inference_pool` as a selectable label dimension.

The prometheus-adapter rule for Direct HPA therefore looks like:

```yaml
rules:
  external:
  - seriesQuery: 'llm_d_router_epp_flow_control_queue_size{inference_pool!=""}'
    resources:
      overrides:
        exported_namespace: {resource: "namespace"}
        inference_pool:     {resource: "deployment"}
    name:
      as: "epp_flow_control_queue_size"
    metricsQuery: >-
      sum(llm_d_router_epp_flow_control_queue_size{<<.LabelMatchers>>})
      by (inference_pool, exported_namespace)

  - seriesQuery: 'llm_d_router_epp_running_requests{model_name!=""}'
    resources:
      overrides:
        exported_namespace: {resource: "namespace"}
    name:
      as: "epp_running_requests"
    metricsQuery: >-
      sum(llm_d_router_epp_running_requests{<<.LabelMatchers>>})
      by (model_name, exported_namespace)
```

---

## HPA manifest

The rendered `HorizontalPodAutoscaler` for a Direct HPA stack looks like:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: <model_id_label>-decode-direct
  namespace: <deploy_namespace>
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: <model_id_label>-decode
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: External
      external:
        metric:
          name: epp_flow_control_queue_size
          selector:
            matchLabels:
              inference_pool: <model_id_label>   # scopes to this pool only
        target:
          type: Value
          value: "250"
    - type: External
      external:
        metric:
          name: epp_running_requests
          selector:
            matchLabels:
              exported_namespace: <deploy_namespace>
        target:
          type: AverageValue
          averageValue: "250"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0    # react immediately to queue pressure
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 240  # wait 4 min before scaling down
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

The target values (`250` for queue size, `250` for running requests) mirror
the inferno-autoscaler reference implementation. These are the values at which
the HPA triggers a scale-up event. Tune them based on your SLO targets and
model serving latency budget.

---

## Scenario configuration

Enable Direct HPA by setting `directHpa.enabled: true` and `wva.enabled: false`
in your scenario YAML. Both cannot be `true` simultaneously.

```yaml
scenario:
  - name: "inference-scheduling-direct-hpa"

    wva:
      enabled: false   # must be false when directHpa is enabled

    directHpa:
      enabled: true
      minReplicas: 1
      maxReplicas: 10
      queueSizeTarget: 250         # value at which scale-up triggers
      runningRequestsTarget: 250   # averageValue at which scale-up triggers
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 0
          policies:
            - type: Percent
              value: 100
              periodSeconds: 15
        scaleDown:
          stabilizationWindowSeconds: 240
          policies:
            - type: Percent
              value: 100
              periodSeconds: 15

    # Same model / deployment config as inference-scheduling-wva
    model:
      name: Qwen/Qwen3-32B
      ...
    decode:
      replicas: 2
      monitoring:
        podmonitor:
          enabled: true   # required: EPP metrics must be scraped
      ...
```

A complete reference scenario is at
[config/scenarios/guides/direct-hpa.yaml](../config/scenarios/guides/direct-hpa.yaml).

---

## CLI usage

```bash
# Standup using the scenario YAML (directHpa.enabled: true already set)
llmdbenchmark --spec guides/direct-hpa standup -p <namespace>

# Or override via CLI flag without editing YAML
llmdbenchmark --spec guides/inference-scheduling standup -p <namespace> --direct-hpa
```

The `-u / --wva` and `--direct-hpa` flags are mutually exclusive. Passing both
is an error.

---

## Comparison with WVA mode

| Aspect | Direct HPA | WVA |
|---|---|---|
| Controller | None (pure K8s HPA) | WVA controller pod required |
| Scaling signal | Raw EPP queue size + running requests | WVA-computed `wva_desired_replicas` (KV cache + queue saturation) |
| Metric routing | `llm_d_router_epp_flow_control_queue_size` | `wva_desired_replicas` |
| Stabilization | Fixed window (default 0s up / 240s down) | Configurable via VariantAutoscaling SLO knobs |
| Multi-pool label scoping | `inference_pool` label on queue metric | `variant_name` + `exported_namespace` labels |
| Platform support | Any cluster with prometheus-adapter | OpenShift only (currently) |
| Use case | Baseline; simplicity | Production; KV-cache-aware scaling |

Direct HPA is a **baseline** — it proves that vanilla HPA can track load, but
it cannot make the saturation-aware decisions (e.g., delay scale-down when KV
cache is still warm) that WVA provides. The benchmark report score formula
quantifies this gap.

---

## Prerequisites

1. **prometheus-adapter** installed and pointed at your Prometheus/Thanos endpoint.
   Direct HPA reuses the same prometheus-adapter instance as WVA.
2. **EPP PodMonitor** (`monitoring.podmonitor.enabled: true` in the decode
   block) so Prometheus scrapes the EPP pod's `/metrics` endpoint.
3. **flowControl feature gate** enabled in the EPP's `EndpointPickerConfig`:

   ```yaml
   apiVersion: inference.networking.x-k8s.io/v1alpha1
   kind: EndpointPickerConfig
   featureGates:
     - flowControl
   plugins:
     - type: queue-scorer
     - type: kv-cache-utilization-scorer
   ```

   Without this gate, `llm_d_router_epp_flow_control_queue_size` is never
   emitted and the HPA `TARGETS` column stays `<unknown>`.

---

## Troubleshooting

### HPA TARGETS shows `<unknown>`

The external metric is not resolving. Check in order:

1. **EPP flowControl gate**: confirm `featureGates: [flowControl]` is in the
   `EndpointPickerConfig`. Without it the metric is never emitted.

   ```bash
   kubectl get configmap -n <namespace> -o yaml | grep -A5 featureGates
   ```

2. **EPP pod is scraping**: verify Prometheus has a target for the EPP pod.
   The PodMonitor must exist and the EPP pod must be Running.

   ```bash
   kubectl get podmonitor -n <namespace>
   kubectl get pods -n <namespace> -l inferencepool
   ```

3. **prometheus-adapter rule**: confirm the `epp_flow_control_queue_size`
   rule is present and `seriesQuery` matches at least one active series.

   ```bash
   kubectl get --raw '/apis/external.metrics.k8s.io/v1beta1/' | jq .
   kubectl get --raw '/apis/external.metrics.k8s.io/v1beta1/namespaces/<namespace>/epp_flow_control_queue_size'
   ```

4. **Label mismatch**: confirm `inference_pool` label on the Prometheus series
   matches `{model_id_label}` used in the HPA `selector.matchLabels`.

   ```bash
   # Query Prometheus directly to see what inference_pool values exist
   kubectl exec -n openshift-monitoring deploy/prometheus-adapter -- \
     wget -qO- 'http://localhost:9090/api/v1/query?query=llm_d_router_epp_flow_control_queue_size'
   ```

### Replica count never changes

- Confirm the HPA `scaleTargetRef` name matches the exact decode Deployment name.
- Check HPA events: `kubectl describe hpa <name> -n <namespace>`.
- Verify `minReplicas` / `maxReplicas` are not both `1`.

### Direct HPA and WVA both enabled

standup will fail with a validation error. Set exactly one of
`wva.enabled: true` or `directHpa.enabled: true`.
