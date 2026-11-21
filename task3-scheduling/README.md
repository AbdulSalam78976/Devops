# Task 3 – Scheduling, Affinity and Failure Recovery

## Files
- `hello-deployment.yaml` – original: 2 replicas with required pod anti-affinity.
- `hello-deployment_updated.yaml` – anti-affinity removed, `nodeSelector: disktype=ssd`, resource requests/limits.

## 14. Field description (`hello-deployment.yaml`)
| Field | Meaning |
|---|---|
| `kind: Deployment` | A controller that manages a ReplicaSet and supports rolling updates/rollbacks. |
| `replicas: 2` | Two identical `hello` pods must always be running. |
| `strategy` | `RollingUpdate` with `maxUnavailable: 100%` – during an update all old pods may be stopped at once (fast, but causes downtime). |
| `spec.template.spec.affinity` | `podAntiAffinity` `requiredDuringSchedulingIgnoredDuringExecution` on label `app=hello` with `topologyKey: kubernetes.io/hostname` – two `hello` pods can never share a node. |
| `spec.template.spec.containers` | One container `hello-from`, image `pbitty/hello-from:latest`, named port `http` 80. |

## Steps
```bash
# 15. Original deployment – pods land on different nodes
kubectl apply -f hello-deployment.yaml
kubectl get pods -l app=hello -o wide         # -> outputs/t3-antiaffinity-pods.txt

# a. delete
kubectl delete -f hello-deployment.yaml
# b-c. label preferred node
kubectl label nodes minikube-m02 disktype=ssd
kubectl get nodes --show-labels               # / -L disktype -> outputs/t3-node-labels.txt
# d. updated file (anti-affinity deleted, nodeSelector added)
kubectl apply -f hello-deployment_updated.yaml
kubectl get pods -l app=hello -o wide         # -> outputs/t3-nodeselector-pods.txt

# 16-17. Node labels, resources, requests/limits
kubectl describe node minikube-m02            # -> outputs/t3-node-describe.txt
minikube addons enable metrics-server
kubectl top nodes ; kubectl top pods          # -> outputs/t3-top-*.txt
kubectl describe pod <hello-pod>              # -> outputs/t3-pod-describe.txt (QoS Burstable)

# 18. Node failure / drain
kubectl drain minikube-m02 --ignore-daemonsets --delete-emptydir-data
kubectl get pods -o wide                      # -> outputs/t3-drained-pods.txt
kubectl uncordon minikube-m02
kubectl get pods -o wide                      # -> outputs/t3-recovered-pods.txt
```

## Results
- Original file: one pod on `minikube`, one on `minikube-m02` (anti-affinity).
- Updated file: both pods on `minikube-m02` (the only node with `disktype=ssd`).
- Requests `cpu 100m / mem 64Mi`, limits `cpu 200m / mem 128Mi` → QoS class **Burstable**.
  `kubectl describe node` shows them under *Allocated resources*.

## Drain and recovery
`kubectl drain` cordons the node (`SchedulingDisabled`) and evicts its pods. Deployment-owned pods
(nginx, web, redis) were recreated by their ReplicaSets on `minikube`. The `hello` pods stayed
**Pending** because their `nodeSelector` only matches the drained node. After
`kubectl uncordon minikube-m02` the scheduler placed them back on `minikube-m02` and they became Running.
Pods that were moved do not migrate back automatically – only new pods use the uncordoned node.

## Issue met
`FailedScheduling: 0/2 nodes are available: 2 node(s) had untolerated taint(s)` – appeared while
`minikube-m02` was briefly `NotReady` (node gets the `node.kubernetes.io/not-ready` taint). Once the
kubelet reported Ready the taint was removed and the pod was scheduled automatically.
