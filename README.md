# Snipe-IT on Kubernetes

A containerized deployment of [Snipe-IT](https://snipeitapp.com/) (open-source IT asset management), backed by MariaDB and a dedicated Redis instance for cache/session/queue.

These manifests are written for a **local/on-prem Kubernetes cluster** (bare-metal nodes, Calico CNI, no cloud load balancer). If you're deploying to **AKS** instead, see the callout notes under each file below — the changes are small and isolated to storage and networking.

## Files

| File | Purpose |
|---|---|
| `mariadb-pv.yaml` | Static local PersistentVolume for MariaDB data (pinned to a specific node) |
| `mariadb-pvc.yaml` | Claims the above PV |
| `mariadb-deployment.yaml` | MariaDB deployment |
| `mariadb-service.yaml` | Internal ClusterIP service for MariaDB |
| `snipeit-redis-deployment.yaml` | Dedicated Redis instance for Snipe-IT (cache/session/queue) |
| `snipeit-redis-service.yaml` | Internal ClusterIP service for the above |
| `snipeit-deployment.yaml` | Snipe-IT web app deployment |
| `snipeit-service.yaml` | Exposes the app (NodePort on local cluster) |

## Prerequisites

- A Kubernetes cluster with `kubectl` context configured
- On the node you pin MariaDB's PV to, create the storage path first:
  ```bash
  sudo mkdir -p /mnt/mariadb-data
  sudo chmod 777 /mnt/mariadb-data
  ```
  (Edit the `nodeAffinity` hostname in `mariadb-pv.yaml` to match whichever node you actually want to use.)

## Required secrets

Create these before deploying — **do not commit secret values to this repo**:

```bash
kubectl create secret generic mariadb-secret \
  --from-literal=MYSQL_PASSWORD='<your-db-password>'

kubectl create secret generic snipeit-secret \
  --from-literal=APP_KEY='<your-laravel-app-key>'
```

Generate a fresh `APP_KEY` per environment — never reuse a key across deployments once real data exists.

## Deploying

Apply in this order:

```bash
kubectl apply -f mariadb-pv.yaml
kubectl apply -f mariadb-pvc.yaml
kubectl apply -f mariadb-deployment.yaml
kubectl apply -f mariadb-service.yaml
kubectl apply -f snipeit-redis-deployment.yaml
kubectl apply -f snipeit-redis-service.yaml
kubectl apply -f snipeit-deployment.yaml
kubectl apply -f snipeit-service.yaml
```

Check the app is reachable:

```bash
kubectl get svc snipeit-nginx
```

On a local cluster (`NodePort`), the app is reachable at `http://<any-node-IP>:<assigned-port>` — grab the port from the `PORT(S)` column (e.g. `80:32098/TCP` means port `32098`). Update `APP_URL` in `snipeit-deployment.yaml` to match this exact address, then reapply:

```bash
kubectl apply -f snipeit-deployment.yaml
```

Snipe-IT/Laravel uses `APP_URL` to build every absolute link (assets, redirects, API calls) — if it doesn't match how you're actually reaching the app, expect broken redirects even though the pod is healthy.

## Deploying to AKS instead

Three files need changes if you're deploying to AKS rather than a local cluster:

**`mariadb-pvc.yaml`** — AKS provides its own managed storage; you don't need `mariadb-pv.yaml` at all on AKS. Change:
```yaml
storageClassName: local-storage
```
to:
```yaml
storageClassName: managed-csi
```
and skip applying `mariadb-pv.yaml` entirely.

**`snipeit-service.yaml`** — AKS integrates with Azure's Load Balancer, so use that instead of NodePort:
```yaml
type: LoadBalancer
```
Get the assigned public IP with `kubectl get svc snipeit-nginx --watch` (takes a minute or two to populate `EXTERNAL-IP`).

**`snipeit-deployment.yaml`** — Set `APP_URL` to that LoadBalancer's `EXTERNAL-IP` instead of a NodePort address:
```yaml
- name: APP_URL
  value: "http://<load-balancer-external-ip>"
```

Everything else (MariaDB, Redis, secrets, deployment order) is identical between the two environments.

## Verifying Redis is actually being used

```bash
kubectl exec -it deploy/snipeit-redis -- redis-cli MONITOR
```
Leave this running and use the app in another tab — you should see live `SET`/`GET` commands streaming through. A stronger test: log into the app, then `kubectl delete pod -l app=snipeit-nginx` and refresh — if you're still logged in after the pod is fully recreated, sessions are confirmed stored in Redis rather than on the pod's local disk.
