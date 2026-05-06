# Planned experiments

Project: Chaos Engineering: Resilience Testing with Chaos Mesh [lab-ready]

*Deploy a multi-service application on Kubernetes (Kind), instrument it with basic health metrics, and demonstrate systematic failure injection: pod kills, network partitions, latency injection, and CPU/memory stress. Show how each failure mode manifests in the metrics and how the system recovers*

---

## Objectives

* Evaluate system resilience under controlled failures
* Observe failure propagation across microservices
* Validate Kubernetes self-healing mechanisms
* Analyze impact on user-facing metrics
* Planned Experiments
* Deliver project as a working lab exercise (lab-ready)

## Details

### Environment

* Infrastructure Setup: Deployment of a local Kubernetes cluster using Kind.
* Application Deployment: Implementation of a multi-service microservices architecture using
the Google Online Boutique demo application (https://github.com/GoogleCloudPlatform/microservices-demo ).
* Observability & Monitoring: Integration of the Prometheus and Grafana stack to collect and
visualize real-time system health metrics, including CPU/memory usage, network traffic, and
pod status.
* Chaos Tooling Integration: Installation and configuration of Chaos Mesh (including RBAC
authorization).

### Analysis

Monitoring the system's behavior during the injected faults via Grafana dashboards and documenting the automated recovery (self-healing) capabilities of Kubernetes. Key metrics observed in dashboards:

* CPU / memory usage
* request rate
* error rate
* latency percentiles

Considered criteria:

* Mean time to recovery (MTTR)
* Service availability during faults
* System stability under repeated failures

### Experiments

#### Pod Failure (Pod Kill) - Simulating unexpected instance crashes

Random or targeted pod termination. Measures:

* restart time
* request error rate
* latency spikes

Expected:

* short disruption
* automatic recovery via Kubernetes

#### Network Partition - Blocking communication between specific microservices

Inject network loss between services. Example: frontend <-> checkoutservice. Measures:

* timeout errors
* retry behavior

Expected:

* increased latency
* partial system degradation

#### Network Latency Injection - Artificially delaying network packets

Add artificial delay (eg. +200ms). Measures:

* response time distribution
* tail latency (p95/p99)

Expected:

* degraded UX
* possible cascading slowdowns

#### CPU Stress - Inducing CPU bottleneck

Increase CPU load in selected pods. Measures:

* CPU saturation
* request throughput

Expected:

* throttling
* slower responses

#### Memory Stress - Inducing memory bottlenecks

Simulate memory pressure. Measures:

* OOM kills
* pod restarts

Expected:

* instability in affected services
* Metrics & Observability
