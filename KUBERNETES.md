
## Kubernetes Components
Kubernetes has many components:

- **API server** (`kube-apiserver`)
- **Key/value store (K/V store)** (`etcd`)
- **Agent** (`kubelet`)
- **Container Runtime** (e.g. Docker)
- **Controllers** (e.g Replication Controllers, )
- **Scheduler**

The **API server** acts as frontend for Kubernetes. All communications from users pass through the API server (such as `kubectl` commands).

The **K/V store** stores all data used to manage the cluster. It implements locks within the cluster to ensure there are no conflicts between master nodes.

The **Scheduler** distributes work (pods) across nodes.

A **Controller** is responsible for noticing when nodes, containers, or endpoints go down. It makes decisions to bring up new things in those instances

The **Container Runtime** is the underlying software used to run containers (e.g. Docker)

The **Agent** runs on each node in the cluster. It monitors and makes sure the containers are running on the nodes as expected.

---

### `kubectl`

`kubectl` is a CLI tool that you use to deploy and manage applications deployed to a cluster, get status information, etc. Example kubectl commands:

#### Examples

These are the aliases for common Kubernetes objects:

* **deploy** for deployment
* **netpol** for networkpolicy
* **ns** for namespace
* **po** for pod
* **pv** for persistentvolume
* **pvc** for persistentvolumeclaim
* **rs** for replicaset
* **sa** for serviceaccount
* **svc** for service

#### Commands

Get all the things:

- `kubectl get all`

Get the current context:

- `kubectl config current-context`


Explain things:

- `kubectl explain pod`
- `kubectl explain deployment`
- `kubectl explain pod.spec.containers.env.valueFrom`

Run `kubectl` as a reverse proxy:

```
kubectl proxy --port=8080 &
curl http://localhost:8080/api
```

Filter by label:

* `kubectl get <object> --selector <key>=<value>`

---

## Kubernetes Concepts

### Pods and Containers

A *pod* is the smallest object you can create in Kubernetes. It represents a single unit of work (e.g., an instance of an application)

Inside each *pod*, there is at least one *container*.

- *Pods* usually have a 1:1 relationship with *containers*; it is usually convenient to *containerize* an application into a single *container*. Such a *container* could be referred to as a "containerized application", "application container", "application", or just "app".
- Everything in a *pod* is bound to the same lifecycle.
  - `Pending`: The *pod* is waiting to be scheduled on a *node*
  - `ContainerCreating`: The container(s) on the pod are in the process of starting up
  - `Running`: The *containers* are all running normally
  - In addition, pods have true/false *conditions*: `PodScheduled`, `Initialized`, `ContainersReady`, and `Ready`.
- To scale an application, you create additional *pods* on one or more *worker nodes*, usually as *replicas* within a *replica set*. It is non-standard to scale instances of an application by adding *containers* within a *pod*, since they would all be bound to the same lifecycle.
- If a *pod* is running many containers, they usually consist of an application container and one or more "helper containers". They share the same networking and use the same storage space. Therefore, they can communicate over `localhost`, or read and write files to the same volume.

*Containers* can't be created directly, but they can be configured. In a *pod* definition's `spec.containers[*]` block, you can specify:

- `name`: a name for the container
- `image`: the container image (syntax according to your **Container Runtime**)
- `commands`: an optional override for the `ENTRYPOINT` of the container
- `args`: an optional override for the `CMD` of the container
- `env`/`envFrom`: a collection of `name` and `value`/`valueFrom` pairs, or hooks to a *config map*.


#### Examples

* Pod definition file: [pod-definition.yml](./pod-definition.yml)

#### Commands

Run an application:

* `kubectl run <application-name> --image <image-name>` deploys an application to a cluster by creating a single pod on the fly.
  * The `<application-name>` maps to the `metadata.name` of the pod definition.
  * The `<image-name>` maps to the `spec.containers[0].image` of the pod definition
* `kubectl run --generator=run-pod/v1 nginx --image=nginx`

Generate a pod file:

* `kubectl run --generator=run-pod/v1 nginx --image=nginx --dry-run -o yaml`

Get info about pods:

* `kubectl get pods`
* `kubectl get pod <pod-name>`
* `kubectl describe pod <pod-name>`

Get definition, delete, recreate:

1. `kubectl get pod <pod-name> -o yaml > pod-definition.yaml`
1. Make edits to `pod-definition.yml`
1. `kubectl delete pod <pod-name>`
1. `kubectl create -f pod-definition.yml`

Edit a pod on the fly (lots of stuff can't be changed, like image, name, kind, API version):

* `kubectl edit pod <pod-name>`

Execute a command on a running pod:

* `kubectl exec -it <pod-name> -- echo hi`

Follow logs for a running pod:

* `kubectl logs -f <pod-name> <container-name>

### Volumes

*Volumes* are a mechanism for persisting data outside of the lifecycle of a *pod*.

When you create a *volume*, you can configure its storage in a number of ways.

Volumes are mounted into a *pod* via `Pod.spec.containers.volumeMounts`. You can define a volume via `Pod.volumes`; there are lots of different types.

A *persistent volume* is a cluster-wide pool of storage that pods can make *persistent volume claims* to. Then, you can use a *persistent volume claim* in a *pod* by specifying `Pod.spec.volumes.persistentVolumeClaim.claimName`. (Similarly for binding a claim in a *deployment* or *replica set*.)

#### Storage Classes

*Volumes* have to be created before they are available; this requires "static provisioning" if you are using an external storage solution such as hosted cloud storage. *Storage classes* are a solution for "dynamic provisioning" of volumes. *Storage classes* have an associated *provisioner* (in `StorageClass.provisioner`), but there are no associated *persistent volumes*. *Persistent volume claims* refer to an available *storage class* via `PersistentVolumeClaim.spec.storageClassName`.

#### Examples

Persistent volume definition: [pv-definition.yaml](./pv-definition.yaml)
Persistent volume claim definition: [pvc-definition.yaml](./pvc-definition.yaml)
Storage class (GCE): [sc-definition.yaml](./sc-definition.yaml)

### Resource requests

The default *resource request* for a container is 0.5 CPU, and 256 MiB memory (for a default Kubernetes environment). This can be modified in `Pod.spec.containers.resources.requests`. Limits can be set with `Pod.spec.containers.resources.limits`. Pods that try to consume more CPU than their limit will be throttled; pods that exceed their memory limit can continue on for a while, but may eventually be terminated.

### Config Maps and Secrets

*Config Maps* are collections of environment variables that containers can consume.

*Secrets* are encoded *Config Map*s. When you provide values in a definition file, you have to `base64`-encode them:

```
echo -n "opensesame" | base64
echo -n "b3BlbnNlc2FtZQ==" | base64 --decode
```

#### Examples

`Pod.spec.containers` can take several different forms of environment configuration.

* Raw key-value pairs:

  ```
  env:
  - name: FOO
    value: hello
  - name: BAR
    value: world
  ```

* With values referencing a *config map*:

  ```
  env:
  - name: FOO
    valueFrom:
      configMapKeyRef:
        name: my-app-config
        key: FOO
  - name: BAR
    value: world
  ```

* With values referencing a *secret*:

  ```
  env:
  - name: FOO
    valueFrom:
      secretKeyRef:
        name: my-app-config
        key: FOO
  ```

* By referencing an entire *config map*:

  ```
  envFrom:
  - configMapRef:
      name: my-app-config
  ```

* By referencing a *secret*:

  ```
  envFrom:
  - secretRef:
      name: my-secrets
  ```

* As a file in a *volume*:

  ```
  volumes:
  - name: app-config-volume
    configMap:
      name: my-app-config
  ```

#### Commands

Make config maps:

* `kubectl create configmap my-config-map --from-literal=FOO=bar`
* `kubectl create configmap my-config-map --from-file=<path-to-file>`
* `kubectl create -f config-map.yaml`

Make secrets:

* `kubectl create secret generic my-secret --from-literal="PASSWORD=opensesame"`
* `kubectl create secret generic my-secret --from-file=secrets.properties
* `kubectl create -f my-secrets.yaml

Get values of secrets:

* `kubectl get secret my-secrets -o yaml`

### Service Accounts

*Service accounts* provide a mechanism for programs to access Kubernetes. When you create a *service account*, a secret with a *token* is created and bound to that *service account*. You can authenticate with the Kubernetes API by using the *token* as a bearer token in a request.

```
curl -X GET -H "Authorization: Bearer $TOKEN" $APISERVER/api --insecure
```

You can mount a service account to a container by setting `pod.spec.serviceAccount`.

#### Examples

View information for the default service account in a running pod:

* `kubectl exec -it <pod> ls /var/run/secrets/kubernetes.io/serviceaccount`

Make a service account:

- `kubectl create serviceaccount my-sa`

Get the API token for the default service account:

* `kubectl get secrets -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='default')].data.token}" | base64 decode`
* `kubectl get secret $(kubectl get serviceaccount default -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 --decode`


### Replicas

A **Replication Controller** monitors *replicas*. *Replicas* are *pods* defined by a template. The **Replication Controller** ensures that the specified number of *replicas* of the *pod*. It is responsible for driving toward the defined target state, by creating or destroying *replicas*.

A **Replica Set** monitors *replicas*, and is a slightly higher-level abstraction. It serves the same purpose as the **Replication Controller**, except that it can take ownership of monitoring *pods* that didn't create itself. It uses a **Replication Controller** under the hood.

#### Examples

* Replication Controller definition file: [rc-definition.yml](./rc-definition.yml)
  * The pod template is under `spec.template`.
* Replica Set definition file: [replicaset-definiton.yml](./replicaset-definition.yml)

#### Commands

Get info:

* `kubectl get replicationcontrollers`
* `kubectl get replicasets`

Delete:

* `kubectl delete replicationcontroller <name>`
* `kubectl delete replicaset <name>`

Scale instances of a replica set:

* Edit a replica set definition file, then run `kubectl replace -f replicaset-definition.yml`

* `kubectl scale --replicas=6 -f replicaset-definition.yml`
* `kubectl scale --replicas=6 replicaset <replicaset-name>`

### Jobs and CronJobs

*Jobs* manage a workload that is meant to run for some fixed amount of time and then exit.

You can scale workloads across `jobs` by setting `Job.spec.completions`. By default these will run serially; modify this with `Job.spec.parallelism`.

#### Examples

Job definition: [job-definition.yaml](./job-definition.yaml)

Cron job definition: [cron-job-definition.yaml](./cron-job-definition.yaml)

### Deployments

When you create a new *deployment*, you trigger a *rollout*, which creates a new *revision*. If the *pod* definition in the deployment is updated with a new *container* version, a new *revision* is created. The way in which the new *revision* is created depends on the *strategy* (`Recreate` or `RollingUpdate`)

#### Examples

Look at the status of a *rollout*:

* `kubectl rollout status deployment/<deployment-name>`

Look at the rollout history of a deployment:

* `kubectl rollout history deployment/<deployment-name>`

#### Commands

Create a deployment imperatively (DEPRECATED)

* `kubectl run --generator=deployment/apps.v1 webapp --image=kodekloud/webapp-color --replicas=3`

Manually set a *container* version on a *deployment*:

* Edit the definition file and run `kubectl apply`
* `kubectl set image deployment/<name> nginx=nginx:1.9.1`

Undo a rollout ("rollback"):

* `kubectl rollout undo deployment/<deployment-name>`

### Stateful Sets

*Stateful sets* are a variant of *deployments*. Unlike deployments:

* *Pods* are created serially
* *Pod names* are uniquely identifiable and stable
* The `kind` is `StatefulSet`
* They specify a *headless service* (via `StatefulSet.spec.serviceName`)

Scaling also works in an ordered way (by default; configurable via `StatefulSet.spec.podManagementPolicy`).

A *headless service* creates stable DNS entries for pods, without providing the other features of a *service*, like load balancing.

#### Examples

Create a *headless service* by setting `Service.spec.clusterIP` to `None` in service definition:

```
apiVersion: v1
kind: Service
metadata:
  name: mysql-h
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
```

Deploy a `Pod` manually to this headless service by defining a `Pod.spec.subdomain` that matches the `Service.metadata.name`, and `Pod.spec.hostname` set as well:

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - image: mysql
    name: mysql
  subdomain: mysql-h
  hostname: mysql-pod
```

The DNS entry for this pod will be `mysql-pod.mysql-h.default.svc.cluster.local` (assuming it's deployed to the `default` namespace).

In a `StatefulSet` definition, you don't need the subdomain or the hostname specified on the *pod* template (but the *headless service* must be provided):

```
apiVersion: v1
kind: StatefulSet
metadata:
  name: mysql-deployment
  labels:
    app: mysql
spec:
  serviceName: mysql-h
  replicas: 3
  matchLabels:
    app: mysql
  template:
    metadata:
      name: myapp-pod
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql
        name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data-volume
    spec:
      accessModes: [ReadWriteMany]
      storageClassName: google-storage
      resources:
        requests:
          storage: 500Mi
```

The pods in the stateful set will have DNS entries of `mysql-<x>.mysql-h.default.svc.cluster.local`, where `<x>` is the order in which they were deployed.

This *stateful set* also uses a *persistent volume claim* template to assign an individual volume claim to each pod. This makes sense for e.g. data replication: if a pod is terminated and recreated, the persistent volume won't be released, and the PVC will be re-bound to the pod that restarts with the same identity. In other words, *stateful sets* ensure stable storage for *pods*.

### Services

*Services* enable communication between components within and outside of an application. A *cluster IP service* unifies communication between multiple pods into a single interface. A *node port service* is a *cluster IP service* that also exposes a port on the *node*.

A *node port service* (`Service.spec.type` of `NodePort`) needs three port configurations (in `Service.spec.ports`):
* The `nodePort`, the port on the node's network that it will receive requests through. Node ports (by default) can only be in the 30000-32767 range. If not specified, a random available port is selected in the range.
* The `port`, where the service is running on its cluster IP
* The `targetPort`, where the pod is waiting for requests. If not specified, it defaults to `port`.

Link a `service` to `pods` using `Service.spec.selector` to identify the pods. If the labels identify multiple pods (e.g. there are replicas), it will pick one using an algorithm. Be default, this algorithm selects a random pod with session affinity. If identified *pods* are distributed across *nodes*, the *service* will span all the necessary *nodes*.

By defualt, a *service* assumes a *pod* is ready for traffic after the pod's container has been created. You can modify this behavior with a pod's *readiness probe*.


#### Examples

Configure a custom *readiness probe* for a pod by configuring `Pod.spec.containers.readinessProbe`:

```
spec:
  containers:
  - image: nginx
    name: nginx
    readinessProbe:
      httpGet:
        path: /api/ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 8
```

```
spec:
  containers:
  - image: nginx
    name: nginx
    readinessProbe:
      tcpSocket:
        port: 3306
```

```
spec:
  containers:
  - image: nginx
    name: nginx
    readinessProbe:
      exec:
        command:
        - cat
        - /app/is_ready
```

#### Commands

Expose a service imperatively:

* `kubectl expose pod redis --port=6379 --name redis-service`

### Ingresses

An *ingress* is an object for routing requests to different *services*.

To deploy an *ingress*, there must be an *ingress controller* in your cluster. Kubernetes doesn't deploy one by default.

#### Examples

An ingress definition looks kinda like this:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - http:
      paths:
      - path: /app-one
        backend:
          serviceName: app-one-service
          servicePort: 8080
  - http:
      paths:
      - path: /app-two
        backend:
          serviceName: app-two-service
          servicePort: 8080
```

This kind of ingress would map `<ingress-controller>/app-one` to `app-one-service:8080`, and `<ingress-controller>/app-two` to `app-two-service:8080`. Details depend on the configuration of the ingress controller (e.g. NGINX provides a "rewrite targets" configuration option to strip the paths of the original requests).

### Network Policies

By default, any *pod* can talk to any other *pod* within the Kubernetes private network. You can impose *network policies* to modify this behavior and control *ingress* and *egress* rules to and from a *pod*.

A *network policy* identifies a *pod* by selecting against its labels.

Not all networking solutions support network policies; depends on the configuration of the cluster.

#### Examples

Network policy definition: [network-policy-definition.yaml](./network-policy-definition.yaml)

### Nodes

A *node* is a machine, physical or virtual, where some Kubernetes *components* are installed

A *master node* is a *node* in a cluster that monitors the *worker nodes* in your cluster.

- The **API server** runs on the master node
- etcd runs on the master node
- The Controller runs on the master node
- The Scheduler runs on the master node

A *worker node* runs workloads (i.e. containers).

- The **Container Runtime** runs on *worker nodes*.
- The **Agent** (`kubelet`) runs on *worker nodes*.

A *worker node* is sometimes called a "minion." A *master node* is sometimes referred to as the "master."

A single *node* can be both a *master* and a *worker*, for example, in the default Docker Desktop deployment

*Nodes* are given *pods* to run. A *node* can have many *pods* at once.

#### Taints, Tolerations, and Affinities

You can control which *pods* get placed on which *nodes* using *taints* and *tolerations*. If a node is given a *taint*, only pods that are *tolerant* of that *taint* can be placed on it. The three taint effects are:

- `NoSchedule`: don't schedule intolerant pods on this node, but let things keep running even if they are intolerant
- `PreferNoSchedule`: avoid scheduling intolerant nodes on this pod if possible
- `NoExecute`: don't schedule intolerant pods on this node, and evict currently running pods if they are intolerant

Tainting a *node* does not guarantee that tolerant *pods* will be placed on that *node*. *Affinity* is an expression by a *pod* to prefer deployment to a certain *node*.

There are only two kinds of node affinities: `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution`. There is one more planned: `requiredDuringSchedulingRequiredDuringExecution`.

#### Examples

Pod definition with *affinity* for a *node* that has been labeled `size: Large`:

```
apiVersion: v1
kind: Pod
metadata:
  name: gimme-large
spec:
  containers:
  - image: big-and-beefy
  nodeSelector:
    size: Large
```

Pod definition with *affinity* for a *node* that has been labeled `size: Large` or `size: Medium`:

```
apiVersion: v1
kind: Pod
metadata:
  name: gimme-large-or-medium
spec:
  containers:
  - image: big-and-beefy
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values: [Large, Medium]
    size: Large
```

#### Commands

Get info about nodes:

- `kubectl get nodes`
- `kubectl get node <node-name>`
- `kubectl describe node <node-name>`

Label a node:

- `kubectl label node <node-name> <key>=<value>

Taint a node:

- `kubectl taint nodes <node-name> <key>=<value>:<taint-effect>`
  - `kubectl taint nodes <node-name> app=blue:NoSchedule`

Schedule a pod to a node like this using a `Pod.spec.tolerations` block:

```
kubectl create -f <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: tolerant-pod
spec:
  containers:
  - image: nginx
    name: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
EOF
```


### Namespaces

There are three namespaces by default: `default`, `kube-system`, and `kube-public`. Namespaces provide isolation to entities within that namespace.

Namespaces can have their own sets of policies and resource limits.

Namespaces provide DNS lookup by name. To access resources within the same namespace, you can use the resource name (e.g)

When a service is created, a DNS entry is added automatically in the format `<service-name>.<namespace>.svc.<cluster-domain-name>`. The default cluster's domain name is `cluster.local`.

Every *namespace* contains a default *service account* named `default`. The `default` service account is automatically mounted as a *volume* to any *pod* deployed in that *namespace*. (You can disable this by setting `pod.spec.automountServiceAccountToken` to `false`, or providing `pod.spec.serviceAccount`.

#### Examples

Connect to a resource called `db-service` from an application in the same namespace:

```
mysql.connect("db-service")
```

Connect to a resource called `db-service` in the `dev` namespace from an application outside of the `dev` namespace:

```
mysql.connect("db-service.dev.svc.cluster.local")
```

Resource quota definition: [resource-quota-dev.yml](./resource-quota-dev.yml)

#### Commands

Create a namespace:

* `kubectl create namespace dev`
* `kubectl create -f namespace-dev.yml`

Run a command in a specific namespace:

* `kubectl create -f pod-definition.yml --namespace=dev`
* Specify `metadata.namespace` in a definition file

Change the namespace `kubectl` targets by default:

* `kubectl config set-context $(kubectl config current-context) --namespace=dev`

Do things in all namespaces with the `--all-namespaces` flag:

* `kubectl get pods --all-namespaces`
* `kubectl get deployments --all-namespaces`


### Clusters

A *cluster* is a grouped set of *nodes*.

#### Commands

Show information about a cluster:

- `kubectl cluster-info`


