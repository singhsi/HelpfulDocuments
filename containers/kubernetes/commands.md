# Kubernetes Commands

> For concepts and best practices see [README.md](README.md).

---

## Contexts and Namespaces

#### show the current context (which cluster you are talking to)
`kubectl config current-context`

#### list all contexts
`kubectl config get-contexts`

#### switch to another context
`kubectl config use-context <context-name>`

#### set the default namespace for the current context
`kubectl config set-context --current --namespace=<namespace>`

#### list namespaces
`kubectl get namespaces`

#### create a namespace
`kubectl create namespace <namespace>`

---

## Viewing Resources

#### list pods in the current namespace
`kubectl get pods`

#### list pods in all namespaces
`kubectl get pods -A`

#### list pods with node and IP details
`kubectl get pods -o wide`

#### watch pods update live
`kubectl get pods -w`

#### list pods matching a label
`kubectl get pods -l app=<app-name>`

#### list deployments, services, and pods together
`kubectl get deployments,services,pods`

#### show detailed info and recent events for a resource
`kubectl describe pod <pod-name>`

#### show a resource as YAML
`kubectl get deployment <deployment-name> -o yaml`

#### list recent events sorted by time
`kubectl get events --sort-by=.metadata.creationTimestamp`

#### explain the fields of a resource type
`kubectl explain deployment.spec`

---

## Applying and Deleting

#### create or update resources from a file
`kubectl apply -f <file>.yaml`

#### apply every YAML file in a directory
`kubectl apply -f <directory>/`

#### preview what apply would change
`kubectl diff -f <file>.yaml`

#### delete resources defined in a file
`kubectl delete -f <file>.yaml`

#### delete a single resource
`kubectl delete pod <pod-name>`

#### generate deployment YAML without creating it
`kubectl create deployment <name> --image=<image> --dry-run=client -o yaml > <file>.yaml`

---

## Deployments and Rollouts

#### scale a deployment
`kubectl scale deployment <deployment-name> --replicas=<count>`

#### update the image of a deployment
`kubectl set image deployment/<deployment-name> <container-name>=<image>:<tag>`

#### watch a rollout until it finishes
`kubectl rollout status deployment/<deployment-name>`

#### show rollout history
`kubectl rollout history deployment/<deployment-name>`

#### roll back to the previous version
`kubectl rollout undo deployment/<deployment-name>`

#### restart all pods of a deployment
`kubectl rollout restart deployment/<deployment-name>`

---

## Logs and Debugging

#### show logs of a pod
`kubectl logs <pod-name>`

#### follow logs
`kubectl logs -f <pod-name>`

#### show logs of a specific container in a multi-container pod
`kubectl logs <pod-name> -c <container-name>`

#### show logs from the previous (crashed) container
`kubectl logs <pod-name> --previous`

#### show logs from all pods of a deployment
`kubectl logs deployment/<deployment-name>`

#### open a shell inside a running pod
`kubectl exec -it <pod-name> -- /bin/sh`

#### run a one-off command in a pod
`kubectl exec <pod-name> -- env`

#### forward a local port to a service
`kubectl port-forward service/<service-name> <local-port>:<service-port>`

#### forward a local port to a pod
`kubectl port-forward <pod-name> <local-port>:<container-port>`

#### check which pods a service routes to
`kubectl get endpoints <service-name>`

#### copy a file out of a pod
`kubectl cp <pod-name>:<path-in-pod> <local-path>`

#### show CPU and memory usage (requires metrics-server)
`kubectl top pods`

---

## ConfigMaps and Secrets

#### create a configmap from literal values
`kubectl create configmap <name> --from-literal=<key>=<value>`

#### create a configmap from a file
`kubectl create configmap <name> --from-file=<file>`

#### create a secret from literal values
`kubectl create secret generic <name> --from-literal=<key>=<value>`

#### create a registry pull secret
```bash
kubectl create secret docker-registry <name> \
  --docker-server=<registry> \
  --docker-username=<username> \
  --docker-password=<password>
```

#### decode a secret value
`kubectl get secret <name> -o jsonpath='{.data.<key>}' | base64 --decode`
