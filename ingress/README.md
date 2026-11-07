# Ingress with 3 nginx apps on kind – Notes

Hands-on notes from routing traffic to **three different apps** through **one
entry point** using an **Ingress** and the **ingress-nginx controller** on a
local **kind** cluster. Each app also has its own **PV + PVC** for its web page.

Goal:

```
curl localhost:8083/app1  →  "Hello from App 1"
curl localhost:8083/app2  →  "Hello from App 2"
curl localhost:8083/app3  →  "Hello from App 3"
```

---

## 1. Why do we need Ingress?

Without Ingress, every app needs its **own** way in from outside the cluster:

```
localhost:8080 → NodePort 30080 → app1
localhost:8081 → NodePort 30081 → app2
localhost:8082 → NodePort 30082 → app3
```

Problems:
- One port per app – doesn't scale (10 apps = 10 ports, 10 firewall rules, 10 load balancers in the cloud).
- No routing by URL path or hostname.
- No central place for TLS/HTTPS, redirects, rewrites, auth, etc.

**Ingress = one entry point + rules that decide which Service gets the request.**

```
                          ┌── /app1 → ingress-service-1 → app1 pod
localhost:8083 → Ingress ─┼── /app2 → ingress-service-2 → app2 pod
                          └── /app3 → ingress-service-3 → app3 pod
```

---

## 2. Ingress resource vs Ingress controller (most important concept)

| | What it is | Who creates it |
|---|---|---|
| **Ingress resource** ([ingress.yaml](./ingress.yaml)) | Just a set of **rules** (YAML) – "send `/app1` to `ingress-service-1`" | You |
| **Ingress controller** (ingress-nginx) | A real **pod running nginx** that reads all Ingress rules and actually proxies the traffic | You install it once per cluster |
| **IngressClass** (`nginx`) | Links an Ingress resource to a specific controller | Created when the controller is installed |

> An Ingress resource **without a controller does nothing**. Kubernetes accepts
> the YAML, but nobody listens for traffic. (We hit exactly this – see
> Troubleshooting.)

Two different "nginx" in this setup – don't mix them up:
- **ingress-nginx controller** → the router / reverse proxy (in namespace `ingress-nginx`).
- **nginx in app1/app2/app3** → plain web servers that serve `index.html`.

---

## 3. Full request flow

```
Mac (curl localhost:8083/app1)
   │
   │  Docker port mapping 8083 → 80   (kind-cluster/config.yaml)
   ▼
k8s-cluster-control-plane node :80
   │
   │  ingress-nginx controller pod listens on hostPort 80 on this node
   ▼
ingress-nginx controller
   │  reads Ingress rules: path /app1 → ingress-service-1:80
   │  rewrite-target: /app1 → /
   ▼
ingress-service-1 :80   (Service, selector app=app1)
   ▼
app1 pod (nginx) :80
   │  serves /usr/share/nginx/html/index.html
   ▼
PVC ingress-pvc-app1 → PV ingress-pv-app1 → node folder /data/app1
```

---

## 4. Files in this folder

| File | What it does |
|---|---|
| [kustomization.yaml](./kustomization.yaml) | Lists all files so everything is applied with `kubectl apply -k .` |
| [persistentVolume.yaml](./persistentVolume.yaml) | 3 PVs (`ingress-pv-app1/2/3`) backed by `hostPath` `/data/app1/2/3` on the node |
| [persistentVolumeClaim.yaml](./persistentVolumeClaim.yaml) | 3 PVCs (`ingress-pvc-app1/2/3`) that bind to those PVs |
| [deployment.yaml](./deployment.yaml) | 3 Deployments (`app1/2/3-deployment`), each an nginx pod mounting its PVC at `/usr/share/nginx/html` |
| [service.yaml](./service.yaml) | 3 Services (`ingress-service-1/2/3`), each selecting one app |
| [ingress.yaml](./ingress.yaml) | Path-based routing rules `/app1`, `/app2`, `/app3` |
| [../kind-cluster/config.yaml](../kind-cluster/config.yaml) | kind cluster: port mapping + `ingress-ready` label |

---

## 5. kind cluster config – how traffic gets from the Mac into the cluster

```yaml
nodes:
- role: control-plane
  labels:
    ingress-ready: "true"        # controller is pinned to this node
  extraPortMappings:
    - containerPort: 80          # port inside the kind node (controller listens here)
      hostPort: 8083             # port on the Mac
      protocol: TCP
- role: worker
- role: worker
```

Key points:
- **kind nodes are Docker containers.** Your Mac can't reach them directly, so
  `extraPortMappings` publishes a node port to the Mac (like `docker run -p 8083:80`).
- **Only ONE port is needed** – the ingress controller's port 80. Ingress does
  all the routing behind it. No need to expose each app's NodePort.
- The mapping exists **only on the control-plane node**, so the controller pod
  **must run on that node**. The `ingress-ready: "true"` label is how we pin it there.
- `extraPortMappings` and `labels` are applied **only when the cluster is
  created** → changing them requires `kind delete cluster` + `kind create cluster`.
- `hostPort` must be free on the Mac. `8080` was already in use, so we used `8083`.

---

## 6. Installing the ingress-nginx controller

```bash
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml

kubectl wait -n ingress-nginx \
  --for=condition=ready pod \
  -l app.kubernetes.io/component=controller \
  --timeout=180s
```

This creates in namespace `ingress-nginx`:
- `ingress-nginx-controller` Deployment (the router)
- `ingress-nginx-admission-create` / `-patch` Jobs → show `Completed`, that's normal (one-time setup of the validating webhook)
- IngressClass `nginx`

### Pin the controller to the control-plane node

In our cluster the controller first landed on a **worker** node → the port
mapping (on control-plane) had nobody listening → `Connection reset by peer`.

Fix: force it onto the node with `ingress-ready=true` and allow it to run on
the control-plane (which has a `NoSchedule` taint by default):

```bash
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --type merge -p '{
  "spec": {"template": {"spec": {
    "nodeSelector": {"ingress-ready": "true", "kubernetes.io/os": "linux"},
    "tolerations": [{"key": "node-role.kubernetes.io/control-plane", "operator": "Exists", "effect": "NoSchedule"}]
  }}}
}'

kubectl get pods -n ingress-nginx -o wide   # NODE must be k8s-cluster-control-plane
```

- **nodeSelector** → "only schedule on nodes with this label".
- **toleration** → "I'm allowed on nodes with this taint" (control-plane is tainted so normal pods avoid it).
- This patch lives only in the cluster – re-run it after recreating the cluster.

Verify:

```bash
kubectl get ingressclass        # nginx
kubectl get ingress             # ADDRESS = localhost
```

---

## 7. The Ingress resource explained

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: ingress-service-1
            port:
              number: 80
      # ... /app2, /app3
```

| Field | Meaning |
|---|---|
| `ingressClassName: nginx` | Which controller handles this Ingress. Without it (and no default class) nothing picks it up. |
| `rules:` | **A list** → must start with `- http:`. (Writing `rules: http:` is invalid YAML for Ingress.) |
| no `host:` | Matches any hostname (`localhost`, IP, …). Add `host: app.local` for host-based routing. |
| `path: /app1` | URL path to match. |
| `pathType: Prefix` | Matches `/app1`, `/app1/`, `/app1/anything`. Other options: `Exact` (only `/app1`), `ImplementationSpecific`. |
| `backend.service.name` | Service to send traffic to (same namespace as the Ingress). |
| `backend.service.port.number` | The Service's **`port`** (80) – **not** the `nodePort` and not the container port. |

### Why `rewrite-target: /`?

nginx in the app pod only has `/index.html`. Without rewrite:

```
curl localhost:8083/app1  →  app1 pod receives GET /app1  →  404 Not Found
```

With `rewrite-target: /` the controller strips the path:

```
curl localhost:8083/app1  →  app1 pod receives GET /       →  index.html ✅
```

> Note: this simple form rewrites **every** sub-path to `/` (e.g. `/app1/style.css` → `/`).
> For real apps use a regex capture:
> `path: /app1(/|$)(.*)` + `pathType: ImplementationSpecific` + `rewrite-target: /$2`.

---

## 8. Services – the 3 kinds of ports

```yaml
spec:
  selector:
    app: app1          # sends traffic to pods with label app=app1
  ports:
    - port: 80         # Service port inside the cluster  ← Ingress uses this
      targetPort: 80   # container port in the pod
      nodePort: 30080  # port on every node (only for type NodePort)
  type: NodePort
```

| Field | Who uses it | Value |
|---|---|---|
| `targetPort` | The container (nginx listens on 80) | 80 |
| `port` | Anything inside the cluster, **including the Ingress controller** | 80 |
| `nodePort` | Direct access from outside via node IP (range 30000–32767) | 30080/1/2 |

Lessons:
- `nodePort` goes **inside** the `ports` list item, not at `spec` level.
- Each Service needs a **different selector**. Originally all 3 used `app: ingress`
  → all 3 Services pointed to the same pod, so routing was pointless.
- With Ingress, `NodePort` is **not required**. Services could be plain
  `ClusterIP` (delete `type` and `nodePort`) – Ingress reaches them from inside the cluster.

Check a Service has pods behind it:

```bash
kubectl get endpoints     # each ingress-service-X should show a pod IP:80
```

---

## 9. Deployments – why 3 Deployments and not 3 containers in 1 pod

Original mistake: one pod with containers `nginx-1`, `nginx-2`, `nginx-3`.

- All containers in a pod **share one network namespace** (same IP, same ports).
- All three nginx try to bind port **80** → two crash with `address already in use`.
- Also, one pod = one label set → Services can't tell the apps apart.

Fix: **one Deployment per app**, each with its own label (`app: app1`, `app2`, `app3`).

```yaml
volumeMounts:
- mountPath: /usr/share/nginx/html   # nginx's web root
  name: ingress-pv-app1
volumes:
- name: ingress-pv-app1
  persistentVolumeClaim:
    claimName: ingress-pvc-app1
```

> `mountPath` must be **`/usr/share/nginx/html`**. When it was `/data/app1`,
> the PVC was mounted somewhere nginx never looks → we kept seeing the default
> "Welcome to nginx!" page.

---

## 10. PV and PVC (per app)

```
node folder /data/app1  ←  PV ingress-pv-app1  ←  PVC ingress-pvc-app1  ←  app1 pod
```

- **PV** = the actual storage (here a `hostPath` folder on the node), `100Mi`, `ReadWriteOnce`, `storageClassName: manual`.
- **PVC** = the pod's request for storage. It binds to a PV with matching `storageClassName`, access mode and enough capacity.
- `hostPath` is **node-local**: the files exist only on the node where the pod runs. If the pod moves to another node, the folder there is empty. Fine for learning, not for production.

```bash
kubectl get pv,pvc        # STATUS should be Bound
```

### Writing a different index.html for each app

From inside each pod (writes into the PV):

```bash
kubectl exec deploy/app1-deployment -- sh -c 'echo "<h1>Hello from App 1</h1>" > /usr/share/nginx/html/index.html'
kubectl exec deploy/app2-deployment -- sh -c 'echo "<h1>Hello from App 2</h1>" > /usr/share/nginx/html/index.html'
kubectl exec deploy/app3-deployment -- sh -c 'echo "<h1>Hello from App 3</h1>" > /usr/share/nginx/html/index.html'
```

Or directly on the kind node (nodes are Docker containers):

```bash
kubectl get pods -o wide                 # find NODE of the pod
docker exec k8s-cluster-worker2 sh -c 'echo "<h1>Hello from App 1</h1>" > /data/app1/index.html'
```

> An empty volume mounted over the web root → nginx returns **403 Forbidden** until `index.html` exists.

---

## 11. Kustomize

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - persistentVolume.yaml
  - persistentVolumeClaim.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

```bash
kubectl kustomize .     # preview the combined YAML
kubectl apply -k .      # apply everything
kubectl delete -k .     # delete everything
```

Lessons:
- **No duplicates** in `resources` – listing `deployment.yaml` twice caused:
  `may not add resource with an already registered id: Deployment.v1.apps/app1-deployment`.
- **Filenames are case-sensitive on Linux/CI.** macOS ignores case, so
  `persistentvolume.yaml` "worked" locally but would break elsewhere.
- **Order doesn't matter** – Kustomize sorts by kind (Namespace → … → Service → PV → PVC → Deployment → Ingress …) before applying.

---

## 12. Complete run – from zero

```bash
# 1. Create the cluster (from kind-cluster/)
cd kind-cluster
kind create cluster --config=config.yaml
kind get clusters

# 2. Install + pin the ingress controller
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --type merge -p '{
  "spec": {"template": {"spec": {
    "nodeSelector": {"ingress-ready": "true", "kubernetes.io/os": "linux"},
    "tolerations": [{"key": "node-role.kubernetes.io/control-plane", "operator": "Exists", "effect": "NoSchedule"}]
  }}}
}'
kubectl wait -n ingress-nginx --for=condition=ready pod \
  -l app.kubernetes.io/component=controller --timeout=180s

# 3. Deploy the apps (from ingress/)
cd ../ingress
kubectl apply -k .
kubectl get pods,svc,pv,pvc,ingress

# 4. Add a page per app
kubectl exec deploy/app1-deployment -- sh -c 'echo "<h1>Hello from App 1</h1>" > /usr/share/nginx/html/index.html'
kubectl exec deploy/app2-deployment -- sh -c 'echo "<h1>Hello from App 2</h1>" > /usr/share/nginx/html/index.html'
kubectl exec deploy/app3-deployment -- sh -c 'echo "<h1>Hello from App 3</h1>" > /usr/share/nginx/html/index.html'

# 5. Test
curl localhost:8083/app1
curl localhost:8083/app2
curl localhost:8083/app3
```

### Cleanup

```bash
kubectl delete -k .                       # only the apps
kind delete cluster --name k8s-cluster    # whole cluster
```

---

## 13. Troubleshooting – problems we actually hit

| Symptom | Cause | Fix |
|---|---|---|
| `error reading file: open config.yaml: no such file or directory` | Ran `kind create` from the wrong folder | `cd kind-cluster` or use `--config=kind-cluster/config.yaml` |
| `node(s) already exist for a cluster with the name "k8s-cluster"` | Cluster already exists | `kind delete cluster --name k8s-cluster` |
| `unknown flag: --config` / `unknown command "k8s-cluster"` on delete | `kind delete` only takes `--name` | `kind delete cluster --name k8s-cluster` |
| `Deleting cluster "k8s0-cluster"` but cluster still exists | Typo in name – `kind delete` succeeds silently even if the name doesn't exist | `kind get clusters` to check the exact name |
| `bind: address already in use` on `0.0.0.0:8080` | Something on the Mac already uses 8080 | Change `hostPort` (we used 8083) or stop the other process (`lsof -i :8083`) |
| Kustomize `already registered id` | Same file listed twice in `resources` | Remove the duplicate |
| `kubectl get ingress` ADDRESS empty + `Connection reset by peer` | No ingress controller installed (namespace existed but was empty) | Install ingress-nginx |
| Controller Running but still `Connection reset by peer` | Controller on a **worker** node; port mapping is on **control-plane** | Patch nodeSelector + toleration (section 6) |
| "Welcome to nginx!" default page | PVC mounted at `/data/appX`, not nginx's web root | `mountPath: /usr/share/nginx/html` |
| `403 Forbidden` | Web root is an empty volume – no `index.html` | Write `index.html` (section 10) |
| `404 Not Found` from the app | Path not rewritten – pod got `/app1` | `rewrite-target: /` annotation |
| `503 Service Temporarily Unavailable` | Service has no ready pods | `kubectl get endpoints`, check selector/labels |

### Useful debug commands

```bash
kubectl get pods -A -o wide                                   # where is everything running
kubectl get nodes --show-labels | grep ingress-ready          # which node is labelled
docker ps --format '{{.Names}}\t{{.Ports}}'                   # kind port mappings
kubectl describe ingress example-ingress                      # rules + backends
kubectl get endpoints                                         # Service → pod IPs
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller # controller access/error logs
kubectl exec deploy/app1-deployment -- ls /usr/share/nginx/html
```

---

## 14. Quick revision checklist

- [ ] Ingress = rules; Ingress **controller** = the pod that applies them. Need both.
- [ ] kind node is a Docker container → `extraPortMappings` exposes it to the Mac.
- [ ] Only one port needed (controller's 80 → Mac 8083).
- [ ] Controller must run on the node that has the port mapping (`ingress-ready` label + toleration).
- [ ] Ingress backend uses the Service **`port`**, not `nodePort`.
- [ ] `rules` is a list (`- http:`), set `ingressClassName: nginx`.
- [ ] `rewrite-target: /` so `/app1` reaches the pod as `/`.
- [ ] One Deployment per app; containers in one pod share ports.
- [ ] Each Service selects a different label.
- [ ] Mount content at `/usr/share/nginx/html` for nginx.
- [ ] `hostPath` PV is node-local.
- [ ] Kustomize: no duplicates, exact filenames, order doesn't matter.

---

## 15. Next steps to learn

1. **Host-based routing** – `host: app1.local`, `host: app2.local` + entries in `/etc/hosts` (`127.0.0.1 app1.local`).
2. **TLS** – self-signed cert → `kubectl create secret tls` → `tls:` section in the Ingress.
3. **Default backend** – custom page for unmatched paths.
4. **Regex rewrites** – keep sub-paths: `/app1(/|$)(.*)` → `/$2`.
5. **ConfigMap for content** – mount `index.html` from a ConfigMap instead of a PV.
6. **Gateway API** – the newer replacement for Ingress (ingress-nginx is retired upstream); try Envoy Gateway or NGINX Gateway Fabric.
