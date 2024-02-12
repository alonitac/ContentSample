# Pod design

In this tutorial we will take a closer look on Pods characteristics, how they are designed and scheduled on nodes properly. Please fasten your seat belt. 

Throughout this tutorial, we will work with the `frontend` service (the main service of the [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) app).

Let's create a `frontend` Pod:

```yaml
# k8s/resources-demo.yaml 

apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:
    app: frontend-demo
spec:
  containers:
  - name: server
    image: gcr.io/google-samples/microservices-demo/frontend:v0.8.0
    ports:
      - containerPort: 8080
    env:
      - name: PORT
        value: "8080"
      - name: PRODUCT_CATALOG_SERVICE_ADDR
        value: "productcatalogservice:3550"
      - name: CURRENCY_SERVICE_ADDR
        value: "currencyservice:7000"
      - name: CART_SERVICE_ADDR
        value: "cartservice:7070"
      - name: RECOMMENDATION_SERVICE_ADDR
        value: "recommendationservice:8080"
      - name: SHIPPING_SERVICE_ADDR
        value: "shippingservice:50051"
      - name: CHECKOUT_SERVICE_ADDR
        value: "checkoutservice:5050"
      - name: AD_SERVICE_ADDR
        value: "adservice:9555"
      - name: ENABLE_PROFILER
        value: "0"
    resources:
      requests:
        cpu: 50m
        memory: 5Mi
      limits:
        cpu: 100m
        memory: 128Mi
```

## Describe Pod

The `kubectl describe` command displays a comprehensive set of information about the specified pod:

```console
$ kubectl describe pod frontend-pod
Name:             frontend-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Thu, 26 Oct 2023 18:53:00 +0000
Labels:           app=frontend-demo
Annotations:      <none>
Status:           Running
IP:               10.244.0.48
IPs:
  IP:  10.244.0.48
Containers:
  server:
    Container ID:   docker://04601c1f153a380555d343613db121e445bdcdc02963d549638c50fac7e770cd
    Image:          gcr.io/google-samples/microservices-demo/frontend:v0.8.0
    Image ID:       docker-pullable://gcr.io/google-samples/microservices-demo/frontend@sha256:048d9344b759afffd52f4585fb11ee5842f013857eec1136b60973f9ee2fc8f0
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 26 Aug 2023 18:53:01 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     100m
      memory:  128Mi
    Requests:
      cpu:     50m
      memory:  5Mi
    Environment:
      PORT:                          8080
      PRODUCT_CATALOG_SERVICE_ADDR:  productcatalogservice:3550
      CURRENCY_SERVICE_ADDR:         currencyservice:7000
      CART_SERVICE_ADDR:             cartservice:7070
      RECOMMENDATION_SERVICE_ADDR:   recommendationservice:8080
      SHIPPING_SERVICE_ADDR:         shippingservice:50051
      CHECKOUT_SERVICE_ADDR:         checkoutservice:5050
      AD_SERVICE_ADDR:               adservice:9555
      ENABLE_PROFILER:               0
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-8t2bj (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-8t2bj:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  33m   default-scheduler  Successfully assigned default/frontend-pod to minikube
  Normal  Pulled     33m   kubelet            Container image "gcr.io/google-samples/microservices-demo/frontend:v0.8.0" already present on machine
  Normal  Created    33m   kubelet            Created container server
  Normal  Started    33m   kubelet            Started container server
```


This command is very useful for diagnosing issues, understanding the current [state of a pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase) and [containers](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-states), and gathering information for debugging purposes, [Pod conditions](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions), Volumes, Network and Events.

## Resource management for Pods and containers

As can be seen from the above YAML manifest and Pod description, when you specify a Pod, you can optionally specify how much of each resource (CPU and memory) a container needs: 

```text 
Limits:
  cpu:     100m
  memory:  128Mi
Requests:
  cpu:     50m
  memory:  5Mi
```

What do the **Limits** and **Requests** mean? 

### Resource limits

When you specify a resource **Limits** for a container, the **kubelet** enforces those limits, as follows:

- If the container tries to consume more than the allowed amount of memory, the system kernel terminates the process that attempted the allocation, with an out of memory (OOM) error.
- If the container tries to consume more than the allowed amount of CPU, it is just not allowed to do it.

Let's connect to the Pod and create an artificial CPU load, using he `stress` command:

```console 
$ kubectl exec -it frontend-pod -- /bin/sh
/src # apk update
...

/src # apk add stress-ng
(1/5) Installing judy (1.0.5-r1)
(2/5) Installing libmd (1.0.4-r2)
(3/5) Installing libbsd (0.11.7-r1)
(4/5) Installing liblksctp (1.0.19-r3)
(5/5) Installing stress-ng (0.14.00-r0)
Executing busybox-1.36.0-r9.trigger
OK: 25 MiB in 37 packages

/src # stress-ng --cpu 2 --vm 1 --vm-bytes 50M -v
stress-ng: info:  [723] defaulting to a 86400 second (1 day, 0.00 secs) run per stressor
stress-ng: info:  [723] dispatching hogs: 2 cpu, 1 vm
```

Watch the Pod's CPU and memory metrics in the Kubernetes UI Dashboard. Although the stress test tries to use 2 cpus, it's limited to 100m (100 mili-cpu, which is equivalent to 0.1 cpu), as specified in the `.resources.limits.cpu` entry.
In addition, the Pod **is** able to use a `50M` of memory as it's below the specified limit (128 megabytes). 

Let's try to use more than the memory limit:


```console
/src # stress-ng --cpu 2 --vm 1 --vm-bytes 500M -v
stress-ng: info:  [723] defaulting to a 86400 second (1 day, 0.00 secs) run per stressor
stress-ng: info:  [723] dispatching hogs: 2 cpu, 1 vm
```

Watch how the processed is killed by the kernel using a `SIGKILL` signal. 

### Resources requests 

While resources limits is quite straight forward, resources request has a completely different purpose. 

When you specify the resource **Requests** for containers in a Pod, the **kube-scheduler** uses this information to decide which node to place the Pod on. How?   

When a Pod is created, the kube-scheduler should select a Node for the Pod to run on. Each node has a maximum capacity for CPU and RAM. 
The scheduler ensures that the resource CPU and memory requests of the scheduled containers is less than the capacity of the node.

```console
$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   15d   v1.27.4

$ kubectl describe nodes minikube
Name:               minikube
[ ... lines removed for clarity ...]
Capacity:
  cpu:                2
  ephemeral-storage:  30297152Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3956276Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  30297152Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3956276Ki
  pods:               110
[ ... lines removed for clarity ...]
Non-terminated Pods:          (23 in total)
  Namespace                   Name                                          CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                          ------------  ----------  ---------------  -------------  ---
  default                     adservice-746b758986-vh7zt                    100m (5%)     300m (15%)  180Mi (4%)       300Mi (7%)     15d
  default                     cartservice-5d844fc8b7-rl7fp                  200m (10%)    300m (15%)  64Mi (1%)        128Mi (3%)     15d
  default                     checkoutservice-5b8645f5f4-gbpwn              50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     currencyservice-79b446569d-dqq7l              50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     emailservice-55df5dcf48-5ksdz                 50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     frontend-66b6775756-bxhbq                     50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     frontend-pod                                  50m (2%)      100m (5%)   5Mi (0%)         128Mi (3%)     50m
  default                     loadgenerator-78964b9495-rphvs                50m (2%)      500m (25%)  256Mi (6%)       512Mi (13%)    15d
  default                     paymentservice-8f98685c6-cjcmx                50m (2%)      200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     productcatalogservice-5b9df8d49b-ng6pz        100m (5%)     200m (10%)  64Mi (1%)        128Mi (3%)     15d
  default                     recommendationservice-5b4bbc7cd4-rkdft        50m (2%)      200m (10%)  220Mi (5%)       450Mi (11%)    15d
  default                     redis-cart-76b9545755-nr785                   70m (3%)      125m (6%)   200Mi (5%)       256Mi (6%)     15d
  default                     shippingservice-648c56798-m8vjh               100m (5%)     200m (10%)  64Mi (1%)        128Mi (3%)     15d
  kube-system                 coredns-5d78c9869d-jkx8m                      100m (5%)     0 (0%)      70Mi (1%)        170Mi (4%)     15d
  kube-system                 etcd-minikube                                 100m (5%)     0 (0%)      100Mi (2%)       0 (0%)         15d
  kube-system                 kube-apiserver-minikube                       250m (12%)    0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                 kube-controller-manager-minikube              200m (10%)    0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                 kube-proxy-kv6m6                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                 kube-scheduler-minikube                       100m (5%)     0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                 metrics-server-7746886d4f-t5btd               100m (5%)     0 (0%)      200Mi (5%)       0 (0%)         15d
  kube-system                 storage-provisioner                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kubernetes-dashboard        dashboard-metrics-scraper-5dd9cbfd69-lm4s2    0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kubernetes-dashboard        kubernetes-dashboard-5c5cfc8747-86kzb         0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                1820m (91%)   2925m (146%)
  memory             1743Mi (45%)  2840Mi (73%)
```

By looking at the “Pods” section, you can see which Pods are taking up space on the node.

Note that although actual memory resource usage on nodes is low (`45%`), the scheduler still refuses to place a Pod on a node if the memory request is more than the available capacity of the node. 
This protects against a resource shortage on a node when resource usage later increases, for example, during a daily peak in request rate.

Test it yourself! Try to change the `.resources.request.cpu` to a larger value (e.g. `300m` instead of `50m`), and re-apply the Pod. What happened? 

Since the total CPU requests on that node is `91%` (`1820m` out of `2000m` CPU), you can see that if a new Pod requests more than `180m` or more than `1.2Gi` of memory, that Pod will not fit on that node.

> [!NOTE]
> In Production, always specify resource request and limit. It allows an efficient resource utilization and ensures that containers have the necessary resources to run effectively and scale as needed within the cluster.

## Configure Quality of Service for Pods

What happened if a container exceeds its memory request and the node that it runs on becomes short of memory overall? it is likely that the Pod the container belongs to will be **evicted**.

When you create a Pod, Kubernetes assigns a **Quality of Service (QoS) class** to each Pod as a consequence of the resource constraints that you specify for the containers in that Pod.
QoS classes are used by Kubernetes to decide which Pods to evict from a Node experiencing [Node Pressure](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/). 

1. [Guaranteed](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/#guaranteed) - The Pod specifies resources request and limit. CPU and Memory requests and limit are equal.
2. [Burstable](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/#burstable) - The Pod specifies resources request and limit. But CPU and Memory requests and limit are not equal.
3. [BestEffort](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/#besteffort) - The Pod does not specify resources request or limit for some of its containers.


When a Node runs out of resources, Kubernetes will first evict `BestEffort` Pods running on that Node, followed by `Burstable` and finally `Guaranteed` Pods.
When this eviction is due to resource pressure, **only Pods exceeding resource requests are candidates** for eviction.

Try to simulate Node pressure by overloading the node from a `Guaranteed` Pod, `Burstable` Pod and `BestEffort` Pod, and see the behaviour. 

## Container probes - Liveness and Readiness probes

A **probe** is a diagnostic performed periodically by the **kubelet** on a container, usually by an HTTP request, to check the container status. 

- **Liveness probes** are used to determine the health of a container (a.k.a. health check), by a periodic checks. The container is restarting if the probe fails.
- **Readiness probes** are used to determine whether a Pod is ready to receive traffic (when it is prepared to accept requests, preventing traffic from being sent to Pods that are not fully operational, mostly because the pod is still initializing, or is about to terminate as part of a rolling update).

> [!NOTE]
> In Production, always specify Liveness and Readiness probes, they are essential to ensure the availability and proper functioning of applications within Pods.

### Define a Liveness probe

The `frontend` service exposes an endpoint called `/_healthz`. Upon an HTTP request, if the endpoint returns a `200` status code, it means "the server is alive".

In the below example, the `livenessProbe` entry defines a liveness probe for our Pod. 
Note the `command` and `args` entries, which override the default container command to intentionally terminate the server process after 60 seconds.
This behaviour simulates the case where the server is becoming unhealthy, and we expect the liveness probe to fail.  

```yaml
# k8s/liveness-demo.yaml

apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:
    app: frontend-demo
spec:
  containers:
  - name: server
    image: gcr.io/google-samples/microservices-demo/frontend:v0.8.0
    command: ["sh", "-c"]
    args: ["timeout 60s /src/server; sleep 6000"]
    ports:
      - containerPort: 8080
    env:
      - name: PORT
        value: "8080"
      - name: PRODUCT_CATALOG_SERVICE_ADDR
        value: "productcatalogservice:3550"
      - name: CURRENCY_SERVICE_ADDR
        value: "currencyservice:7000"
      - name: CART_SERVICE_ADDR
        value: "cartservice:7070"
      - name: RECOMMENDATION_SERVICE_ADDR
        value: "recommendationservice:8080"
      - name: SHIPPING_SERVICE_ADDR
        value: "shippingservice:50051"
      - name: CHECKOUT_SERVICE_ADDR
        value: "checkoutservice:5050"
      - name: AD_SERVICE_ADDR
        value: "adservice:9555"
      - name: ENABLE_PROFILER
        value: "0"
    resources:
      requests:
        cpu: 50m
        memory: 5Mi
      limits:
        cpu: 100m
        memory: 128Mi
    livenessProbe:
      initialDelaySeconds: 10
      httpGet:
        path: "/_healthz"
        port: 8080
```

After 60 seconds, the prob HTTP requests to `_healthz` will fail, thus, the **kubelet** will restart the container.

This restarting mechanism helps the application to be more available.
For example, when the server is entering a deadlock (the application is running, but unable to make progress),
restarting the container in such a state can help to make the application more available despite bugs.

By default, the HTTP probe is done every 10 seconds, and should be low-cost, it shouldn't affect the application performance. 

### Define a Readiness probe

Sometimes, applications are temporarily unable to serve traffic.
For example, an application might need to load large data or configuration files during startup, or depend on external services after startup. 
In such cases, you don't want to kill the application, but you don't want to send it requests either.

Kubernetes provides readiness probes to detect and mitigate these situations. 
A pod with containers reporting that they are not ready does not receive traffic through Kubernetes Services.

Readiness probes are configured similarly to liveness probes:

```yaml
# k8s/readiness-demo.yaml

apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:
    app: frontend-demo
spec:
  containers:
  - name: server
    image: gcr.io/google-samples/microservices-demo/frontend:v0.8.0
#    command: ["sh", "-c"]
#    args: ["timeout 60s /src/server; sleep 6000"]
    ports:
      - containerPort: 8080
    env:
      - name: PORT
        value: "8080"
      - name: PRODUCT_CATALOG_SERVICE_ADDR
        value: "productcatalogservice:3550"
      - name: CURRENCY_SERVICE_ADDR
        value: "currencyservice:7000"
      - name: CART_SERVICE_ADDR
        value: "cartservice:7070"
      - name: RECOMMENDATION_SERVICE_ADDR
        value: "recommendationservice:8080"
      - name: SHIPPING_SERVICE_ADDR
        value: "shippingservice:50051"
      - name: CHECKOUT_SERVICE_ADDR
        value: "checkoutservice:5050"
      - name: AD_SERVICE_ADDR
        value: "adservice:9555"
      - name: ENABLE_PROFILER
        value: "0"
    resources:
      requests:
        cpu: 50m
        memory: 5Mi
      limits:
        cpu: 100m
        memory: 128Mi
    livenessProbe:
      initialDelaySeconds: 10
      httpGet:
        path: "/_healthz"
        port: 8080
    readinessProbe:
      initialDelaySeconds: 10
      httpGet:
        path: "/"
        port: 8080
```

Usually a server has a dedicated endpoint for readiness probes. 
But in the above example, we use the default endpoint (`/`) to perform the readiness probe HTTP request.

Since the content served by the `/` endpoint depends on the currency service (`currencyservice`), if you scale it down to **0**, you'll notice how the readiness probe is failing:

```bash
kubectl scale deployment currencyservice --replicas=0
```

#### Readiness probes are critical to perform a successful rolling update 

In the context of a rolling update, readiness probes play a crucial role in ensuring a seamless transition.
When old replicaset Pods receive the SIGTERM signal from the kubelet, the Pod should start a graceful termination process. First of all it should stop receiving new requests.
By failing the readiness probes, the Pod indicates that it's not ready to receive new traffic. 

## Assign Pods to Nodes

There are some circumstances where you may want to control which node the Pod is deployed on.
For example, to ensure that a Pod ends up on a node with an SSD attached to it, or to co-locate Pods from two different services that communicate a lot into the same availability zone.

You can constrain a Pod so that it is restricted to run on particular node(s), or to prefer to run on particular nodes. 

First, let's add another node to your Minikube cluster:

```bash 
minikube node add
```

> [!WARNING]
> As multinode clusters in Minikube is currently experimental, there might be networking and DNS issues.
> Make sure to delete the node when you're done. 

Like many other Kubernetes objects, nodes have **labels**. We will use  [label selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) to assign Pods to Nodes. 
Kubernetes populates a standard set of labels on all nodes in a cluster:

```console
$ kubectl get nodes --show-labels
minikube       Ready    control-plane   22d   v1.27.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=minikube,kubernetes.io/os=linux,minikube.k8s.io/commit=fd7ecd9c4599bef9f04c0986c4a0187f98a4396e,minikube.k8s.io/name=minikube,minikube.k8s.io/primary=true,minikube.k8s.io/updated_at=2023_10_11T07_10_41_0700,minikube.k8s.io/version=v1.31.2,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
minikube-m02   Ready    <none>          69s   v1.27.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=minikube-m02,kubernetes.io/os=linux
```

Let's manually attach the label `gpu=true` to one of your nodes:

```console
$ kubectl label nodes minikube-m02 gpu=true
node/minikube-m02 labeled
```

### Node selectors

There are [several ways](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) to facilitate the Node selection, [nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) is the simplest recommended form. 
You can add the `nodeSelector` field to your Pod specification and specify the node labels you want the target node to have.
Kubernetes only schedules the Pod onto nodes that have each of the labels you specify.

```yaml
# k8s/node-selector-demo.yaml

apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:
    app: frontend-demo
spec:
  nodeSelector:
    gpu: 'true'
  containers:
  - name: server
    image: gcr.io/google-samples/microservices-demo/frontend:v0.8.0
    ports:
      - containerPort: 8080
    env:
      - name: PORT
        value: "8080"
      - name: PRODUCT_CATALOG_SERVICE_ADDR
        value: "productcatalogservice:3550"
      - name: CURRENCY_SERVICE_ADDR
        value: "currencyservice:7000"
      - name: CART_SERVICE_ADDR
        value: "cartservice:7070"
      - name: RECOMMENDATION_SERVICE_ADDR
        value: "recommendationservice:8080"
      - name: SHIPPING_SERVICE_ADDR
        value: "shippingservice:50051"
      - name: CHECKOUT_SERVICE_ADDR
        value: "checkoutservice:5050"
      - name: AD_SERVICE_ADDR
        value: "adservice:9555"
      - name: ENABLE_PROFILER
        value: "0"
    resources:
      requests:
        cpu: 50m
        memory: 5Mi
      limits:
        cpu: 100m
        memory: 128Mi
```

Make sure the Pod has been scheduled on the labelled Node. 

Try to delete the Node and re-apply the Pod. What happened? 

### Taints and Tolerations

We've seen that Node Selector is a simple mechanism to attract a Pod on specific Node (or group of nodes) by a label.
This technique is useful, for example, to schedule Pods on Nodes with specific hardware specification.

But what if we want to ensure that **only** those specific Pods use the dedicated Node?
For example, Nodes with an expensive GPU, it is desirable to keep pods that don't need the GPU off of those nodes, thus leaving room for later-arriving pods that do need the GPU. 

**Taints and Tolerations** can help us.

[Taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) allow a node to **repel** a set of pods.

Let's add a **taint** to a node using `kubectl taint`:

```bash 
kubectl taint nodes minikube-m02 gpu=true:NoSchedule
```

The taint has key `gpu`, value `true`, and taint effect `NoSchedule`.
This means that **no pod will be able to schedule** onto `minikube-m02` unless it has a matching **toleration**.

A **Tolerations** allow the scheduler to schedule pods with matching taints.
To tolerate the above taint, you can specify toleration in the pod specification.

```yaml
# k8s/taint-toleration-demo.yaml

apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:
    app: frontend-demo
spec:
  nodeSelector:
    gpu: true
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: server
    image: gcr.io/google-samples/microservices-demo/frontend:v0.8.0
    ports:
      - containerPort: 8080
    env:
      - name: PORT
        value: "8080"
      - name: PRODUCT_CATALOG_SERVICE_ADDR
        value: "productcatalogservice:3550"
      - name: CURRENCY_SERVICE_ADDR
        value: "currencyservice:7000"
      - name: CART_SERVICE_ADDR
        value: "cartservice:7070"
      - name: RECOMMENDATION_SERVICE_ADDR
        value: "recommendationservice:8080"
      - name: SHIPPING_SERVICE_ADDR
        value: "shippingservice:50051"
      - name: CHECKOUT_SERVICE_ADDR
        value: "checkoutservice:5050"
      - name: AD_SERVICE_ADDR
        value: "adservice:9555"
      - name: ENABLE_PROFILER
        value: "0"
    resources:
      requests:
        cpu: 50m
        memory: 5Mi
      limits:
        cpu: 100m
        memory: 128Mi
```

The above defined toleration would allow to schedule the Pod onto `minikube-m02`.

**Note**: Taints and Tolerations **do not** guarantee that the pods only prefer these nodes. 
So the pods may be scheduled on the other untainted nodes. 
To guarantee that group of pods would be scheduled on a node, and **only** them, both **node selector** and **taint and tolerations** should be used.

To remove the taint: 

```bash 
kubectl taint nodes minikube-m02 gpu=true:NoSchedule-
```

## Horizontal Pod autoscaling

> [!NOTE]
> Before starting make sure the [Metrics Server](https://github.com/kubernetes-sigs/metrics-server#readme) is installed on your cluster.

A **HorizontalPodAutoscaler** (HPA for short) automatically updates the number of Pods to match demand, usually in a Deployment or StatefulSet.

The HorizontalPodAutoscaler controller periodically (by default every 15 seconds) adjusts the desired scale of its target (for example, a Deployment) to match observed metrics such as average CPU utilization, average memory utilization, or any other custom metric you specify.
The common use for HPA is to configure it to fetch metrics from a [Metrics Server](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/#metrics-server) API (should be installed as an add-on). 
Every 15 seconds, the HPA controller queries the Metric Server and fetches resource utilization metrics (e.g. CPU, Memory) for each Pod that are part of the HPA. 
The controller then calculates the desired replicas, and scale in/out based on the current replicas.

To demonstrate a HorizontalPodAutoscaler, you will first create a Deployment and a Service:

```yaml
# k8s/hpa-demo.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-hpa-demo
spec:
  selector:
    matchLabels:
      app: frontend-hpa-demo
  template:
    metadata:
      labels:
        app: frontend-hpa-demo
    spec:
      containers:
        - name: server
          image: gcr.io/google-samples/microservices-demo/frontend:v0.8.0
          ports:
          - containerPort: 8080
          env:
          - name: PORT
            value: "8080"
          - name: PRODUCT_CATALOG_SERVICE_ADDR
            value: "productcatalogservice:3550"
          - name: CURRENCY_SERVICE_ADDR
            value: "currencyservice:7000"
          - name: CART_SERVICE_ADDR
            value: "cartservice:7070"
          - name: RECOMMENDATION_SERVICE_ADDR
            value: "recommendationservice:8080"
          - name: SHIPPING_SERVICE_ADDR
            value: "shippingservice:50051"
          - name: CHECKOUT_SERVICE_ADDR
            value: "checkoutservice:5050"
          - name: AD_SERVICE_ADDR
            value: "adservice:9555"
          - name: ENABLE_PROFILER
            value: "0"
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 80m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-hpa-demo-service
spec:
  selector:
    app: frontend-hpa-demo
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

Now that the `frontend-hpa-demo` server is running, create the autoscaler:

```yaml
# k8s/hpa-autoscaler-demo.yaml

apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend-hpa-demo
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 60
```

When defining the pod specification, the resource **Requests**, like CPU and memory must be specified.
This is used to determine the resource utilization and used by the HPA controller to scale the target up or down.
In the above example, the HPA will scale up once the Pod is reaching 60% of the `.resources.requests.cpu`. 


Next, let's see how the autoscaler reacts to increased load. 
To do this, you'll start a different Pod to act as a client. 
The container within the client Pod runs in an infinite loop, sending queries to the `frontend-hpa-demo-service` service.

```bash 
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.05; do (wget -q -O- http://frontend-hpa-demo-service &); done"
```


## Multiple containers pod

As we said, a Pod is a group of one or more containers.
The containers in a Pod are automatically co-located and co-scheduled on the same physical or virtual machine in the cluster.

Grouping containers in a single Pod is a relatively advanced and **uncommon** use case. 
You should use this pattern only in specific cases in which your containers are tightly coupled.

For example, you might have an `nginx` container that acts as a web server for files in a shared volume, and a separate "helper" container that updates those files from a remote source:

<img src="../.img/k8s_multi-container-pod.png" width="35%" />

This pattern is called a **Sidecar**:

```yaml
# k8s/sidecar-demo.yaml

apiVersion: v1
kind: Pod
metadata:
  name: mc1
spec:
  containers:
    - name: webserver
      image: nginx
      volumeMounts:
      - name: html
        mountPath: /usr/share/nginx/html
    - name: helper
      image: debian
      volumeMounts:
      - name: html
        mountPath: /html
      command: ["/bin/sh", "-c"]
      args:
        - while true; do
            date >> /html/index.html;
            sleep 1;
          done
  volumes:
    - name: html
      emptyDir: {}
```

Apply the above manifest and observe the server behaviour. 

Other multi-container Pod patterns are **Proxies** and **Adapters**:

- **Proxies:** Acts as a proxy to the mian container. For example, an Nginx container that pass the traffic to a Flask backend server.
- **Adapter:** Used to standardize and normalize application input and output. For example, integrating a legacy system with a new framework by introducing an adapter container that translates calls between the two interfaces.

# Self-check questions

[Enter the interactive self-check page](https://alonitac.github.io/__REPO_NAME__/multichoice-questions/k8s_pod_deep_dive.html)


# Exercises 

### :pencil2: Zero downtime during scale

Your goal in this exercise is to achieve zero downtime during scale up/down events of a simple webserver.

The code can be found in `k8s/zero_downtime_node` (a simple Nodejs webserver).

1. Build the image and push it to a dedicated ECR repo that you'll create.
2. Deploy the app as a simple `Deployment` (with the corresponding `Service`). 
3. Generate some incoming traffic (20 requests per second) from a dedicated pod: 
   ```bash
   kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.05; do (wget -q -O- http://SERVICE_URL &); done"
   ```
   Chane `SERVICE_URL` accordingly.    

4. Increase the webserver's Deployment replicas while watching the `load-generator` pod logs. Did you lose requests for a short moment? Why?
5. Decrease the number of replicas, lost requests again? Why? 
6. Define a `liveness` and `readiness` probes in your Deployment. Change the server code in the `/ready` endpoint, and the `SIGTERM` handler to support zero down-time during scale up/down. You may also want to specify the `terminationGracePeriodSeconds` entry for the container.
8. Create a horizontal pod autoscaler (HPA) for the deployment. Base your HPA on CPU utilization, set the `resources.requests.cpu` and `resources.limits.cpu` entries `300m`. Maximum number of pods to 10.
7. Generate some load as described in step 3. Let the HPA to increase and decrease the number of pods. Make sure you don't lose even a single requests during **scale up and scale down** events.

**Bonus**: Make sure you are able to perform a rolling update with zero downtime, during traffic load. 

### :pencil2: Pod troubleshoot

In the file `k8s/pod-troubleshoot.yaml`, a PostgreSQL database Deployment is provided with intentionally injected issues for troubleshooting purposes.

Fix the issues while complying with the below requirements: 

- You should be able to communicate with the DB by (the command creates a temporary Pod that lists the existed databases in Postgres):

  ```bash
  kubectl run -i --tty --rm postgres-client --image=postgres:11.22-bullseye --restart=Never -- /bin/sh -c "PGPASSWORD=<your-pg-pass> psql -h postgres-service -U postgres -c '\list'"
  ```
  
  While `<your-pg-pass>` is your postgres DB password.

- The DB must be running on a node labeled `disktype=ssd`.
- The `postgres:11.22-bullseye` container must be scheduled on a node with at least 50 milli-cpu.
- The `postgres:11.22-bullseye` container must have liveness and readiness probed configured and working properly. 

### :pencil2: Init containers

A Pod can have multiple containers running apps within it, but it can also have one or more [init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/), which are run before the app containers are started.

Create a Pod with an init container that clones the [2048 game](https://github.com/gabrielecirulli/2048) source code repo game into a **shared volume** with the Pod's container - an [nginx](https://hub.docker.com/_/nginx) image - that will serve the game.   





