# Containers

Guides for building, running, and orchestrating containers. **Podman** builds and runs containers on one machine; **Kubernetes** runs and manages many containers across a cluster of machines.

---

## Contents

### [`containers/podman/`](podman/)
Container basics with Podman (compatible with Docker).

| File | Description |
|---|---|
| [README.md](podman/README.md) | What containers are, Podman vs Docker, key concepts, writing Dockerfiles, best practices |
| [commands.md](podman/commands.md) | Quick-lookup reference for common Podman commands |

---

### [`containers/kubernetes/`](kubernetes/)
Container orchestration with Kubernetes.

| File | Description |
|---|---|
| [README.md](kubernetes/README.md) | What Kubernetes is, core objects (Pods, Deployments, Services, ConfigMaps, Secrets, Namespaces), and a first deployment |
| [commands.md](kubernetes/commands.md) | Quick-lookup reference for common `kubectl` commands |

---

## How They Fit Together

```
Dockerfile ──podman build──▶ Image ──podman push──▶ Registry ──kubectl apply──▶ Pods running in a cluster
```

1. Write a `Dockerfile` and build an image with Podman.
2. Push the image to a registry (Docker Hub, Quay, a private registry).
3. Describe how to run it in Kubernetes YAML and apply it with `kubectl`.
