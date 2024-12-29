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

