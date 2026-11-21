# DevOps – Assignment 3 (Kubernetes)

**Student:** Abdul Salam  **Reg. No:** 04072312009

Environment: Fedora host, Minikube v1.39 (Docker driver), Kubernetes v1.37.0, containerd 2.3.4, 2 nodes (`minikube`, `minikube-m02`).

| Folder | Task |
|---|---|
| [task1-cluster](task1-cluster/) | Two-node cluster, 3-replica deployment, self-healing |
| [task2-compose-to-k8s](task2-compose-to-k8s/) | Docker Compose → Kubernetes with Kompose, scaling, load balancing |
| [task3-scheduling](task3-scheduling/) | Anti-affinity, nodeSelector, resources, drain/recovery |
| [task4-mysql-phpmyadmin](task4-mysql-phpmyadmin/) | MySQL StatefulSet + PV/PVC, phpMyAdmin |
| [task5-configmap-secret](task5-configmap-secret/) | ConfigMap & Secret as env vars and volumes |
| [task6-services-ingress](task6-services-ingress/) | ClusterIP, NodePort, Ingress |
| [task7-troubleshooting](task7-troubleshooting/) | Fixing a broken manifest |

- `outputs/` – raw command outputs captured with `tee` during the lab.
- `Screenshots/` – terminal screenshots.
- `report/` – LaTeX report (`report.tex`) and compiled `report.pdf`.

## Quick start

```bash
minikube start --driver=docker --nodes=2
minikube addons enable ingress
minikube addons enable metrics-server
```

Then follow the README inside each task folder.
