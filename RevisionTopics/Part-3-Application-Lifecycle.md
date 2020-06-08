
# CKA Curriculum Part 3 - Application Lifecycle Management

## Understand deployments and how to perform rolling updates and rollbacks

Deployments are intended to replace Replication Controllers.  They provide the same replication functions (through Replica Sets) and also the ability to rollout changes and roll them back if necessary. An example configuration is shown below:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: nginx-frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.14
        ports:
        - containerPort: 80
```

We can then describe it with` kubectl describe deployment nginx-deployment`

### Upgrade

To update an existing deployment, we have two main options:

*   Rolling Update
*   Recreate

A rolling update, as the name implies, will swap out containers in a deployment with one created by a new image.

Use a rolling update when the application supports having a mix of different pods (aka application versions). This method will also involve no downtime of the service, but will take longer to bring up the deployment to the requested version

A recreate will delete all the existing pods and then spin up new ones. This method will involve downtime. Consider this a “bing bang” approach

Examples listed in the Kubernetes documentation are largely imperative, but I prefer to be declarative. As an example, create a new yaml file and make the required changes, in this example, the version of the nginx container is incremented.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: nginx-frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.15
        ports:
        - containerPort: 80
```

We can then apply this file` kubectl apply -f updateddeployment.yaml --record=true`

Followed by the following:

```
kubectl rollout status deployment/nginx-deployment

Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 4 of 5 updated replicas are available...
deployment "nginx-deployment" successfully rolled out
```

We can also use the kubectl rollout history to look at the revision history of a deployment

```
kubectl rollout history deployment/nginx-deployment

deployment.extensions/nginx-deployment
REVISION  CHANGE-CAUSE
1     	<none>
2     	<none>
4     	<none>
5     	kubectl apply --filename=updateddeployment.yaml --record=true
```

Alternatively, we can also do this imperatively:

```
kubectl --record deployments/nginx-deployment set image deployments/nginx-deployment nginx=nginx:1.9.1

deployment.extensions/nginx-deployment image updated
deployment.extensions/nginx-deployment image updated
```

### Rollback

To rollback to the previous version:

```
kubectl rollout undo deployment/nginx-deployment 
```

To rollback to a specific version:

```
kubectl rollout undo deployment/nginx-deployment --to-revision 5
```

Where revision number comes from `kubectl rollout history deployment/nginx-deployment`

## Know various ways to configure applications

I personally find this subject a bit ambiguous, but we can attempt to break this down into the following objet types:
* Commands
* Jobs
* Configmaps
* Secrets
* Environments variables

### Commands

#### Difference between CMD and ENTRYPOINT in Docker

By default the ubuntu image launches bash as command. Since it cannot find a terminal. It exits. But it is possible to specify a different command.

```
docker run ubuntu [COMMAND]

docker run ubuntu sleep 5
```

If you want the image to always run the sleep command when it starts. You can create your own image from the ubuntu image.

```dockerfile
# ubuntu-sleeper image
FROM Ubuntu

CMD sleep 5
----
CMD command param1
CMD ["command", "param1"]
CMD ["sleep", "5"]
```

But what if you want to change the number of seconds ?

First option:

You specify a command that will override the default `CMD`

```
docker run ubuntu-sleeper sleep 10
```

But it doesn't look good. When if we want 

```
docker run ubuntu-sleeper 10
```

You can use the docker `ENTRYPOINT`

```dockerfile
# ubuntu-sleeper image
FROM Ubuntu

ENTRYPOINT ["sleep"]
```

`ENTRYPOINT` is like CMD but it is not overriden by the docker set in `docker run`. Whatever you put in the `docker run` command it will be appended to the `ENTRYPOINT`.

If you want a default value, use both `ENTRYPOINT` and `CMD`

```dockerfile
# ubuntu-sleeper image
FROM Ubuntu

ENTRYPOINT ["sleep"]

CMD ["5"]
```

It is possible to override the `ENTRYPOINT` by using the `--entrypoint` option

```
docker run --entrypoint sleep2.0 ubuntu-sleeper 10
```

#### Kubernetes use case

##### Override the CMD

Example below is equivalent to `docker run ubuntu-sleeper 10`

```yaml
apiVersion: v1 
kind: Pod 
metadata:
  name: ubuntu-sleeper-pod 
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    args: ["10"]
```

`args` will override the `CMD` in the Dockerfile.

##### Override the ENTRYPOINT

Example below is equivalent to `docker run --entrypoint sleep2.0 ubuntu-sleeper 10`

```yaml
apiVersion: v1 
kind: Pod 
metadata:
  name: ubuntu-sleeper-pod 
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    command: ["sleep2.0"]
    args: ["10"]
```

### Jobs

A job is  simply an application that runs to completion within a pod. Use case for this would be something like batch processing.

The example below executes a perl command inside a container that calculates pi to 2000 places, printing out the result. After it has completed, the pod will terminate.


```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

We can check the status of the job with the following:

```
kubectl get jobs

NAME   COMPLETIONS   DURATION   AGE
pi     1/1           97s        102s
```

And the respective container:

```
kubectl get pods
NAME       READY   STATUS      RESTARTS   AGE
pi-vjk5m   0/1     Completed   0          2m54s
```

To view the output from this pod:

```
kubectl logs pi-vjk5m

3.141592653589793238462643383279502884197169399375105820974944592307816406286208998628034825342117067982148086513282306647093844609550582231725359………..
```

### Config maps

#### Create a configmap

Configmaps are a way to decouple configuration from pod manifest file. Obviously, the first step is to create a config map before we can get pods to use them:

```
kubectl create configmap <map-name> <data-source>
```

“Map-name” is a arbitrary name we give to this particular map, and “data-source” corresponds to a key-value pair that resides in the config map.

```
kubectl create configmap vt-cm --from-literal=blog=virtualthoughts.co.uk
```

At which point we can then describe it:

```
kubectl describe configmap vt-cm
Name:         vt-cm
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
blog:
----
virtualthoughts.co.uk
```

You can also create a configmap from a file

```
kubectl create configmap <config-name> --from-file=<path-to-file>
```

#### Configmap in Pods

#### ENV

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: config-test-pod
spec:
 containers:
 - name: test-container
   image: busybox
   command: [ "/bin/sh", "-c", "env" ]
   envFrom:
      - configMapRef:
          name: vt-cm
 restartPolicy: Never
```

#### SINGLE ENV

To reference this config map in a pod, we declare it in the respective yaml:

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: config-test-pod
spec:
 containers:
 - name: test-container
   image: busybox
   command: [ "/bin/sh", "-c", "env" ]
   env:
     - name: BLOG_NAME
       valueFrom:
         configMapKeyRef:
           name: vt-cm
           key: blog
 restartPolicy: Never
```


The pod above will output the environment variables, so we can validate it’s leveraged the config map by extracting the logs from the pod:

```
kubectl logs config-test-pod
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT=443
HOSTNAME=config-test-pod
SHLVL=1
HOME=/root
BLOG_NAME=virtualthoughts.co.uk
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
```

#### Volume

```yaml
volumes:
- name: app-config-volume
  configmap:
    name: app-config
```

### Secrets

Secrets allow us to store and manage sensitive information pertaining to our applications, which can take the form of a variety of objects such as usernames, passwords, ssh keys and much more. Similarly to configmaps, secrets are designed to decouple this information directly from the pod declaration.

As part of a Kubernetes cluster stand up, secrets are already leveraged, and we can view them by executing the following:

```
kubectl get secrets --all-namespaces
```

#### Creating secrets

##### Imperative way

```
kubectl create secret generic my-secret --from-literal=username=dbu --from-literal=pass=dbp
```

**generic** is important

##### Declarative way

Values must be encoded in base64 when using declarative way

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret 
data:
  DB_Host: bXlzcWw=
  DB_User: cm9vdA==
  DB_Password: cGFzd3Jk
```
 
Encryption / Decryption commands

```
echo –n ‘root’ | base64
echo –n ‘bXlzcWw=’ | base64 --decode
```

#### Using secrets

##### ENV

```
envFrom:
  - secretRef:
      name: app-config
```

##### Single ENV

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: secret-test-pod
spec:
 containers:
 - name: test-container
   image: busybox
   command: [ "/bin/sh", "-c", "env" ]
   env:
     - name: DB_Username
       valueFrom:
         secretKeyRef:
           name: my-secret
           key: username
     - name: DB_Pass
       valueFrom:
         secretKeyRef:
           name: my-secret
           key: pass
 restartPolicy: Never

kubectl logs secret-test-pod
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
HOSTNAME=secret-test-pod
DB_Username=dbu
SHLVL=1
HOME=/root
DB_Pass=dbp
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
```

##### Volumes

```yaml
volumes:
  - name: app-secret-volume
    secret:
      secretName: app-secret
```

When mounting secret as a volume, each attribute in the secret is created as a file with the value of the secret as its content.

```
> ls /opt/app-secret-volumes
DB_Host DB_Password DB_User 
---
> cat /opt/app-secret-volumes/DB_Password
paswrd
```

## Environment Variables

For more simple, direct configuration we can manipulate the environment variables directly within the pod manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: secret-test-pod
spec:
 containers:
 - name: test-container
   image: busybox
   command: [ "/bin/sh", "-c", "env" ]
   env:
     - name: DB_Username
       value:  "some username"
     - name: DB_Pass
       Value: "some password"
 restartPolicy: Never
```


## Know how to Scale Applications

Constantly adding more, individual pods is not a sustainable model for scaling an application. To facilitate applications at scale, we need to leverage higher level constructs such as replicasets or deployments.

As an example, if the following is deployed:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
 name: nginx-deployment
spec:
 replicas: 5
 template:
   metadata:
     labels:
       app: nginx-frontend
   spec:
     containers:
     - name: nginx
       image: nginx:1.14
       ports:
       - containerPort: 80
```

If we wanted to scale this, we can simply modify the yaml file and scale up/down the deployment by modifying the “replicas” field, or modify it in the fly:

```
kubectl scale deployment nginx-deployment --replicas 10
```

## Understand the primitives necessary to create a self-healing application

Kubernetes supports self-healing applications through ReplicaSets and Replication Controllers. The replication controller helps in ensuring that a POD is re-created automatically when the application within the POD crashes. It helps in ensuring enough replicas of the application are running at all times.

Stateful Sets are similar to deployments, for example they manage the deployment and scaling of a series of pods. However, in addition to deployments they also provide guarantees about the ordering and uniqueness of Pods. A StatefulSet maintains a sticky identity for each of their Pods. These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.

StatefulSets are valuable for applications that require one or more of the following.


*   Stable, unique network identifiers.
*   Stable, persistent storage.
*   Ordered, graceful deployment and scaling.
*   Ordered, automated rolling updates.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
 name: nginx-statefulset
spec:
 selector:
   matchLabels:
     app: vt-nginx
 serviceName: "nginx"
 replicas: 2
 template:
   metadata:
     labels:
       app: vt-nginx
   spec:
     containers:
     - name: vt-nginx
       image: nginx:1.7.9
       ports:
       - containerPort: 80
```

## Multi container Pods

```yaml
apiVersion: v1 
kind: Pod 
metadata:
  name: simple-webapp 
  labels:
    name: simple-webapp 
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp 
    ports:
    - containerPort: 8080
  - name: log-agent 
    image: log-agent
```

## Init containers

In a multi-container pod, each container is expected to run a process that stays alive as long as the POD's lifecycle. For example in the multi-container pod that we talked about earlier that has a web application and logging agent, both the containers are expected to stay alive at all times. The process running in the log agent container is expected to stay alive as long as the web application is running. If any of them fails, the POD restarts.

But at times you may want to run a process that runs to completion in a container. For example a process that pulls a code or binary from a repository that will be used by the main web application. That is a task that will be run only  one time when the pod is first created. Or a process that waits  for an external service or database to be up before the actual application starts. That's where initContainers comes in.

An initContainer is configured in a pod like all other containers, except that it is specified inside a initContainers section,  like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'git clone <some-repository-that-will-be-used-by-application> ; done;']
```

When a POD is first created the initContainer is run, and the process in the initContainer must run to a completion before the real container hosting the application starts. 

You can configure multiple such initContainers as well, like how we did for multi-pod containers. In that case each init container is run one at a time in sequential order.

If any of the initContainers fail to complete, Kubernetes restarts the Pod repeatedly until the Init Container succeeds.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```