# Task 5 – ConfigMaps and Secrets

## Files
- `config-secret.yaml` – ConfigMap `app-config` (`APP_COLOR`, `APP_MODE`) and Secret `app-secret` (`DB_USER`, `DB_PASSWORD`).
- `config-demo.yaml` – busybox Deployment that prints `$APP_COLOR` (env), `$DB_USER` (env) and `/etc/config/APP_COLOR` (file) every 10 s.

How they are consumed:
| Source | As env var | As volume |
|---|---|---|
| ConfigMap `app-config` | `APP_COLOR` via `configMapKeyRef` | `/etc/config/APP_COLOR`, `/etc/config/APP_MODE` |
| Secret `app-secret` | all keys via `envFrom.secretRef` | `/etc/secret/DB_USER`, `/etc/secret/DB_PASSWORD` (readOnly) |

## Steps
```bash
kubectl apply -f config-secret.yaml      # APP_COLOR: blue initially
kubectl apply -f config-demo.yaml
kubectl logs deploy/config-demo          # -> outputs/t5-initial-logs.txt
kubectl exec deploy/config-demo -- ls /etc/config /etc/secret   # -> outputs/t5-mounts.txt
kubectl exec deploy/config-demo -- env | grep -E "APP_|DB_"     # -> outputs/t5-env.txt

# 28. Update the ConfigMap
sed -i 's/APP_COLOR: blue/APP_COLOR: green/' config-secret.yaml
kubectl apply -f config-secret.yaml
kubectl logs -f deploy/config-demo       # -> outputs/t5-restarted-logs.txt

kubectl rollout restart deploy/config-demo   # env picks up new value only after restart

# 29. Secret is only base64-encoded
kubectl get secret app-secret -o yaml    # -> outputs/t5-secret-yaml.txt
kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

## Observed behaviour
After changing `APP_COLOR` to `green`, the logs changed from `ENV color=blue | FILE color=blue` to
`ENV color=blue | FILE color=green` after about a minute:
- **Volume-mounted** keys are refreshed automatically by the kubelet (sync period ~60 s).
- **Environment variables** are fixed when the container starts; they only change after the pod is
  recreated (`kubectl rollout restart`).

## ConfigMap vs Secret
| | ConfigMap | Secret |
|---|---|---|
| Purpose | Non-sensitive config (colours, modes, URLs) | Passwords, tokens, keys |
| Storage | Plain text | Base64-encoded (`S3cr3tPass` → `UzNjcjN0UGFzcw==`); can be encrypted at rest |
| Volume | Normal files | tmpfs (memory), can be `readOnly` |
| Access control | Normal RBAC | Usually stricter RBAC |
| Size limit | 1 MiB | 1 MiB |

Base64 is encoding, not encryption – anyone with `get secret` permission can decode it.

## Issue met
After `kubectl rollout restart`, `kubectl logs deploy/config-demo` printed
`Found 2 pods, using pod/config-demo-84c4fcb56d-gz78d` – it read the **old, terminating** pod, so the
output still showed `ENV color=blue`. Fix: wait for `kubectl rollout status deploy/config-demo` to
finish, or read logs from the new pod by name/label.
