We can schedule a pod into a specific node by using the **nodeName** attribute in the pod definition file. After provisioning this pod, we cannot change this attribute.

To change the node of a pod after provisioning, we can use the **binding** file to do it.

```
apiVersion: v1
kind: Binding
metadata:
  name: nginx
target:
  apiVersion: v1
  kind: Node
  name: node02
```

Convert this file into **JSON** and send a POST request to kube-api for changing of node.

**Annotations** are usually used for adding information related to pod for other purposes like integration or build-version etc.

```
kubectl get pods --selector env=dev

kubectl get pods --selector env=prod,bu=finance,tier=frontend
```
**--selector** flag filters objects using their **labels**

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
   name: replicaset-1
spec:
   replicas: 2
   selector:
      matchLabels:
        tier: front-end
   template:
     metadata:
       labels:
        tier: front-end
     spec:
       containers:
       - name: front-end-pod
         image: nginx
```

**Taints** are used to disable scheduling of pods on nodes that are not **tolerant** to that particular taint.
```
kubectl taint nodes <node-name> key=value:<taint-effect>
kubectl taint nodes node1 app=blue:NoSchedule

taint-effect options = NoSchedule | PreferNoSchedule | NoExecute
```
**taint-effect options = NoSchedule | PreferNoSchedule | NoExecute**

Difference between **NoSchedule** and **NoExecute** is that NoSchedule won't have have effect on pods that are already provisioned in the node, but NoExecute will in addition to no schedule will evict the pods provisioned in the node until they have the necessary tolerations.

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    "effect": "NoSchedule"
```
This pod can now be scheduled on the node with a taint with `app=blue:NoSchedule`

```
kubectl label nodes <node-name> <label-key>=<label-value>
```
We can use **node selector** for provisioning pods on specific labelled nodes

We cannot use advanced selectors like OR, AND etc with node selectors. So, for advanced selection we use **Node Affinity**

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: data-processor
	image: data-processor
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
            - key: size
              operator: In / NotIn  (Optional)
              values:
              - Large
              - Medium
```
We have different operators in Node Affinity
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: data-processor
	image: data-processor
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
            - key: size
              operator: Exists
```

**Exists** operator will just check if the label "size" exists and then, provision the node.

In taints and toleration, our pod with toleration can be scheduled in another node, but that node won't accept any other pod except the pod that has the toleration

This issue can be resolved using **Node Affinity**, all the pods that we want to be scheduled in the right node are now scheduled. However, other pods can still be provisioned in our dedicated nodes for specific pods.

**Node Affinity** is for provisioning of our pods to dedicated nodes, while **taints and toleration** are for repelling pods that should not be scheduled in our nodes.

```
apiversion: v1
kind: Pod
metadata:  
  name: simple-webapp-color  
  labels:    
    name: simple-webapp-color
spec:  
  containers:    
    - name: simple-webapp-color      
      image: simple-webapp-color      
      ports:        
        - containerPort: 8080      
      resources:        
      requests:          
        memory: "1Gi"          
        cpu: 1        
      limits:          
        memory: "2Gi"          
        cpu: 2
```

When **limits** are used, for CPU limits, the system throttles CPU so that the usage doesn't go beyond that specified limit. This is **not** the case with memory, a pod can use more memory than its limit, but kubernetes will then terminate the pod, with **OOM (Out of Memory)** sign.

**Requests** with **no limits** is ideal. That's because, if other pods doesn't need CPU of the instance and one pod needs it, so it should be given the resources. But, the **requests** ensure, that other pods have baseline resources to run.

We cannot throttle memory unlike CPU, once memory is assigned, we cannot take it back until we kill it.

For default limits & requests, we can use **LimitRange**

```
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
spec:
  limits:
  - default:
	  cpu: 500m              (LIMIT)
	defaultRequest:
	  cpu: 500m              (REQUEST)
	max:
	  cpu: "1"               (LIMIT)
	min:
	  cpu: 100m              (REQUEST)
	type: Container
```
This is applicable on **Namespace** level. Existing pods are **not** affected.

**Resource Quotas** can be used to set hard limits for total resources in a **namespace**

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-quota
spec:
  hard:
    requests.cpu: 4
    requests.memory: 4Gi
    limits.cpu: 10
    limits.memory: 10Gi
```

These resources are total resources used by all pods in a namespace.

**We cannot edit running pod properties except for:**
		Image, activeDeadlineSeconds, Tolerations, initContainersImage

```
kubectl get pod elephant -o yaml > elephant.yaml
```

**Daemon Sets** provision a copy of pod on each node.
Good for deploying monitoring/logging agents.
Kube-proxy is deployed as daemon set

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
spec:
  selector:
    matchLabels:
      app: monitoring-agent
    template:
      metadata:
        labels:
          apps: monitoring-agent
      spec:
        containers:
        - name: monitoring-agent
          image: monitoring-agent
```

Similar to Replica Set.
**Daemon Set** uses Node Affinity & default scheduler to provision the desired pods.

We can configure our **kubelet** to read manifest files from **/etc/kubernetes/manifests**. Kubelet only understands PODS, so it can only provision pods from those files. This configuration can be done, if you want to provision pods via kubelet directly instead of going through the API server. These pods are called **static pods**.

API server has the visibility of these pods but for it, they are just **read-only** pods. **Static pods** names are by default appended with node name.

**Static pods** are used to provision pods of master plane by its kubelet.

We can write our own **scheduler**
```
---------------THIS IS THE SCHEDULER CONFIG FILE---------------------

apiVersion: kube-scheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: my-scheduler-2
leaderElection:
  leaderElect: true
  resourceNamespace: kube-system
  resourceName: lock-object-my-scheduler
```
Leader election is used if we are deploying scheduler as in a high availability environment with multiple master nodes. Only **one** scheduler can work at a time, so one is chosen using election.

Deploying scheduler as a pod: (**my-scheduler-config.yaml** is the config file mentioned above)
```
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --config=/etc/kubernetes/my-scheduler-config.yaml
    image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
    name: kube-scheduler
```
Pod that is to be scheduled by **custom scheduler**:
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  schedulerName: my-custom-scheduler
```

Setting **pod profiles**:
```
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "To be used for setting priority"
```

```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  priorityClassName: high-priority
```
This way pods can be ordered in the **scheduling queue**, then nodes are filtered, then nodes are scored according to their available resources and finally pod is bind to the node.

![[Pasted image 20241027225456.png]] 
![[Pasted image 20241027225644.png]]
**Extension points** are used to connect the **plugins** to a particular stage of scheduler.
Deploying multiple schedulers have a catch, they can get into race condition, when another scheduler is in process of scheduling a pod in the same node that is selected by another scheduler.

Kubernetes released an update of **profiles**, now in one scheduler we can have multiple profiles of different schedulers. This way only one scheduler will run.
```
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: my-scheduler-2
  plugins:
    score:
      disabled:
      - name: TaintToleration
      enabled:
      - name: MyCustomPluginA
      - name: MyCustomPluginB
- schedulerName: my-scheduler-3
  plugins:
    preScore:
      disabled:
      - name: '*'
    score:
      disabled:
      - name: '*'
- schedulerName: my-scheduler-4
```

___DONE WITH SCHEDULING___ 