# StatefulSet with MongoDB Replica Set on kind – Notes

Hands-on notes from running a 3-member **MongoDB replica set** on a local
**kind** cluster using a **StatefulSet**, a **headless Service**,
**volumeClaimTemplates** and **Kustomize**.

## Files in this folder

| File | What it does |
|---|---|
| [kustomization.yaml](./kustomization.yaml) | Lists all the files so they can be applied with one command |
| [namespace.yaml](./namespace.yaml) | Creates the `stateful` namespace |
| [service.yaml](./service.yaml) | Headless Service `mongo-service` (`clusterIP: None`) giving each pod a stable DNS name |
| [deployment.yaml](./deployment.yaml) | StatefulSet running 3 MongoDB pods with `--replSet rs0`, one PVC per pod |

Flow:

```
                     mongo-service (headless, clusterIP: None)
                                    │
        ┌───────────────────────────┼───────────────────────────┐
  stateful-set-0              stateful-set-1              stateful-set-2
        │                           │                           │
 mongo-storage-stateful-set-0  mongo-storage-...-1   mongo-storage-...-2   (PVC per pod)
        │                           │                           │
   PV (auto, local-path)      PV (auto, local-path)      PV (auto, local-path)
```

---

## 1. Deployment vs StatefulSet

| | Deployment | StatefulSet |
|---|---|---|
| Pod names | random (`nginx-7fcf...-dv4g4`) | stable, ordered (`stateful-set-0`, `-1`, `-2`) |
| Start / stop order | all at once | one by one, in order (0 → 1 → 2) |
| Storage | all pods share the same PVC | each pod gets **its own** PVC |
| Network identity | only via Service IP | stable DNS name per pod (needs headless Service) |
| Use for | stateless apps (nginx, APIs) | databases, queues (MongoDB, Kafka, etc.) |

Pod `stateful-set-1` is only created after `stateful-set-0` is **Ready**.

---

## 2. Why one shared PVC was wrong

The first version used a single PV + PVC (`mongo-pvc`) for all 3 replicas:

```yaml
volumes:
- name: mongo-storage
  persistentVolumeClaim:
    claimName: mongo-pvc     # ❌ same claim for every pod
```

Problems:
- All 3 `mongod` processes write to the **same** `/data/db` → lock error
  (`DBPathInUse`) or data corruption.
- `ReadWriteOnce` = one **node** only → pods on other nodes cannot mount it.
- Defeats the point of a StatefulSet (per-pod stable storage).

Fix: `volumeClaimTemplates` – the StatefulSet creates one PVC per pod:

```yaml
volumeClaimTemplates:
- metadata:
    name: mongo-storage
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

PVC names follow `<template-name>-<statefulset-name>-<ordinal>`:

```
mongo-storage-stateful-set-0
mongo-storage-stateful-set-1
mongo-storage-stateful-set-2
```

- No `storageClassName` → uses the default kind StorageClass `standard`
  (`rancher.io/local-path`), which creates PVs **automatically**. No PV file
  needed.
- PVCs are **not deleted** when pods or the StatefulSet are deleted – data
  survives.

> Separate PV files are fine for a single-replica Deployment
> (see [../persistent-volume](../persistent-volume/README.md)). For a
> StatefulSet with static PVs you would need **one PV per pod** and still keep
> the `volumeClaimTemplates`.

---

## 3. Headless Service

`serviceName` in the StatefulSet must match a Service with `clusterIP: None`
and a selector that matches the pod labels:

```yaml
# service.yaml
metadata:
  name: mongo-service
spec:
  clusterIP: None
  selector:
    app: stateful-set
```

```yaml
# deployment.yaml
spec:
  serviceName: "mongo-service"
```

Each pod then gets a stable DNS name:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
stateful-set-0.mongo-service.stateful.svc.cluster.local
```

---

## 4. Other fixes made

| Problem | Fix |
|---|---|
| `kind: namespace` | `kind: Namespace` (kinds are case-sensitive) |
| Container name `smongoDb` | `mongodb` – names must be lowercase (DNS-1123) |
| Service selector `app: mongo` | `app: stateful-set` – must match pod labels |
| `namespace:` on a PV | PVs are cluster-scoped, the field is ignored |

---

## 5. Creating the MongoDB replica set

Start `mongod` in replica set mode:

```yaml
containers:
- name: mongodb
  image: mongo
  args: ["--replSet", "rs0", "--bind_ip_all"]
```

Before initiating, logs show these warnings – they are **normal**:

```
Collection [local.oplog.rs] not found
No primary exists currently
ReadConcernMajorityNotAvailableYet
```

Wait for all pods to be `1/1 Running`, then initiate **once**:

```bash
kubectl exec stateful-set-0 -- mongosh --quiet --eval '
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "stateful-set-0.mongo-service.stateful.svc.cluster.local:27017" },
    { _id: 1, host: "stateful-set-1.mongo-service.stateful.svc.cluster.local:27017" },
    { _id: 2, host: "stateful-set-2.mongo-service.stateful.svc.cluster.local:27017" }
  ]
})'
```

Check:

```bash
kubectl exec stateful-set-0 -- mongosh --quiet --eval 'rs.status().members.map(m => m.name.split(".")[0] + " " + m.stateStr)'
```

```
[ 'stateful-set-0 PRIMARY', 'stateful-set-1 SECONDARY', 'stateful-set-2 SECONDARY' ]
```

The config is stored in each pod's PVC, so restarted pods rejoin
automatically.

Connection string for apps:

```
mongodb://stateful-set-0.mongo-service.stateful,stateful-set-1.mongo-service.stateful,stateful-set-2.mongo-service.stateful:27017/mydatabase?replicaSet=rs0
```

> Auth was removed for now: replica set + username/password requires a shared
> **keyFile** (`--keyFile`), otherwise `mongod` refuses to start.

---

## 6. Testing replication and failover

```bash
# write on the primary
kubectl exec stateful-set-0 -- mongosh --quiet mydatabase --eval 'db.users.insertOne({name:"prince"})'

# read from a secondary
kubectl exec stateful-set-1 -- mongosh --quiet mydatabase --eval 'db.getMongo().setReadPref("secondary"); db.users.find()'
```

Failover:

```bash
kubectl delete pod stateful-set-0
kubectl exec stateful-set-1 -- mongosh --quiet --eval 'rs.status().members.map(m => m.name.split(".")[0] + " " + m.stateStr)'
```

Result:

```
[ 'stateful-set-0 SECONDARY', 'stateful-set-1 PRIMARY', 'stateful-set-2 SECONDARY' ]
```

- A new primary is elected from the remaining members.
- `stateful-set-0` comes back with the **same name and same PVC** and rejoins
  as `SECONDARY`. The primary does not move back automatically.

---

## 7. `ContainerCreating` for a long time

```bash
kubectl logs -f stateful-set-0      # "waiting to start: ContainerCreating"
kubectl describe pod stateful-set-0 # Events: Pulling image "mongo"
```

- Container has not started yet → no logs. Check **Events** in `describe`.
- Images are cached **per node**, not per pod. kind nodes are separate Docker
  containers, so each node downloads the image once.
- `image: mongo` = `:latest` → `imagePullPolicy: Always` (checks the registry
  on every pod start).

Download once and load into all kind nodes:

```bash
docker pull mongo:8.0
kind load docker-image mongo:8.0 --name k8s-cluster
```

Then use `image: mongo:8.0` (a fixed tag → `IfNotPresent`).

---

## 8. Useful commands

```bash
kubectl apply -k .                                          # apply everything
kubectl config set-context --current --namespace=stateful   # skip -n stateful
kubectl get pods -o wide -w                                 # watch pods + nodes
kubectl get pvc                                             # one PVC per pod
kubectl get storageclass                                    # default: standard
kubectl delete -k .                                         # PVCs are kept
kubectl delete pvc -l app=stateful-set                      # delete the data too (PVCs get the selector labels)
```

- `kubectl logs` / `exec` default to the `default` namespace – use
  `-n stateful` or set the context namespace.
- Pasting multiple commands can merge them onto one line (e.g.
  `--namespace=statefulkubectl delete ...`) – run them one at a time.

---

## 9. TODO

- Add credentials via a **Secret** (not a ConfigMap – those are for
  non-sensitive config) plus a keyFile Secret for `--keyFile`.
- Pin the image tag (`mongo:8.0`).
