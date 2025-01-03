Docker filesystem exists at **/var/lib/docker**

![[Pasted image 20241220160333.png]]

If there is no change in layers, then your docker images using the same layers will be sharing those layers, hence saving your disk space.

When you run a docker image as a container, then docker creates a **new layer** known as **container layer**. This is a temporary layer and is **Read Write Layer** unlike other image layers which are **Read Only** layers.
Any runtime changes made into container are added into **container layer**.
If you modify any file from **Image Layers**, then docker creates a replica of that and you actually modify that replica.
![[Pasted image 20241226152136.png]]
**Container Layer** gets deleted when the container is destroyed.

To create a **docker volume** and use it:
```
docker volume create <volume_name>
This volume will be created in /var/lib/docker/volumes/<volume_name>

docker run -v <volume_name>:<container_directory> <image_name>
(It is not necessary to run docker volume create command explicitly)
```

For **bind mounting**:
```
docker run -v <host_directory>:<container_directory> <image_name>
```
Docker uses **storage drivers** for storing layered docker images.
For volumes, **volume drivers** are used.

Kubernetes follows **CRI (Container Runtime Interface)** standards. For networking, it follows **CNI (Container Network Interface)** standards. For storage, it follows **CSI**.

Volumes in kubernetes:

```
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
  - image: alpine
    name: alpine
    volumeMounts:
    - mountPath: /opt  (container Path)
      name: data-volume

  volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: Directory
  - name: data-volume-aws
    awsElasticBlockStore:
      volumeID: <volume-id>
      fsType: ext4
```

**Persistent Volume** is a cluster wide pool of storage volumes. Pods can claim volumes from this **PV** by using **Persistent Volume Claims (PVC)**.
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/data
  persistentVolumeReclaimPolicy: Retain/Delete/Recycle

--------- FOR AWS -----------
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1 Gi
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
  persistentVolumeReclaimPolicy: Retain/Delete/Recycle
```
**Recycle** policy means that the data will be deleted before assigning PV to another PVC.

Each **PVC** can be binded to only **one PV**. They have one to one relationship meaning that if a PVC is granted a larger PV than its need, then the remaining **PV** will remain unused and won't be given to any other PVC.

We can use **Labels & Selectors** for specific binding of PVC with PV.
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500 Mi
```

**PVC** with **Pods**:
```
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: event-simulator
    image: kodekloud/event-simulator
    env:
    - name: LOG_HANDLERS
      value: file
    volumeMounts:
    - mountPath: /log
      name: log-volume

  volumes:
  - name: log-volume
    persistentVolumeClaim:
      claimName: claim-log-1
```
If the **Retain** policy PV by a PVC, then it is **NOT** made available to other **PVC**, it's state becomes **released**. To make it available again use policy **Recycle**.

**Static Provisioning** of volume is manually creating a volume for every PV, while **Dynamic Provisioning** is to use a **storage class** for provisioning of volumes.

![[Pasted image 20250103151044.png]]

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: none

------------
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: delayed-volume-sc
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
