# Core Workloads & Networking

> **Goal**: Master the objects you'll interact with every single day in production - Services, ConfigMaps, Secrets, and Storage. These are the building blocks every real application needs beyond just running containers.

---

## 1. The Problem Services Solve

Pod IPs are ephemeral. Every time a Pod dies and a new one is created by the ReplicaSet controller, it gets a fressh IP address. Any service harcoded to talk to a specific Pod IP will break.

Services solve this with two things:
- A **stable ClusterIP** - a virtual IP that never changes
- A **stable DNS name** - `<service-name>.<namespace>.svc.cluster.local`

**The full traffic chain:**

```
"web-svc" -> DNS -> ClusterIP (virtual, stable) -> kube-proxy iptables -> Pod IP (ephemeral)
```

Your app never needs to know Pod IPs exist. It just calls `http://web-svc`.

---

## 2. How Services Find Pods - Label Matching

A Service has **no direct relationship** to a Deployment or ReplicaSet. It only cares about Pod lables. The `selector` field defines which Pods receive traffic.

```yaml
spec:
  selector:
    app: web     # sends traffic to any Pod with this lable
```

**Production gotcha:** If you mislabel a Pod, it silently gets no traffic - no error, just excluded from the Service. Always verify with:

```bash
kubectl get endpoints <service-name> -n <namespace>
```

The Endpoints object is the live list of healthy Pod IPs behind a Service. It's maintained by the **Endpoint controller** - another reconciliation loop in the controller manager. A Pod is only added to Endpoints when its **readines probe is passing**. This is why readiness probes and Services are deeply connected.

---

## 3. The Four Service Types

Each type builds on top of the previous - they are extensions, not replacements.

```
LoadBalancer
  └── NodePort
        └── ClusterIP
```

### 3.1 ClusterIP (default)

Reachable only from **inside the cluster**. Every Service gets a ClusterIP automatically unless specified otherwise.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: dev
spec:
  selector:
    app: web
  ports:
  - port: 80          # Service port (what clients hit)
    targetPort: 80    # Pod port (where traffic lands)
  type: ClusterIP
```
Use for: all internal service-to-service communication.

### 3.2 NodePort

Opens a port on **every node** in the cluster. Reachable from outside via `nodeIP: nodePort`.

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080     # must be in range 30000-32767
```

```bash
curl http://localhost:30080     # works with OrbStack
```
Use for: local development and quick testing. Not for production - exposes a port on every node.

### 3.3 LoadBalancer

Provisions a **cloud load balancer** with a real external IP. Builds on NodePort underneath.

```yaml
spec:
  type: LoadBalancer
```
Use for: production internet-facing traffic on cloud providers (EKS, GKE, AKS).

### 3.4 Headless Service

A ClusterIP Service with `clusterIP: None`. Instead of one virtual IP, DNS returns individual Pod IPs directly. Each Pod gets its own stable DNS name.

```yaml
spec:
  clusterIP: None
  selector:
    app: db
```

Pod DNS names: `db-0.my-db.dev.svc.cluster.local`, `db-1.my-db.dev.svc.cluster.local`

Use for: StatefulSets, databases - anywhere Pods need individual stable identities.

---

## 4. kube-proxy and How Traffic Actually Routes

`kube-proxy` runs on every node. It writes `iptables` (or IPVS) rules so that any traffic hitting a ClusterIP gets forwarded to one of the healthy Pod IPs behind it.

When a Pod dies:
1. Endpoint controller removes its IP from the Endpoints object
2. kube-proxy updates iptables rules on all nodes
3. Traffic stops going to the dead Pod - no client change needed

```bash
# Verify DNS resolution from inside the cluster
kubectl run dns-test --image=busybox:1.28 --rm --it --restart=Never -n dev -- nslookup web.svc.dev.svc.cluster.local

# Short form works within same namespace
kubectl run dns-test --image=busybox:1.28 --rm --it --restart=Never -n dev -- nslookup web-svc
```

Both return the same ClusterIP - DNS is just a stable name pointing to the stable virtual IP.

---

## 5. ConfigMaps

Decouples non-sensitive configuration from the container image. Same image, different config per environment.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
  namespace: dev
data:
  APP_ENV: "development"
  LOG_LEVEL: "debug"
  MAX_CONNECTIONS: "100"
```

---

## 6. Secrets

Stores sensitive data. Structurally identical to ConfigMap but values are **base64 encoded**.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-secret
  namespace: dev
type: Opaque
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQ=    # base64 of "supersecret"
  API_KEY: bXlhcGlrZXkxMjM=        # base64 of "myapikey123"
```

**Encode values:**

```bash
echo -n "supersecret" | base64
```

**Critical warning:** Base64 is encoding, not encryption. Anyone with `kubectl get secret` permission can decode it instantly:

```bash
echo "c3VwZXJzZWNyZXQ=" | base64 -d
# supersecret
```

Real secret security requires either **etcd encryption at rest** or an external secrets manager (Vault, AWS Secrets Manager, External Secrets Operator).

---

## 7. Injecting Config into Pods

Two methods - each with different behviour when the source changes.

### 7.1 Environment Variables

Injected at Pod start. **Changing the ConfigMap/Secret does not update running Pods - requires restart.**

```yaml
# Inject all keys from a ConfigMap as env vars
envFrom:
  - configMapRef:
      name: web-config

# Inject a single Secret key
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: web-secret
        key: DB_PASSWORD
```

### 7.2 Volume Mounts

Mounted as files inside the container. **Kubernetes updates the files automatically within ~60 seconds when the ConfigMap changes - no restart needed.**

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /etc/config      # each ConfigMap key becomes a file here

volumes:
  - name: config-volume
    configMap:
      name: web-config
```

**When to use which**

| Use env vars for | Use volume mounts for |
|------------------|-----------------------|
| Simple key-value config | Config files (nginx.conf, app.yaml) |
| Values that rarely change | Values that change at runtime |
| 12-factor app style | Apps that watch the filesystem for changes |

**Verify injection**

```bash
# Check env vars inside a running Pod
kubectl exec -it <pod-name> -n dev -- env | grep -E "APP_ENV|LOG_LEVEL|DB_PASSWORD"

# Check mounted config files
kubectl exec -it <pod-name> -n dev -- ls /etc/config
kubectl exec -it <pod-name> -n dev -- cat /etc/config/APP_ENV
```

---

## 8. Storage - The Three Objects

Kubernetes separates *what storage is available from what storage is requested* from *what storage is used.*

```
StorageClass -> the landlord (defines what kind of storage can be provisioned)
PersistentVolume (PV) -> the building (actual storage resource on disk)
PersistentVolumeClaim (PVC) -> your lease (Pod's request for storage)
```

The Pod always talks to a PVC - never directly to a PV. This means your app spec is identical whether the underlying storage is a local disk, AWS EBS, or GCP Persistent Disk.

### 8.1 StorageClass

Defines how storage is provisioned. OrbStack ships with `local-path` as the default.

```bash
kubectl get storageclass
```

Key annotation: `WaitForFirstConsumer` - don't provision storgae until a Pod claims the PVC. This ensures the disk is created on the same node the Pod lands on.

### 8.2 PersistenVolumeClaim

Your request for storage. Kubernetes finds (or creates) a PV that satisfies it.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-data
  namespace: dev
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

PVC stays `Pending` until a Pod mounts it (with `WaitForFirstConsumer`) or a matching PV exists.

### 8.3 Access Modes

| Mode | Abbreviation | Meaning | Use case |
|------|--------------|---------|----------|
| ReadWriteOnce | RWO | One node at a time, read+write | Databases |
| ReadOnlyMany | ROX | Many nodes, read only | Shared config, static assets |
| ReadWriteMany | RWX | Many nodes, read + write | Shared file storage (needs NFS) |

Most production databases use `ReadWriteOnce`.

### 8.4 Mounting a PVC into a Pod

```yaml
volumeMounts:
  - name: data-volume
    mountPath: /data

volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: web-data
```

### 8.5 Verify persistence

```bash
# Write data to the volume
kubectl exec -it <pod-name> -n dev -- sh -c "echo 'survived!' > /data/test.txt"

# Delete the Pod
kubectl delete pod <pod-name> -n dev

# Check the replacement Pod
kubectl exec -it <new-pod-name> -n dev -- cat /data/test.txt
# Output: survived!
```

---

## 9. Deployments vs StatefulSets

Deployments with shared PVCs have a hidden problem: `ReadWriteOnce` PVCs can only be written from one node. Multiple Pods in a Deployment on different nodes will fight over the same PVC.

**StatefulSets** solve this by giving each Pod its own dedicated PVC automatically.

|  | Deployment | StatefulSet |
| Pod names | Random (`web-7664656f85-6c55b`) | Stable and ordered (`db-0`, `db-1`) |
| Storage | Shared PVC | One PVC per Pod (`data-db-0`, `data-db-1`) |
| Start/stop order | Parallel | Ordered (0->1->2) |
| Pod identity | Interchangeable | Unique and stable |
| Use case | Stateless apps | Databases, message queues, caches |

**Rule:** Always use StatefulSets for databases. Never use Deployments for anything that needs per-Pod persistent storage.

---

## 10. Key Commands Learned

```bash
# Services
kubectl get service -n dev
kubectl get endpoints <service-name> -n dev
kubectl describe service <service-name> -n dev

# DNS testing from inside the cluster
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -n dev -- nslookup web-svc.dev.svc.cluster.local

# ConfigMaps and Secrets
kubectl get configmap -n dev
kubectl get secret -n dev
kubectl get secret <name> -n dev -o yaml      # see base64 values
kubectl describe configmap <name> -n dev

# Storage
kubectl get pvc -n dev
kubectl get pv
kubectl get storageclass

# Exec into a pod
kubectl exec -it <pod-name> -n dev -- env
kubectl exec -it <pod-name> -n dev -- sh
kubectl exec -it <pod-name> -n dev -- df -h
```

---

## 11. Mental Models to Keep
> 1. **Services find Pods by labels, not by Deployment.** A service with the right selector will pick up any Pod with matching labels - regardless of which Deployment created it.

> 2. **Always verify traffic flow with kubectl get endpoints.** No endpoints == no traffic. Nine times out of ten it's a label mismatch or a failing readiness probe.

> 3. **Secrets are not secret by default.** Base64 is encoding, not encryption. Real security requires RBAC + etcd encryption or an external secrets manager.

> 4. **Env vars are static, volume mounts are live.** Change a ConfigMap -> volume mount updates in ~60s. Env vars require a Pod restart to pick up changes.

> 5. **PVC -> StorageClass -> PV. The Pod never touches the PV directly.** This abstraction means your app spec is portable across any cloud provider.

> 6. **Databases belong in StatefulSets, not Deployments.** StatefulSets give each Pod a stable name, stable storage, and ordered startup/shutdown.

---

## 12. Full Deployment Spec - Everything Together

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 5
          failureThreshold: 2
        envFrom:
        - configMapRef:
            name: web-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: web-secret
              key: DB_PASSWORD
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
        - name: data-volume
          mountPath: /data
      volumes:
      - name: config-volume
        configMap:
          name: web-config
      - name: data-volume
        persistentVolumeClaim:
          claimName: web-data
```
