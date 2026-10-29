# Day 19 - Storage & StatefulSets

DevOps Winter Arc | Day 19/90 | Week 3: Kubernetes

## Objective

Run PostgreSQL as a StatefulSet with persistent storage, insert a test
record, delete the Pod, and prove that the record survives Pod recreation.

## Key concepts

| Concept | Meaning | In this lab |
|---|---|---|
| PV | Storage resource available to the cluster | Created automatically for `data-pg-0` |
| PVC | A workload's request for storage (size, access mode) | `data-pg-0`, 1Gi, ReadWriteOnce |
| StorageClass | Describes a class of storage and enables dynamic provisioning | `[your StorageClass name]` |
| StatefulSet | Pods with stable identity and per-Pod storage | `pg` creating `pg-0` |
| Headless Service | `clusterIP: None`, DNS records for individual Pods | `pg-headless` |
| volumeClaimTemplates | Creates one PVC per StatefulSet Pod identity | Template named `data` |

A container's writable layer is tied to the container lifecycle, so it is not
a safe place for database files. Persistent storage is designed to outlive a Pod.

## Storage chain

```
StatefulSet Pod (pg-0) -> PVC (data-pg-0) -> PV (pvc-...) -> storage provisioner -> node path/disk
```

- The StatefulSet's `volumeClaimTemplates` creates the PVC `data-pg-0`.
- The StorageClass triggers dynamic provisioning of a PV for that claim.
- The PV is bound to the PVC, and the PVC is mounted into `pg-0` at
  `/var/lib/postgresql/data`.
- PV name in my cluster: **[from `kubectl get pv`]**
- Provisioner: **[from `kubectl get storageclass`]**
- Binding mode: **[Immediate / WaitForFirstConsumer]**

## Files

- `pg-headless.yaml`: Headless Service for stable Pod DNS identity
- `postgres-statefulset.yaml`: PostgreSQL 16 StatefulSet with a 1Gi PVC
- `evidence/`: screenshots

The database password comes from a Kubernetes Secret created with
`kubectl create secret` using a dummy lab value. It is not stored in any
file in this repository.

## Commands and steps

### 1. Inspect the cluster storage

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc -A
kubectl describe storageclass [name]
```

Default StorageClass found: **[name]**

### 2. Namespace and Secret

```bash
kubectl create namespace day19-storage
kubectl create secret generic pg-secret \
  --from-literal=POSTGRES_PASSWORD='<dummy-password>' \
  -n day19-storage
```

### 3. Headless Service

```bash
kubectl apply -f pg-headless.yaml
kubectl get svc -n day19-storage
```

`CLUSTER-IP` shows `None`, so there is no virtual IP. DNS returns Pod
addresses and gives each Pod a stable name such as
`pg-0.pg-headless.day19-storage.svc.cluster.local`.

### 4. PostgreSQL StatefulSet

```bash
kubectl apply -f postgres-statefulset.yaml
kubectl get statefulset,pods,pvc -n day19-storage -w
```

Result: PVC `data-pg-0` became **[Bound]** and `pg-0` became **[Running 1/1]**.

Evidence: `evidence/[pvc-bound.png]`

### 5. Insert a record

```bash
kubectl exec -it -n day19-storage pg-0 -- psql -U appuser -d winterarc
```

```sql
CREATE TABLE lab_notes (id INT PRIMARY KEY, note TEXT);
INSERT INTO lab_notes VALUES (1, 'data-survives-pod-delete');
SELECT * FROM lab_notes;
```

Row before deletion: **[paste the SELECT output]**

Evidence: `evidence/[row-before.png]`

### 6. Delete the Pod and verify persistence

```bash
kubectl get pvc -n day19-storage
kubectl delete pod pg-0 -n day19-storage
kubectl get pods -n day19-storage -w
kubectl exec -n day19-storage pg-0 -- \
  psql -U appuser -d winterarc -c "SELECT * FROM lab_notes;"
```

- PVC after Pod deletion: **still Bound**
- Pod recreation: **[describe what you saw, same name pg-0]**
- Row after recreation: **[paste the SELECT output]**

Evidence: `evidence/[pod-recreation.png]`, `evidence/[row-after.png]`

The data survived because the StatefulSet recreated the Pod with the same
identity (`pg-0`), and the replacement reattached to the same PVC
(`data-pg-0`), which is still bound to the same PV.

### 7. Inspect the storage chain

```bash
kubectl get statefulset,pods,pvc,pv -n day19-storage
kubectl describe pvc data-pg-0 -n day19-storage
kubectl get events -n day19-storage --sort-by=.lastTimestamp
```

From `describe pvc`: Status **[Bound**, Volume **pvc-9c0888e6-6ec9-42e1-9e49-c18e0ea0b51c**,
StorageClass **Standard**, Used By **pg-0**.



## Why a PVC can stay Pending

I would inspect `kubectl describe pvc data-pg-0 -n day19-storage` first and
read the Events before changing anything. Possible causes:

1. **Missing or mismatched StorageClass.** The `storageClassName` in the
   manifest does not exist in the cluster.
2. **Provisioner problem.** The component that creates volumes is not running
   or is failing, so no PV can be provisioned.
3. **Unsatisfiable request.** The requested size or access mode cannot be
   provided by the storage backend.
4. **WaitForFirstConsumer binding mode.** The claim stays Pending until a Pod
   using it is scheduled. This is expected behavior in some clusters.

What I observed in my cluster: ** that the PVC bound without problems**

## Troubleshooting / lessons learned
- Deleting only a Pod does not delete its PVC.
- Check the StorageClass name in your cluster before applying the manifest.
- Postgres needs a few seconds after the container starts. Check
  `kubectl logs pg-0 -n day19-storage` for "ready to accept connections".

## Interview questions

**Q1. PV vs PVC?**
A PV represents storage in the cluster. A PVC is a workload's request for
storage. Kubernetes binds a matching claim to a volume.

**Q2. Why not store database files in the container filesystem?**
The writable layer is tied to the container lifecycle. Persistent storage is
designed to outlive a Pod.

**Q3. What does a StorageClass do?**
It describes a class of storage and can trigger dynamic provisioning when a
PVC requests storage.

**Q4. What does volumeClaimTemplates do in a StatefulSet?**
It creates a dedicated PVC for each Pod identity, so a replacement Pod can
reuse its claim.

**Q5. Why did the data survive deleting pg-0?**
The StatefulSet recreated the Pod with the same identity, and it reattached
to the same PVC and volume.

**Q6. Deployment vs StatefulSet?**
Deployment replicas are generally interchangeable. StatefulSets give stable
identities and ordered management, often with per-Pod storage.

**Q7. What does ReadWriteOnce mean?**
The volume can be mounted read-write by one node at a time. It does not simply
mean only one Pod can ever use it. Details depend on the storage driver.

**Q8. Why use a Headless Service?**
It provides DNS identities for individual StatefulSet Pods instead of a single
virtual service IP.

**Q9. Why can a PVC stay Pending?**
The StorageClass may be missing or wrong, the provisioner may be unhealthy, or
the request may not be satisfiable. Inspect PVC Events first.

**Q10. Does deleting a StatefulSet Pod delete its PVC?**
Normally no. The claim from volumeClaimTemplates stays. Verify retention
behavior before cleanup.

**Q11. What is the risk of deleting a PVC?**
Depending on the reclaim policy and storage setup, the underlying data may be
deleted or retained. Treat PVC deletion as potentially destructive.

**Q12. Is persistent storage the same as a backup?**
No. It protects against Pod and container failures but does not replace
backups, restore tests, or disaster recovery.

## Security notes

- Only a dummy lab password was used, created with `kubectl create secret`.
- No Secret values, kubeconfig files, or tokens are committed.
- Screenshots do not show the password value.

## Cleanup

Run only after all evidence is saved. This is a disposable local lab.

```bash
kubectl delete namespace day19-storage
kubectl get pv
```

Deleting the namespace removes the StatefulSet and PVC. What happens to the
underlying PV and data depends on the reclaim policy and provisioner.
After cleanup, `kubectl get pv` showed: **[working when I delete ]**
