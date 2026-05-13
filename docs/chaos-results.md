# Experiment 1: Pod Failure (Pod Kill)

**Objective:** Simulate an unexpected frontend pod crash and observe Kubernetes self-healing behavior, measuring the impact on request error rate and system recovery time.

**Target service:** `frontend` (namespace: `default`)

**Tool:** Chaos Mesh — `PodChaos` action `pod-kill`, mode `one`

## Baseline (pre-experiment)

Prior to the experiment, all pods were ready and running.
<img width="890" height="359" alt="image" src="https://github.com/user-attachments/assets/3b665321-f8f0-4154-adaf-83aad3fd1232" />


The frontend pod (`frontend-759775d795-qfd6l`) was operating within normal parameters. CPU usage stood at 0.047 cores (47% of requested, 23.5% of limit), with CPU throttling at 28.1%. The load generator recorded 937 requests with a 0% failure rate, an average response time of 82 ms.

<img width="945" height="410" alt="image" src="https://github.com/user-attachments/assets/5ca2b401-8875-4ce2-807f-a060423facc3" />

<img width="945" height="250" alt="image" src="https://github.com/user-attachments/assets/8effda90-a6e9-4d72-8396-de5fb0321ccc" />


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
<img width="945" height="99" alt="image" src="https://github.com/user-attachments/assets/fb37be10-47c2-4430-9f9c-b9773d812443" />


| Time | Event |
|---|---|
| T+0s | Killing — pod `frontend-759775d795-qfd6l` stopped |
| T+0s | SuccessfulCreate — new pod `frontend-759775d795-khbmz` created by ReplicaSet |
| T+1s | Scheduled — new pod assigned to node |
| T+4s | Created + Pulled — container created, image already cached locally |
| T+6s | Started — container running |

**Mean Time To Recovery (MTTR):** 7m52s − 7m46s = **6 seconds**

Recovery was exceptionally fast due to the container image being present in the local node cache, eliminating the image pull step entirely.
In Grafana we can see that new pod was created almost immediately:
<img width="865" height="500" alt="image" src="https://github.com/user-attachments/assets/cd949f80-84c1-4c9b-9c65-4cd1cc9db91b" />


## Load Generator Comparison

| Metrics | Before | After | Change |
|---|---|---|---|
| Total requests | 937 | 3275 | — |
| Failures | 0 (0.00%) | 21 (0.64%) | spike |
| Avg response time | 82 ms | 128 ms | +56% |
| Max response time | 4670 ms | 19685 ms | +322% |

<img width="945" height="254" alt="image" src="https://github.com/user-attachments/assets/a2fa0a9d-b91e-4ef7-9b82-01694e3a4e70" />


## Error Rate and Latency — Impact Observed

After the experiment, the load generator recorded a cumulative failure rate of 0.64% (21 out of 3275 requests), compared to 0% before the experiment. Average response time increased from 82 ms to 128 ms (+56%), reflecting the brief period during which the frontend was unavailable. Max response time reached 19685 ms (+322%) after running the experiment.

Importantly, the failures were limited to the disruption window only.

## Conclusions

- Kubernetes self-healing operated automatically and without manual intervention, restarting the frontend pod within 6 seconds.
- The brief unavailability window caused a measurable but contained error spike (0.64% of total requests), with no persistent degradation after recovery.
- The elevated average response time (128 ms vs 82 ms baseline) reflects request failures and retries during the disruption window rather than ongoing latency degradation.
- The absence of image pull delay (cached image) was a key factor in achieving sub-10-second MTTR. In a cold-start scenario (no cached image), MTTR would take much longer.


## Injection mechanism 

Chaos Mesh implements pod kill by issuing a direct deletion request to the Kubernetes API server targeting the selected pod. The controller identifies the pod matching the label selector and calls the same API endpoint that kubectl delete pod would invoke. The container runtime (containerd) receives a SIGTERM signal and immediately stops the container. Kubernetes detects the pod as terminated and the owning ReplicaSet controller responds by creating a replacement pod to satisfy the declared replica count.
The injection operates entirely through the Kubernetes control plane — no process-level or kernel-level manipulation is involved. This means the failure is clean and instantaneous, with no partial state or corrupted data left behind.

---




# Experiment 2: Network Partition

**Objective:** Simulate a network-level communication failure between the frontend and checkoutservice, and observe the system's partial degradation behavior and recovery.

**Tool:** Chaos Mesh — `NetworkChaos`, action `partition`, direction `to`, duration `60s`

## Baseline (pre-experiment)

Prior to the experiment, all 14 monitored endpoints reported a 0% failure rate. The checkout endpoint (`POST /cart/checkout`) recorded an average response time of 139ms with no failures across 119 requests. Overall system throughput was stable at 1.80 req/s.

Prior to conducting network partition experiments, liveness and readiness probe timeouts were increased from the default 1 second to 10 seconds across all application deployments. This adjustment was necessary to prevent probe-induced cascading failures in the single-node Kind environment, where iptables-based network rules affect all pod-to-pod traffic on the node. Without this change, health probes would begin failing within ~15 seconds of partition injection, triggering Kubernetes to kill and restart unrelated pods.

<img width="1123" height="310" alt="image" src="https://github.com/user-attachments/assets/c5171c65-f721-4cb0-9d36-c78f95fd7360" />


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

<img width="1090" height="255" alt="image" src="https://github.com/user-attachments/assets/80e72532-9d07-4e29-8a87-9f40124c5285" />

## Impact Observed During Partition

The network partition produced a clean, isolated failure confined exclusively to the `POST /cart/checkout` endpoint. All other 13 endpoints maintained a 0% failure rate throughout the experiment, confirming the expected partial degradation behavior.

<img width="1091" height="303" alt="image" src="https://github.com/user-attachments/assets/24390221-1d1f-493d-839e-72a29619816e" />


| Metric | Baseline | During Partition | Change |
|---|---|---|---|
| `POST /cart/checkout` failure rate | 0.00% | 2.45% | increase |
| `POST /cart/checkout` avg response time | 139ms | 366ms | +163% |
| `POST /cart/checkout` max response time | 2498ms | 20046ms | timeout |
| All other endpoints failure rate | 0.00% | 0.00% | — |
| Overall failure rate | 0.00% | 0.11% | increase |
| failures/s | 0.00 | 0.10 | increase |

The maximum observed response time of 20,046ms corresponds to the frontend's HTTP timeout threshold — requests to checkoutservice were held open until the connection timed out, rather than failing immediately.

No pod restarts occurred during the experiment:
<img width="872" height="403" alt="image" src="https://github.com/user-attachments/assets/0754fd51-1258-4ca9-af9d-b9d29439e29b" />


## Recovery

Upon automatic removal of the partition at T+60s, the failure rate dropped to 0.00 failures/s immediately. The 8 failures recorded during the experiment represent the total blast radius — no additional failures were observed post-recovery.
<img width="1114" height="305" alt="image" src="https://github.com/user-attachments/assets/6de4c845-74cf-442f-ae53-af880a0936e2" />

## Conclusions

- **Partial degradation confirmed.** The partition isolated failures exclusively to `POST /cart/checkout` (2.45% failure rate) while all 13 remaining endpoints maintained 0% failure rate, demonstrating effective blast radius containment.
- **Kubernetes self-healing does not apply to network failures.** Zero pod restarts occurred. Unlike crashed pods, a running-but-unreachable service is invisible to Kubernetes lifecycle mechanisms.
- **Absence of circuit breakers amplifies user impact.** Requests to the partitioned path were held open for the full 20-second timeout before failing, rather than fast-failing. A circuit breaker implementation would reduce this to milliseconds.
- **Recovery was instantaneous.** Upon partition removal at T+60s, `failures/s` returned to 0.00 immediately with no retry storms or state corruption observed.
- **Default probe timeouts (1s) caused unintended cluster-wide cascading failures in initial attempts.** Increasing timeouts to 10s was necessary to isolate the experiment to its intended scope.


## Injection Mechanism

Chaos Mesh implements network partition by injecting iptables rules directly into the network namespace of the targeted pod on the cluster node. When the experiment is applied, the Chaos Mesh daemon running on the node executes the equivalent of:
```
iptables -A OUTPUT -s <frontend-pod-IP> -d <checkoutservice-pod-IP> -j DROP
iptables -A INPUT  -d <frontend-pod-IP> -s <checkoutservice-pod-IP> -j DROP
```
These rules cause the kernel to silently drop all IP packets travelling between the two pods in the specified direction. Neither pod is aware of the rule — from the application's perspective, packets are sent but never arrive, and no TCP connection is ever established. This results in connection timeouts rather than immediate connection refused errors, which is why response times reached the full 20-second HTTP timeout threshold rather than failing fast.
When the experiment duration elapses, Chaos Mesh removes the injected iptables rules, restoring full network connectivity between the pods. Because the rules are stateless and operate at the packet level, recovery is instantaneous — no application restart or reconnection handshake is required.

## CPU Stress Mechanism

### Theory

Chaos Mesh implements CPU stress by injecting a stress workload directly into the target pod’s Linux cgroup and namespace using the embedded `stress-ng` utility. When the experiment is applied, the Chaos Mesh daemon running on the node launches CPU-intensive worker processes inside the selected container, equivalent to:

```bash
stress-ng --cpu 2 --cpu-load 90
```

*Note: If `stress-ng` is not available, install it with: `sudo apt-get install stress-ng`*

This causes the injected worker threads to continuously execute computational operations in order to maintain approximately 90% utilization across the specified number of CPU cores. Because the stress process runs inside the same cgroup as the application container, it competes directly with the application for CPU scheduling time enforced by the Linux Completely Fair Scheduler (CFS).

From the application’s perspective, no explicit failure occurs — the service remains reachable and functional, but receives significantly less CPU time for processing requests. As CPU saturation increases, request handling slows down, latency rises, and throughput decreases. Under sustained load, Kubernetes may additionally apply CPU throttling if container CPU limits are configured, further amplifying response delays.

When the experiment duration elapses, Chaos Mesh terminates the injected stress processes, immediately releasing CPU resources back to the application container. Since no application state or network configuration is modified, recovery is near-instantaneous and does not require pod restarts or connection re-establishment.

### Deployment

1. Create a CPU stress experiment manifest (Chaos Mesh `yaml`):

  ```yaml
  apiVersion: chaos-mesh.org/v1alpha1
  kind: StressChaos
  metadata:
    name: recommendation-cpu-stress
    namespace: chaos-testing
  spec:
    mode: one
    selector:
      namespaces:
        - boutique
      labelSelectors:
        app: recommendationservice
    stressors:
      cpu:
        workers: 2
        load: 90
    duration: '120s'
  ```

2. Apply the experiment:

  ```bash
  kubectl apply -f recommendation-cpu-stress.yaml
  ```

3. Observe CPU utilization:

  ```bash
  kubectl top pod -n boutique
  ```

3. Generate traffic against the application:

  ```bash
  kubectl port-forward svc/frontend 8080:80 -n boutique
  ```

4. Continuously execute:

```bash
curl http://localhost:8080
```

*. Useful Grafana metrics to observe:
* CPU saturation
* container CPU throttling
* request throughput
* latency increase
* service response times

5. Remove the experiment:

  ```bash
  kubectl delete -f recommendation-cpu-stress.yaml
  ```

## Memory Stress Mechanism

### Theory

Chaos Mesh implements memory stress by injecting artificial memory pressure into the target pod using `stress-ng` memory allocation workers executed inside the container namespace. When the experiment is applied, the Chaos Mesh daemon launches processes equivalent to:

```bash
stress-ng --vm 1 --vm-bytes 512M
```

These worker processes continuously allocate and access large regions of memory, forcing the container to consume a significant portion of its available RAM. Because the stress workload executes within the same Kubernetes cgroup as the application, the memory usage is accounted against the pod’s configured memory limits.

From the application’s perspective, the service initially continues to operate normally, but available memory gradually decreases. As memory pressure increases, the Linux kernel may reclaim page cache, trigger swap activity (if enabled), or invoke the Out-Of-Memory (OOM) killer once the container exceeds its memory limit. In Kubernetes, this typically results in container termination and automatic pod restart according to the deployment policy.

Unlike network-based chaos experiments, memory stress can therefore lead to hard application failures rather than degraded performance alone. During the experiment, Grafana metrics typically show rapidly increasing memory utilization, potential OOM kill events, elevated restart counts, and temporary spikes in request errors while the affected pod is recreated.

### Deployment

1. Create a memory stress experiment manifest (Chaos Mesh `yaml`):

  ```yaml
  apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cart-memory-stress
  namespace: chaos-testing
spec:
  mode: one
  selector:
    namespaces:
      - boutique
    labelSelectors:
      app: cartservice
  stressors:
    memory:
      workers: 1
      size: '512MB'
  duration: '120s'
  ```

2. Apply the experiment:
   ```bash
   kubectl apply -f cart-memory-stress.yaml
   ```

3. Observe pod resource usage:
   ```bash
    kubectl top pod -n boutique
   ```

4. Watch for OOM restarts:
   ```bash
   kubectl get pods -n boutique -w
   ```

5. Inspect pod termination reason:
  ```bash
  kubectl describe pod <cartservice-pod-name> -n boutique
  ```

  *. Useful Grafana metrics to observe:
  * memory utilization
  * OOM kill events
  * pod restart count
  * request error rate
  * service instability

6. Remove the experiment:
   ```bash
   kubectl delete -f cart-memory-stress.yaml
   ```
