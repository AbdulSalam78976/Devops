# Task 1 – Cluster Deployment and Investigation

## Files
- `nginx-deployment.yaml` – Deployment `nginx-deploy`, 3 replicas of `nginx:1.27`.

## Steps

```bash
# 1-2. Two-node Minikube cluster with the Docker driver
minikube start --driver=docker --nodes=2
minikube status
kubectl get nodes -o wide            # -> outputs/t1-nodes.txt

# 3. Deployment with 3 replicas (+ ClusterIP service to get a Cluster IP)
kubectl apply -f nginx-deployment.yaml
kubectl expose deployment nginx-deploy --name=nginx-svc --port=80

# 4. Investigation
kubectl get pods -o wide             # -> outputs/t1-pods.txt
kubectl describe deployment nginx-deploy   # -> outputs/t1-deploy.txt
kubectl describe pod <pod-name>      # -> outputs/t1-pod.txt
kubectl get svc nginx-svc            # -> outputs/t1-svc.txt

# 6. Self-healing
kubectl delete pod <pod-name>
kubectl get pods -o wide -w
```

## 5. Identified values

| Item | Value |
|---|---|
| Pod IP (`nginx-deploy-75cc477c85-5lhv8`) | 10.244.0.3 |
| Node IP (`minikube`) / (`minikube-m02`) | 192.168.49.2 / 192.168.49.3 |
| Cluster IP (`nginx-svc`) | 10.97.74.3 |
| Container image | nginx:1.27 |

Pods on `minikube` get IPs from `10.244.0.0/24`, pods on `minikube-m02` from `10.244.1.0/24`.

## 6. Self-healing

Pod `nginx-deploy-75cc477c85-5lhv8` was deleted. The ReplicaSet controller noticed that only
2 of the 3 desired replicas existed and immediately created `nginx-deploy-75cc477c85-fqhd9`
(Running within ~1 s, new IP 10.244.0.4). The desired state stored in etcd (`replicas: 3`) is
continuously reconciled against the actual state, so pods are replaced, never "repaired".

## 7. Architecture

```
                kubectl ──► API server (minikube, control-plane, 192.168.49.2)
                               │ etcd · scheduler · controller-manager
             ┌─────────────────┴──────────────────┐
   Node minikube (192.168.49.2)        Node minikube-m02 (192.168.49.3)
   kubelet · kube-proxy · containerd   kubelet · kube-proxy · containerd
   └ pod nginx 10.244.0.x              ├ pod nginx 10.244.1.2
                                       └ pod nginx 10.244.1.3
             └──── Service nginx-svc (ClusterIP 10.97.74.3:80) ────┘
```
