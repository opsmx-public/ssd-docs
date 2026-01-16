# SSD Instance – Data Reset Guide

This document describes the **standard and safe procedure to reset data for an SSD instance** running on Kubernetes. This is typically required for:
- Demo environment cleanup
- Fresh re-install / re-test
- Corrupted or inconsistent data

---

## ⚠️ Important Notes

- This process **permanently deletes data**.
- Take backups if data needs to be preserved.
- Run commands with appropriate cluster-admin permissions.
- Replace `<NAMESPACE>` with your SSD namespace.

---

## Step 1: Scale Down SSD Workloads

Scale down **all StatefulSets and Deployments**, **except** `license-generator`.

```bash
kubectl scale sts,deploy --all --replicas=0 -n <NAMESPACE>
```

### Verify

```bash
kubectl get pods -n <NAMESPACE>
```

Ensure all pods are stopped except `license-generator`.

---

## Step 2: Delete Persistent Volume Claims (PVCs)

Delete PVCs related to databases and stateful components.

### List PVCs

```bash
kubectl get pvc -n <NAMESPACE>
```

### Common PVCs to Delete

```bash
kubectl delete pvc datadir-dgraph-0 -n <NAMESPACE>

kubectl delete pvc redis-data-ssd-redis-master-0 -n <NAMESPACE>

kubectl delete pvc ssd-db-postgresql-ssd-db-0 -n <NAMESPACE>
```

> ⚠️ These PVC deletions will reset:
> - Dgraph data
> - Redis cache/state
> - PostgreSQL (SSD DB)

---

## Step 3: Reset MinIO (Temporal & Scan Data)

SSD stores scan artifacts and Temporal data in **MinIO** under the `ssd-temporal` path.

You can reset MinIO data using **any one** of the following approaches.

---

### Option A: Delete and Recreate MinIO PVC (Recommended for Full Reset)

```bash
kubectl delete pvc ssd-minio -n <NAMESPACE>
```

- This wipes **all MinIO data**
- PVC will be recreated automatically when MinIO pod starts
- If required, you can recreate PVC using YAML from a working instance

---

### Option B: Delete Data via MinIO UI (Selective Cleanup)

1. Port-forward the MinIO pod:

```bash
kubectl port-forward pod/ssd-minio-0 9000:9000 -n <NAMESPACE>
```

2. Open browser:

```text
http://localhost:9000
```

3. Login using MinIO credentials
4. Navigate to the bucket / folder:

```text
ssd-temporal/
```

5. Delete all subfolders such as:

```text
sbom/
codeSecretScan/
codeLicenseScan/
codeVulnScan/
```

> Use this option if you want to keep MinIO but reset only SSD scan data

---

### Option C: If Using External S3 (AWS / Compatible Object Store)

If SSD Temporal data is stored in **S3** instead of in-cluster MinIO:

1. Login to S3 console / CLI
2. Navigate to:

```text
ssd-temporal/
```

3. Delete all subfolders:

```text
sbom/
codeSecretScan/
codeLicenseScan/
codeVulnScan/
```

---

## Step 4: Scale Up SSD Components

After cleanup is complete, scale SSD workloads back up.

```bash
kubectl scale sts,deploy --all --replicas=1 -n <NAMESPACE>
```

> Adjust replica count if required for HA setups

---

## Step 5: Verify SSD Health

### Check Pods

```bash
kubectl get pods -n <NAMESPACE>
```

### Check Logs (Optional)

```bash
kubectl logs deploy/ssd-gate -n <NAMESPACE>
```

Ensure:
- No DB migration errors
- SSD UI loads successfully
- New scans start cleanly

---

## Cleanup Summary

| Component | Action |
|---------|-------|
| StatefulSets / Deployments | Scaled down |
| PostgreSQL (ssd-db) | PVC deleted |
| Redis | PVC deleted |
| Dgraph | PVC deleted |
| MinIO | PVC deleted OR folders cleaned |

---

## Best Practices

- Prefer **Option A** (MinIO PVC delete) for demos and test resets
- Use **grouped PVC deletion** carefully in shared clusters
- Always verify namespace before running delete commands
- Keep this guide handy for demo-day recoveries

---

**Maintained by:** OpsMx DevOps Team  
**Applies to:** SSD Demo / Dev Environments
