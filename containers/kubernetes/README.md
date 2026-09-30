# Kubernetes — New Developer Guide

A beginner-friendly introduction to Kubernetes (K8s). Assumes you know what a container is — if not, start with [../podman/README.md](../podman/README.md).

> **See also:** [commands.md](commands.md)

---

## Table of Contents

- [What is Kubernetes?](#what-is-kubernetes)
- [Cluster Architecture](#cluster-architecture)
- [Core Objects](#core-objects)
- [Contexts and Namespaces](#contexts-and-namespaces)
- [Your First Deployment](#your-first-deployment)
- [Configuration and Secrets](#configuration-and-secrets)
- [Health Checks](#health-checks)
- [Best Practices](#best-practices)
- [Common Mistakes to Avoid](#common-mistakes-to-avoid)

---

## What is Kubernetes?

Podman runs containers on **one machine**. Kubernetes runs containers across **many machines** and keeps them running the way you described.

You write a YAML file that says *"I want 3 copies of my app, reachable on port 80"*. Kubernetes continuously compares that **desired state** with what is actually running and fixes any difference — restarting crashed containers, replacing failed machines, and rolling out new versions.

**What Kubernetes gives you:**
- **Self-healing** — crashed containers are restarted automatically
- **Scaling** — change the replica count and Kubernetes adds or removes copies
- **Rolling updates** — deploy a new version with no downtime, and roll back if needed
- **Service discovery and load balancing** — apps find each other by name
- **Config and secret management** — keep settings out of your image

---

## Cluster Architecture

| Component | Where it runs | What it does |
|---|---|---|
| **API server** | Control plane | Front door of the cluster; `kubectl` talks to it |
| **etcd** | Control plane | Key-value store holding the cluster's state |
| **Scheduler** | Control plane | Decides which node a new Pod runs on |
| **Controller manager** | Control plane | Runs the loops that move actual state toward desired state |
| **kubelet** | Every node | Starts and monitors the containers assigned to its node |
| **kube-proxy** | Every node | Routes Service traffic to the right Pods |
| **Container runtime** | Every node | Actually runs the containers (e.g. containerd, CRI-O) |

> **Tip:** For local learning use a single-node cluster such as [minikube](https://minikube.sigs.k8s.io/), [kind](https://kind.sigs.k8s.io/), or the Kubernetes option in Podman Desktop.

---

## Core Objects

| Object | What it is | Analogy |
|---|---|---|
| **Pod** | Smallest unit; one or more containers sharing network and storage | One running instance of your app |
| **Deployment** | Manages a set of identical Pods and rolling updates | "Keep 3 copies of this Pod running" |
| **ReplicaSet** | Created by a Deployment to hold the Pod count | Rarely used directly |
| **Service** | Stable name and IP that load-balances to matching Pods | An internal phone number that always reaches someone |
| **Ingress** | HTTP routing from outside the cluster to Services | Reverse proxy / front desk |
| **ConfigMap** | Non-secret configuration (key-value or files) | `application.properties` |
| **Secret** | Sensitive values (passwords, tokens) | Password vault (base64-encoded, not encrypted by default) |
| **Namespace** | Logical partition of a cluster | A folder for grouping resources |
| **PersistentVolumeClaim** | Request for durable storage | Mounting a disk that survives Pod restarts |

**Labels and selectors** tie objects together. A Deployment labels its Pods `app: my-app`, and a Service selects Pods with `app: my-app` to send traffic to them.

**Service types:**

| Type | Reachable from | Use for |
|---|---|---|
| `ClusterIP` (default) | Inside the cluster only | Service-to-service calls |
| `NodePort` | `<node-ip>:<30000-32767>` | Quick local testing |
| `LoadBalancer` | External IP from the cloud provider | Exposing an app on a cloud cluster |

---

## Contexts and Namespaces

A **namespace** groups resources *inside* a cluster. A **context** decides *which cluster* (and which namespace) `kubectl` talks to.

### Namespaces

- Resource names only need to be unique within a namespace, so `dev` and `test` can each have a Deployment called `my-app`.
- If you don't specify one, objects go into the `default` namespace.
- Kubernetes keeps its own components in system namespaces such as `kube-system` — leave those alone.
- Some objects are cluster-wide and belong to no namespace (e.g. Nodes, PersistentVolumes, Namespaces themselves).

```bash
kubectl create namespace dev
kubectl apply -f my-app.yaml -n dev    # create objects in "dev"
kubectl get pods -n dev                # look in "dev"
kubectl get pods -A                    # look in all namespaces
```

### Contexts

`kubectl` reads its connection settings from a **kubeconfig** file (`~/.kube/config` by default). A context is a named entry in that file that combines three things:

| Part | What it is |
|---|---|
| **Cluster** | The API server address to connect to |
| **User** | The credentials to log in with |
| **Namespace** | The default namespace for commands (optional; `default` if unset) |

```bash
kubectl config current-context                       # which cluster am I on?
kubectl config get-contexts                          # list all contexts
kubectl config use-context <context-name>            # switch cluster
kubectl config set-context --current --namespace=dev # change default namespace
```

> **Warning:** Every `kubectl` command runs against the current context. Check it with `kubectl config current-context` before applying or deleting anything, so you don't change production by mistake.

---

## Your First Deployment

Save this as `my-app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: nginx:1.27
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
```

Apply it and check the result:

```bash
kubectl apply -f my-app.yaml
kubectl get deployments,pods,services
kubectl port-forward service/my-app 8080:80   # then open http://localhost:8080
```

Roll out a new version and watch it:

```bash
kubectl set image deployment/my-app my-app=nginx:1.28
kubectl rollout status deployment/my-app
kubectl rollout undo deployment/my-app        # go back if something is wrong
```

Clean up:

```bash
kubectl delete -f my-app.yaml
```

---

## Configuration and Secrets

Create them:

```bash
kubectl create configmap my-app-config --from-literal=LOG_LEVEL=info
kubectl create secret generic my-app-secret --from-literal=DB_PASSWORD=<password>
```

Use them as environment variables in the container spec:

```yaml
      containers:
        - name: my-app
          image: <image>
          envFrom:
            - configMapRef:
                name: my-app-config
            - secretRef:
                name: my-app-secret
```

> **Warning:** Secret values are only base64-encoded. Anyone who can read Secrets in the namespace can decode them. Never commit Secret YAML with real values to Git.

---

## Health Checks

| Probe | Question it answers | On failure |
|---|---|---|
| `livenessProbe` | Is the app alive? | Container is restarted |
| `readinessProbe` | Is the app ready for traffic? | Pod is removed from the Service until it passes |
| `startupProbe` | Has a slow app finished starting? | Other probes wait until it passes |

```yaml
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
```

> **Tip:** Spring Boot Actuator exposes `/actuator/health/liveness` and `/actuator/health/readiness` when running in Kubernetes.

---

## Best Practices

- **Pin image tags** (`my-app:1.4.2`), never `latest` — so rollouts and rollbacks are predictable.
- **Set resource requests and limits** so the scheduler can place Pods and one app can't starve others:
  ```yaml
  resources:
    requests: { cpu: "250m", memory: "256Mi" }
    limits:   { memory: "512Mi" }
  ```
- **Always add readiness and liveness probes.**
- **Keep YAML in Git** and apply it with `kubectl apply -f` (declarative), not one-off `kubectl run` / `kubectl edit` commands.
- **Use Namespaces** to separate teams or environments.
- **Run as non-root** — set `securityContext.runAsNonRoot: true`.
- **Keep config out of the image** — use ConfigMaps and Secrets.

---

## Common Mistakes to Avoid

| Mistake | Symptom | Fix |
|---|---|---|
| Service `selector` doesn't match Pod labels | Service has no endpoints; requests fail | Check with `kubectl get endpoints <service>` and align labels |
| Wrong image name or missing registry credentials | Pod stuck in `ImagePullBackOff` | `kubectl describe pod <pod>`; fix the image or add an `imagePullSecret` |
| App crashes on start | Pod in `CrashLoopBackOff` | `kubectl logs <pod> --previous` to see the last crash |
| No memory limit | Node runs out of memory; other Pods evicted | Set `resources.limits.memory` |
| Liveness probe too aggressive | Healthy-but-slow app keeps restarting | Add a `startupProbe` or increase `initialDelaySeconds` |
| Editing live objects with `kubectl edit` | Changes lost on next `apply` | Change the YAML in Git and re-apply |
