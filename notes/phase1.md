# Foundations & Mental Model

> Goal: Understand how Kubernetes works internally - not just what commands to run, but *why* the system behaves the way it does. Everything in production Kubernetes flows from these foundations.

---

## 1. The Core Idea: Desired State vs Actual State

Kubernetes is a **reconciliation engine**. You never tell it *how* to do something - you declare *what you want*, and Kubernetes continuously drives reality toward that declaration.

```
You declare: "I want 2 replicas of this app running"
Kubernetes:  "Current actual state has 1. Let me fix that."
```
This loop never stops. It fires on every deviation - whether caused by you, a crash, a node failure, or anything else. This is why Kubernetes is self-healing: not because it detects failures specifically, but because it is always asking *"does actual match desired?"*

---

## 2. The Control Plane

The control plane is the brain of Kubernetes. It runs on master nodes and consists of four components. **None of them run your application containers** - they only manage the cluster.

### 2.1 API Server (`kube-apiserver`)

The singel entry point for all cluster operations. Every read and write - from `kubectl`, from controllers, from kubelets - goes through the API server.

**Hard rule:** Nothing touches etcd directly except the API server.

The API server runs every request through a pipeline:

```
Request -> Authentication -> Authorization -> Admission Controllers -> Schema Validation -> Write to etcd
```

- **Authentication** - who are you? (certificates, tokens, service accounts)
- **Authorization** - are you allowed to do this? (RBAC)
- **Admission Controllers** - mutate or validate the object before it's stored (e.g. inject defaults, enforce policies)
- **Schema Validation** - is the object structurally valid?

### 2.2 etcd

A distributed key-value store. The **single source of truth** for all cluster state - every object (Pods, Deployments, Services, Secrets) is stored here.

**Key properties:**
- Only the API server reads from and writes to etcd
- All controllers watch etcd (via the API server) for changes
- If etcd is lost without a backup, the cluster state is lost
-

### 2.3 Scheduler (`kube-scheduler`)

Watches for Pods with no `nodeName` assigned and decides which node they should run on.

The scheduler runs two phases:

| Phase | What it does |
|-------|--------------|
| **Filter** | Eliminate nodes that *cannot* run the pod (insufficient CPU/memory, wrong taints, affinity rules) |
| **Score**  | Ranks remaining nodes and pick the best one |

After scoring, the scheduler writes the chosen `nodeName` back to the pod Object in etcd. **Its job ends there**. The Scheduler never starts containers, never rremoves Pods, never touches the container runtime.

### 2.4 Controller Manager (`kube-controller-manager`)

Runs dozens of control loops - each one responsible for a specific resource type. Each controller watches etcd for its resource and reconcilies desired vs actual state.

**Key controllers:**

| Controller | Responsibility |
|-------------|-----------------|
| Deployment controller | Creates and updates ReplicaSets when Deployments change |
| ReplicaSet controller | Creates and deletes Pods to match `spec.replicas` |
| Node controller | Monitors node health and marks nodes as unavaiable |
| Job controller | Manages one-off batch Pods |

---

## 3. Worker Nodes

Worker nodes are where your application containers actually run. Each node has three components.

### 3.1 kubelet

An agent that runs on every worker node. It is the only component that actually start and stops containers.

**Responsibilities:**

- Watches the API server for pods assigned to its node (`nodeName == this node`)
- Instructs the container runtime to pull images and start containers
- Mounts volumes and sets up environment variables
- Runs liveness/ readiness probes
- Reports Pod status back to the API server

### 3.2 Container Runtime

The software that actually manages containers. kubelet talks to it via the CRI (Container Runtime Interface). Common runtimes: `containerd`, `CRI-0`.

### kube-proxy

Maintains network rules on each node so that traffic to a Service IP gets routed to the correct Pod IPs. Uses iptables or IPVS under the hood.

---

## 4. The Full Flow: `kubectl apply` to Running Pod

This is the most important mental model in Kubernetes. Understanding this one flow covers every major components.

> **Key insight:** No component talks directly to another. Everything communicates through etcd via the API server. This is what makes Kubernetes resilient

![`kubectl apply` to Running Pod](./images/reconciliation_loop.png)

**Step 1 - kubectl sends the request**
- Kubectl reads your YAML, does client-side schema validation
- Reads `~/.kube/config` for the cluster address and credentials
- Sends an authenticated HTTPS request to the API server

**Step 2 - API server validates and persits**
- Runs the request through authn -> authz -> admission -> validation
- Writes the Deployment object to etcd
- Returns `201 Created` to kubectl
- **At this point the Deployment exists in etcd - but nothing is running yet**

**Step 3 - Controller Manager reconciles**
- The **Deployment controller** is watching etcd for new Deployments
- It sees `desired ReplicaSets = 1, actual = 0` -> writes a ReplicaSet to etcd
- The **ReplicaSet controller** then sees `desired Pods = 2, actual = 0` -> writes 2 Pod object to etcd
- These Pod objects have **no** `nodeName` - they are unscheduled

**Step 4 - Scheduler assigns nodes**
- The scheduler watches for Pods with no `nodeName`
- Runs filter -> score for each Pod
- Writes the chosen `nodeName` back to each Pod object in etcd
- Its job is now done

**Step 5 - kubelet starts the containers**
- kubelet on the assigned node is watching for Pods with `nodeName == its node`
- It sees the new Pod, calls the container runtime (via CRI) to pull the image
- Sets up networking (via CNI), mounts volumes, starts the container
- Reports status back to the API server -> Pod transitions: `Pending -> ContainerCreating -> Running`

**Step 6 - The reconciliation loop never stops**
- All controllers continue watching etcd
- If a Pod is deleted -> ReplicaSet controller sees `desired=2, actual=1` -> creates a new Pod immediately
- This is Kubernetes self-healing in action

---

## 5. The Ownership Chain

Kubernetes tracks parent-child relationships between objects using `ownerReferences` in each object's metadata.

```
Deployment -> owns -> ReplicaSet -> owns -> Pods
```

### Verify this yourself:

```bash
kubectl get replicaset -n dev -o yaml | grep -A5 ownerReferences
```

### Implications:
- Deleting a Deployment cascases: its ReplicaSets and Pods are deleted too
- Deleting a ReplicaSet directly -> the Deployment controller recreates it immediately
- Each ReplicaSet's name suffix is a **hash of the Pod template** - this is how rollouts are tracked. A new image -> new hash -> new ReplicaSet -> old one scales to 0

---

## 6. Production Pod Spec: Resources

Setting resources correctly is one of the most important things you can do for production stability.

### 6.1 Requests vs Limits

| Field | Purpose | Who uses it |
|-------|---------|-------------|
| `requests` | Reserved capacity - used for **scheduling decisions** | Scheduler |
| `limits` | Hard ceiling enforced at **runtime** | Linux kernel (cgroups) |

**Crtitical distinction:**
- The scheduler only looks at `requests` when placing a Pod on a node
- At runtime, a Pod can use *more* than its `requests` (up to `limits`)
- This means a node can be **overcommited** - all Pod's request fit, but if they all spike to limits simultaneously, the node runs out of real memory

**What happens when limits are exceeded:**
- **CPU limit exceeded** -> container is throttled (slowed down, not killed)
- **Memory limit exceeded** -> container is OOMKilled (killed immediately)

```yaml
resources:
  requests:
    cpu: "50m"      # 50 millicores - used by scheduler
    memory: "64Mi"  # used by scheduler
  limits:
    cpu: "220m"     # kernel throttles if exceeded
    memory: "128Mi" # kernel kills container if exceeded
```

### 6.2 QoS Classes

Kubernetes assigns every Pod a QoS class based on its resource configuration. This determines **eviction order** when a node runs low on memory.

| QoS Class | Condition | Eviction order |
|-----------|-----------|----------------|
| Guaranteed | `requests == limits` for all containers | Killed last |
| Burstable | `requests < limits` (at least one container) | Killed second |
| BestEffort | No requests or limits set at all | Killed first |

**Production rule**: For any critical application, set `requests == limits`. You trade flexibility for eviction safety - the node always knows exactly what the Pod needs and it will never be sacrificed under pressure.

---

## 7. Prdoduction Pod Spec: Probes

Kubernetes doesn't know if your application is healthy just because the container process is running. Probes give Kubernetes a way to check.

### 7.1 The Three Probes

| Probe | Question it answers | Failure action |
|-------|---------------------|----------------|
| `livenessProbe` | Is the container still alive and functional? | **Restart** the container |
| `readinessProbe` | Is the container ready to receive traffic? | **Remove from Service endpoints** (not restarted) |
| `startupProbe` | Has the container finished its initial startup? | Disables liveness/readiness until it passes |

### 7.2 When to use each

**livenessProbe** - protects against stuck processes. A container can be running but internally deadlocked, memory-leaked, or in an infinite loop. Without liveness, that zombie serves traffic forever. The liveness probe detects it and triggers a restart.

**readinessProbe** - protects users. Your app may be alive but not yet ready: still connecting to a database, warming a cache, or temporarily overloaded. A failed readiness prove removes the Pod from the load balancer without restarting it.

**startupProbe** - protects slow starters. If `initialDelaySeconds` on a liveness probe is set too low, the probe fires before the app has booted, kills the container, it restarts - and you get a **CrashLoopBackOff**. The startupProbe disables liveness entirely until the app has successfully started.

### 7.3 Probe configuration

```yaml
livenessProbe:
  httpGet:
    path: /               # endpoint to check
    port: 80
  initialDelaySeconds: 5  # wait before first check
  periodSeconds: 10       # check every 10s
  failureThreshold: 3     # fail 3 times before acting

readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 3
  periodSeconds: 5
  failureThreshold: 2

startupProbe:
  httpGet:
    path: /
    port: 80
  failureThreshold: 30    # 30 * 10s = 5 minutes max startup time
  periodSeconds: 10
```

### 7.4 CrashLoopBackOff

The most common probe-related failure. Symptoms: `RESTARTS` column climbing in `kubectl get pods`, status show `CrashLoopBackOff`.

**Common causes:**
- `initialDelaySeconds` too low on liveness probe for a slow-starting app
- Wrong probe path (e.g. `/healthz` when the app serves `/health`)
- App genuinely crashing on startup (check `kubectl logs <pod>`)

**Diagnose with:**

```bash
kubectl describe pod <pod-name> -n <namespace>
# Look at the Events section at the bottom
# You'll see: "container failed liveness probe, will be restarted"
```

---

## 8. Key Commands Learned

```bash
# Cluester info
kubectl cluster-info

# Namespace management
kubectl create namespace dev
kubens dev                    # switch default namespace

# Deployments
kubectl apply -f deployment.yaml
kubectl get deployment -n dev
kubectl scale deployment web -n dev --replicas=0
kubectl rollout status deployment/web -n dev

# ReplicaSets
kubectl get replicaset -n dev
kubectl get rs -n dev -o yaml | grep -A5 ownerReference

# Pods
kubectl get pods -n dev
kubectl get pods -n dev -w      # watch mode
kubectl delete pod <name> -n dev
kubectl describe pod <name> -n dev  # events, probe status, resource usage
kubectl logs <pod-name> -n dev

# Patching
kubectl patch deployment web -n dev --type=json \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/livenessProbe/httpGet/path", "value":"/"}]'
```

---

## 9. Mental Models to Keep

> 1. **The reconciliation loop is everything.** Controllers don't react to events. They watch desired state and continuously reconcile it against actual state. The trigger doesn't matter - the loop just fires.

> 2. **Nothing touches etcd except the API server**. Every read and write goes through the API server. This is a hard rule, not a guideline.

> 3. **The schedulre only assigns - it never removes.** The schedulre writes a `nodeName` to an unscheduled Pod. It never deletes Pods or does anything else.

> 4. **Kubelet is the only component that starts containers.** THe control plane only writes objects to etcd. kubelet on the node is what actually runs `containerd` to pull images and start processes.

> 5. **Requests are for scheduling. Limits are for runtime.** The scheduler sees requests. The kernel enforces limits. A node can be overcommited if requests are low but limits are high.

> 6. **Set requests == limits for critical apps.** This gets you Guaranteed QoS - the Pod is the last to be evicted under memory pressure.
