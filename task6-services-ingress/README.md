# Task 6 – Services and Ingress

## Files
- `webapp.yaml` – Deployment `webapp` (3 × `nginxdemos/hello:plain-text`), Service `webapp-clusterip`,
  Service `webapp-nodeport` (30080) and Ingress `webapp-ingress` (host `webapp.local`, class `nginx`).

## Steps
```bash
minikube addons enable ingress
kubectl apply -f webapp.yaml
kubectl wait --for=condition=Ready pod -l app=webapp --timeout=90s
kubectl get svc,ingress,endpoints              # -> outputs/t6-svc-ingress.txt

# 1. ClusterIP – only reachable inside the cluster
kubectl run tester --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://webapp-clusterip
                                               # -> outputs/t6-clusterip.txt
# 2. NodePort – reachable on every node IP
curl -s http://$(minikube ip):30080            # -> outputs/t6-nodeport.txt

# 3. Ingress
echo "$(minikube ip) webapp.local" | sudo tee -a /etc/hosts
curl -s http://webapp.local                    # -> outputs/t6-ingress.txt
for i in $(seq 1 6); do curl -s http://webapp.local | grep "Server name"; done   # -> outputs/t6-ingress-lb.txt
```

## Comparison
| Type | Address | Reachable from |
|---|---|---|
| ClusterIP | 10.108.123.18:80 | Pods inside the cluster only |
| NodePort | 192.168.49.2:30080 | Anything that can reach a node IP |
| Ingress | http://webapp.local (192.168.49.2:80) | Outside, by host name / path (L7) |

## Request flow: Browser → Ingress → Service → Pod
1. Browser resolves `webapp.local` to `192.168.49.2` (`/etc/hosts`) and sends `GET /` with `Host: webapp.local`.
2. The **ingress-nginx controller** pod on port 80 matches the host and path `/` in `webapp-ingress`.
3. The rule's backend is Service `webapp-clusterip:80`; the controller reads its **endpoints**
   (`10.244.0.19, 10.244.1.16, 10.244.1.17`).
4. The request is proxied to one of those **pod IPs** on port 80, possibly on another node via the CNI (kindnet).
5. The pod responds; the reply returns through the controller to the browser.

The six Ingress requests were answered by all three pods (`mgvqq`, `zzkhz`, `kcb5g`), confirming load balancing.
