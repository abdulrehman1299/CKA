In **Recreate** strategy, all the pods of previous deployment is first taken down and then new pods are provisioned. In this deployment strategy, there will be a slight downtime.

```
kubectl rollout history deployment/<deployment_name>
```
This command will show the previous revisions of deployments. We can limit storing of older revisions by setting a parameter of **RevisionHistoryLimit** in our deployment files.

By **default kubernetes** follows a **rolling update** strategy, in this strategy, one by one pods are taken down and taken up. This ensures that users does not feel any downtime.
When first pod is taken down, then a new pod is taken up before taking down another pod.

Instead of updating the deployment image in your deployment files, you can use the command:

```
kubectl set image deployment/<deployment_name> <container-name>=nginx:1.9.1
```

Whenever a new deployment is being made, a new **replicaset** is created. If your **RevisionHistoryLimit=10**, this means that you will have **10** replicasets stored in your kubernetes cluster, out of which **9** will be the older ones. Though, they won't be having any pods.

To undo a new deployment:
```
kubectl rollout undo deployment/<deployment_name>
```

For checking rollout status:
```
kubectl rollout status deployment/<deployment_name>
```

This way we can manipulate **entrypoint** and **cmd** instructions of a Dockerfile:
![[Pasted image 20241117214909.png]]

**command** in manifest file, **overwrites** the entrypoint and **args** overwrites the **cmd** of dockerfile.

For giving **environment files** in manifest files:


```
kubectl create configmap <config-name> --from-literal=<key>=<value>
---
kubectl create configmap <config-name> --from-file=<path-to-file>
kubectl create configmap app-config --from-file=app.properties
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP__COLOR: blue
  APP__MODE: prod

---
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-color
    ports:
      - containerPort: 8080
    env:
      - name: APP__COLOR                         (From Direct Entry)
        value: pink
      - name: APP__COLOR                         (From ConfigMap)
        valueFrom:
          configMapKeyRef:
            key: APP__COLOR
            name: app-config
      - name: APP__COLOR                         (From Secrets)
        valueFrom:
          secretKeyRef:
            key: APP__COLOR
            name: app-color
    envFrom:                                     (All env from ConfigMap)
      - configMapRef:
          name: app-config
      - secretRef:
          name: db-secret

	volumes:                                     (Inject env as a volume)
	- name: app-config-volume
	  configMap:
	    name: app-config

```

For creating secrets using imperative way:
```
kubectl create secret generic \
	<secret-name> --from-literal=<key>=<value>
	--from-literal=<key>=<value>

kubectl create secret generic \
	<secret-name> --from-file=<file_name>.properties
```

For creating secrets using declarative method:
```
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_HOST: mysql
  DB_USER: root
  DB_PASSWORD: paswrd
```
For storing secrets in **base64** encoding, use command:
```
echo -n 'msql' | base64
```
To see the secrets in non-encrypted format:
```
kubectl get secret <secret-name> -o yaml
```
In yaml format, we can see the secrets but in **base64** encoding, we will have to decode it by using **base64 --decode**

We can mount secrets as a **volume**:
```
volumes:
- name: app-secret-volume
  secret:
    secretName: app-secret


ls /opt/app-secret-volume  (Location in container where secret is stored)
```
Secrets are not encrypted anywhere, nor in ETCD. For encryption in ETCD, you will have to write a separate file of kind **EncryptionConfiguration** and pass it to API Server.

We can restrict permissions based on namespaces.

In multi-container pods, the containers share the same volumes and resources
In multi container pods, if either of a container goes down, the whole pod goes down. So both the containers are expected to stay alive.

If there is a requirement of running a container in Pod only once when a pod is started. Then we can use **init containers**.
```
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
    command: ['sh', '-c', 'git clone  ;']
```
When a POD is first created the initContainer is run, and the process in the initContainer must run to a completion before the real container hosting the application starts.
You can configure multiple such initContainers as well, like how we did for multi-containers pod. In that case, each init container is run one at a time in sequential order.

Kubernetes supports self-healing with use of ReplicaSets to ensure that the necessary number of pods are up and running.
Kubernetes provides additional support to check the health of applications running within PODs and take necessary actions through Liveness and Readiness Probes.

___DONE___

