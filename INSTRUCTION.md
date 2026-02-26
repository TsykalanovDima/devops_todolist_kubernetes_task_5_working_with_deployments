# Deployment Instructions — ToDo App on Kubernetes

## Prerequisites

- A running Kubernetes cluster (local via `minikube` / `kind`, or cloud-managed like GKE, EKS, AKS)
- `kubectl` configured and pointing at your cluster

> The Docker image `ikulyk404/todoapp:3.0.0` is already published on Docker Hub — no build or push needed.

---

## 1. Apply All Manifests

All required files are already in the project. Apply them in this order:

```bash
# 1. Create the namespace first
kubectl apply -f namespace.yml

# 2. Deploy the app
kubectl apply -f deployment.yml

# 3. Set up autoscaling
kubectl apply -f hpa.yml
```

---

## 2. Enable Metrics Server (required for HPA)

HPA needs the Metrics Server to read CPU and Memory usage. If it's not already running:

```bash
# Generic cluster
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# minikube
minikube addons enable metrics-server
```

---

## 3. Verify Everything is Running

```bash
# Expect 2 pods in Running state
kubectl get pods -n mateapp

# Check deployment status
kubectl get deployment todoapp -n mateapp

# Check HPA (TARGETS column should show real values after ~1 min)
kubectl get hpa todoapp-hpa -n mateapp
```

---

## 4. Access the App

### Option A — ClusterIP + Port Forward (quick local access)

```bash
kubectl apply -f clusterIp.yml
kubectl port-forward svc/todoapp 8080:80 -n mateapp
```

Then open http://localhost:8080 in your browser.

### Option B — NodePort (access via cluster node IP)

```bash
kubectl apply -f nodeport.yml
```

Then access the app at:
```
http://<NODE_IP>:30080
```

To find your node IP:
```bash
kubectl get nodes -o wide
# minikube shortcut:
minikube service todoapp -n mateapp
```

---

## 5. Clean Up

To remove everything at once:
```bash
kubectl delete namespace mateapp
```

---

## Design Decisions

### Resource Requests & Limits

| | CPU | Memory |
|---|---|---|
| **Request** | 100m | 128Mi |
| **Limit** | 500m | 256Mi |

**Requests** are the minimum resources Kubernetes guarantees and reserves on a node for each pod. 100m CPU (0.1 core) and 128Mi memory are enough for a lightly loaded Django app at rest, while keeping node scheduling flexible.

**Limits** cap each pod at 500m CPU and 256Mi memory to prevent a misbehaving pod from starving other workloads. The 5× headroom between request and limit lets the app absorb traffic bursts without wasting resources in idle periods.

With 2 pods running at idle, the cluster guarantees 200m CPU / 256Mi memory total — comfortable for this Django application.

---

### HPA Configuration

| Parameter | Value | Reason |
|---|---|---|
| `minReplicas` | 2 | Guarantees high availability at all times — no single point of failure even at zero traffic. Both pods stay warm so spikes are handled immediately without cold-start delay. |
| `maxReplicas` | 5 | A sensible ceiling for a small app. Scaling beyond 5 would require reviewing the database and caching layer first. |
| CPU threshold | 70% | Triggers scale-out before pods become saturated, giving new pods time to pass readiness checks before load peaks. |
| Memory threshold | 80% | Memory grows more gradually than CPU in Django workloads. 80% catches real pressure without over-reacting to short-lived allocations. |

---

### Rolling Update Strategy

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

- **`maxUnavailable: 0`** — at no point during a rollout will fewer than 2 pods be serving traffic. Zero downtime guaranteed.
- **`maxSurge: 1`** — allows 1 extra pod to be created above the desired count during the update. With 2 replicas, at most 3 pods exist at the same time — a small, controlled resource spike.

The new pod must pass its `readinessProbe` (`api/ready`) before the old pod is terminated. This ensures users never hit an unready instance.