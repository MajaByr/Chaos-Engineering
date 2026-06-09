# Results

## Experiment 1: Pod Failure (Pod Kill)

### Objective 
Simulate an unexpected frontend pod crash and observe Kubernetes self-healing behavior, measuring the impact on request error rate and system recovery time.

### Theory 
Chaos Mesh implements a pod kill by issuing a direct deletion request to the Kubernetes API server, targeting the selected pod. The controller identifies the pod matching the label selector and calls the exact same API endpoint that executing kubectl delete pod would invoke.

Upon receiving this request, the container runtime (e.g., containerd) receives a SIGTERM signal and immediately stops the container. Because Kubernetes constantly monitors the state of the cluster, it quickly detects the pod as terminated. The owning ReplicaSet controller immediately responds to this state change by creating a replacement pod to satisfy the declared replica count.

The injection operates entirely through the standard Kubernetes control plane—no direct process-level or kernel-level manipulation is involved. This ensures that the failure is clean and instantaneous, with no partial state or corrupted data left behind. The primary goal of this experiment is to observe how quickly the system's self-healing mechanisms restore the required number of replicas (Mean Time To Recovery) and how the brief unavailability impacts user traffic.

### Baseline (pre-experiment)

Prior to the experiment, all pods were ready and running.
<img width="890" height="359" alt="image" src="screenshots/1.png" />


The frontend pod (`frontend-759775d795-qfd6l`) was operating within normal parameters. CPU usage stood at 0.047 cores (47% of requested, 23.5% of limit), with CPU throttling at 28.1%. The load generator recorded 937 requests with a 0% failure rate, an average response time of 82 ms.

<img width="945" height="410" alt="image" src="screenshots/2.png" />

<img width="945" height="250" alt="image" src="screenshots/3.png" />


### Experiment Execution

1. **Creating the manifest**
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

2. **Applying the experiment:**
```bash
kubectl apply -f network-partition.yaml
```

3. **Observing the system:**
```bash
kubectl get pods -n default --sort-by='.lastTimestamp' | Select-String "frontend"
```
The following sequence was observed:
<img width="945" height="99" alt="image" src="screenshots/4.png" />


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
<img width="865" height="500" alt="image" src="screenshots/5.png" />


### Load Generator Comparison

```bash
kubectl logs -n default deployment/loadgenerator --tail=20
```

| Metrics | Before | After | Change |
|---|---|---|---|
| Total requests | 937 | 3275 | — |
| Failures | 0 (0.00%) | 21 (0.64%) | spike |
| Avg response time | 82 ms | 128 ms | +56% |
| Max response time | 4670 ms | 19685 ms | +322% |

<img width="945" height="254" alt="image" src="screenshots/6.png" />


### Error Rate and Latency — Impact Observed

After the experiment, the load generator recorded a cumulative failure rate of 0.64% (21 out of 3275 requests), compared to 0% before the experiment. Average response time increased from 82 ms to 128 ms (+56%), reflecting the brief period during which the frontend was unavailable. Max response time reached 19685 ms (+322%) after running the experiment.

Importantly, the failures were limited to the disruption window only.

### Conclusions

- Kubernetes self-healing operated automatically and without manual intervention, restarting the frontend pod within 6 seconds.
- The brief unavailability window caused a measurable but contained error spike (0.64% of total requests), with no persistent degradation after recovery.
- The elevated average response time (128 ms vs 82 ms baseline) reflects request failures and retries during the disruption window rather than ongoing latency degradation.
- The absence of image pull delay (cached image) was a key factor in achieving sub-10-second MTTR. In a cold-start scenario (no cached image), MTTR would take much longer.


## Experiment 2: CPU Stress

### Theory

Chaos Mesh implements CPU stress by injecting a stress workload directly into the target pod’s Linux cgroup and namespace using the embedded `stress-ng` utility. When the experiment is applied, the Chaos Mesh daemon running on the node launches CPU-intensive worker processes inside the selected container, equivalent to:

```bash
stress-ng --cpu 2 --cpu-load 90
```

*Note: If `stress-ng` is not available, it can be installed with: `sudo apt-get install stress-ng`*

This causes the injected worker threads to continuously execute computational operations in order to maintain approximately 90% utilization across the specified number of CPU cores. Because the stress process runs inside the same cgroup as the application container, it competes directly with the application for CPU scheduling time enforced by the Linux Completely Fair Scheduler (CFS).

From the application's perspective, no explicit failure occurs — the service remains reachable and functional, but receives significantly less CPU time for processing requests. As CPU saturation increases, request handling slows down, latency rises, and throughput decreases. Under sustained load, Kubernetes may additionally apply CPU throttling if container CPU limits are configured, further amplifying response delays.

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

  CPU usage before:
  <img alt="image" src="screenshots/cpu-stress/cpu-stress-disabled.png" />

  CPU usage after:
  <img alt="image" src="screenshots/cpu-stress/cpu-stress-active.png" />

  <img alt="image" src="screenshots/cpu-stress/cpu-grafana.png" />

Application worked slower, but no failures were observed.

Useful Grafana metrics to observe:
* CPU saturation
* container CPU throttling
* request throughput
* latency increase
* service response times

4. Remove the experiment:

  ```bash
  kubectl delete -f recommendation-cpu-stress.yaml
  ```

## Experiment 3: Memory Stress

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

  <img alt="image" src="screenshots/memory-stress/logs.png" />

  <img alt="image" src="screenshots/memory-stress/pods-oom.png" />

5. Inspect pod termination reason:
  
  ```bash
  kubectl describe pod <cartservice-pod-name> -n boutique
  ```

  <img alt="image" src="screenshots/memory-stress/cart-oom-logs.png" />

Useful Grafana metrics to observe:

  * memory utilization
  * OOM kill events
  * pod restart count
  * request error rate
  * service instability

6. Remove the experiment:
   ```bash
   kubectl delete -f cart-memory-stress.yaml
   ```
