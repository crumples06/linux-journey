# Week-19
*19/10/26 - 25/10/26*

## Kubernetes

### Kubernetes vs Docker
They solve different problems and are used together.
- **Docker** builds images and runs containers on **one machine**.
- **Kubernetes** orchestrates containers **across many machines**: keeps N copies running, replaces dead ones, does rolling updates, load balances, scales, manages config and secrets.
- Kubernetes doesn't build images and doesn't run containers itself. It hands that to a container runtime (containerd). Docker-built images work fine because they follow the OCI standard.
- Closest comparison is Compose, but Compose only works on one host and has no continuous self-healing.

## Setup

- Docker Desktop already bundles `kubectl`, and also has a built-in Kubernetes cluster (Settings -> Kubernetes -> Enable). 
- `kubectl get nodes` showed one node, `Ready`, and `kubectl config current-context` showed `docker-desktop`. The node name was `desktop-control-plane`, so the cluster itself is a container.
- Always check the context before running commands: `kubectl config current-context`.
- Use `curl.exe` in PowerShell, since plain `curl` is an alias for something else.

## Fundamentals

### The cluster
- **Control plane** (the brain):
  - *API server*: the only front door. `kubectl` and every other component talk to it.
  - *etcd*: database holding the desired state.
  - *Scheduler*: decides which node a new Pod runs on.
  - *Controller manager*: runs the control loops that make reality match the desired state.
- **Worker nodes** (run the containers):
  - *kubelet*: agent that makes sure the containers it's told about are running.
  - *Container runtime*: actually runs them (containerd).
  - *kube-proxy*: sets up the forwarding rules for Services.

### The declarative model
I never say "start a container". I submit a description of what should exist, it's stored in etcd, and controllers keep working until the cluster matches it. Every object has `apiVersion`, `kind`, `metadata`, `spec` (what I want). The cluster fills in `status` (what's actually true).

### Core objects
- **Pod**: smallest unit. Disposable; a replacement gets a new name and IP.
- **Deployment**: keeps N replicas running, handles rolling updates and rollbacks. Uses a **ReplicaSet** underneath.
- **Service**: a stable address in front of changing Pods.
- **Namespace**: grouping of resources inside one cluster.

### What's new compared to Docker
1. **Reconciliation.** `docker compose up` does something once and exits. Kubernetes controllers run permanently and keep correcting any difference between declared and actual state.
2. **Pods are disposable.** Never address one directly.
3. **Services exist because of that.** Stable name and IP in front of changing Pods.
4. **Labels and selectors.** Objects find each other by matching labels, not by name. A mismatch fails silently.
5. **Scheduling across machines.** I don't pick which node a container runs on.

## How it actually works

What happened when I ran `kubectl apply -f deployment.yaml`:
1. `kubectl` sent the YAML to the **API server**, which saved a Deployment record in **etcd**. No containers exist yet.
2. The **Deployment controller** saw the record and created a **ReplicaSet** record ("3 Pods from this template").
3. The **ReplicaSet controller** saw 0 Pods but 3 wanted, and created 3 **Pod records** (database entries, no node yet).
4. The **scheduler** saw Pods with no node and wrote a node name onto each.
5. The **kubelet** on that node saw a Pod assigned to it, told containerd to pull the image and start the container, and reported back. That's `ContainerCreating` -> `Running`.

Key point: the components never talk to each other. They only read and write records through the API server, each reacting to the records it cares about.

- **Deleting a Pod brings one back** because the ReplicaSet controller sees 2 Pods when it wants 3 and creates another Pod record. The old Pod isn't restarted, it's gone. That's why the replacement has a new name.
- **A rolling update replaces Pods** because changing the image changes the Pod template, which produces a new template hash, so a **second ReplicaSet** is created. It scales up while the old one scales down. Changing only `replicas` doesn't change the template, so the same ReplicaSet just adds Pods.
- **A Service finds Pods** through an endpoints controller. It collects the IPs of Ready Pods matching the selector and writes an Endpoints record. **kube-proxy** on each node reads those and sets up forwarding rules. No match means nothing to forward to, and nothing reports an error.
- **Pod names** have three parts: `web` (Deployment), `7887448d46` (ReplicaSet hash), `94h5k` (random suffix).

## What a Service is

A Service is a stable address that forwards to whichever Pods currently match a label. It isn't a program or a container. It's a record (fixed IP + DNS name) plus forwarding rules set up by kube-proxy. Inside the cluster other Pods can reach it as just `web`, or `web.default.svc.cluster.local`. Same job as the service name in Compose, but split into its own object because Pods come and go constantly.

## YAML structure

### deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web                 # the Deployment's identity
spec:
  replicas: 3               # how many Pods
  selector:                 # "which Pods are mine?"
    matchLabels:
      app: web
  template:                 # blueprint for each Pod
    metadata:
      labels:
        app: web            # label stamped on every Pod
    spec:
      containers:           # a list
        - name: nginx
          image: nginx:1.27
```
- Three nested layers: Deployment -> `template` (a Pod description) -> `containers`.
- `template` is a standalone Pod file without `apiVersion`, `kind` and the Pod name (the Deployment generates names).
- `selector.matchLabels` asks "which Pods are mine?" and `template.metadata.labels` answers it. They must agree, or `apply` is rejected. The selector can't be changed after creation.
- `containers` is a list because a Pod can hold several (sidecars). The container `name` is unrelated to the image name.

### services.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web              # Service identity AND DNS name
spec:
  selector:
    app: web             # forward to Pods with this label
  ports:
    - port: 80           # port the Service listens on
      targetPort: 80     # port on the container to forward to
```
- `port` and `targetPort` are the two halves of Docker's `-p 8080:80`.
- The Service's `selector` is a plain map; the Deployment's is wrapped in `matchLabels`. Same job, two formats.
- `type` defaults to `ClusterIP` (internal only).
- The Service knows nothing about the Deployment. It only sees Pods carrying a label.
- `app` is only a convention. The word `web` appears in three unrelated roles (Deployment name, label value, Service name).
- `apiVersion`: Pods, Services, ConfigMaps, Secrets use `v1`; Deployments and ReplicaSets use `apps/v1`.

## Commands

Pattern: `kubectl <verb> <type> <name>`

| Command | What it does |
|---|---|
| `kubectl apply -f file.yaml` | Create or update whatever the file defines |
| `kubectl get pods` | List objects (also `nodes`, `services`, `deployments`, `rs`, `endpoints`) |
| `kubectl describe pod <name>` | Full details plus the **Events** log at the bottom |
| `kubectl delete -f file.yaml` | Delete what the file defines |
| `kubectl delete pod <name>` | Delete one object |
| `kubectl create deployment web --image=nginx --replicas=3` | Imperative version; fine for tests, the file is what I keep |
| `kubectl rollout status deployment/web` | Watch an update progress |
| `kubectl rollout undo deployment/web` | Roll back to the previous ReplicaSet |
| `kubectl port-forward service/web 8080:80` | `localhost:8080` -> port 80. Local port : remote port. Runs until Ctrl+C |
| `kubectl get endpoints web` | Pod IPs a Service currently points at |
| `kubectl get rs` | Show ReplicaSets (both old and new during a rollout) |
| `kubectl config current-context` | Which cluster commands go to |
| `kubectl explain deployment.spec.selector` | Documentation of any field from the terminal |
| `kubectl get deployment web -o yaml` | Full object including `status` |

**Debugging habit:** `kubectl get pods`, then `describe pod`, then `logs`, then `get events`. Most debugging is reading those four.

## Hands-on log

### Stage 1: reconciliation
- `kubectl create deployment web --image=nginx --replicas=3`, then deleted one Pod.
- A new Pod appeared within seconds with a new name; the other two kept their names and ages. `ContainerCreating` at the start was the image pull, so the work network isn't blocking nginx.

### Stage 2: manifests
- Wrote `deployment.yaml` and applied it. Changed replicas 3 -> 5 and re-applied: the 3 existing Pods stayed, 2 new ones were added, all sharing the same hash.
- Changed the image tag and applied: the rollout log showed a new ReplicaSet scaling up while the old one scaled down ("2 old replicas are pending termination"). Pods are **replaced, not modified in place**.
- `rollout undo` warned that the resource was managed with `kubectl apply`. It changes the live object but **not** my YAML file or the recorded last-applied configuration. If the file still says the new image, the next `apply` rolls forward again. **The file is the source of truth**, like Ansible. `rollout undo` is the emergency lever; the real fix is editing the file back and applying.

### Stage 3: Services and labels
- Applied `deployment.yaml` and `services.yaml`. `describe service web` listed 5 Pod IPs under `Endpoints:`.
- `port-forward` + `curl` returned the nginx welcome page.
- Changed the selector to `app: wbe` and applied: **no error**, but `Endpoints:` came back empty. Kubernetes validates syntax, not whether a selector matches anything. This is the silent failure.
- The `curl: (7) Failed to connect` afterwards was because I'd stopped the port-forward, not because of the broken selector. `port-forward` tunnels to **one specific Pod** chosen when it starts and skips the Service forwarding rules, so it's a poor way to test a selector. Better check: `kubectl get endpoints web`.
