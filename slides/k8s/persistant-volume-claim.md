# Persistent Volumes

## Revisiting volumes

- [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) are used for many purposes:

  - sharing data between containers in a pod

  - exposing configuration information and secrets to containers

  - accessing storage systems

- Let's see examples of the latter usage

---

## Volumes types

- There are many [types of volumes](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes) available:

  - public cloud storage (GCEPersistentDisk, AWSElasticBlockStore, AzureDisk...)

  - private cloud storage (Cinder, VsphereVolume...)

  - traditional storage systems (NFS, iSCSI, FC...)

  - distributed storage (Ceph, Glusterfs, Portworx...)

- Using a persistent volume requires:

  - creating the volume out-of-band (outside of the Kubernetes API)

  - referencing the volume in the pod description, with all its parameters

---

## Using a cloud volume

Here is a pod definition using an AWS EBS volume (that has to be created first):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-using-my-ebs-volume
spec:
  containers:
  - image: ...
    name: container-using-my-ebs-volume
    volumeMounts:
    - mountPath: /my-ebs
      name: my-ebs-volume
  volumes:
  - name: my-ebs-volume
    awsElasticBlockStore:
      volumeID: vol-049df61146c4d7901
      fsType: ext4
```

---

## Using an NFS volume

Here is another example using a volume on an NFS server:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-using-my-nfs-volume
spec:
  containers:
  - image: ...
    name: container-using-my-nfs-volume
    volumeMounts:
    - mountPath: /my-nfs
      name: my-nfs-volume
  volumes:
  - name: my-nfs-volume
      nfs:
        server: 192.168.0.55
        path: "/exports/assets"
```

---

## Shortcomings of volumes

- Their lifecycle (creation, deletion...) is managed outside of the Kubernetes API

  (we can't just use `kubectl apply/create/delete/...` to manage them)

- If a Deployment uses a volume, all replicas end up using the same volume

- That volume must then support concurrent access

  - some volumes do (e.g. NFS servers support multiple read/write access)

  - some volumes support concurrent reads

  - some volumes support concurrent access for colocated pods

- What we really need is a way for each replica to have its own volume

---

## Persistent Volume Claims

- To abstract the different types of storage, a pod can use a special volume type

- This type is a *Persistent Volume Claim*

- A Persistent Volume Claim (PVC) is a resource type

  (visible with `kubectl get persistentvolumeclaims` or `kubectl get pvc`)

- A PVC is not a volume; it is a *request for a volume*

---

## Persistent Volume Claims in practice

- Using a Persistent Volume Claim is a two-step process:

  - creating the claim

  - using the claim in a pod (as if it were any other kind of volume)

- A PVC starts by being Unbound (without an associated volume)

- Once it is associated with a Persistent Volume, it becomes Bound

- A Pod referring an unbound PVC will not start

  (but as soon as the PVC is bound, the Pod can start)

---

## Binding PV and PVC

- A Kubernetes controller continuously watches PV and PVC objects

- When it notices an unbound PVC, it tries to find a satisfactory PV

  ("satisfactory" in terms of size and other characteristics; see next slide)

- If no PV fits the PVC, a PV can be created dynamically

  (this requires to configure a *dynamic provisioner*, more on that later)

- Otherwise, the PVC remains unbound indefinitely

  (until we manually create a PV or setup dynamic provisioning)

---

## What's in a Persistent Volume Claim?

- At the very least, the claim should indicate:

  - the size of the volume (e.g. "5 GiB")

  - the access mode (e.g. "read-write by a single pod")

- Optionally, it can also specify a Storage Class

- The Storage Class indicates:

  - which storage system to use (e.g. Portworx, EBS...)

  - extra parameters for that storage system

    e.g.: "replicate the data 3 times, and use SSD media"

---

## What's a Storage Class?

- A Storage Class is yet another Kubernetes API resource

  (visible with e.g. `kubectl get storageclass` or `kubectl get sc`)

- It indicates which *provisioner* to use

  (which controller will create the actual volume)

- And arbitrary parameters for that provisioner

  (replication levels, type of disk ... anything relevant!)

- Storage Classes are required if we want to use [dynamic provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)

  (but we can also create volumes manually, and ignore Storage Classes)

---

## Defining a Persistent Volume Claim

Here is a minimal PVC:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
   name: my-claim
spec:
   accessModes:
     - ReadWriteOnce
   resources:
     requests:
       storage: 1Gi
```

---

## Using a Persistent Volume Claim

Here is a Pod definition like the ones shown earlier, but using a PVC:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-using-a-claim
spec:
  containers:
  - image: ...
    name: container-using-a-claim
    volumeMounts:
    - mountPath: /my-vol
      name: my-volume
  volumes:
  - name: my-volume
    persistentVolumeClaim:
      claimName: my-claim
```
