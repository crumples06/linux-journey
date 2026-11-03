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


## StatefulSet

