# LAB: Chaos Engineering: Resilience Testing with Chaos Mesh
## Table of Contents
1. [What is Chaos Engineering?](#1-what-is-chaos-engineering)
2. [Why Do It? (Core Objectives)](#2-why-do-it-core-objectives)
3. [What is Chaos Mesh?](#3-what-is-chaos-mesh)
4. [Technology Stack](#4-technology-stack)
5. [Getting Started (Lab Setup)](#5-getting-started-lab-setup)
6. [Student Lab Guide: Experiments](#6-student-lab-guide-experiments)
7. [Conclusions](#7-conclusions)

---

## 1. What is Chaos Engineering?

**Chaos Engineering** is the practice of intentionally breaking your systems in a controlled way to find weak spots before they cause real outages. 

Because modern software is complex, failures are inevitable—it's not a matter of *"if"* a system will fail, but *"when"*. Chaos Engineering lets you fail on purpose so you can learn, adapt, and fix issues before your customers ever notice.

## 2. Why Do It? (Core Objectives)

Organizations use Chaos Engineering to:
* **Reduce Downtime:** Find bugs and fix vulnerabilities early.
* **Train the Team:** Give engineers hands-on practice responding to live system alerts in a safe environment.
* **Build Confidence:** Prove that your system can actually survive unexpected disasters.

## 3. What is Chaos Mesh?

**Chaos Mesh** is a free, open-source tool built specifically for testing Kubernetes. It makes it easy to inject different types of failures directly into your cluster. 

In this lab, we will use it to simulate:
* **Pod crashes** (`PodChaos`)
* **Network drops and delays** (`NetworkChaos`)
* **High CPU and Memory usage** (`StressChaos`)

---

## 4. Technology Stack

Our lab environment is built using industry-standard cloud-native tools:

| Component | Technology | Purpose |
| :--- | :--- | :--- |
| **Infrastructure** | Kubernetes (Kind) | Lightweight local cluster deployment for testing |
| **Application** | Google Online Boutique | Multi-service microservices demo application |
| **Observability** | Prometheus | Time-series database for collecting health metrics |
| **Visualization** | Grafana | Dashboards for monitoring CPU, latency, and errors |
| **Chaos Tooling** | Chaos Mesh | Fault injection and experiment orchestration |

---

## 5. Getting Started (Lab Setup)

To run this lab on your local machine, you must have **Docker** installed and running. Download it here: [Install Docker Desktop](https://docs.docker.com/desktop/setup/install).

Install the remaining CLI tools (`kubectl`, `kind`, `helm`) using the instructions for your operating system below:

###  macOS 
```bash
brew install kubectl
brew install kind
brew install helm
```

###  Linux 
```bash
# Install kubectl
curl -LO "[https://dl.k8s.io/release/$](https://dl.k8s.io/release/$)(curl -L -s [https://dl.k8s.io/release/stable.txt](https://dl.k8s.io/release/stable.txt))/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install kind
[ $(uname -m) = x86_64 ] && curl -Lo ./kind [https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64](https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64)
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Install helm
curl [https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3) | bash
```

###  Windows 
Open PowerShell or Command Prompt and run:
```bash
winget install Kubernetes.kubectl
winget install Kubernetes.kind
winget install Helm.Helm
```

**Verify your installations (All OS):**
```bash
docker --version
kubectl version --client
kind version
helm version
```

---

### Lab Setup Steps

**Step 1: Create the Kubernetes Cluster (Kind)**
```bash
kind create cluster --name chaos-lab
```
Check if the node is Ready:
```bash
kubectl get nodes
```
*Troubleshooting:* If you face resource issues or the cluster fails to spin up correctly, apply these limits:
```bash
kubectl set resources deployment emailservice --limits=cpu=1,memory=256Mi
kubectl set resources deployment recommendationservice --limits=cpu=1,memory=256Mi
```

**Step 2: Install Prometheus & Grafana (Observability)**
```bash
helm repo add prometheus-community [https://prometheus-community.github.io/helm-charts](https://prometheus-community.github.io/helm-charts)
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack
```

**Step 3: Access Grafana**
```bash
kubectl port-forward svc/monitoring-grafana 9090:80
```
Open `http://localhost:9090/` in your browser.
* **Login:** `admin`
* **Password:** Retrieve the default password by opening a new terminal tab and running the command for your OS:

  **Mac/Linux (Bash/Zsh):**
  ```bash
  kubectl get secret monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
  ```
  **Windows (PowerShell):**
  ```powershell
  $haslo = kubectl get secret monitoring-grafana -o jsonpath="{.data.admin-password}"; [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($haslo))
  ```

**Step 4: Install Chaos Mesh**
```bash
helm repo add chaos-mesh [https://charts.chaos-mesh.org](https://charts.chaos-mesh.org)
helm repo update
helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --create-namespace --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```
Verify the installation (ensure pods are in `Ready` status) and access the dashboard:
```bash
kubectl get pods -n chaos-mesh
kubectl port-forward svc/chaos-dashboard -n chaos-mesh 2333:2333
```

**Step 5: Create a Chaos Mesh Account**
Follow the official Chaos Mesh instructions to create an account for `namespace = chaos-mesh` and `role = manager`.

**Step 6: Start the Online Boutique Application**
```bash
kubectl port-forward svc/frontend 8082:80
```

**Step 7: Generate Chaos Mesh Token**
Generate the token to log into the Chaos Mesh dashboard:
```bash
kubectl create token chaos-admin -n chaos-mesh
```

---

## 6. Student Lab Guide: Experiments

Before starting the experiments, ensure you have multiple terminal windows open. Use the following commands to monitor the system's baseline health and observe changes during fault injection:

* **Watch cluster events in real-time:**
  ```bash
  kubectl get events -n default --sort-by='.lastTimestamp' -w
  ```
* **Watch pod status changes:**
  ```bash
  kubectl get pods -n default -w
  ```
* **Monitor application traffic (Load Generator logs):**
  ```bash
  kubectl logs -n default deployment/loadgenerator --tail=20 -f
  ```

---

### Experiment 1: Pod Failure (Pod Kill)

**Objective:** Simulate an unexpected frontend pod crash and observe Kubernetes self-healing behavior. Calculate how long it takes for the system to recover (MTTR).

**1. Create the Chaos Manifest**
Create a file named `pod-kill.yaml` with the following content:
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

**2. Execute the Experiment**
```bash
kubectl apply -f pod-kill.yaml
```

**3. Observe the System**
* Check your `kubectl get pods -w` terminal. You should see the frontend pod transition to `Terminating` and a new pod spinning up to `ContainerCreating` and then `Running`.
* Look at your Grafana dashboard. Did the error rate spike? Did latency increase temporarily while the pod was down?

**4. Cleanup**
```bash
kubectl delete -f pod-kill.yaml
```

---

### Experiment 2: Network Partition

**Objective:** Simulate a network-level communication failure between the `frontend` and the `checkoutservice`. Observe how the system degrades partially rather than failing entirely.

**1. Create the Chaos Manifest**
Create a file named `network-partition.yaml` with the following content:
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

**2. Execute the Experiment**
```bash
kubectl apply -f network-partition.yaml
```

**3. Observe the System**
* Note that no pods are killed during this experiment. 
* Check the Load Generator logs. You should see HTTP timeout errors specifically for the `/cart/checkout` endpoint.
* Open Grafana. Observe the response times for the checkout service. Without a circuit breaker, requests will hang until they hit the application's maximum timeout threshold.

**4. Cleanup**
*Wait 60 seconds for the experiment to automatically finish, or run:*
```bash
kubectl delete -f network-partition.yaml
```

---

### Experiment 3: CPU Stress

**Objective:** Induce a CPU bottleneck in the `recommendationservice` to observe how resource starvation affects request throughput and latency.

**1. Create the Chaos Manifest**
Create a file named `cpu-stress.yaml` with the following content:
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: recommendation-cpu-stress
  namespace: default
spec:
  mode: one
  selector:
    namespaces:
      - default
    labelSelectors:
      app: recommendationservice
  stressors:
    cpu:
      workers: 2
      load: 90
  duration: '120s'
```

**2. Execute the Experiment**
```bash
kubectl apply -f cpu-stress.yaml
```

**3. Observe the System**
* Run the following command to observe CPU utilization in real-time:
  ```bash
  kubectl top pods -n default
  ```
* You should see the `recommendationservice` pod consuming significantly more CPU.
* Check Grafana. Notice how CPU throttling affects the service response times and overall user experience.

**4. Cleanup**
```bash
kubectl delete -f cpu-stress.yaml
```

---

### Experiment 4: Memory Stress

**Objective:** Simulate severe memory pressure within the `cartservice` container to trigger an Out-Of-Memory (OOM) kill by the Linux kernel.

**1. Create the Chaos Manifest**
Create a file named `memory-stress.yaml` with the following content:
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cart-memory-stress
  namespace: default
spec:
  mode: one
  selector:
    namespaces:
      - default
    labelSelectors:
      app: cartservice
  stressors:
    memory:
      workers: 1
      size: '512MB'
  duration: '120s'
```

**2. Execute the Experiment**
```bash
kubectl apply -f memory-stress.yaml
```

**3. Observe the System**
* Monitor memory usage:
  ```bash
  kubectl top pods -n default
  ```
* Watch your pod status terminal (`kubectl get pods -w`). Unlike the CPU test, memory starvation will eventually cause the pod to crash. Look for the status to change to `OOMKilled`.
* To verify why the pod crashed, describe the pod (replace `<pod-name>` with your actual cartservice pod name):
  ```bash
  kubectl describe pod <pod-name> -n default
  ```

**4. Cleanup**
```bash
kubectl delete -f memory-stress.yaml
```

---

## 7. Conclusions

By systematically subjecting our infrastructure to controlled chaos, we transition from hoping our systems work to knowing exactly how they fail. 

* **Automatic Healing:** Kubernetes handles basic infrastructure failures (like pod crashes) exceptionally well, demonstrating low MTTR.
* **Partial Degradation:** Network partitions prove that blast radii can be contained. However, without proper circuit breakers, requests hang until timing out, degrading the user experience.
* **Resource Starvation:** CPU and Memory stress testing highlights the critical need for appropriate Kubernetes resource limits and readiness probes.

Chaos Engineering is not a one-time event, but a continuous practice of learning and hardening modern software systems.
