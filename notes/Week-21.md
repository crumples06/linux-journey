## Week-21
*2/11/26 - 8/11/26*


## Kubernetes
All the configuration in kubernetes is written in yaml files. There is a format for many different things like deployment, service, secrets, etc. 

Every Kubernetes object has the same skeleton:
```yaml
apiVersion: ...  # which API group/version defines this kind of object
kind: ...            # type of object (Deployment, Service, etc.)
metadata: ...   # identity: name, labels
spec: ...          # the desired state, and the contents depend on the kind
```

Generating a starting point with kubectl example:
```
kubectl create deployment guestbook-api --image=guestbook-api:1.0 --replicas=3 --dry-run=client -o yaml
```
`--dry-run=client` creates nothing, It prints the YAML in the terminal which can be edited as needed. The same trick works for other kinds, for example `kubectl create configmap`, `kubectl create secret generic` and `kubectl create ingress`.

There a lot of field in the many different objects in kubernetes, I can look up any field with `kubectl explain`, like `kubectl explain deployment`, `kubectl explain spec.containers.readinessProbe`, It will give a short description of what it is and it's fields.

Objects in kubernetes (values in `kind`):

### Deployment
It provides a way to manage pods and their lifecycles. A Deployment does not manage Pods directly, deployment is used to define the desired state. The Deployment automatically creates and manages a ReplicaSet for each version of the application. The ReplicaSet's sole job is to maintain a stable set of running Pods.
The ReplicaSet creates and deletes individual Pods to match the replica count.

Some important fields in Deployment's YAML file:
`apiVersion: app/v1` is the current stable API version for deployments.

`spec` is the desired state.
- `replicas: 3` is the number of pods that should exist.
- `selector.matchLabels: ` tells the Deployment which Pods it owns. It must match the labels defined in `spec.template.metadata.labels: `

`spec.template` defines the pod blueprint
- `containers :` defines the containers to run.
- `containers.image: ` defines the image of the container
- `containers.resources :` defines CPU and memory requests and limits
- `containers.readinessProbe :` tells Kubernetes when the Pod is ready to receive traffic.
- 'containers.envFrom.configMapRef.name: ` is used to provide the config file for environment varibles.


### Service
A Service is an abstraction that defines a stable network endpoint for a set of Pods. Pods are ephemeral i.e. they're created, destroyed, rescheduled, and get new IPs each time. A Service solves this by providing a fixed virtual IP, a DNS name, and load balancing in front of a dynamic set of Pods.
A Service provides one stable address regardless of how many Pods are running.

A Service works through three interlocking pieces:
1. Label Selector -> EndpointSlice
The service's `selector` matches Pods by labels. Kubernetes automatically creates `EndpointSlice` objects listing the IPs and ports of all ready Pods.

2. ClusterIP
Kubernetes assigns the Service a virtual IP from the cluster's service CIDR. This IP does not belong to any real interface or node and is programmed into kube-proxy on every node. 

3. DNS
Every Service automatically gets a DNS record. For example in my guestbook project, `guestbook-api.default.svc.cluster.local`. Pods can then talk to `guestbook-api` or `guestbook-api.default` instead of hardcoded IPs.

There are 4 service types:
1. `ClusterIP` (default): Exposes to inside the cluster only.
2. `NodePort`: Every node's IP on a static port.
3. `LoadBalancer`: External, via cloud provider's LB
4. `ExternalName`: CNAME alias to an external DNS name

Some fields:
- `spec.selector.app`: Label query to find backend Pods, basically the same value as `spec.template.metadata.labels`.
- `spec.ports`: `port` is the port clients use to reach the Service, `targetPort` is the port on the Pod to forward to.


### ConfigMap
A `ConfigMap` is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume.

In the YAML file for `ConfigMap` there is `data` where key pair values can be written.


### Secret
A Secret is an object that contains a small amount of sensitive data such as a password, a token, or a key. Because Secrets can be created independently of the Pods that use them, there is less risk of the Secret (and its data) being exposed during the workflow of creating, viewing, and editing Pods.


### Ingress
It is used to manages external access to the services in a cluster. Ingress may provide load balancing, SSL termination and name-based virtual hosting. 
The Ingress spec has all the information needed to configure a load balancer or proxy server. Most importantly, it contains a list of rules matched against all incoming requests. 

`spec` fields:
- `ingressClassName`: It is the name of an IngressClass cluster resource. Ingress
    controller implementations use this field to know whether they should be
    serving this Ingress resource. for ex: `ingressClassName: nginx`
- `rules`: It is a list of host rules used to configure the Ingress. 
- `rules.host`: It is the domain name of the host.

`rules.http` contains a list of paths (for example, `/testpath`), each of which has an associated `backend` defined with a `service.name` and a `service.port.name`
Each path in an Ingress is required to have a corresponding path type. Paths that do not include an explicit `pathType` (`rules.host.http.paths.path.pathType`) will fail validation. 
- `Exact`: Matches the URL path exactly and with case sensitivity.
- `Prefix`: Matches based on a URL path prefix split by `/`.


### StatefulSet
A StatefulSet runs a group of Pods, and maintains a sticky identity for each of those Pods. This is useful for managing applications that need persistent storage or a stable, unique network identity.
Deleting and/or scaling a StatefulSet down will not delete the volumes associated with the StatefulSet.

Important fields:
`serviceName: ` is the governing service. It tells Kubernetes which Service owns the pods' network identity.
Just like in Deployment, there is a `template` it's `metadata.labeld.app` and `spec` and it's `spec.containers`.
In the `containers` the container is described like in deployment. with `name`, `image`, `ports`, `env`, `readinessProbe`, etc. Importantly for this particulat object there is one more thing in `containers`, `volumeMounts`. It mounts the named volume.

After the `template` there is `volumeClassTemplates`, which has `metadata.name` (i.e. the name of the volume which is also used in the `volumeMounts` in `containers`), `spec.accessModes`, `spec.resources`, etc.


## k8s-guestbook

In this project there are the following components:
- A simple flask server with database connection
- dockerfile to put the server file in an image
- `api.yaml` that contains the guestbook `deployment` and it's corresponding `service`
- `config.yaml` that contains a `ConfigMap` for mysql environment variables and a `Secret` that contains mysql and root password
- `ingress.yaml` that contains nginx `ingress`
- 'mysql.yaml` that contains a `StatefulSet` for the mysql database and it's corresponding `service`


The app is made of four YAML files. Together they describe the whole system, and traffic flows through them like this:
`Ingress -> Service (guestbook-api) -> API Pods -> Service (mysql) -> MySQL Pod -> PVC`

1. `config.yaml`: configuration and secrets

This file holds two objects separated by `---`.

ConfigMap `guestbook-config` stores non-sensitive settings: `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`. The value `DB_HOST: mysql` works because it is the name of the MySQL Service, and Kubernetes DNS resolves Service names inside the cluster. Port is quoted (`"3306"`) because ConfigMap values must be strings.

Secret `mysql-secret` stores `MYSQL_ROOT_PASSWORD` and `DB_PASSWORD`. I wrote them under `stringData`, so I could use plain text, and Kubernetes stores them base64-encoded. Base64 is encoding, not encryption, so anyone with access to the Secret can decode it. 

2. `mysql.yaml`: the database

Service `mysql` gives the database a stable name and virtual IP. Pods come and go and change IP addresses, but `mysql:3306` stays fixed. The `selector: app: mysql` decides which Pods receive the traffic.

StatefulSet `mysql` runs the database itself. I used a StatefulSet instead of a Deployment because a database needs stable identity and storage. The Pod is always called `mysql-0`, and it keeps the same volume when it is recreated.

- The `env` section builds MySQL’s setup variables from the Secret and ConfigMap, so MySQL creates the `guestbook` database and user on first start.
- The `readinessProbe` runs `mysqladmin ping`. The Pod only counts as Ready (and only receives traffic) once MySQL answers.
- `volumeClaimTemplates` creates a 1Gi `PersistentVolumeClaim` named `data-mysql-0` and mounts it at `/var/lib/mysql`. The data lives in that volume, so deleting the Pod does not delete the data.

3. `api.yaml`: the application

Deployment `guestbook-api` keeps 3 identical Pods running.

- `image: guestbook-api:1.0` with `imagePullPolicy: IfNotPresent` uses the image I loaded into the cluster instead of pulling from a registry.
- `envFrom` loads every key of the ConfigMap as an environment variable. `env` with `secretKeyRef` pulls only `DB_PASSWORD` from the Secret. The two styles do the same job, with `envFrom` being bulk and `secretKeyRef` being per key.
- The readiness probe (`/healthz`) controls whether a Pod receives traffic. The liveness probe controls whether the container gets restarted if it hangs, and its `initialDelaySeconds: 20` gives the app time to start so it isn’t killed early.
- `resources` sets a CPU and memory request (what the scheduler reserves) and a memory limit (exceed it and the container is `OOMKilled`). I set no CPU limit.

Service `guestbook-api` load-balances across the Pods matching `app: guestbook-api`. It listens on port 80 and forwards to the container’s port 3000 (`targetPort`). This is where the pod field in the responses started changing.

4. `ingress.yaml`: the entry from outside

Ingress guestbook is a set of routing rules for the `ingress-nginx` controller. 
It says: requests for host `guestbook.local` with any path starting with `/` (`pathType: Prefix`) go to Service `guestbook-api` on port 80. `ingressClassName: nginx` tells the right controller to pick it up. On its own, an Ingress does nothing, because the controller (running in the `ingress-nginx` namespace) reads the rules and configures nginx.

### Testing
I installed the ingress-nginx controller, then ran `kubectl apply -f k8s/`. I tested with curl: a POST followed by several GETs. The pod field changed between requests, so the Service was spreading traffic across replicas.

`Host header:` the single controller routes by the Host header (name-based virtual hosting). `localhost` doesn’t match my `guestbook.local` rule, so curl needs `-H "Host: guestbook.local"`. In production, DNS does this automatically. 
Instead of having to set the Host everytime in `curl`, i changed the `host: guestbook.local` in `ingress.yaml` to `host: localhost`, so now the default Host as `localhost` works and I don't have to change it to `guestbook.local`.


