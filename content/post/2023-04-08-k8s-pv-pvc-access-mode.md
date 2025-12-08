---
title:       "Explore Kubernetes Local PV & PVC access mode"
subtitle:    ""
description: "A local volume represents a mounted local storage device such as a disk, partition or directory.
Local volumes can only be used as a statically created PersistentVolume. Dynamic provisioning is not supported."
date:        "2023-04-08"
author:      "Jamie Zhang"
image:       "/img/background-10.jpg"
tags:        ["Kubernetes"]
categories:  ["Cloud" ]
---
A local volume represents a mounted local storage device such as a disk, partition or directory.
Local volumes can only be used as a statically created PersistentVolume. Dynamic provisioning is not supported.
Compared to hostPath volumes, local volumes are used in a durable and portable manner without manually scheduling pods
to nodes. The system is aware of the volume's node constraints by looking at the node affinity on the PersistentVolume.

# Local Storage vs Dynamic/remove storage

| Storage type | Storage class/provisioner                                                                                                         | Pros                                                                      | Cons                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| :--- |:----------------------------------------------------------------------------------------------------------------------------------|:--------------------------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| HostPath| Not applicable                                                                                                                    | **Not recommended to use.**                                               | - HostPaths can expose privileged system credentials (such as for the Kubelet) or privileged APIs (such as container runtime socket), which can be used for container escape or to attack other parts of the cluster. <br> - Pods with identical configuration (such as created from a PodTemplate) may behave differently on different nodes due to different files on the nodes. <br> - The files or directories created on the underlying hosts are only writable by root. You either need to run your process as root in a privileged Container or modify the file permissions on the host to be able to write to a hostPath volume. | 
| Local PV | kubernetes.io/no-provisioner                                                                                                      | - Better I/O <br> - Better R/W performance <br> - Especially on SSD disk. | - Reduced availability. <br> - Potential data loss.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Dynamic PV | [Built-in internal provisioner or external third party provisioner](https://kubernetes.io/docs/concepts/storage/storage-classes/) | - High available and durability <br> - Automated PV/PVC configuration     | Rely on independent storage system/solution, e.g. NFS, Ceph.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |

---
**NOTE**

1. Each StorageClass has a provisioner that determines what volume plugin is used for provisioning PVs.
2. Local volumes do not currently support dynamic provisioning, however a StorageClass should still be created to delay
   volume binding until Pod scheduling.

---

# Test cases

## Presets knowledge or setting

1. Access modes

- RWO - ReadWriteOnce  
  >The volume can be mounted as read-write by a single node. ReadWriteOnce access mode still can allow multiple pods to
  access the volume when the pods are running on the same node.
- RWX - ReadWriteMany
  >The volume can be mounted as read-write by many nodes.
- ROX - ReadOnlyMany
  >The volume can be mounted as read-only by many nodes.
- RWOP - ReadWriteOncePod
  >The volume can be mounted as read-write by a single Pod.

> Access mode control binding at node level, regarding whatever single process or multi processes access the volume
> mounted is controlled by pod/application itself.

2. Reclaim Policy  
   Current reclaim policies are:

- Retain -- manual reclamation
- Recycle -- basic scrub (rm -rf /thevolume/*)
- Delete -- associated storage asset such as AWS EBS, GCE PD, Azure Disk, or OpenStack Cinder volume is deleted
>Currently, only NFS and HostPath support recycling. AWS EBS, GCE PD, Azure Disk, and Cinder volumes support deletion.

3. Phase of PV
   A volume will be in one of the following phases:

- Available -- a free resource that is not yet bound to a claim
- Bound -- the volume is bound to a claim
- Released -- the claim has been deleted, but the resource is not yet reclaimed by the cluster
- Failed -- the volume has failed its automatic reclamation

4. Create local storage class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
> Delaying volume binding allows the scheduler to consider all of a Pod's scheduling constraints when choosing an
appropriate PersistentVolume for a PersistentVolumeClaim.

## Test scenarios 
Will try below cases one by one:
| PV/C | Path | PV Access Modes | nodeAffinity.*.hostname | Comment |
| -- | --- |-----------------| --- |---------|
| local-pv-rwo | /data/local | ReadWriteOnce | worker1.it-meta.space| Cannot specify more than one hostname, otherwise will face pod scheduling issue|
| local-pv-rwx | /data/local| ReadWriteMany| worker1.it-meta.space, worker2.it-meta.space | |
| local-pv-rox | /data/local| ReadOnlyMany | worker1.it-meta.space, worker2.it-meta.space | |
| local-pv-rwop| /data/local | ReadWriteOncePod| worker1.it-meta.space| |

### Local PV RWO
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
  namespace: test
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /data/local/
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker1.it-meta.space
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-local-pvc
  namespace: test
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-storage
---
kind: Pod
apiVersion: v1
metadata:
  name: nginx-test-local-pvc-1
  namespace: test
spec:
  containers:
    - name: nginx-test-local-pvc-1
      image: nginx
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: local-pvc
  volumes:
    - name: local-pvc
      persistentVolumeClaim:
        claimName: test-local-pvc
---
kind: Pod
apiVersion: v1
metadata:
  name: nginx-test-local-pvc-2
  namespace: test
spec:
  containers:
    - name: nginx-test-local-pvc-2
      image: nginx
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: local-pvc
  volumes:
    - name: local-pvc
      persistentVolumeClaim:
        claimName: test-local-pvc
```
Example output:
```shell
root@master:/opt/k8s/config-repo/workload-test/local-pv# k get pod -n test -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP             NODE                    NOMINATED NODE   READINESS GATES
nginx-test-local-pvc-1   1/1     Running   0          3m48s   10.244.46.33   worker1.it-meta.space   <none>           <none>
nginx-test-local-pvc-2   1/1     Running   0          3m48s   10.244.46.35   worker1.it-meta.space   <none>           <none>

# write to volume from pod1
k exec -it nginx-test-local-pvc-1 -n test -- /bin/bash
cd /usr/share/nginx/html
cat > index.html <<EOF
<html>
<head></head>
<body>
<div>Welcome to Jamie's testing space about kubernetes</dir>
</body>
</html>
EOF

root@master:/opt/k8s/config-repo/workload-test/local-pv# curl 10.244.46.33
<html>
<head></head>
<body>
<div>Welcome to Jamie's testing space about kubernetes</dir>
</body>
</html>

# write to volume from pod2
k exec -it nginx-test-local-pvc-2 -n test -- /bin/bash
cd /usr/share/nginx/html
cat > storageclass.html <<EOF
<html>
<head></head>
<body>
<div>Welcome to Jamie's testing space about kubernetes</dir>
<textarea rows=15 cols=40 style="display:box">
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-volume
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
</textarea>
</body>
</html>
EOF

root@master:/opt/k8s/config-repo/workload-test/local-pv# curl 10.244.46.35/storageclass.html
<html>
<head></head>
<body>
<div>Welcome to Jamie's testing space about kubernetes</dir>
<textarea rows=15 cols=40 style="display:box">
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-volume
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
</textarea>
</body>
</html>
```
> Notes:  
> Two pods are scheduled onto worker1.it-meta.space only.
> Clean up disk files.
### Local PV RWX
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
  namespace: test
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /data/local/
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker1.it-meta.space
          - worker2.it-meta.space
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-local-pvc
  namespace: test
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-storage
---
kind: Pod
apiVersion: v1
metadata:
  name: nginx-test-local-pvc-1
  namespace: test
spec:
  containers:
    - name: nginx-test-local-pvc-1
      image: nginx
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: local-pvc
  volumes:
    - name: local-pvc
      persistentVolumeClaim:
        claimName: test-local-pvc
---
kind: Pod
apiVersion: v1
metadata:
  name: nginx-test-local-pvc-2
  namespace: test
spec:
  containers:
    - name: nginx-test-local-pvc-2
      image: nginx
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: local-pvc
  volumes:
    - name: local-pvc
      persistentVolumeClaim:
        claimName: test-local-pvc
```
Example output:
```shell
root@master:/opt/k8s/config-repo/workload-test/local-pv# k get pod -n test -o wide
NAME                     READY   STATUS    RESTARTS   AGE    IP               NODE                    NOMINATED NODE   READINESS GATES
nginx-test-local-pvc-1   1/1     Running   0          109s   10.244.164.188   worker2.it-meta.space   <none>           <none>
nginx-test-local-pvc-2   1/1     Running   0          32s    10.244.46.49     worker1.it-meta.space   <none>           <none>

# write to volume from pod1
k exec -it nginx-test-local-pvc-1 -n test -- /bin/bash
cd /usr/share/nginx/html
cat > index.html <<EOF
<html>
<head></head>
<body>
<div>Welcome to Jamie's testing space about kubernetes</dir>
</body>
</html>
EOF

root@master:/opt/k8s/config-repo/workload-test/local-pv# curl 10.244.164.188
<html>
<head></head>
<body>
<div>Welcome to Jamie's testing space about kubernetes</dir>
</body>
</html>

# write to volume from pod2
k exec -it nginx-test-local-pvc-2 -n test -- /bin/bash
cd /usr/share/nginx/html
cat > storageclass.html <<EOF
<html>
<head></head>
<body>
<div>Welcome to Jamie's testing space about kubernetes</dir>
<textarea rows=15 cols=40 style="display:box">
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-volume
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
</textarea>
</body>
</html>
EOF

root@master:/opt/k8s/config-repo/workload-test/local-pv# curl 10.244.46.49/storageclass.html
<html>
<head></head>
<body>
<div>Welcome to Jamie's testing space about kubernetes</dir>
<textarea rows=15 cols=40 style="display:box">
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-volume
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
</textarea>
</body>
</html>
```

### Local PV ROX
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
  namespace: test
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /data/local/
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker1.it-meta.space
          - worker2.it-meta.space
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-local-pvc
  namespace: test
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-storage
---
kind: Pod
apiVersion: v1
metadata:
  name: nginx-test-local-pvc-1
  namespace: test
spec:
  containers:
    - name: nginx-test-local-pvc-1
      image: nginx
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: local-pvc
  volumes:
    - name: local-pvc
      persistentVolumeClaim:
        claimName: test-local-pvc
---
kind: Pod
apiVersion: v1
metadata:
  name: nginx-test-local-pvc-2
  namespace: test
spec:
  containers:
    - name: nginx-test-local-pvc-2
      image: nginx
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: local-pvc
  volumes:
    - name: local-pvc
      persistentVolumeClaim:
        claimName: test-local-pvc
```
Example output:
```shell
root@master:/opt/k8s/config-repo/workload-test/local-pv# k get pod -n test -o wide
NAME                     READY   STATUS    RESTARTS   AGE    IP               NODE                    NOMINATED NODE   READINESS GATES
nginx-test-local-pvc-1   1/1     Running   0          4m8s   10.244.164.147   worker2.it-meta.space   <none>           <none>
nginx-test-local-pvc-2   1/1     Running   0          62s    10.244.164.149   worker2.it-meta.space   <none>           <none>

# write to volume from pod1
k exec -it nginx-test-local-pvc-1 -n test -- /bin/bash
cd /usr/share/nginx/html
cat > index.html <<EOF
<html>
<head></head>
<body>
<div>Welcome to Jamie's testing space about kubernetes</dir>
</body>
</html>
EOF

root@nginx-test-local-pvc-1:/usr/share/nginx/html# cat > index.html <<EOF
<html>
<head></head>
<body>
<div>Welcome to Jamie's testing space about kubernetes</dir>
</body>
</html>
EOF

root@nginx-test-local-pvc-1:/usr/share/nginx/html# ls
index.html
root@nginx-test-local-pvc-1:/usr/share/nginx/html# ls -al
total 12
drwxr-xr-x 2 root root 4096 Apr  8 15:20 .
drwxr-xr-x 3 root root 4096 Dec 29  2021 ..
-rw-r--r-- 1 root root  105 Apr  8 15:31 index.html
```
| :boom: Failed                                                          |
|:-----------------------------------------------------------------------|
| Test failed, seems still can write, suppose it's a access setup issue. |

### Local PV RXOP
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
  namespace: test
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOncePod
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /data/local/
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker2.it-meta.space
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-local-pvc
  namespace: test
spec:
  accessModes:
    - ReadWriteOncePod
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-storage
---
kind: Pod
apiVersion: v1
metadata:
  name: nginx-test-local-pvc-1
  namespace: test
spec:
  containers:
    - name: nginx-test-local-pvc-1
      image: nginx
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: local-pvc
  volumes:
    - name: local-pvc
      persistentVolumeClaim:
        claimName: test-local-pvc
---
kind: Pod
apiVersion: v1
metadata:
  name: nginx-test-local-pvc-2
  namespace: test
spec:
  containers:
    - name: nginx-test-local-pvc-2
      image: nginx
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: local-pvc
  volumes:
    - name: local-pvc
      persistentVolumeClaim:
        claimName: test-local-pvc
```
Example output:
```shell
root@master:/opt/k8s/config-repo/workload-test/local-pv# k apply -f local-pv-rwop.yaml
pod/nginx-test-local-pvc-1 created
pod/nginx-test-local-pvc-2 created
Error from server (Invalid): error when creating "local-pv-rwop.yaml": PersistentVolume "local-pv" is invalid: spec.accessModes: Unsupported value: "ReadWriteOncePod": supported values: "ReadOnlyMany", "ReadWriteMany", "ReadWriteOnce"
Error from server (Invalid): error when creating "local-pv-rwop.yaml": PersistentVolumeClaim "test-local-pvc" is invalid: spec.accessModes: Unsupported value: "ReadWriteOncePod": supported values: "ReadOnlyMany", "ReadWriteMany", "ReadWriteOnce"
```
| :boom: Failed                            |
|:-----------------------------------------|
| Current K8S cluster cannot support ReadWriteOncePod  |

# Ends
Local PV is still complex for ops engineer, dynamic pv(storage class) approach is Strongly recommended for production.