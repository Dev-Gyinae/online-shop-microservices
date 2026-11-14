# 📘 README Section — Concepts & Incidents

---

```markdown
## Concepts — Kubernetes Terms Used in This Project

Quick reference for the vocabulary that appears in `kubectl` output and the troubleshooting below.

### Core objects

| Term | What it is |
|---|---|
| **Pod** | Smallest deployable unit. One or more containers sharing a network namespace and storage. Pods are ephemeral — they die and are replaced. |
| **Deployment** | Declares *what* a pod should look like and *how many* should run. Manages rolling updates. |
| **ReplicaSet** | Created by a Deployment. Owns the actual pods. Every pod template change creates a **new** ReplicaSet — old ones stick around until their pods terminate. |
| **Service** | Stable virtual IP + DNS name in front of a set of pods. `frontend` is reachable at `frontend:8080` even as pods come and go. |
| **Endpoints / EndpointSlice** | The actual pod IPs a Service routes to. If this is `<none>`, the Service has no healthy backing pods. |
| **Namespace** | Logical partition. This demo runs everything in `default`; k3s system pods live in `kube-system`. |

### Pod lifecycle states

| State | Meaning |
|---|---|
| `Pending` | Scheduler can't place the pod — usually **insufficient CPU or memory** on any node. |
| `ContainerCreating` | Scheduled, but the container runtime is still setting up (pulling image, mounting volumes, creating network). |
| `Running 0/1` | Container is up, but a **readiness probe** is failing — it is not yet receiving traffic. |
| `Running 1/1` | Ready. Included in the Service's endpoints. |
| `Completed` | Container exited with code 0 (normal for jobs like `helm-install-gateway-api-crd`). |
| **`CrashLoopBackOff`** | Container starts, exits non-zero, and is restarted repeatedly. The **backoff** means Kubernetes waits longer between each attempt (10s → 20s → 40s → … up to 5 min) to avoid thrashing. |
| **`ErrImagePull`** | The image could not be pulled — registry unreachable, tag doesn't exist, or auth failed. |
| **`ImagePullBackOff`** | Same as above, but Kubernetes is now backing off between retries. |
| **`OOMKilled`** | The kernel killed the container for exceeding its memory limit. Always shows `exitCode: 137`. |
| **`Terminating`** | Pod is shutting down (grace period or `kubectl delete`). |

### Probes

| Term | Purpose | Failure consequence |
|---|---|---|
| **Readiness probe** | "Is this pod ready to receive traffic?" | Pod stays at `0/1`, removed from the Service's endpoints, but **not** restarted. |
| **Liveness probe** | "Is this pod still alive?" | Pod is **killed and restarted**. Repeated failures → `CrashLoopBackOff`. |
| **`timeoutSeconds`** | How long the probe waits before declaring failure. **Default: 1s.** | A slow-starting app fails every probe and gets killed. |
| **`initialDelaySeconds`** | Grace period before the first probe runs. | Without it, a slow app is killed before it even starts. |
| **`periodSeconds`** | How often the probe runs. | — |

### Container exit codes

| Code | Meaning |
|---|---|
| `0` | Clean exit |
| `1` | Application error (uncaught exception, failed assertion) |
| `137` | `SIGKILL` — almost always **OOMKilled** (kernel killed for memory) |
| `143` | `SIGTERM` — graceful shutdown request (kubelet killing on probe failure counts here) |

### Tools & components

| Term | What it is |
|---|---|
| **k3s** | A CNCF-certified Kubernetes distribution packaged as a single binary with containerd, CoreDNS, and a local storage provisioner baked in. |
| **containerd** | The container runtime k3s uses. Runs containers; images stored in its content store. |
| **`ctr`** | containerd's low-level CLI. Used in this project to pre-pull images. Unlike `docker pull`, it **requires fully qualified image paths**. |
| **`crictl`** | Kubernetes-facing CLI for containerd (k3s symlinks it). |
| **`kubectl port-forward`** | Tunnels a local port to a Service or Pod via the Kubernetes API server. No ingress needed. `--address 0.0.0.0` makes it listen on all interfaces. |
| **QoS class** | How Kubernetes prioritizes pods for eviction. `Burstable` = requests ≠ limits (what most demo pods are). `Guaranteed` = requests == limits. `BestEffort` = no resources set. |
| **Rolling update** | Deployment strategy: create a new ReplicaSet, scale it up, scale the old one down. Paused or deadlocked if new pods never become ready. |

---

## Incidents — What Broke, Why, and How It Was Fixed

Six real failures from this deployment. Each entry has the same structure: **symptom → root cause → fix → why the fix works**.

---

### Incident 1 — Full deployment on a 2 GB VM

**Symptom**

```
NAME                            READY   STATUS              RESTARTS
adservice-679967db4b-tmmch      0/1     Running             0
adservice-84b79f4569-zx5s6      0/1     Running             0     ← two ReplicaSets
cartservice-574c7f4cdd-fb9bq    0/1     Pending             0
frontend-65c9648564-h5ss8       0/1     CrashLoopBackOff    5
redis-cart-586648f4ff-tbv8n     0/1     ErrImagePull        0
...
```

Dozens of `Pending` pods, every Deployment had **two** ReplicaSets running simultaneously, and `frontend`/`recommendationservice` were crash-looping.

**Root cause**

k3s control plane (~480 MiB) + 11 demo pods (~800 MiB requests, ~1.75 GiB limits) exceeds 2 GB of physical RAM.

A rolling update deadlocked: new pods could not schedule (`Pending`), so old pods were never terminated. Both ReplicaSets kept a pod alive, consuming memory in a loop. The `ErrImagePull` on `redis-cart` and `CrashLoopBackOff` on `frontend` were **cascading symptoms** of memory exhaustion, not separate bugs.

**Fix**

Move to a VM with ≥ 8 GB RAM.

**Why the fix works**

The scheduler can now place every pod. With no `Pending` pods, rolling updates complete normally and old ReplicaSets scale to zero on their own. Swap alone would not have helped — kubelet does not schedule against swap, and the demo's resource requests are hard values.

---

### Incident 2 — `adservice` `ErrImagePull: not found`

**Symptom**

```
Error: ImagePullBackOff
Failed to pull image "gcr.io/google-samples/microservices-demo/adservice:v0.2.3":
  not found
```

**Root cause**

Google's sample registry retired the `v0.2.3` tag for `adservice`. The manifest was pinned to a tag that no longer exists. The other 10 services on `v0.2.3` still exist.

**Fix**

```bash
sudo k3s ctr images pull gcr.io/google-samples/microservices-demo/adservice:v0.4.1
sed -i 's|adservice:v0\.2\.3|adservice:v0.4.1|' config.yaml
```

**Why the fix works**

The `AdService.GetAds` gRPC RPC signature is unchanged between `v0.2.3` and `v0.4.1`, so `frontend:v0.2.3` can call the newer `adservice` without code changes. Pre-pulling with `ctr` **before** `kubectl apply` isolates this failure from the rest of the deployment — the pull either succeeds or fails immediately, instead of blocking a pod that then needs to be debugged.

---

### Incident 3 — `redis-cart` `invalid port ":alpine" after host`

**Symptom**

```
ctr: failed to resolve image: parse "dummy://redis:alpine":
     invalid port ":alpine" after host
```

**Root cause**

`ctr` requires fully qualified image references. The string `redis:alpine` is parsed as **host `redis`, port `alpine`** — not as image `redis`, tag `alpine`. `docker pull` accepts short names because Docker performs registry resolution; `ctr` does not.

**Fix**

```bash
sudo k3s ctr images pull docker.io/library/redis:alpine
sed -i 's|image: redis:alpine|image: docker.io/library/redis:alpine|' config.yaml
```

**Why the fix works**

`docker.io/library/redis:alpine` is a fully qualified reference: registry `docker.io`, namespace `library`, image `redis`, tag `alpine`. containerd stores the image under this name, and the Deployment spec references the same name — so the kubelet finds the pre-pulled image and never pulls at runtime.

> Note: `redis:alpine` in the original manifest would have worked inside the cluster, because kubelet/containerd perform their own short-name resolution. It was only the `ctr` pre-pull that failed. Aligning the manifest to the full path keeps the pre-pull step meaningful.

---

### Incident 4 — `recommendationservice` `CrashLoopBackOff` (the hard one)

**Symptom**

```
recommendationservice-5995f559bb-9g7g7   0/1   CrashLoopBackOff   6 (1s ago)   5m14s
```

Logs from `kubectl logs --previous`:

```
{"message": "initializing recommendationservice"}
{"message": "Profiler enabled."}
WARNING:google.auth.compute_engine._metadata:Compute Engine Metadata server
        unavailable on attempt 1 of 3. Reason: timed out
WARNING:google.auth.compute_engine._metadata:... attempt 2 of 3 ...
WARNING:google.auth.compute_engine._metadata:... attempt 3 of 3 ...
{"message": "Unable to start Stackdriver Profiler Python agent."}
{"message": "Sleeping 10 seconds to retry Stackdriver Profiler agent initialization"}
```

Probe failures in `kubectl describe`:

```
Readiness probe failed: command timed out:
  "/bin/grpc_health_probe -addr=:8080" timed out after 1s
Liveness probe failed: ... timed out after 1s
Killing: Container server failed liveness probe, will be restarted
```

**Root cause**

Three separate problems chained together.

1. **The `v0.2.3` Python image enables three Google Cloud agents by default** — Profiler, Tracing, and Debugger. Each tries to contact the GCP metadata server at `169.254.169.254` to obtain credentials. On a non-GCP VM that endpoint times out.

2. **Each agent handles the timeout differently.** Profiler retries three times, then sleeps 10 seconds before retrying again. Tracing gives up quietly. **Debugger hangs indefinitely** — the log stalls at `Initializing Cloud Debugger Python agent` and never returns.

3. **The probe timeout was 1 second.** During that startup phase, the app has not yet bound to `:8080`. Every liveness probe fails → kubelet sends `SIGTERM` → container exits → restart. The 10-second profiler sleep means the app could never have passed a 1-second probe.

The pod was killed before it ever had a chance to start.

**Fix — three changes, all required**

```bash
# 1. Disable all three GCP agents
kubectl set env deployment/recommendationservice \
  DISABLE_PROFILER=1 \
  DISABLE_TRACING=1 \
  DISABLE_DEBUGGER=1
```

```bash
# 2. Raise probe timeouts from 1s to 10s
kubectl patch deployment recommendationservice --type=json -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/timeoutSeconds","value":10},
  {"op":"replace","path":"/spec/template/spec/containers/0/livenessProbe/timeoutSeconds","value":10}
]'
```

```bash
# 3. Raise memory limit — 250Mi OOMs during Python import
kubectl set resources deployment recommendationservice \
  --requests=memory=220Mi --limits=memory=450Mi
```

```bash
# 4. Reset to a single clean pod (clears accumulated ReplicaSets)
kubectl scale deployment recommendationservice --replicas=0
# wait for pods to disappear
kubectl scale deployment recommendationservice --replicas=1
```

**Why the fix works**

- With `DISABLE_*` flags set, the service skips GCP agent initialization entirely. The app proceeds straight to binding `:8080`. It now starts in ~5 seconds instead of never.
- 10-second probe timeouts give the Python interpreter room to import libraries and initialize the gRPC server, so the readiness probe succeeds on the first attempt.
- 450 MiB memory limit prevents `OOMKilled` during the initial import spike (implicit ML libraries can peak above 250 MiB).
- Scaling to zero and back **clears the accumulated ReplicaSets** from previous patch attempts. All patches converge in the Deployment spec before the new pod is created, eliminating races between competing ReplicaSets.

**Verify the flags actually landed** (a critical step — see Incident 6):

```bash
kubectl get deployment recommendationservice \
  -o jsonpath='{.spec.template.spec.containers[0].env}' | python3 -m json.tool
```

Expected:

```json
[
  {"name": "PORT", "value": "8080"},
  {"name": "PRODUCT_CATALOG_SERVICE_ADDR", "value": "productcatalogservice:3550"},
  {"name": "DISABLE_PROFILER", "value": "1"},
  {"name": "DISABLE_TRACING", "value": "1"},
  {"name": "DISABLE_DEBUGGER", "value": "1"}
]
```

If any of the three `DISABLE_*` flags is missing, the pod will crash-loop again even if the probe and memory fixes are in place.

---

### Incident 5 — Duplicate ReplicaSets, rolling-update deadlock

**Symptom**

```
NAME                               DESIRED   CURRENT   READY   AGE
recommendationservice-56c58657c5   1         1         0       4m7s
recommendationservice-56c79bbfb    1         1         0       4m18s
recommendationservice-5995f559bb   0         0         0       10m
recommendationservice-85d9d5dccb   0         0         0       6m21s
```

Two ReplicaSets each holding a live, crash-looping pod. Neither reaching `1/1`, so neither one is terminating.

**Root cause**

Every `kubectl patch` or `kubectl set env` **changes the pod template** → creates a new ReplicaSet → triggers a rolling update. If the new pods never become ready (because the bug isn't fixed yet), the old ReplicaSet keeps its pods alive. Each patch attempt adds another ReplicaSet. After four patches, there were four ReplicaSets competing for the same node.

**Fix**

```bash
kubectl scale deployment recommendationservice --replicas=0

# wait until no pods remain
while kubectl get pods -l app=recommendationservice --no-headers 2>/dev/null | grep -q .; do
  sleep 2
done

kubectl scale deployment recommendationservice --replicas=1
```

Optionally delete stale ReplicaSets (only those with `DESIRED=0`):

```bash
kubectl get rs -l app=recommendationservice
kubectl delete rs recommendationservice-56c58657c5 recommendationservice-56c79bbfb
```

**Why the fix works**

Scaling to zero forces **all** ReplicaSets to zero pods. Scaling back to one creates **exactly one** pod — from the newest ReplicaSet, whose spec includes every patch that was applied before the scale-down. The state converges because there is nothing left to compete.

**Lesson:** never run `kubectl apply` or `kubectl patch` on a deployment that is already failing a rolling update. Either fix the root cause and scale to zero/one, or delete the deployment entirely and re-apply.

---

### Incident 6 — JSON patch on `env` array silently dropped entries

**Symptom**

A single JSON patch with three `{"op":"add","path":".../env/-","value":{...}}` operations applied only the **first** entry.

```bash
# Command that looked correct
kubectl patch deployment recommendationservice --type=json -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"DISABLE_TRACING","value":"1"}},
  {"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"DISABLE_DEBUGGER","value":"1"}}
]'

# Result
kubectl get deployment recommendationservice -o jsonpath='{.spec.template.spec.containers[0].env}'
# → [PORT, PRODUCT_CATALOG_SERVICE_ADDR, DISABLE_PROFILER]
# DISABLE_TRACING and DISABLE_DEBUGGER are missing
```

**Root cause**

The `-` path component means "append to the end of the array." Multiple appends in a single JSON patch against the **same array path** can race during ReplicaSet serialization — the API server serializes the patch into the Deployment, then the Deployment controller serializes it into the ReplicaSet. If two appends resolve to the same index in the intermediate state, one overwrites the other.

This is a known sharp edge of JSON Patch (RFC 6902) on arrays.

**Fix**

Use `kubectl set env` — it computes the final desired state server-side and is idempotent:

```bash
kubectl set env deployment/recommendationservice \
  DISABLE_PROFILER=1 \
  DISABLE_TRACING=1 \
  DISABLE_DEBUGGER=1
```

**Why the fix works**

`kubectl set env` performs a strategic **merge** rather than an array append. It resolves to "ensure these keys exist with these values" — no indexing, no race. Running it twice produces the same result, which JSON Patch does not (running it twice adds duplicates).

**Lesson:** use `kubectl set env` for environment variables. Reserve raw JSON Patch for scalar fields where the path is unambiguous (`/timeoutSeconds`, `/limits/memory`).

**Diagnostic pattern:** after any env patch, always verify:

```bash
kubectl get deployment <name> \
  -o jsonpath='{.spec.template.spec.containers[0].env}' | python3 -m json.tool
```

Never assume a patch that returned "deployment.apps/foo patched" actually applied every operation.

---

## Failure Triage Cheat Sheet

When something is broken, run these in order — each answer narrows the search space.

```bash
# 1. What's broken?
kubectl get pods
```

| Status | Next step |
|---|---|
| `Pending` | `kubectl describe pod <name>` → check `Events` for `Insufficient memory` / `Insufficient cpu` |
| `ContainerCreating` (stuck > 30s) | `kubectl describe pod <name>` → check image pull, volume mounts, network |
| `Running 0/1` | `kubectl logs <name>` → readiness probe failing (app not ready or wrong port) |
| `CrashLoopBackOff` | `kubectl logs <name> --previous` → app crashed; read the last error |
| `ImagePullBackOff` / `ErrImagePull` | `kubectl describe pod <name>` → image name wrong or registry unreachable |
| `OOMKilled` (in `describe`) | Raise `limits.memory` on the Deployment |

```bash
# 2. For any pod not Running — this is your primary diagnostic
kubectl describe pod <name> | tail -40
```

Look at `Events:` at the bottom. Probe failures, image pull errors, scheduling failures, and OOMKills all show up here with timestamps.

```bash
# 3. For application-level failures
kubectl logs <name>              # current run
kubectl logs <name> --previous   # last crashed run (essential for CrashLoopBackOff)
```

```bash
# 4. For networking / DNS / service-to-service failures
kubectl get endpoints                 # is the Service backed by pods?
kubectl exec -it <pod> -- nslookup <service>   # can this pod resolve that service?
kubectl get svc                       # is the port what you expect?
```

```bash
# 5. For cluster-wide state
kubectl get pods -A          # is kube-system healthy?
kubectl get events --sort-by=.metadata.creationTimestamp | tail -20
free -h                      # is the node out of memory?
df -h /var/lib/rancher       # is the disk full?
```

**Rule of thumb:** the answer is in `kubectl describe` **Events** or `kubectl logs --previous` ~90% of the time. If neither tells you, it's a networking or scheduling problem — check endpoints and node resources.
```

---

