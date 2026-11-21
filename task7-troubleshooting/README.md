# Task 7 – Troubleshooting Challenge

## Files
- `broken.yaml` – intentionally broken Deployment + Service (5 bugs, marked with comments).
- `fixed.yaml` – corrected manifest (includes the missing ConfigMap).

## Issues, diagnosis and resolution

| # | Symptom | Diagnosis (command) | Cause | Fix |
|---|---|---|---|---|
| 1 | `kubectl apply` rejected the Deployment: `selector does not match template labels` | API server validation message | Pod label `app: brokenapp` ≠ selector `app: broken-app` | Pod label → `app: broken-app` |
| 2 | Pods `ErrImagePull` / `ImagePullBackOff` | `kubectl describe pod` → `nginx:1.999: not found` | Image tag does not exist | Image → `nginx:1.27` |
| 3 | New pod `CreateContainerConfigError` | `kubectl describe pod` → `configmap "app-settings" not found` | Env var references a missing ConfigMap | `kubectl create configmap app-settings --from-literal=mode=prod` (also added to `fixed.yaml`) |
| 4 | Service has no endpoints | `kubectl get endpoints broken-svc` → `<none>` | Service selector `app: broken` matches no pods | Selector → `app: broken-app` |
| 5 | Connection refused through the Service | Compare `targetPort` with `containerPort` | `targetPort: 8080`, nginx listens on 80 | `targetPort: 80` |

## Commands
```bash
kubectl apply -f broken.yaml                    # bug 1 error
kubectl get pods -l app=broken-app
kubectl describe pod <pod> | tail -n 15         # bugs 2, 3
kubectl get endpoints broken-svc                # bug 4
kubectl apply -f fixed.yaml
kubectl get deploy,svc,endpoints,pods -l app=broken-app -o wide
kubectl run tester --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://broken-svc
                                                # -> outputs/t7-fixed-output.txt
```

## Evidence of the fix
- Both pods `broken-app-c6bf9df65-*` are `1/1 Running` (on `minikube` and `minikube-m02`).
- `broken-svc` endpoints: `10.244.0.21:80, 10.244.1.20:80`.
- `wget http://broken-svc` returns the **"Welcome to nginx!"** page.
