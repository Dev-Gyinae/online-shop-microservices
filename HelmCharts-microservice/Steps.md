# README.md — Helm Deployment

---

```markdown
# Online Boutique on k3s via Helm

A Helm-based deployment of Google's [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) (11 microservices) on a single **7.6 GB Linux VM** using **k3s**. Fully declarative: one values file, one install command, no post-install patching.

**Result:** 11/11 pods `Running`, ~800 MiB idle, `helm status` = `deployed` in ~10 seconds.

This is the second deployment path in this repo. The first uses raw Kubernetes manifests (`config.yaml` + `kubectl apply`); this one packages the same stack through Helm for idempotent, parameterized installs.

---

## Prerequisites

| Resource | Minimum | This deployment |
|---|---|---|
| RAM | 4 GB (8 GB safe) | 7.6 GB |
| vCPU | 4 | 4 |
| Disk | 20 GB | 858 GB free |
| OS | Ubuntu 22.04+ / Mint 22+ | Linux Mint 22.2 |
| Network | Outbound to `us-central1-docker.pkg.dev`, `docker.io` | Required |

Cluster prerequisites (from the manifest README, Steps 1–2):

- k3s v1.37 installed with `--disable traefik --disable servicelb --disable metrics-server`
- Node `Ready`, system pods (`coredns`, `local-path-provisioner`) `1/1 Running`
- Helm v3 installed
- No demo workloads currently running (`kubectl get pods` shows only system pods)

---

## Step 1 — Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Expected:

```
version.BuildInfo{Version:"v3.22.1", ...}
```

Helm reads the kubeconfig from `~/.kube/config` by default. k3s writes its kubeconfig to `/etc/rancher/k3s/k3s.yaml`, which is **not** in Helm's default search path.

Fix before proceeding:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER:$USER" ~/.kube/config
chmod 600 ~/.kube/config

# Verify
kubectl get nodes
helm list
```

If `helm list` returns without `connection refused`, the config is in place.

---

## Step 2 — Clone the Source Repository

The chart is inside the Online Boutique source tree, not published as a standalone artifact for this deployment.

```bash
cd ~/Desktop
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo

ls helm-chart/
# → Chart.yaml  README.md  templates/  values.yaml
```

---

## Step 3 — Inspect the Chart

Before writing overrides, confirm what the chart accepts. Helm **silently ignores** unknown keys in a values file — a wrong path does not raise an error, it just does nothing.

```bash
# Top-level keys
grep -E '^[a-zA-Z][a-zA-Z0-9_-]*:' helm-chart/values.yaml

# The three services that have caused trouble in the manifest path
grep -n -A10 '^recommendationService:' helm-chart/values.yaml
grep -n -A8  '^adService:'            helm-chart/values.yaml
grep -n -A8  '^loadGenerator:'        helm-chart/values.yaml

# How images are templated
grep -n -B1 -A4 '^images:' helm-chart/values.yaml
```

Key observations for this chart (v0.10.7):

| Default | Value | Implication |
|---|---|---|
| `images.repository` | `us-central1-docker.pkg.dev/online-boutique-ci/microservices-demo` | Not GCR; uses Google's active CI registry with current tags |
| `images.tag` | `""` | Empty means fall back to `Chart.AppVersion` (v0.10.7) |
| `googleCloudOperations.profiler/tracing/metrics` | all `false` | The three GCP agents are already off — no `DISABLE_*` env var needed |
| `recommendationService.resources` | 220Mi req / 450Mi limit | The memory tuning that was hand-discovered in the manifest path is the chart default |
| `loadGenerator.create` | `true` | This is the one thing that needs to change for a single-VM demo |

---

## Step 4 — Write `values-k3s.yaml`

With those defaults, the override is minimal.

```yaml
# values-k3s.yaml — Online Boutique on single-node k3s (non-GCP)
#
# The chart's defaults already include:
#   - googleCloudOperations: profiler/tracing/metrics all false
#   - recommendationService resources: 220Mi requests / 450Mi limits
#   - adService resources: 180Mi requests / 300Mi limits
#
# Only two overrides are needed for this environment.

# 1. Disable the synthetic-traffic load generator
loadGenerator:
  create: false

# 2. Confirm GCP agents are off (belt and suspenders — the
#    chart default is already false, but this keeps the intent
#    explicit and version-controlled)
googleCloudOperations:
  profiler: false
  tracing: false
  metrics: false
```

Save in the repo root:

```bash
cd ~/Desktop/microservices-demo
# create values-k3s.yaml with the content above
```

---

## Step 5 — Verify the Render Before Touching the Cluster

`helm template` renders the chart locally, with no cluster contact. This catches wrong-key overrides in seconds.

```bash
helm template onlineboutique ./helm-chart -f values-k3s.yaml > /tmp/render.yaml

# What images will be pulled
grep 'image:' /tmp/render.yaml | sed 's/.*image: //' | sort -u

# How many Deployments (10 services + redis = 11; loadgenerator should be absent)
grep -c '^kind: Deployment' /tmp/render.yaml

# Confirm loadgenerator is not a Deployment
grep -B2 -A2 'name: loadgenerator' /tmp/render.yaml | grep '^kind:'
```

Expected:

- 11 image references (10 app services + redis)
- `11` Deployments
- No `kind: Deployment` under a `name: loadgenerator` block

**On the "11" count:** the chart's `cartDatabase.inClusterRedis.create: true` adds a `redis-cart` Deployment. The 10 application services plus redis totals 11. An earlier expectation of "10" would have been wrong.

---

## Step 6 — Pre-Pull Images

Pre-pulling isolates registry failures from deployment failures. The chart pins redis to a specific digest; the app images use `v0.10.7`.

```bash
# Pull everything the render references
for img in $(grep 'image:' /tmp/render.yaml | sed 's/.*image: //' | sort -u); do
  printf '\n=== Pulling %s ===\n' "$img"
  sudo k3s ctr images pull "$img" && echo "OK: $img" || echo "FAILED: $img"
done
```

Two known `ctr` gotchas:

- **`ctr` requires fully qualified image references.** `redis:alpine` fails with `invalid port ":alpine" after host` because `ctr` parses `redis` as a hostname and `alpine` as a port. The chart renders redis with its full path (`redis:alpine@sha256:...`), but the digest-qualified form also needs the registry: `docker.io/library/redis:alpine@sha256:...`.
- **Short-name resolution is a Docker feature, not a containerd feature.** If the chart's rendered image uses a short name, adjust the pre-pull command to include the full path, or rely on the kubelet to resolve it at pod start.

Verify all images are cached:

```bash
sudo k3s ctr images list | grep -cE 'online-boutique-ci|redis'
# → 11
```

---

## Step 7 — Install (No `--wait`)

```bash
cd ~/Desktop/microservices-demo

helm upgrade onlineboutique ./helm-chart \
  --install \
  --namespace default \
  -f values-k3s.yaml \
  --timeout 5m
```

Expected — returns in ~10 seconds:

```
Release "onlineboutique" does not exist. Installing it now.
NAME: onlineboutique
LAST DEPLOYED: ...
NAMESPACE: default
STATUS: deployed
REVISION: 1
```

**`--wait` is deliberately omitted.** See Incident 2 in the Troubleshooting section for why it deadlocks on this cluster.

---

## Step 8 — Watch Pods

```bash
kubectl get pods -w
```

| Elapsed | State |
|---|---|
| 0–20 s | `ContainerCreating` (redis pull only, rest cached) |
| 20–60 s | most `Running 0/1` → `1/1` |
| 60–120 s | `adservice` (JVM) reaches `1/1` |
| 2–3 min | all running |

Confirm:

```bash
kubectl get pods --no-headers | awk '{print $2}' | sort | uniq -c
# → 11  1/1

helm list
# → onlineboutique   1   deployed   onlineboutique-0.10.7
```

---

## Step 9 — Expose the Frontend

The chart creates `frontend-external` as `type: LoadBalancer`. On k3s with `--disable servicelb`, this Service never receives an external IP — `EXTERNAL-IP` stays `<pending>` permanently. This is expected, not an error.

Use `port-forward`, which connects through the Kubernetes API server directly to the pod, bypassing the unprovisioned LoadBalancer:

```bash
kubectl port-forward --address 0.0.0.0 svc/frontend-external 8080:80
```

Verify from the VM:

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8080/
curl -s http://localhost:8080/ | grep -o '<title>[^<]*</title>'
# → HTTP 200
# → <title>Online Boutique</title>
```

Browse from your laptop to `http://<tailscale-ip>:8080`.

---

## Verification

| Action | Service exercised |
|---|---|
| Home page loads with prices | productcatalog + currency |
| Change currency (top-right) | currencyservice |
| Click a product | productcatalog + recommendations |
| Add to cart | cartservice + redis-cart |
| View cart | cartservice + redis-cart |
| Checkout flow | checkout → shipping + payment + email |

---

## Concepts — Helm Terms

| Term | Meaning |
|---|---|
| **Chart** | A packaged application: templates + default `values.yaml` + metadata (`Chart.yaml`). |
| **Release** | One deployed instance of a chart. Same chart, multiple releases, multiple namespaces = multiple independent deployments. |
| **Revision** | Every `helm upgrade` bumps the revision. `helm rollback <release> <revision>` reverts. |
| **Values** | User-supplied overrides merged with chart defaults. Merged in order: chart `values.yaml` → `-f file1.yaml` → `-f file2.yaml` → `--set`. Last wins. |
| **Template** | Go template (`{{ .Values.x }}`). Renders to Kubernetes YAML. `helm template` runs this offline. |
| **`--install`** | Idempotent flag: if the release exists, upgrade; else install. |
| **`--wait`** | Block until every rendered resource reports ready. Not always appropriate — see Incident 2. |
| **`--timeout`** | How long Helm's API operations may take. Distinct from resource readiness. |
| **`helm status`** | Current state of a release: `deployed`, `pending-install`, `pending-upgrade`, `failed`, `uninstalled`. |
| **OCI registry** | Helm charts distributed via OCI. `oci://registry/path/chart`. The Online Boutique chart is available this way (`oci://us-docker.pkg.dev/online-boutique-ci/charts/onlineboutique`), but the local chart path is preferred here for reproducibility. |

---

## Incident Log — What Broke and Why

Six failures across the manifest and Helm deployments. Each entry shows symptom, root cause, fix, and reasoning.

---

### Incident 1 — Helm cannot reach the cluster

**Symptom**

```
Error: Kubernetes cluster unreachable: Get "http://localhost:8080/version":
  dial tcp 127.0.0.1:8080: connect: connection refused
```

**Root cause**

`kubectl` works because the `k3s` binary is symlinked as `kubectl` and auto-reads `/etc/rancher/k3s/k3s.yaml`. Helm has no such fallback: it looks for `~/.kube/config` or `$KUBECONFIG`. Neither existed. Helm then defaults to `http://localhost:8080`, the legacy kubectl default, which is not a k3s API endpoint.

**Fix**

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER:$USER" ~/.kube/config
chmod 600 ~/.kube/config
```

Optionally also export the path permanently:

```bash
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
```

**Why the fix works**

Helm and `kubectl` share the same client-go library. When `$KUBECONFIG` resolves to a readable file with a valid server URL and client certificate, they can both authenticate against the k3s API server. The copy makes the config location standard; the export defends against future `$HOME` differences.

---

### Incident 2 — `helm install --wait` hangs in `pending-install`

**Symptom**

```bash
helm upgrade onlineboutique ./helm-chart --install --wait --timeout 10m
```

Blocks for 10 minutes. Every pod is `1/1 Running` throughout. Final output:

```
Error: timed out waiting for the condition
```

`helm status onlineboutique`:

```
STATUS: pending-install
```

**Root cause**

The chart creates `frontend-external` as `type: LoadBalancer`. `--wait` blocks until every rendered resource is "ready." For a `LoadBalancer` Service on a k3s cluster installed with `--disable servicelb`, no external IP is ever assigned: `EXTERNAL-IP` stays `<pending>` permanently. Helm interprets this as "not ready" and waits until `--timeout`.

The pods are not the problem. The Service is.

**Fix**

Two options:

1. **Drop `--wait`** and watch pods manually:
   ```bash
   helm upgrade onlineboutique ./helm-chart --install --namespace default \
     -f values-k3s.yaml --timeout 5m
   kubectl get pods -w
   ```

2. **Re-enable ServiceLB** in the k3s install (removes `--disable servicelb`) so `LoadBalancer` services get a real IP from the node. Costs ~15 MiB and adds klipper-lb pods.

For a demo VM where `port-forward` already provides access, (1) is cheaper.

**Why the fix works**

Helm's `--wait` is an all-or-nothing gate on every rendered resource. On minimal clusters, some resources (LoadBalancer Services, PersistentVolumeClaims backed by slow storage, custom resources without a reconciler) never reach "ready" within the timeout window. `kubectl get pods -w` gives the same readiness signal for the parts that matter, without blocking on components that will not become ready by design.

**Recovery if a release is stuck in `pending-install`:** scale the release to a clean state before the next attempt.

```bash
helm uninstall onlineboutique

while kubectl get pods --no-headers 2>/dev/null | grep -qE 'Running|Pending|ContainerCreating'; do
  sleep 2
done

helm list    # empty
```

Only after `helm list` is empty can the reinstall succeed.

---

### Incident 3 — Deployment count is 11, not 10

**Symptom**

Render verification expected 10 Deployments (10 application services, loadgenerator disabled) but the count returned 11.

**Root cause**

The chart's `cartDatabase.inClusterRedis.create: true` renders a `redis-cart` Deployment alongside the 10 application services. The `frontend-external` Service is a Service, not a Deployment, so it does not increment the count. Correct total is 10 app services + 1 redis = 11.

**Fix**

Correct the expectation in the verification step. There is nothing to change in the values file.

**Lesson**

Always count from the render, not from an assumed manifest structure. `helm template` output is the truth.

---

### Incident 4 — `ctr images pull` fails on short names

**Symptom**

```
ctr: failed to resolve image: parse "dummy://redis:alpine":
  invalid port ":alpine" after host
```

**Root cause**

`ctr` requires fully qualified image references. `redis:alpine` is parsed as host `redis`, port `alpine`. `docker pull` accepts short names because Docker performs registry resolution; containerd's `ctr` does not.

**Fix**

```bash
sudo k3s ctr images pull docker.io/library/redis:alpine
```

Or, for a digest-pinned image, add the registry:

```bash
sudo k3s ctr images pull docker.io/library/redis:alpine@sha256:<digest>
```

**Why the fix works**

`docker.io/library/redis:alpine` is a fully qualified reference. containerd stores the image under this name, and the Deployment spec references the same name, so the kubelet finds the pre-pulled image and never pulls at runtime.

Note: the chart itself renders `redis:alpine` with its full path and digest, so the *deployment* was fine. Only the pre-pull command needed adjustment.

---

### Incident 5 — `recommendationservice` crash-loops (manifest path only)

**Symptom**

Pod stuck at `0/1 CrashLoopBackOff`. Logs from `kubectl logs --previous`:

```
{"message": "initializing recommendationservice"}
{"message": "Profiler enabled."}
WARNING:google.auth.compute_engine._metadata:Compute Engine Metadata
  server unavailable on attempt 1 of 3. Reason: timed out
{"message": "Sleeping 10 seconds to retry Stackdriver Profiler agent initialization"}
```

Then:

```
"Debugger enabled."
Initializing Cloud Debugger Python agent version: 2.15
```

(no further output — the log stalls)

**Root cause**

The `v0.2.3` Python image enables three Google Cloud agents by default — **Profiler**, **Tracing**, **Debugger**. On a non-GCP host, each attempts to contact the GCP metadata server at `169.254.169.254` and times out.

Profiler retries three times, then sleeps 10 seconds and retries again. Tracing gives up quietly. Debugger hangs indefinitely — the log stalls at `Initializing Cloud Debugger Python agent`.

Meanwhile, the readiness probe has a 1-second timeout. The service has not yet bound to `:8080`. Probe fails, kubelet sends SIGTERM, container exits, restart.

The pod is killed before it can ever finish starting.

**Fix (manifest path only)**

Three changes, all required:

```bash
# 1. Disable all three GCP agents
kubectl set env deployment/recommendationservice \
  DISABLE_PROFILER=1 \
  DISABLE_TRACING=1 \
  DISABLE_DEBUGGER=1

# 2. Raise probe timeouts from 1s to 10s
kubectl patch deployment recommendationservice --type=json -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/timeoutSeconds","value":10},
  {"op":"replace","path":"/spec/template/spec/containers/0/livenessProbe/timeoutSeconds","value":10}
]'

# 3. Raise memory limit (250Mi OOMs during Python import)
kubectl set resources deployment recommendationservice \
  --requests=memory=220Mi --limits=memory=450Mi

# 4. Reset to a clean pod
kubectl scale deployment recommendationservice --replicas=0
# wait until no pods remain, then:
kubectl scale deployment recommendationservice --replicas=1
```

**Why this is not needed in the Helm path**

The chart uses `recommendationservice:v0.10.7`. In that version Google changed the default: `googleCloudOperations.profiler/tracing/metrics` all default to `false`, so no GCP agents are initialized at startup. The `values-k3s.yaml` sets them to `false` explicitly for clarity, but even with chart defaults the pod starts cleanly. `v0.10.7` also ships with `resources.requests.memory: 220Mi` and `limits.memory: 450Mi` as defaults — the same values that were hand-discovered in the manifest path.

This is a **material improvement** of the chart path over the manifest path: the newer image eliminates three separate patches.

---

### Incident 6 — JSON patch on `env` array silently drops entries

**Symptom**

A single JSON patch with multiple `{"op":"add","path":".../env/-","value":{...}}` operations applies only the first entry.

**Root cause**

The `-` path component means "append to the end of the array." Multiple appends against the same array in one JSON patch can race during ReplicaSet serialization. The API server serializes the patch, then the Deployment controller serializes to a ReplicaSet. Concurrent appends can resolve to the same index, and one overwrites the other. This is a known sharp edge of RFC 6902 JSON Patch on arrays.

**Fix**

Use `kubectl set env` — it performs a strategic merge server-side:

```bash
kubectl set env deployment/<name> VAR1=1 VAR2=1 VAR3=1
```

**Why the fix works**

`kubectl set env` computes the final desired state rather than issuing array operations. It resolves to "ensure these keys exist with these values," which is idempotent and race-free. Running it twice produces the same result; running a JSON Patch twice on an append path does not (it adds duplicates).

**Lesson**

Use `kubectl set env` for environment variables. Reserve raw JSON Patch for scalar paths (`/timeoutSeconds`, `/limits/memory`) where the target is unambiguous.

**Diagnostic pattern** — always verify after an env patch:

```bash
kubectl get deployment <name> \
  -o jsonpath='{.spec.template.spec.containers[0].env}' | python3 -m json.tool
```

Never assume a "deployment.apps/foo patched" message means every operation applied.

---

## Failure Triage

When something breaks, run in this order. Each answer narrows the search space.

```bash
# 1. What is broken?
kubectl get pods
```

| Status | Next step |
|---|---|
| `Pending` | `kubectl describe pod <name>` → check Events for `Insufficient memory` / `Insufficient cpu` |
| `ContainerCreating` (> 30 s) | `kubectl describe pod <name>` → image pull, volume mount, or network |
| `Running 0/1` | `kubectl logs <name>` → readiness probe failing (app not ready or wrong port) |
| `CrashLoopBackOff` | `kubectl logs <name> --previous` → last crash log |
| `ImagePullBackOff` / `ErrImagePull` | `kubectl describe pod <name>` → image name wrong or registry unreachable |
| `OOMKilled` (in describe) | Raise `limits.memory` on the Deployment |

```bash
# 2. For any pod not Running
kubectl describe pod <name> | tail -40
```

The `Events:` block at the bottom shows probe failures, image pull errors, scheduling failures, and OOMKills with timestamps. This is the primary diagnostic source.

```bash
# 3. For application-level failures
kubectl logs <name>              # current run
kubectl logs <name> --previous   # last crashed run — essential for CrashLoopBackOff
```

```bash
# 4. For networking, DNS, service-to-service
kubectl get endpoints                          # Service backed by pods?
kubectl exec -it <pod> -- nslookup <service>   # DNS resolution from a pod?
kubectl get svc                                # port matches what you expect?
```

```bash
# 5. For Helm-level state
helm status <release>                # deployed / pending-install / failed
helm get manifest <release>          # what Helm actually applied
helm get values <release>            # user-supplied values (not computed)
helm history <release>               # revision history for rollback
```

**Rule of thumb:** the answer is in `kubectl describe` Events or `kubectl logs --previous` about 90% of the time. If neither tells you, it is a networking or scheduling problem — check endpoints and node resources.

---

## Operations

```bash
# Stop the demo (scale pods to 0, keep objects)
kubectl scale deployment --all --replicas=0

# Restart the demo
kubectl scale deployment --all --replicas=1

# Upgrade to a new chart version (after updating helm-chart/)
helm upgrade onlineboutique ./helm-chart \
  --namespace default \
  -f values-k3s.yaml \
  --timeout 5m

# Roll back to the previous revision
helm rollback onlineboutique

# Roll back to a specific revision
helm history onlineboutique
helm rollback onlineboutique 2

# Remove the release and all its resources
helm uninstall onlineboutique

# Remove the entire cluster
/usr/local/bin/k3s-uninstall.sh
```

---

## Helm vs. Manifest — When to Use Which

| Aspect | Raw Manifest | Helm |
|---|---|---|
| **Auditability** | `git diff` shows every YAML line change | `values-k3s.yaml` shows only the overrides |
| **Reusability** | Copy the file, edit by hand | `helm upgrade --install -f values.yaml` is idempotent |
| **Multi-environment** | Maintain separate manifest files | One chart + `values-dev.yaml`, `values-prod.yaml` |
| **Rollback** | `kubectl apply` previous commit | `helm rollback onlineboutique <revision>` |
| **Upstream updates** | Manually diff and merge | `git pull` the chart source, `helm upgrade` |
| **Number of patches for this demo** | 4 manual patches (adservice, redis, recommendationservice env, memory) | 0 patches — chart defaults + values file cover everything |
| **Learning curve** | Low (Kubernetes YAML only) | Moderate (Go templates, values schema) |

For a portfolio, both matter: the manifest path demonstrates understanding of Kubernetes primitives; the Helm path demonstrates packaging for repeatability.

---

## Design Decisions

| Choice | Reason |
|---|---|
| **Local chart over OCI registry** | No network dependency at install time; the chart is versioned in the repo; `helm show values ./helm-chart` works offline |
| **`values-k3s.yaml` with only two overrides** | The chart's defaults for `v0.10.7` already include the memory tuning and GCP-agents-off configuration that had to be discovered by hand for the manifest path |
| **`--wait` omitted** | `LoadBalancer` readiness never resolves on a cluster without ServiceLB; the pods are ready long before Helm gives up |
| **`port-forward` over the `frontend-external` LoadBalancer** | Bypasses the unprovisioned LoadBalancer entirely; also avoids the `EXTERNAL-IP <pending>` state that would confuse a casual observer |
| **k3s with Traefik, ServiceLB, and metrics-server disabled** | Saves ~200 MiB; for a single-VM demo none of these add value |
| **Pre-pull images before install** | Turns a hard-to-diagnose `ErrImagePull` mid-deployment into a single-command failure at pull time |

