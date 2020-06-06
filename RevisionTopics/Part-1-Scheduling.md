

# CKA Curriculum Part 1 - Scheduling

## Use labelselectors to schedule Pods

Labels are key/value pairs that are applied to objects in Kubernetes, such as pods, nodes, services and much more. After we’ve defined and applied labels, we can leverage labels to help us identify and distinguish components, including (but not limited to):

*   Application Environment
    *   Environment=Prod, Environment=Test, Environment=Dev
*   Tier
    *   tier=web, tier=app, tier=db
*   Distinguish nodes, or a collection of nodes based on location
    *   location=GB, location=GER
*   Distinguish nodes based on features
    *   CPU=AMD, CPU=Intel

The example below labels a pod as belonging in the Dev environment, App Tier

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
  labels: 
    environment: dev
    tier: web
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000000"
```

Validate with `kubectl get pods --show-labels`

```
NAME            READY   STATUS    RESTARTS   AGE   LABELS
busybox-sleep   1/1     Running   0          25s   environment=dev,tier=web
```

Labels can also be used to schedule specific pods on specific nodes based on labels.

As a example, a node can be labelled with having a AMD cpu with the following:

```
kubectl label node k8s-worker-03 CPU=AMD
```

Validate with `kubectl get nodes --show-labels`

```
k8s-worker-03   Ready    <none>   11d   v1.14.1   CPU=AMD
```

Specify in a pod manifest to run on a node with an appropriate label

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
  labels:
    Environment : Dev
    Tier : Web
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000000"
  nodeSelector:
    CPU: AMD
```
For the pod to be eligible to run on a node, the node must have each of the indicated key-value pairs as labels (it can have additional labels as well). The most common usage is one key-value pair.

To validate, we can check kubectl get pods -o wide. Note how all three pods are running on the same node, which is what we’ve labelled with CPU=AMD.

```
busybox-amd     1/1     Running   0          8s      10.244.1.56   k8s-worker-03 
```

Further validation can be obtained by executing a describe pod [podname] and identifying the nodeselector value:

```
Node-Selectors:  CPU=AMD
```

### No Scheduler

`nodeName` is the simplest form of node selection constraint, but due to its limitations it is typically not used. `nodeName` is a field of PodSpec. If it is non-empty, the scheduler ignores the pod and the kubelet running on the named node tries to run the pod. Thus, if `nodeName` is provided in the PodSpec, it takes precedence over the above methods for node selection.

Some of the limitations of using `nodeName` to select nodes are:

If the named node does not exist, the pod will not be run, and in some cases may be automatically deleted.
If the named node does not have the resources to accommodate the pod, the pod will fail and its reason will indicate why, for example OutOfmemory or OutOfcpu.
Node names in cloud environments are not always predictable or stable.

Here is an example of a pod config file using the `nodeName` field:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: kube-01
```

## Taints and Tolerations

### Taints

```
# Places a taint on node node1. The taint has key key, value value, and taint effect NoSchedule
kubectl taint nodes node1 key=value:NoSchedule
---
# Remove a taint
kubectl taint nodes node1 key:NoSchedule-
```

**NoSchedule**: No Pod will be able to schedule onto node1 unless it has a matching toleration

### Tolerations

You specify a toleration for a pod in the PodSpec. Both of the following tolerations “match” the taint created by the kubectl taint line above, and thus a pod with either toleration would be able to schedule onto node1

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
-----
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```

Example for a Pod that uses tolerations

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
```

**PreferNoSchedule**: the system will try to avoid placing a pod that does not tolerate the taint on the node, but it is not required

You can put multiple taints on the same node and multiple tolerations on the same pod. The way Kubernetes processes multiple taints and tolerations is like a filter: start with all of a node’s taints, then ignore the ones for which the pod has a matching toleration; the remaining un-ignored taints have the indicated effects on the pod. In particular,

* If there is at least one un-ignored taint with effect NoSchedule then Kubernetes will not schedule the pod onto that node
* If there is no un-ignored taint with effect NoSchedule but there is at least one un-ignored taint with effect PreferNoSchedule then Kubernetes will try to not schedule the pod onto the node
* If there is at least one un-ignored taint with effect NoExecute then the pod will be evicted from the node (if it is already running on the node), and will not be scheduled onto the node (if it is not yet running on the node).

For example, imagine you taint a node like this
```
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```

And a pod has two tolerations:

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

In this case, the pod will not be able to schedule onto the node, because there is no toleration matching the third taint. But it will be able to continue running if it is already running on the node when the taint is added, because the third taint is the only one of the three that is not tolerated by the pod.

Normally, if a taint with effect NoExecute is added to a node, then any pods that do not tolerate the taint will be evicted immediately, and pods that do tolerate the taint will never be evicted. However, a toleration with NoExecute effect can specify an optional tolerationSeconds field that dictates how long the pod will stay bound to the node after the taint is added. For example,

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

means that if this pod is running and a matching taint is added to the node, then the pod will stay bound to the node for 3600 seconds, and then be evicted. If the taint is removed before that time, the pod will not be evicted

## Node Affinity

Provides advanced capabilities to limit pod placement on specific nodes.

Node affinity is conceptually similar to `nodeSelector` – it allows you to constrain which nodes your pod is eligible to be scheduled on, based on labels on the node.

There are currently two types of node affinity, called `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution`. You can think of them as “hard” and “soft” respectively, in the sense that the former specifies rules that must be met for a pod to be scheduled onto a node (just like `nodeSelector` but using a more expressive syntax), while the latter specifies preferences that the scheduler will try to enforce but will not guarantee. The “IgnoredDuringExecution” part of the names means that, similar to how `nodeSelector` works, if labels on a node change at runtime such that the affinity rules on a pod are no longer met, the pod will still continue to run on the node. In the future we plan to offer `requiredDuringSchedulingRequiredDuringExecution` which will be just like `requiredDuringSchedulingIgnoredDuringExecution` except that it will evict pods from nodes that cease to satisfy the pods’ node affinity requirements.

Thus an example of `requiredDuringSchedulingIgnoredDuringExecution` would be “only run the pod on nodes with Intel CPUs” and an example `preferredDuringSchedulingIgnoredDuringExecution` would be “try to run this set of pods in failure zone XYZ, but if it’s not possible, then allow some to run elsewhere”.

Node affinity is specified as field `nodeAffinity` of field `affinity` in the PodSpec.

Here’s an example of a pod that uses node affinity:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

This node affinity rule says the pod can only be placed on a node with a label whose key is `kubernetes.io/e2e-az-name` and whose value is either `e2e-az1` or `e2e-az2`. In addition, among nodes that meet that criteria, nodes with a label whose key is `another-node-label-key` and whose value is `another-node-label-value` should be preferred.

You can see the operator `In` being used in the example. The new node affinity syntax supports the following operators: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`. You can use `NotIn` and `DoesNotExist` to achieve node anti-affinity behavior, or use node taints to repel pods from specific nodes.

If you specify both `nodeSelector` and `nodeAffinity`, both must be satisfied for the pod to be scheduled onto a candidate node.

If you specify multiple `nodeSelectorTerms` associated with `nodeAffinity` types, then the pod can be scheduled onto a node **if one** of the `nodeSelectorTerms` can be satisfied.

If you specify multiple `matchExpressions` associated with `nodeSelectorTerms`, then the pod can be scheduled onto a node only **if all** `matchExpressions` is satisfied.

If you remove or change the label of the node where the pod is scheduled, the pod won’t be removed. In other words, the affinity selection works only at the time of scheduling the pod.

The `weight` field in `preferredDuringSchedulingIgnoredDuringExecution` is in the range 1-100. For each node that meets all of the scheduling requirements (resource request, RequiredDuringScheduling affinity expressions, etc.), the scheduler will compute a sum by iterating through the elements of this field and adding “weight” to the sum if the node matches the corresponding MatchExpressions. This score is then combined with the scores of other priority functions for the node. The node(s) with the highest total score are the most preferred.

|                                                 | DuringScheduling | DuringExecution |
| ----------------------------------------------- | ---------------- | --------------- |
| requiredDuringSchedulingIgnoredDuringExecution  | Required         | Ignored         |
| preferredDuringSchedulingIgnoredDuringExecution | Preferred        | Ignored         |
| requiredDuringSchedulingRequiredDuringExecution | Required         | Required        |


## Resource requirements and limits

### Default request and limits

If a container is created in a namespace with a default request/limit value and doesn’t explicitly define these in the manifest, it inherits these values from the namespace.

 For the POD to pick up those defaults you must have first set those as default values for request and limit by creating a LimitRange in that namespace.

 ```yaml
 apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
--- 
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container
```

### Requests 
Kubernetes default requests:

| Resource | Default requests |
| -------- | :--------------: |
| CPU      |       0.5        |
| MEM      |      256Mi       |
| DISK     |                  |

Kubernetes search for nodes that has sufficient amount of resources to hold that container.

When you specify the resource request for Containers in a Pod, the scheduler uses this information to decide which node to place the Pod on.

### Limits

Kubernetes default limits:

| Resource | Default limits |
| -------- | :------------: |
| CPU      |      0.5       |
| MEM      |     256Mi      |
| DISK     |                |

What happens if a limit is exceeded ?

**Memory:** If you set a memory limit of 4GiB for that Container, the kubelet (and container runtime) enforce the limit. The runtime prevents the container from using more than the configured resource limit. For example: when a process in the container tries to consume more than the allowed amount of memory, the system kernel terminates the process that attempted the allocation, with an out of memory (OOM) error.

**CPU** Kubernetes throttles so it cannont go beyond its limit

### Example

Here’s an example. The following Pod has two Containers. Each Container has a request of 0.25 cpu and 64MiB (226 bytes) of memory. Each Container has a limit of 0.5 cpu and 128MiB of memory. You can say the Pod has a request of 0.5 cpu and 128 MiB of memory, and a limit of 1 cpu and 256MiB of memory.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
``` 

Note, if you define a container with a memory/CPU limit, but not a request, Kubernetes will define the limit the same as the request.

## Understand the role of DaemonSets 

A DaemonSet is a way of ensuring that a particular pod runs on all (or a subset) of nodes. A very common use case is a CNI plugin. This should not be confused with a Deployment, or (in my opinion) be considered a mechanism to scale an application. We define a Daemonset like so:


```
apiVersion: apps/v1
kind: DaemonSet
metadata:
 name: nginx
spec:
 selector:
   matchLabels:
     name: nginx-ds
 template:
   metadata:
     labels:
       name: nginx-ds
   spec:
     containers:
     - name: nginx-ds
       image: nginx
```


This basic example will create a Daemonset based on the nginx image. Once executed we can check by executing the following:


```
kubectl get ds

NAME    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
nginx   2         2         2       2            2           <none>          33s
```


These are also listed as individual pods:


```
kubectl get pods -o wide | grep nginx

nginx-97hp4     1/1     Running   0          97s   10.244.1.57   k8s-worker-03  
nginx-hmtzp     1/1     Running   0          97s   10.244.2.43   k8s-worker-04   
```


The Daemonset has created two pods based on the manifest - this is because, in this environment, we have three nodes that facilitate workloads.

## Static Pods

Static Pods are managed directly by the kubelet daemon on a specific node, without the API server observing them. Unlike Pods that are managed by the control plane (for example, a Deployment); instead, the kubelet watches each static Pod (and restarts it if it crashes).

Static Pods are always bound to one Kubelet on a specific node.

The kubelet automatically tries to create a mirror Pod on the Kubernetes API server for each static Pod. This means that the Pods running on a node are visible on the API server, but cannot be controlled from there.

1. Choose a directory, say `/etc/kubelet.d` and place a web server Pod definition there, for example `/etc/kubelet.d/static-web.yaml`

2. Configure your kubelet on the node to use this directory by running it with `--pod-manifest-path=/etc/kubelet.d/` argument

3. Restart the kubelet 
```
# Run this command on the node where the kubelet is running
systemctl restart kubelet
```

Note, to identify the file that needs modifying:

```
sudo systemctl status kubelet                  
● kubelet.service - kubelet: The Kubernetes Node Agent
  Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)

[Service]
ExecStart=/usr/bin/kubelet \
 --pod-manifest-path=/etc/staticpods
```

Note you can also specify a url with the format `--manifest-url=[https://virtualthoughts.co.uk/somepod.yaml](https://virtualthoughts.co.uk/somepod.yaml)` should you wish to load a static pod from a URL.

## Understand how to run multiple schedulers and how to configure Pods to use them

Kubernetes comes with its own scheduler which acts as the default for all related actions. You can, however, define and implement your own to exist in parallel to the default scheduler, should you wish.

The process is defined here [https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/](https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/)

### Deploy and additional scheduler

#### Scheduler running as a service

```
# wget https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-scheduler

# kube-scheduler.service
ExecStart=/usr/local/bin/kube-scheduler \
    --config=/etc/kubernetes/config/kube-scheduler.yaml \
    --scheduler-name=default-scheduler

# my-custom-scheduler.service
ExecStart=/usr/local/bin/kube-scheduler \
    --config=/etc/kubernetes/config/kube-scheduler.yaml \
    --scheduler-name= my-custom-scheduler
```

#### Scheduler running as a POD

```yaml
# Default Kube Scheduler
apiVersion: v1 kind: Pod metadata:
name: kube-scheduler
namespace: kube-system 
spec:
  containers: 
  - command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf 
    - --leader-elect=true
    image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3 
    name: kube-scheduler
---
# Custom Kube Scheduler
apiVersion: v1 kind: Pod metadata:
name: kube-scheduler
namespace: kube-system 
spec:
  containers: 
  - command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf 
    - --leader-elect=true
    - --scheduler-name=my-custom-scheduler
    # In case you have multiple master, add the option below to differenciate the custom scheduler for the default during the leader election process
    - --lock-object-name=my-custom-scheduler
    image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3 
    name: kube-scheduler
```

### View schedulers

```
kubectl get pods -n kube-system
```

### Define which scheduler to use


As an example, a second custom scheduler has been implemented in a kubernetes cluster called “custom-scheduler”. In a pod manifest we can specify a scheduler to use, if different from the default under “spec”.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
spec:
  # Add field scheduler name
  schedulerName: custom-scheduler
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000000"
```

Execute a kubectl describe pod busybox-sleep to validate which scheduler is being used:

![alt_text](https://i.imgur.com/ZqtQUz9.png "image_tooltip")

**Note : to get familiar with the syntax, use “default-scheduler” in the manifest. This is the name for the scheduler that comes with Kubernetes**

### Display scheduler events

Viewing and assessing logs generated by schedulers provide a myriad of functions, including but not limited to - troubleshooting, performance monitoring, and various others. There are several ways to get scheduler events:

```
# List all events in the current namespace
kubectl get events | grep scheduler
```

Note this will only work if you’re using the default scheduler (default-scheduler) or a customer scheduler with “scheduler” in the name. Replace accordingly or don’t pipe anything to grep to get a system wide set of events

Shortly after performing an action, such as deploying a pod, you should see something in the logs resembling the following:

Successfully assigned default/busybox-sleep to 632efcbf-8027-4711-b6f4-e0aea7db9ed3

To acquire logs from the scheduler itself, one of two methods must be used, depending on how the scheduler is deployed.

#### For schedulers running as a pod

Locate the pod facilitating the scheduler:

```
kubectl get pods - kube-system
```

Inspect the logs accordingly:

```
kubectl logs [Name of pod] - kube-system
```

#### For schedulers running locally on a node

Log onto the master node and cat / copy / open the following file:

```
/var/log/kube-scheduler.log
```

## Understand how resource limits can affect Pod scheduling

At a namespace level, we can define resource limits. This enables a restriction in resources, especially helpful in multi-tenancy environments and provides a mechanism to prevent pods from consuming more resources than necessary, which may have a detrimental effect on the environment as a whole.

We can define the following:

Default memory / CPU **requests & limits **for a namespace

Minimum and Maximum memory / CPU **constraints **for a namespace

Memory/CPU **Quotas **for a namespace

### Minimum / Maximum Constraints

If a pod does not meet the range in which the constraints are valued at, it will not be scheduled.

### Quotas

Control the _total_ amount of CPU/memory that can be consumed in the _namespace_ as a whole.

Example: Attempt to schedule a pod that request more memory than defined in the namespace

Create a namespace: kubectl create namespace tenant-mem-limited \

Create a YAML manifest to limit resources:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-max-mem
spec:
  limits:
  - max:
      memory: 250Mi
    type: Container
```

Apply this to the aforementioned namespace: `kubectl apply -f maxmem.yaml --namespace tenant-mem-limited`

To create a pod with a memory request that exceeds the limit:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: too-much-memory
  namespace: tenant-mem-limited 
spec:
  containers:
  - name: too-much-mem
    image: nginx
    resources:
      requests:
        memory: "300Mi"
```

Executing the above will yield the following result:

```
The Pod "too-much-memory" is invalid: spec.containers[0].resources.requests: Invalid value: "300Mi": must be less than or equal to memory limit
```

As we have defined the pod limit of the namespace to 250MiB, a request for 300MiB will fail.
