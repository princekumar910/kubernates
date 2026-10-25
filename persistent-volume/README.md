# Persistent Volume with nginx on kind – Notes

Hands-on notes from running nginx on a local **kind** cluster with a
**Service**, **PersistentVolume (PV)**, **PersistentVolumeClaim (PVC)** and
**Kustomize**.

## Files in this folder

| File | What it does |
|---|---|
| [kustomization.yaml](./kustomization.yaml) | Lists all the files so they can be applied with one command |
| [deployment.yaml](./deployment.yaml) | Runs the nginx pod and mounts the PVC at `/usr/share/nginx/html` (read-only) |
| [service.yaml](./service.yaml) | `NodePort` Service exposing nginx on node port `30080` |
| [persistentVolume.yaml](./persistentVolume.yaml) | PV backed by a `hostPath` folder `/data/nginx` on the node |
| [persistentVolumeClaim.yaml](./persistentVolumeClaim.yaml) | PVC that requests storage from the PV |

Flow:

```
Browser → localhost:8080 → kind node :30080 → nginx-service :80 → nginx pod :80
                                                                   │
                                     /usr/share/nginx/html ← PVC ← PV ← node folder /data/nginx
```

---

## 1. Kustomize – apply many files with one command

Instead of applying each file, list them in `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - persistentVolume.yaml
  - persistentVolumeClaim.yaml
```

```bash
kubectl apply -k .       # apply everything (note: -k, not -f)
kubectl kustomize .      # preview the combined YAML
kubectl delete -k .      # delete everything
```

Alternatives:
- One file with resources separated by `---` → `kubectl apply -f app.yaml`
- Apply a whole folder → `kubectl apply -f .`

---

## 2. `kubectl describe` needs a resource type

```bash
kubectl describe nginx-deployment-7fcfddfbf6-dv4g4             # ❌ no type
kubectl describe deployment/nginx-deployment-7fcfddfbf6-dv4g4  # ❌ that is a pod name
kubectl describe pod nginx-deployment-7fcfddfbf6-dv4g4         # ✅
kubectl describe deployment nginx-deployment                   # ✅
kubectl describe service nginx-service                         # ✅
```

Pod name format: `<deployment-name>-<replicaset-hash>-<random>`.

---

## 3. NodePort on kind & port mapping

- A `NodePort` (range **30000–32767**) opens the port on the **nodes**.
- In kind, nodes are **Docker containers**, so the Mac cannot reach a NodePort
  unless kind maps it with `extraPortMappings` in the cluster config
  ([../kind-cluster/config.yaml](../kind-cluster/config.yaml)):

```yaml
extraPortMappings:
  - containerPort: 30080   # NodePort inside the kind node
    hostPort: 8080         # port on the Mac (localhost)
```

- So `nodePort: 30080` is reached at **http://localhost:8080**, not
  `localhost:30080`.
- `nodePort: 30007` did not work because it was not mapped.
- To use `localhost:30080`, set `hostPort: 30080` and **recreate the
  cluster** (port mappings only apply at cluster creation).
- Ports go up to 65535 – `300080` is not a valid port.
- No mapping needed for quick testing:

```bash
kubectl port-forward service/nginx-service 9090:80   # → http://localhost:9090
```

`nodePort` belongs **inside** the `ports` entry:

```yaml
spec:
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

---

## 4. Access modes

Valid values: `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`,
`ReadWriteOncePod` (`ReadOnly` is **not** valid).

| Mode | Short | Meaning |
|---|---|---|
| ReadWriteOnce | RWO | read-write by one **node** |
| ReadOnlyMany | ROX | read-only by many nodes |
| ReadWriteMany | RWX | read-write by many nodes |
| ReadWriteOncePod | RWOP | read-write by one **pod** |

> ⚠️ Access modes are used for **matching PV ↔ PVC**, they do **not** make the
> mount read-only. With `ReadOnlyMany` you could still write to the files.

To actually enforce read-only, set `readOnly: true` in the Deployment:

```yaml
volumeMounts:
  - mountPath: /usr/share/nginx/html
    name: nginx-volume
    readOnly: true
volumes:
  - name: nginx-volume
    persistentVolumeClaim:
      claimName: nginx-pvc
      readOnly: true
```

Writing now fails with `Read-only file system`.

---

## 5. PVC stuck in `Pending` – StorageClass mismatch

kind has a default StorageClass called `standard`. A PVC without
`storageClassName` gets `standard`, but the PV had none → they never matched.

Fix: use the **same** `storageClassName` on both:

```yaml
# PV and PVC
spec:
  storageClassName: manual
```

A PVC binds to a PV only if **storageClassName, accessModes and size** match
(PV capacity ≥ PVC request).

```bash
kubectl get pv,pvc    # STATUS should be Bound
```

---

## 6. `hostPath` is a folder on the node, not on your Mac

- `kubectl` only sends YAML to the API server – it never creates folders on
  your machine.
- The **kubelet** on the node running the pod creates/uses the folder.
- In kind, that node is a Docker container, so the folder is **inside the
  container**:

```bash
docker exec k8s-cluster-worker2 ls /data/nginx
```

- `hostPath.path` must be **absolute** (`/data/nginx`), not `./data` – the
  node has no idea which folder you ran `kubectl` from.
- `type: DirectoryOrCreate` creates the folder if it does not exist.

To see the folder on the Mac, mount it into the kind nodes (requires
recreating the cluster):

```yaml
- role: worker
  extraMounts:
    - hostPath: /Users/princekumar/Desktop/Prince Kumar/kubernetes/persistent-volume/data
      containerPath: /data/nginx
```

---

## 7. `Forbidden: ... is immutable`

Some fields cannot be changed after creation. Changing them and re-applying
gives `Forbidden`.

| Resource | Immutable fields |
|---|---|
| PersistentVolume | `hostPath` / any volume source, `nodeAffinity` |
| PersistentVolumeClaim | `accessModes`, `storageClassName`, `volumeName` (size can usually only increase) |
| Deployment | `spec.selector` |
| Service | `clusterIP` |

Fix: delete and recreate.

```bash
kubectl delete -k .
kubectl apply -k .
```

---

## 8. nginx `403 Forbidden` & data on different nodes

`403 Forbidden` from nginx = the web folder has **no `index.html`**
(log: `directory index of "/usr/share/nginx/html/" is forbidden`).

It happened because the pod moved to another node. With `hostPath`,
**every node has its own separate `/data/nginx`**, so data is inconsistent:

| Node | `/data/nginx` |
|---|---|
| k8s-cluster-worker2 | `index.html` ✅ |
| k8s-cluster-worker | empty ❌ → 403 |

Ways to fix it:

| Option | How |
|---|---|
| `nodeAffinity` on the PV | Pin the PV (and its pods) to one node |
| `local` PV | Like `hostPath`, but `nodeAffinity` is required |
| kind `standard` StorageClass | Only create a PVC; kind creates a PV pinned to a node |
| Shared storage (NFS, cloud disk) | Any node can mount the same data – used in real clusters |
| kind `extraMounts` | Same Mac folder mounted into every node |

Pin the PV to one node:

```yaml
spec:
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - k8s-cluster-worker2
```

Trade-off: if that node goes down, the pod cannot run anywhere else.

---

## Useful commands

```bash
kubectl apply -k .                         # create / update everything
kubectl delete -k .                        # delete everything
kubectl get pods -o wide                   # pods + which node they run on
kubectl get svc -o wide                    # services + selectors
kubectl get pv,pvc                         # volume binding status
kubectl get endpoints nginx-service        # pods behind the service
kubectl describe pod <pod-name>            # events / errors
kubectl logs <pod-name>                    # nginx logs
kubectl exec -it <pod-name> -- bash        # shell inside the pod
kubectl rollout status deployment/nginx-deployment
docker exec <node-name> ls /data/nginx     # look inside a kind node
```
