# CKA Curriculum Part 4 - Cluster

## Facilitate Operating System Upgrades

Occasionally upgrades have to be made to the underlying host that’s facilitating your pods. To do this in a graceful way, we can do the following.

Get a list of nodes running on which node

```
kubectl get pods -o wide

Drain the pods from the node:

kubectl drain k8s-worker04 --ignore-daemonsets

node/k8s-worker-04 cordoned
WARNING: Ignoring DaemonSet-managed pods: kube-flannel-ds-amd64-hq4wh, kube-proxy-cqpst
pod/nginx-7db9fccd9b-tdpx7 evicted
pod/coredns-fb8b8dccf-hj5km evicted
node/k8s-worker-04 evicted
```

As we can see, the non daemonset pods have been evicted and the host has been cordoned. This will apply the “SchedulingDisabled” flag on the node

```
kubectl get nodes
NAME        	STATUS                	 
k8s-worker-04   Ready,SchedulingDisabled
```

At this point, complete the required maintenance activities. Even if you reboot, the node will still be in a “SchedulingDisabled” state.

To release the node back to scheduling duties, uncordon the node:

```
kubectl uncordon k8s-worker-04
```

At which point it will be ready:

```
kubectl get nodes
NAME        	STATUS                	 
k8s-worker-04   Ready
```

## Kubernetes Releases

### Show current Kubernetes version

```
kubectl get nodes
---
NAME STATUS ROLES AGE VERSION 
master Ready master 1d v1.11.3
node-1 Ready <none> 1d v1.11.3
node-2 Ready <none> 1d v1.11.3
```

Etcd and CoreDNS have their own versions

## Understand the Kubernetes cluster upgrade process

### Versioning

No Kubernetes components can have a version higher than kube-apiserver. We will can this version X.

| Layer |              Components             | Allowed versions |        Example        |
|:-----:|:-----------------------------------:|:----------------:|:---------------------:|
|   1   |            kube-apiserver           |         X        |         v1.10         |
|   2   | controller-manager / kube-scheduler |        X-1       |     v1.9 or v1.10     |
|   3   |         kubelet / kube-proxy        |        X-2       | v1.8 or v1.9 or v1.10 |

kubectl can be at X+1, X or X-1

### When should you upgrade ?

Kubernetes supports the 3 latest minor versions.

It is recommended to upgrade when minor version at a time. If you are in version v1.10 and want to upgrade to v1.12, you should upgrade to 1.11 first.

### Upgrade process

#### GKE

Just click use the interface

#### Kubeadm

Kubeadm allows to plan and upgrade the cluster

```
kubeadm upgrade plan
```

```
kubeadm upgrade apply
```

Upgrade the cluster has 2 major steps:
* Upgrade the master nodes
* Upgrade the worker nodes

|   Role  | Version |
|:-------:|:-------:|
|  Master |  v1.10  |
| Worker1 |  v1.10  |
| Worker2 |  v1.10  |
| Worker3 |  v1.10  |

##### Upgrading the Master Node

First you upgrade your master nodes and then upgrade the worker nodes while the master is being upgraded the control plane components such as the API server scheduler and controller managers go down briefly.

The going down does not mean your work or nodes and applications on the cluster are impacted all workloads hosted on the worker nodes continue to serve users as normal.

Since the Master is down all management functions are down you cannot access the cluster using kubectl or other Kubernetes API you cannot deploy new applications or delete or modify existing ones.
The controller managers don't function either.

If a pod was to fail a new pod won't be automatically created.

But as long as the nodes and the pods are up your applications should be up and users will not be impacted.

Once the upgrade is complete and the cluster is back up it should function normally.


|   Role  | Version |
|:-------:|:-------:|
|  Master |  v1.11  |
| Worker1 |  v1.10  |
| Worker2 |  v1.10  |
| Worker3 |  v1.10  |

##### Upgrading the Worker Node

Different strategy are available:
* Upgrade all at once
  * Issue is that your applications are no longer accessible
  * Downtime
* Upgrade one node at a time
  * Move Pods on available nodes
* Add new node to the cluster
  * Provision new nodes, move Pods to the new node, then delete old nodes

##### Full procedure

1. `kubeadm upgrade plan` to have the current state of kubeadm and the cluster
2. If needed, upgrade kubeadm to the desired version: 
```
apt-get upgrade -y kubeadm=1.12.0-00
```
3. `kubeadm upgrade apply v1.12.0`
4. `kubectl get nodes` will still show the previous versions. This command show the version of kubelet
5. Upgrade Kubelet on the master node:
```
apt-get upgrade -y kubelet=1.12.0-00
systemctl restart kubelet
```
6. `kubectl get nodes` will now show that the master node is upgraded but not the worker nodes yet
7. Upgrade Kubelet on the worker node:
   1. Move the workload to another node with the `kubectl drain node-1` 
   2. Upgrade kubelet `apt-get upgrade -y kubelet=1.12.0-00 && systemctl restart kubelet`
   3. Make node schedulable again: `kubectl uncordon node-1`

For kubelet and kubeadm you can also use the following commands:

```
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.14.x-00 && \
apt-mark hold kubeadm
```

#### From scratch

Each component must be upgraded manually.

## Implement backup and restore methodologies

In a Kubernetes cluster there are two main pieces of data that need backing up:
*   Resource configuration
*   Certificate Files
*   Etcd database

### Resource configuration

Use declarative approach and store the files in a git respository.

Or get current resource configurations:

```
kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml
```
Warning: This will not save all resources.

### Backing up certificate files

Back up the /etc/kubernets/pki directory. This can also be done as a cronjob, and contains the following files:

```
/etc/kubernetes/pki$ ls
apiserver.crt          	
apiserver.key             	
ca.crt  
front-proxy-ca.crt  	
front-proxy-client.key
apiserver-etcd-client.crt  
apiserver-kubelet-client.crt  
ca.key  
front-proxy-ca.key  	
sa.key
apiserver-etcd-client.key  
apiserver-kubelet-client.key  
etcd	front-proxy-client.crt  
Sa.pub
```

### Backing up etcd

While configuring etcd, we specify a location where all the data will be stored, the data directory: `--data-dir=/var/lib/etcd`

```
ExecStart=/usr/local/bin/etcd \\
--name ${ETCD_NAME} \\ 
--cert-file=/etc/etcd/kubernetes.pem \\ 
--key-file=/etc/etcd/kubernetes-key.pem \\ 
--peer-cert-file=/etc/etcd/kubernetes.pem \\ 
--peer-key-file=/etc/etcd/kubernetes-key.pem \\ 
--trusted-ca-file=/etc/etcd/ca.pem \\ 
--peer-trusted-ca-file=/etc/etcd/ca.pem \\ 
--peer-client-cert-auth \\
--client-cert-auth \\
--initial-advertise-peer-urls https://${INTERNAL_IP}: 
--listen-peer-urls https://${INTERNAL_IP}:2380 \\ 
--listen-client-urls https://${INTERNAL_IP}:2379,http 
--advertise-client-urls https://${INTERNAL_IP}:2379 \ 
--initial-cluster-token etcd-cluster-0 \\ 
--initial-cluster controller-0=https://${CONTROLLER0_ 
--initial-cluster-state new \\
--data-dir=/var/lib/etcd
```

#### Take a snapshot of the DB, then store it in a safe location

##### Take a snapshot

```
ETCDCTL_API=3 etcdctl snapshot save snapshot.db
```

##### View the status of the snapshot

```
ETCDCTL_API=3 etcdctl snapshot status snapshot.db
```

##### Restore the snapshot

1. Stop the kube-apiserver: `service kube-apiserver stop`
2. Restore
```
ETCDCTL_API=3 etcdctl \
snapshot restore snapshot.db \ 
--data-dir /var/lib/etcd-from-backup \ # new data location
--initial-cluster master-1=https://192.168.5.11:2380,master-2=https://192.168.5.12:2380 \
--initial-cluster-token etcd-cluster-1 \ # new name must be set. During a restore, etcd creates a new cluster with new members. 
--initial-advertise-peer-urls https://${INTERNAL_IP}:2380
```
3. Reload the daemon: `systemctl daemon-reload`
4. Restart etcd: `service etcd restart`
5. Start kube-apiserver service: `service kube-apiserver start`

##### Important note

With all etcdctl commands, specify the following information

```
_API=3 etcdctl \
snapshot save snapshot.db \
--endpoints=https://127.0.0.1:2379 \ 
--cacert=/etc/etcd/ca.crt \
--cert=/etc/etcd/etcd-server.crt \ 
--key=/etc/etcd/etcd-server.key
```

