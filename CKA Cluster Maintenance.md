If a node goes down for more than 5 minutes, then kubernetes considers the pods in it as **dead**

In order to remove the pods from a particular node, we can use:
```
kubectl drain <node-name>
```
This will terminate the pods and re-schedule them on a different node. This node will be marked as **unschedulable** so that no pods can be scheduled on it.
Kubernetes won't delete any pod that is not managed by ReplicaSet while draining a node, it will give you an error.

To make the node **schedulable**, you need to do:
```
kubectl uncordon <node-name>
```

To just make the node **unschedulable** and **not drain** the pods, we can use:
```
kubectl cordon <node-name>
```

In **alpha** releases of kubernetes API, the upcoming features are disabled by default and may be buggy. This is further promoted to **beta** release, where the code is well-tested and new features are enabled by default. This **beta** release is then further promoted to a **stable** release.

Different components of Kubernetes can have different release versions, but the **Kube-API** should have an equal or higher release version than other components as it is the most important component.
**Kubectl** can be have a variance of **X+1 to X-1**, where X is the version of Kube-API
These permissible skews are documented in Kubernetes documentation.

The **recommended** approach is to be jump one minor version at a time during an upgrade.
First upgrade the master plane, then the worker plane.
When master plane is down, the worker nodes keep on serving the traffic, but the management is gone, so no scaling or replication.
We can upgrade worker nodes one at a time, to avoid downtime.
Or, we can provision new worker node versions first then down the old worker nodes.

We can use **kubeadm** for a upgrade.

To get resource configurations of all resources in your cluster
```
kubectl get all --all-namespaces -o yaml > all-deploy-serv.yaml
```
To get snapshot of **ETCD**
```
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
--endpoints=127.0.0.1:2379 \
--cacert=/etc/etcd/ca.crt \
--cert=/etc/etcd/etcd-server.crt \
--key=/etc/etcd/etcd-server.key

These certificates are important for authentication
```
To restore this ETCD Backup
```
service kube-apiserver stop         (needs to be stopped)
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
--data-dir <backup-restore-location> <backup-location>
systemctl daemon-reload
service etcd restart
service kube-apiserver start
```
A new data-directory is created, whole cluster is freshly initialized and path of data-directory has to be changed in **etcd.service** to avoid any new member to join the old cluster.

We can use **--watch** to see live changes in results of our commands:
```
kubectl get pods -n kube-system --watch 
```
