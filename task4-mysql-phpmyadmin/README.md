# Task 4 – MySQL (StatefulSet) and phpMyAdmin

## Files
- `mysql-pv-pvc.yaml` – PersistentVolume `mysql-pv` (1Gi, hostPath `/data/mysql`, `Retain`, pinned to node `minikube`) and PVC `mysql-pvc`.
- `mysql.yaml` – Secret `mysql-secret` (root password), headless Service `mysql`, StatefulSet `mysql` with the PVC mounted at `/var/lib/mysql`.
- `phpmyadmin.yaml` – Deployment `phpmyadmin` (`PMA_HOST=mysql`) and NodePort Service on port 30090.

## Steps
```bash
kubectl apply -f mysql-pv-pvc.yaml
kubectl apply -f mysql.yaml
kubectl apply -f phpmyadmin.yaml
kubectl get pv,pvc                    # -> outputs/t4-pv-pvc.txt (Bound)
kubectl get pods -w                   # mysql-0 and phpmyadmin Running

# 22. Sample data
kubectl exec -i mysql-0 -- mysql -uroot -prootpass123 <<'SQL'
CREATE DATABASE IF NOT EXISTS school;
USE school;
CREATE TABLE students (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50), reg_no VARCHAR(20));
INSERT INTO students (name, reg_no) VALUES ('Student One','001'), ('Student Two','002');
SELECT * FROM students;
SQL

# 23. Delete pod, StatefulSet recreates mysql-0 with the same PVC
kubectl delete pod mysql-0
kubectl wait --for=condition=Ready pod/mysql-0 --timeout=120s
sleep 20   # give mysqld time to open its socket
kubectl exec -i mysql-0 -- mysql -uroot -prootpass123 -e "SELECT * FROM school.students;"  # -> outputs/t4-persistence.txt

# 24. phpMyAdmin
echo "http://$(minikube ip):30090"    # login root / rootpass123
```

## PV–PVC architecture
```
mysql-0 (StatefulSet pod) ──volumeMount /var/lib/mysql──► volume "mysql-data"
      └─► PVC mysql-pvc (request 1Gi, RWO, class manual)
             └─ bound ─► PV mysql-pv (1Gi, hostPath /data/mysql on node minikube, Retain)
```
- **PV** is a cluster-level piece of storage created by the admin (static provisioning).
- **PVC** is the pod's request for storage; Kubernetes binds it to a matching PV (size, access mode, storageClass).
- The pod only references the PVC, so when `mysql-0` is deleted the new `mysql-0` re-mounts the same
  directory and the data survives. `Retain` keeps the data even if the PVC is deleted.
- The PV's `nodeAffinity` forces MySQL onto node `minikube`, where the hostPath directory lives.

## Issue met
Right after `kubectl wait` reported `mysql-0` Ready, the query failed with
`ERROR 2002 (HY000): Can't connect to local MySQL server through socket`. The container is "Ready"
as soon as it starts (no readiness probe) while mysqld is still initialising. Fix: wait a few
seconds and retry, or add a readiness probe such as `exec: mysqladmin ping -uroot -p$MYSQL_ROOT_PASSWORD`.
