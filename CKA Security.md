All communications between Kubernetes components in control plane and worker plane is secured using TLS certificates.

For authorization in kubernetes cluster, we may use **static password file** or **static token file**. This is NOT a recommended approach.
For setting permissions for these users, we use **rbac**.

Example **static password file**:
```
/tmp/users/user-details.csv

\# User File Contents
password123,user1,u0001
password123,user2,u0002
password123,user3,u0003
password123,user4,u0004
password123,user5,u0005
```

Add this file as a **volume** to the **kube-api server**:
```
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --basic-auth-file=/tmp/users/user-details.csv
    
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
    name: kube-apiserver
    volumeMounts:
    - mountPath: /tmp/users
      name: usr-details
      readOnly: true
  volumes:
  - hostPath:
      path: /tmp/users
      type: DirectoryOrCreate
    name: usr-details
```

Next create a **role** and bind it to the user using **role bindings**:
```
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: \[""\] # "" indicates the core API group
  resources: \["pods"\]
  verbs: \["get", "watch", "list"\]

---
 This role binding allows "jane" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: user1 # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

Communication using key pairs in kubernetes, note that **public key** is written in the **crt file**:
![[Pasted image 20241130203619.png]]

For generating public and private key pairs for web communication:
```
openssl genrsa -out <keyname.key> 2048
^ This will generate the private key

openssl req -new -key <keyname.key> -subj "/CN=KUBERNETES-CA" -out keyname.csr       (CSR=certificate signing request)
^ This will request certificate for the key but with no signatures. CN stands for Common Name and it is used to classify the use of this certificate to be issued.

openssl x509 -req -in ca.csr -signkey keyname.key -out ca.crt
```

For granting administrative permissions to a particular, it's certificate must be in the group of administrators:
```
openssl req -n -key admin.key -subj \
"/CN=kube-admin/O=system:masters" -out admin.csr
```

Instead of mentioning everytime about your ca certificates & private keys, you can add them as parameters in **kubeconfig** file.

To get details of a TLS certificate:
```
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

**CA Server** is just a pair of certificate and private key pair.
Kubernetes has a **certificates API** that automate the task of signing **certificate signing requests (CSR)** 

For generating a **CSR**:
```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: Jane
spec:
  expirationSeconds: 600
  usages:
  - digital signature
  - key encipherment
  - server auth
  request:
    <base 64 encoded csr>
```

To **approve** the request:
```
kubectl certificate approve jane
```
To **share** the approved certificate:
```
kubectl get csr <csr_name> -o yaml
```
**base64 decode** it and share it with end administrator.

All certificates related management is done by **controller manager**, as it has **CSR Signing** and **CSR Approving** controllers.

For anyone to sign these certificates, one needs CA server root certificate and private key.

In **kubeconfig** file, we have three main areas: **clusters**, **contexts** and **users**. In **cluster** section, we have **CA data OR CA certificate path** and **server**. In **contexts**, we have link between **cluster** and **user**. In **users**, we have **client-key** & **client-certificate**. 
CA data must be entered in **base64** format.

For updating current-context kubeconfig:
```
kubectl config use-context prod-user@production
```

In **context** section, we can add a **namespace** field, so that kubectl automatically switches to the namespace.

To view config using kubectl command:
```
kubectl config view

kubectl config view --kubeconfig <path-to-config-file>
```

For running some commands on every boot-up:
```
nano ~/.bashrc    (add commands in bashrc file)
source ~/.bashrc  (reload it if you want effects immediately)
```

Kubernetes API is divided into multiple parts based on their purpose:
![[Pasted image 20241208175816.png]]

![[Pasted image 20241208175747.png]]

There many ways of authorization to our cluster:
**Node**, **ABAC (Attribute based)**, **RBAC** and **Webhook**.
We add nodes to group **SYSTEM:NODES** in our authorization certificate.
![[Pasted image 20241208181610.png]]

For **ABAC**, we need to pass the policy in JSON:
```
{"kind": "Policy", "spec": {"user": "dev-user", "namespace": "*", "resource: "pods", "apiGroup": "*"}}
```

Better approach is **RBAC**, as ABAC is difficult to handle for each user:
In RBAC, we create a role for a group and assigns user to that role with necessary permissions.

![[Pasted image 20241208182713.png]]
Only one authorization mode can be used at a time, though multiple authorization modes can be written. The first one will be tested, if it fails then the next one will be tried until any of the mode gets the approval. Modes are tested in order in which we have written.

To create a role with policy:
```
apiVerion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]   FOR CORE GROUPS like "api", we can leave it as                           blank, for named groups we will have to mention its                      name like "apis"
  resources: ["pods"]
  verbs: ["list", "get", "create"]
  
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
```

To link the role to a user:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

To check if you have access to certain feature of kubernetes:
```
kubectl auth can-i create <feature>    feature like deployments,nodes etc
```

To check for permissions of another user:

```
kubectl auth can-i create deployments --as dev-user

for testing in particular namespace:

kubectl auth can-i create deployments --as dev-user --namespace test
```

for restricting access to certain resources:

```
apiVerion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
  resourceNames: ["blue", "orange"]
```

![[Pasted image 20241208204352.png]]

For permissions of namespaced resources, we use **roles** and for permissions of cluster scoped resources, we use **cluster roles**

**Cluster Role**:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```

**Cluster Role Binding**:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

Imperative command for creating **role** and **cluster role**:
```
kubectl create role my-role \
  --verb=get,list,watch \
  --resource=pods,services \
  --namespace=my-namespace

kubectl create clusterrole my-role \
  --verb=get,list,watch \
  --resource=pods,services \
  --dry-run=client -o yaml
```

Imperative command for creating **role binding** and **cluster role binding**:
```
kubectl create rolebinding my-rolebinding \ 
  --role=my-role \ 
  --user=my-user \ 
  --namespace=my-namespace
```

For kubernetes workloads/items, **service accounts** are used instead of **users**:
```
kubectl create serviceaccount <service-account name>
```
Service account creates a token automatically that is used for authentication with Kubernetes API. This token is stored as a **secret object** that is created simultaneously and linked with service account.

To use the **token** with a curl request for authorization with Kube API:
```
curl https://192.168.0.1:6443/api -insecrue \
-- header "Authorization: Bearer <token>"
```
Permissions can be assigned to a service account using **RBAC** mechanisms.

There is always a **default** service account in every namespace. This service account and its token are automatically mounted to that pod as a volume at **/var/run/secrets/kubernetes.io/serviceaccount**.

To remove this default behavior and not allow kubernetes to mount a token volume:
```
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  containers:
    - name: my-kubernetes-dashboard
      image: my-kubernetes-dashboard
  automountServiceAccountToken: false
```

In version 1.24+, the token volumes now attached are of the type **projected** and they have an expiry time. Projected type means that the volume has multiple sources.
**Plus, no tokens are now automatically created upon creation of service account.** The tokens are now sent by **TokenRequest** API.

If you want to create a token for service account:
```
kubectl create token dashboard-sa

apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: dashboard-sa
(This won't be having any expiry)
```

You shouldn't be creating any service account token unless you are unable to access **TokenRequest API**.

To access a **private docker registry**, we use:
```
kubectl create secret docker-registry regcred \
	--docker-server= <> \
	--docker-username= <> \
	--docker-password=<> \
	--docker-email=<>

(docker-registry is a built-in secret type)

apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: private-registry.io/apps/internal-apps
  imagePullSecrets:
  - name: regcred
```

Docker and other system processes share the same **kernel** but they run in a different namespace such that docker has visibility of its own processes only.
If we run **ps aux** from a container we can see a process of that container, but if we run **ps aux** from our linux terminal it will show the process of container with a **different process id**. Processes can have different IDs in different namespaces.

By default, docker runs its processes using the **root** user.
If you want to change the user for docker process:
```
docker run --user=<user id/name> ubuntu sleep 3600
```

This is a good **security practice** to limit the permissions of docker user.
By default, if root user is being used, this root user won't be having complete access of our Linux system. As docker uses Linux capabilities to limit its permissions by default.

We can see root user capabilities from:
```
/usr/include/linux/capability.h
```

If you want to limit or expand the capabilities of **root user** of docker:
```
docker run --cap-drop KILL ubuntu

docker run --cap-add MAC_ADMIN ubuntu

docker run --privileged ubuntu  (For all root user access to docker root)
```

We can set these **security contexts** at either pod level or container level. At **pod level**, these will be applied to all containers inside a pod.
If we have added **security contexts** at both levels. Then **container level** contexts will override the pod level contexts.

```
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN"]
```