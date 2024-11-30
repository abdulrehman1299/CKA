
**Controllers**: Available in controller-manager. Controllers take care of different departments like node controllers, manage the number of nodes & replication controllers ensure that the desired number of pods are running.

**Kube-proxy** is on worker-nodes and allows communication between services/pods in the cluster.

**ETCD** is a distributed, key-value store that is simple, secure & fast. In the key-value store, we have one key-value document for each item, containing all the information about it.

Every change that we make is stored in ETCD, once the change is written in ETCD, then it is considered complete. ETCD listens at port 2379.

Instead of using kubectl, we can directly invoke the APIs too.

Only Kube-API can interact with ETCD directly.
  
Kubernetes uses certificates for communication between its different components. Each component has its own certificate.

**ETCD** can be installed as a component or as a cluster in the kubernetes cluster in the kube-system namespace.

Similarly, other components can be installed using similar methods.

**Kubeadm** can be used to install kubernetes components as a pod in K8s cluster.

A **controller** is a component that continuously monitors the state of a component and works to bring the whole system to the desired functioning state.

If a node is unhealthy, Kube-API communicates this to the Node Controller. It gives a 40 seconds grace period to the node and 5 minutes for the POD eviction process.

**Kube-scheduler** first filters the nodes that can run the pod depending on its specifications. Then ranks the nodes on which the pod should be provisioned, suppose two nodes can run a pod. One with 12 CPU & other with 16 CPU. Pod needs 10 CPU, Kube-Scheduler will place it on 16 CPU node as it will have a higher number of CPU after pod is scheduled.

**Kube-proxy** is deployed as a daemonset by Kubeadm.
  
**apiVersion** in the YAML file refers to the Kubernetes API version.

```
kubectl run redis --image=redis:latest --dry-run=client -o yaml > redis.yaml

kubectl edit pod redis

kubectl get pods --all-namespaces
```
**ReplicaSet** takes pods already created with the labels mentioned in its **selector**. This feature is the difference between Replication Controller & ReplicaSet.
**![]()**
```
kubectl scale --replicas=6 -f replicaset.yaml

kubectl scale --replias=6 replicaset myapp-replicaset
```
**Deployment** manages a set of pods to run an application workload, usually one that doesn't maintain state.
```
kubectl create deployment --image=nginx nginx

kubectl create deployment --image=nginx nginx --dry-run=client -o yaml

kubectl create deployment --image=nginx --replicas=4

kubectl scale deployment nginx --replicas=4
```
Kubectl **apply** command compares the local file with the live configuration and makes changes in the live configuration. After that the updated change is applied to the last applied configuration.
The last applied configuration is stored in the live configuration in an annotation:
```
apiVersion: v1
kind: Pod

metadata:
  name: myapp-pod
  annotations:
	kubectl.kubernetes.io/last-applied-configuration:{"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"run":"myapp-pod","type":"front-end-service"},"name":"myapp-pod"},"spec":{"containers":[{"image":"nginx:1.18","name":"nginx-container"}]}}

labels:
  run: myapp-pod
```
Only **apply** command creates this annotation as it is a **declarative** command while the rest are **imperative** commands.

```
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
```
This will automatically use the pod's labels as selectors and expose them via a service
```
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml
```
This will not use the pods labels as selectors, instead, it will assume selectors as app=redis
```
kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml

kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```


