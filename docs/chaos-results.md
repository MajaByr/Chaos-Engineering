# Experiment 1: Pod Failure (Pod Kill)

**Objective:** Simulate an unexpected frontend pod crash and observe Kubernetes self-healing behavior, measuring the impact on request error rate and system recovery time.

**Target service:** `frontend` (namespace: `default`)

**Tool:** Chaos Mesh — `PodChaos` action `pod-kill`, mode `one`

## Baseline (pre-experiment)

Prior to the experiment, all pods were ready and running.

The frontend pod (`frontend-759775d795-qfd6l`) was operating within normal parameters. CPU usage stood at 0.047 cores (47% of requested, 23.5% of limit), with CPU throttling at 28.1%. The load generator recorded 937 requests with a 0% failure rate, an average response time of 82 ms.

## Experiment Execution

The pod kill was injected via Chaos Mesh. The injection was specified by the following `.yaml` file:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-frontend
  namespace: chaos-mesh
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - default
    labelSelectors:
      "app": "frontend"
```

The following sequence was observed:

| Time | Event |
|---|---|
| T+0s | Killing — pod `frontend-759775d795-qfd6l` stopped |
| T+0s | SuccessfulCreate — new pod `frontend-759775d795-khbmz` created by ReplicaSet |
| T+1s | Scheduled — new pod assigned to node |
| T+4s | Created + Pulled — container created, image already cached locally |
| T+6s | Started — container running |

**Mean Time To Recovery (MTTR):** 7m52s − 7m46s = **6 seconds**

Recovery was exceptionally fast due to the container image being present in the local node cache, eliminating the image pull step entirely.

## Load Generator Comparison

| Metrics | Before | After | Change |
|---|---|---|---|
| Total requests | 937 | 3275 | — |
| Failures | 0 (0.00%) | 21 (0.64%) | spike |
| Avg response time | 82 ms | 128 ms | +56% |
| Max response time | 4670 ms | 19685 ms | +322% |

## Error Rate and Latency — Impact Observed

After the experiment, the load generator recorded a cumulative failure rate of 0.64% (21 out of 3275 requests), compared to 0% before the experiment. Average response time increased from 82 ms to 128 ms (+56%), reflecting the brief period during which the frontend was unavailable. Max response time reached 19685 ms (+322%) after running the experiment.

Importantly, the failures were limited to the disruption window only.

## Conclusions

- Kubernetes self-healing operated automatically and without manual intervention, restarting the frontend pod within 6 seconds.
- The brief unavailability window caused a measurable but contained error spike (0.64% of total requests), with no persistent degradation after recovery.
- The elevated average response time (128 ms vs 82 ms baseline) reflects request failures and retries during the disruption window rather than ongoing latency degradation.
- The absence of image pull delay (cached image) was a key factor in achieving sub-10-second MTTR. In a cold-start scenario (no cached image), MTTR would take much longer.

---

# Experiment 2: Network Partition

**Objective:** Simulate a network-level communication failure between the frontend and checkoutservice, and observe the system's partial degradation behavior and recovery.

**Tool:** Chaos Mesh — `NetworkChaos`, action `partition`, direction `to`, duration `60s`

## Baseline (pre-experiment)

Prior to the experiment, all 14 monitored endpoints reported a 0% failure rate. The checkout endpoint (`POST /cart/checkout`) recorded an average response time of 139ms with no failures across 119 requests. Overall system throughput was stable at 1.80 req/s.

Prior to conducting network partition experiments, liveness and readiness probe timeouts were increased from the default 1 second to 10 seconds across all application deployments. This adjustment was necessary to prevent probe-induced cascading failures in the single-node Kind environment, where iptables-based network rules affect all pod-to-pod traffic on the node. Without this change, health probes would begin failing within ~15 seconds of partition injection, triggering Kubernetes to kill and restart unrelated pods.

## Experiment Execution

The network partition was injected via Chaos Mesh. The injection was specified by the following `.yaml` file:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: partition-frontend-checkout
  namespace: default
spec:
  action: partition
  mode: all
  duration: "60s"
  selector:
    namespaces:
      - default
    labelSelectors:
      app: frontend
  direction: to
  target:
    mode: all
    selector:
      namespaces:
        - default
      labelSelectors:
        app: checkoutservice
```

| Time | Event |
|---|---|
| T+0s | Experiment started |
| T+2s | Network partition applied to frontend and checkoutservice pods |
| T+59s | Duration elapsed |
| T+60s | Partition automatically removed, services recovered |

## Impact Observed During Partition

The network partition produced a clean, isolated failure confined exclusively to the `POST /cart/checkout` endpoint. All other 13 endpoints maintained a 0% failure rate throughout the experiment, confirming the expected partial degradation behavior.

| Metric | Baseline | During Partition | Change |
|---|---|---|---|
| `POST /cart/checkout` failure rate | 0.00% | 2.45% | increase |
| `POST /cart/checkout` avg response time | 139ms | 366ms | +163% |
| `POST /cart/checkout` max response time | 2498ms | 20046ms | timeout |
| All other endpoints failure rate | 0.00% | 0.00% | — |
| Overall failure rate | 0.00% | 0.11% | increase |
| failures/s | 0.00 | 0.10 | increase |

The maximum observed response time of 20,046ms corresponds to the frontend's HTTP timeout threshold — requests to checkoutservice were held open until the connection timed out, rather than failing immediately.

No pod restarts occurred during the experiment.

## Recovery

Upon automatic removal of the partition at T+60s, the failure rate dropped to 0.00 failures/s immediately. The 8 failures recorded during the experiment represent the total blast radius — no additional failures were observed post-recovery.

## Conclusions

- **Partial degradation confirmed.** The partition isolated failures exclusively to `POST /cart/checkout` (2.45% failure rate) while all 13 remaining endpoints maintained 0% failure rate, demonstrating effective blast radius containment.
- **Kubernetes self-healing does not apply to network failures.** Zero pod restarts occurred. Unlike crashed pods, a running-but-unreachable service is invisible to Kubernetes lifecycle mechanisms.
- **Absence of circuit breakers amplifies user impact.** Requests to the partitioned path were held open for the full 20-second timeout before failing, rather than fast-failing. A circuit breaker implementation would reduce this to milliseconds.
- **Recovery was instantaneous.** Upon partition removal at T+60s, `failures/s` returned to 0.00 immediately with no retry storms or state corruption observed.
- **Default probe timeouts (1s) caused unintended cluster-wide cascading failures in initial attempts.** Increasing timeouts to 10s was necessary to isolate the experiment to its intended scope.
