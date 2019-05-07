
name: inverse
class: center, middle, inverse
layout: true

---

name: admin
class: top, left, admin
layout: true

---

name: user
class: top, left, user
layout: true

---

layout: true
class: top, left
<!-- the default layout -->

---

class: center, middle

# Kubernetes Storage 101
David Zhu, Google <br/>
Jan Šafránek, Red Hat

---

# Kubernetes

* ~~Container~~ Pod orchertrator.
  * Pod = one or more containers.
    * Containers are stateless.
      * Cleared on exit.
      * Unless a *persistent volume* is used.
---

template: user
# Pod

.column1_20[
  .center[
    <img src="pod.png" class="icon"/><br/>
    `Pod`
  ]
]


```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mysql
spec:
  containers:
  - image: mysql:5.6
    name: mysql
    ports:
    - containerPort: 3306
      name: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: opensesame
```

* Database is lost when `mysql` container ends!

---

# Kubernetes Persistent Storage Objects

.column1_20[
  .center[
    <img src="pod-pv-pvc.png" width="150px"/><br/>
  ]
]

.column2[

**Pod**
* Mounts `PersistentVolumeClaim` into container(s).

**PersistentVolumeClaim (PVC)**
* Application request for storage.
* Created by user / devops.
* Binds to single PV.
* Usable in Pods.

**PersistentVolume (PV)**
* Pointer to physical storage.
* Binds to single PVC.
* Created by admin ("pre-provisioning").
* Created by Kubernetes on demand ("dynamic provisioning").
]


---

# Kubernetes Persistent Storage Objects

.column1_20[
  .center[
  <img src="class.png" class="icon"/>
  ]
]

.column2[
`StorageClass`
* Collection of PersistentVolumes with the same characteristics.
  * "Fast", "Cheap", "Replicated", ...
* Parameters for dynamic provisioning.
* Created by admin.
* Subject of quota per namespace.
]

---

# Kubernetes Persistent Storage Objects Portability

.column1_20[
  .center[
    <img src="pod-pv-pvc.png" width="150px"/><br/>
  ]
]

.column2[
**Portable across Kubernetes clusters.**
  * Pod
  * PersistentVolumeClaim (PVC)

**Not portable across Kubernetes clusters.**
  * PersistentVolume (PV)
  * StorageClass
  * Both contain details about the storage:
      * Volume plugin.
      * IP addresses of storage server(s).
      * Paths.
      * Usernames / passwords.
      * ...
]

---

template: user
# Pod

.column1_20[
  .center[
    <img src="pod-gpv-gpvc.png" width="150px"/><br/>
  ]
]

.column2[
Mounts `PersistentVolumeClaim` into container(s).
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mysql
spec:
* volumes:
* - name: data
*   persistentVolumeClaim:
*     claimName: my-mysql-claim
  containers:
  - image: mysql:5.6
    name: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: opensesame
*   volumeMounts:
*   - name:  data
*     mountPath: /var/lib/mysql
```
]

---

template: user
# PersistentVolumeClaim

.column1_20[
  .center[
    <img src="gpod-gpv-pvc.png" width="150px"/><br/>
  ]
]

.column2[
Request for storage.

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-mysql-claim
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
    - ReadWriteOnce
```
]

---

template: user
# PersistentVolumeClaim

.column1_20[
  .center[
    <img src="gpod-gpv-pvc.png" width="150px"/><br/>
  ]
]

.column2[
Request for storage.

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-mysql-claim
spec:
* resources:
*   requests:
*     storage: 1Gi
  accessModes:
    - ReadWriteOnce
```

* *"Give me 1 GiB of storage."*
]

---

template: user
# PersistentVolumeClaim

.column1_20[
  .center[
    <img src="gpod-gpv-pvc.png" width="150px"/><br/>
  ]
]

.column2[
Request for storage.

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-mysql-claim
spec:
  resources:
    requests:
      storage: 1Gi
* accessModes:
*   - ReadWriteOnce
```

* *"Give me 1 GiB of storage."*
* *"That is mountable to single pod as read/write."*
]
---

template: user
# PersistentVolumeClaim

.column1_20[
  .center[
    <img src="gpod-gpv-pvc.png" width="150px"/><br/>
  ]
]

.column2[
Request for storage.

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-mysql-claim
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
    - ReadWriteOnce
```

* *"Give me 1 GiB of storage."*
* *"That is mountable to single pod as read/write."*
* *"And I don't really care about the rest."*
]

---

template: user
# PVC creation

.column1_20[
  .center[
    <img src="gpod-gpv-pvc.png" width="150px"/><br/>
  ]
]

.column2[
```shell
$ kubectl create -f claim.yaml
persistentvolumeclaim/my-mysql-claim created

$ kubectl get pvc
NAME            STATUS VOLUME   CAPACITY ACCESS MODES STORAGECLASS AGE
my-mysql-claim  Bound  pvc-6428 1Gi      RWO          standard     26s
```
]

---

template: user
# Pod creation

.column1_20[
  .center[
    <img src="pod-gpv-gpvc.png" width="150px"/><br/>
  ]
]

.column2[
```shell
$ kubectl create -f pod.yaml
pod/mysql created

$ kubectl get pod
NAME    READY   STATUS    RESTARTS   AGE
mysql   1/1     Running   0          19s
```
]

---

template: user
# PVC debugging

```shell
$ kubectl get pvc
NAME              STATUS 
my-broken-claim   Pending
```

---

template: user
# PVC debugging

```shell
$ kubectl get pvc
NAME              STATUS 
my-broken-claim   Pending

*$ kubectl describe pvc
...
Events:
  Type       Reason              Age               From                         Message
  ----       ------              ----              ----                         -------
  Warning    ProvisioningFailed  8s (x4 over 53s)  persistentvolume-controller  storageclass.storage.k8s
.io "foo" not found

```

---

template: user
# Delayed binding

Depending on `StorageClass`, PVC binding may be delayed until the PVC is consumed by a pod.

```shell
$ kubectl get pvc
NAME              STATUS 
my-delayed-claim  Pending

$ kubectl describe pvc
...
Events:
  Type    Reason                Age   From                         Message
  ----    ------                ----  ----                         -------
  Normal  WaitForFirstConsumer  9s    persistentvolume-controller  waiting for first consumer to
be created before binding
```
--
```shell
$ kubectl create -f pod.yaml
pod/mysql created
```
--
```shell
$ kubectl get pvc
NAME              STATUS 
my-delayed-claim  Bound

```

---

template: admin
# PersistentVolume

.column1_20[
  .center[
    <img src="gpod-pv-gpvc.png" width="150px"/><br/>
  ]
]

.column2[
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
* capacity:
*   storage: 2Gi
* accessModes:
*   - ReadWriteMany
*   - ReadWriteOnce
*   - ReadOnlyMany
* storageClassName: cheap
* persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.121.1
    path: "/vol/share-1"
```

* Some metadata.

]

---

template: admin
# PersistentVolume

.column1_20[
  .center[
    <img src="gpod-pv-gpvc.png" width="150px"/><br/>
  ]
]

.column2[
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
    - ReadWriteOnce
    - ReadOnlyMany
  storageClassName: cheap
  persistentVolumeReclaimPolicy: Retain
* nfs:
*   server: 192.168.121.1
*   path: "/vol/share-1"
```

* Pointer to storage.
  * AWS EBS, Azure DD, Ceph FS & RBD, CSI, FC, Flex, GCE PD, Gluster, iSCSI, NFS, OpenStack Cinder, Photon, Quobyte, StorageOS, vSphere

]

---

template: admin
# PersistentVolume

.column1_20[
  .center[
    <img src="gpod-pv-gpvc.png" width="150px"/><br/>
  ]
]

.column2[
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
* capacity:
*   storage: 2Gi
  accessModes:
    - ReadWriteMany
    - ReadWriteOnce
    - ReadOnlyMany
  storageClassName: cheap
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.121.1
    path: "/vol/share-1"
```

* Size of the volume.

]

---

template: admin
# PersistentVolume

.column1_20[
  .center[
    <img src="gpod-pv-gpvc.png" width="150px"/><br/>
  ]
]

.column2[
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 2Gi
* accessModes:
*   - ReadWriteMany
*   - ReadWriteOnce
*   - ReadOnlyMany
  storageClassName: cheap
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.121.1
    path: "/vol/share-1"
```

* Access modes that the volume supports.

]

---

template: admin
# PersistentVolume

.column1_20[
  .center[
    <img src="gpod-pv-gpvc.png" width="150px"/><br/>
  ]
]

.column2[
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
    - ReadWriteOnce
    - ReadOnlyMany
* storageClassName: cheap
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.121.1
    path: "/vol/share-1"
```

* `StorageClass` where this volume belongs.

]

---

template: admin
# PersistentVolume

.column1_20[
  .center[
    <img src="gpod-pv-gpvc.png" width="150px"/><br/>
  ]
]

.column2[
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
    - ReadWriteOnce
    - ReadOnlyMany
  storageClassName: cheap
* persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.121.1
    path: "/vol/share-1"
```

* What to do when the volume is not needed any longer.
  * `Recycle` (deprecated), `Retain`, `Delete`
]

---

template: admin
# Binding

.column1_20[
  .center[
    <img src="gpod-gpv-gpvc-bound.png" width="150px"/><br/>
  ]
]

.column2[
When PVC is created:
* *Matching* PV exists: they're `Bound` together.
* No matching PV: dynamic provisioning.
  * Based on StorageClass.
* No matching PV && dynamic provisioning not possible / failed: PVC is `Pending`. 
]

TODO: image with flowchart?

---

template: admin
# StorageClass

.column1_20[
  .center[
  <img src="class.png" class="icon"/>
  ]
]

.column2[

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "50"
```

* Collection of PersistentVolumes with the same characteristics.
* Usually admin territory.
* Global, not namespaced.
]


---

template: admin
# StorageClass

.column1_20[
  .center[
  <img src="class.png" class="icon"/>
  ]
]

.column2[

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
*provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "50"
```

* Who dynamically provisions volumes.
  * Name of hardcoded volume plugin.
  * Name of external provisioner.
  * Name of CSI driver.
]

---

template: admin
# StorageClass

.column1_20[
  .center[
  <img src="class.png" class="icon"/>
  ]
]

.column2[

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
*parameters:
* type: io1
* iopsPerGB: "50"
```

* Parameters for dynamic provisioning.
  * Depend on the provisioner.
]

---

template: admin
# StorageClass

.column1_20[
  .center[
  <img src="class.png" class="icon"/>
  ]
]

.column2[

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
* annotations:
*   storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "50"
```

* One `StorageClass` in the cluster can be default.
  * PVC without any `StorageClass` gets the default one.
]

---

template: inverse
# Life of a volume and pod

---

template: admin
# PersistentVolume creation

* Admin can pre-create PVs in advance.
  * Usually for "brownfield" scenarios - volumes with legacy data.
  * PV is `Available` until it matches a PVC. 

---

template: user
# PersistentVolumeClaim creation

User requests some volume for application data.

--
* Default `StorageClass` is set.
  * Admission plugin in `kube-apiserver`.
--
* Matching PV exists -> both are `Bound` together.
  * PersistentVolume controller in `kube-controller-manager`.
--
* Matching PV does not exist -> dynamic provisioning.
  * Initiated by PersistentVolume controller in `kube-controller-manager`.
  * Performed by:
      * `kube-controller-manager` for Kubernetes volume plugins.
      * External provisioner.
      * External CSI provisioner.

TODO: mention `kubectl describe pvc` for debugging somewhere

---

template: user
# Pod creation

User wants to run a container with mounted volume

--
* Pod is scheduled to a node.
--
* Volume is attached to the node.
  * Attachable volumes only (cloud).
  * Attach/Detach controller in `kube-controller-manager`.
--
* Volume is mounted to the node.
  * Volume is formatted if it's empty. 
  * `kubelet`.
--
* Container is started.
  * `kubelet`.
--

## Pod deletion
* Reverse of the above steps.

---

template: user
# PersistentVolumeClaim deletion

User does not want the application data.

* `pv.spec.persistentVolumeReclaimPolicy` is executed by `kube-controller-manager`.
  * `Recycle` (deprecated):
      * **All data from the volume are removed** ("`rm -rf *`").
      * PV is `Available` for new PVCs.
  * `Delete`:
      * **Volume is deleted in the storage backend.**
      * PV is deleted.
      * Usually for dynamically-provisioned volumes
  * `Retain`:
      * PV is kept `Released`.
      * **No PVC can bind to it.**
      * Admin should manually prune `Released` volumes.
* **Volume can be deleted when user deletes PVC!!!** 

---

template: inverse

# Stateful applications

---

template: user
# Pods are not for users

* Pod can be deleted.
  * Preemption.
  * Node is drained (for update, ...)  
  * Node goes down.

-> Users should not create Pod objects.

---

template: user
# Kubernetes hig-level objects

`Deployment`
* Runs X replicas of a single Pod template.
* When a pod is deleted, `Deployment` automatically creates a new one.
* Scalable up & down.
* All pods share the same PVC!

```yaml
TODO: simple deployment with PVC
```

---

template: user
# Deployment
<br/>
.center[
<img src="deployment.png" width="70%">
]

---

template: user
# Deployment
<br/>
.center[
<img src="deployment2.png" width="70%"/>
]

* All three pods can overwrite data of each other!
* Most applications crash / refuse to work.

---

template: user
# Kubernetes hig-level objects

`StatefulSet`
* Runs X replicas of a single Pod template.
  * Each pod gets its own PVC(s) from a PVC template.
* When a pod is deleted, `StatefulSet` automatically creates a new one.
* Each pod has a stable identity.
* Scalable up & down.

```yaml
TODO: simple statefulset?
```
---

template: user
# StatefulSet
.center[
<img src="statefulset.png" width="70%"/>
]
* The pods must be aware of other siblings!
* Usually very complex setup.
  
---

template: inverse
# Storage features

TODO: any interesting feature missing?

---

# [Topology aware scheduling](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)

* PV can be usable only by subset of nodes.
  * Cloud *regions* / *availability zones*.
  * Bare metal datacenters.
  * ...
* Pod must be scheduled:
  * Where the PV is reachable.
  * Where is enough resources to run the pod (CPU, memory, GPU, ...)

PV provisioning is delayed until Pod is created for scheduler to pick a node that matches both PV & Pod.

---

# [Local volumes](https://kubernetes.io/blog/2018/04/13/local-persistent-volumes-beta/)

* Unused local disks can be used as PVs.
  * Extra speed.
  * Lower reliability.
  * No pod scheduling flexibility.

---

# [Resize](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#expanding-persistent-volumes-claims)

* Only expansion is supported.
* Offline.
* Online (alpha). 

---

# [Raw block](https://kubernetes.io/blog/2019/03/07/raw-block-volume-support-to-beta/)

* Pods can get a block device of a PV.
  * For extra speed.
  * For software defined storage. 

---

# [Container Storage Interface (CSI)](https://kubernetes-csi.github.io/docs/)


> *Industry standard that will enable storage vendors (SP) to develop a plugin once and have it work across a number of container orchestration (CO) systems.*

* No change from user perspective, Pods & PVCs as usual.

* Extra work for cluster admin.
  * New Kubernetes external components:
      * `external-attacher`
      * `external-provisioner`
      * `node-driver-registrar`
      * `cluster-driver-registrar`
      * `external-resizer`
      * `external-snapshotter`
      * ...

---

# Snapshots

* Alpha!
* Part of CSI.
* Can take a snapshot of PVC.
* PVC can be provisioned from a snapshot.
 
---

template: inverse
# Summary

---

# Persistent Storage objects

.column1_20[
  .center[
    <img src="pod-pv-pvc.png" width="150px"/><br/>
  ]
]

.column2[
**Pod**
* Mounts `PersistentVolumeClaim` into container(s).

**PersistentVolumeClaim (PVC)**
* Application request for storage.
* Created by user / devops.

**PersistentVolume (PV)**
* Pointer to physical storage.
* Created by Kubernetes on demand ("dynamic provisioning").

**StorageClass**
* Collection of PersistentVolumes with the same characteristics.
* Parameters for dynamic provisioning.
]
---

template: inverse
# Questions?

---

# Junkyard

---
# Complex `PersistentVolumeClaim`

.column1_20[
  .center[
    <img src="gpod-gpv-pvc.png" width="150px"/><br/>
  ]
]

.column2[
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-claim
spec:
  resources:
    requests:
      storage: 1Gi
* storageClassName: cheap
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  volumeMode: Block
```

* *"Give me 1 GiB of storage."*
* "*Of something `cheap`."*
]
---

# Complex `PersistentVolumeClaim`

.column1_20[
  .center[
    <img src="gpod-gpv-pvc.png" width="150px"/><br/>
  ]
]

.column2[
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-claim
spec:
  resources:
    requests:
      storage: 1Gi
  storageClassName: cheap
* accessModes:
*   - ReadWriteOnce
*   - ReadOnlyMany
  volumeMode: Block
```

* *"Give me 1 GiB of storage."*
* *"Of something `cheap`."*
* *"And is usable in single pod as read/write or in several pods as read only."*
]

---

# Complex `PersistentVolumeClaim`

.column1_20[
  .center[
    <img src="gpod-gpv-pvc.png" width="150px"/><br/>
  ]
]

.column2[
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-claim
spec:
  resources:
    requests:
      storage: 1Gi
  storageClassName: cheap
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
* volumeMode: Block
```

* *"Give me 1 GiB of storage."*
* *"Of something `cheap`."*
* *"And is usable in single pod as read/write or in several pods as read only."*
* *"And is presented in Pods as a raw [block device](https://kubernetes.io/blog/2019/03/07/raw-block-volume-support-to-beta/)."*
]

---

# PersistentVolume

.column1_20[
  .center[
    <img src="gpod-pv-gpvc.png" width="150px"/><br/>
  ]
]

.column2[
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
    - ReadWriteOnce
    - ReadOnlyMany
  storageClassName: cheap
  persistentVolumeReclaimPolicy: Retain
* volumeMode: Filesystem
  mountOptions:
    - "
    nfsvers=3"
  nfs:
    server: 192.168.121.1
    path: "/vol/share-1"
```

* How the volume can be accessed.
  * `Filesystem` (default), `Block`
]

---

# PersistentVolume

.column1_20[
  .center[
    <img src="gpod-pv-gpvc.png" width="150px"/><br/>
  ]
]

.column2[
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
    - ReadWriteOnce
    - ReadOnlyMany
  storageClassName: cheap
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
* mountOptions:
*   - "nfsvers=3"
  nfs:
    server: 192.168.121.1
    path: "/vol/share-1"
```

* List of additional mount options.
]

---