# **emptyDir:**

 **emptyDir** are volumes that get created empty when a Pod is created. 
 **emptyDir** exists if pod is running. If a container in a Pod crashes the **emptyDir** content is unaffected.
 Deleting a Pod then deletes all its **emptyDirs**. 

 Means empty Dirs come into live when pod is running and die with it. 

 **emptyDir** are meant for temporary working disk space.

By default, `emptyDir` volumes are stored on whatever medium is backing the node - that might be disk or SSD or network storage, depending on your environment. 

However, you can set the `emptyDir.medium` field to `"Memory"` to tell Kubernetes to mount a tmpfs (RAM-backed filesystem) for you instead.

Let's create our first **emptyDir** volume and use it to learn more.

This tutorial will cover the following topics:

- Basic **emptyDir** example
- Pod with 3 containers sharing emptyDir use
- **emptyDir** created in RAM
- **PersistentVolume** and **PersistentVolumeClaim**

## 1) Basic emptyDir Example

```
nano myVolumes-Pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: myvolumes-pod
spec:
  containers:
  - image: alpine
    imagePullPolicy: IfNotPresent
    name: myvolumes-container
    
    command: [    'sh', '-c', 'echo The Bench Container 1 is Running ; sleep 3600']
    
    volumeMounts:
    - mountPath: /demo
      name: demo-volume
  volumes:
  - name: demo-volume
    emptyDir: {}
```

This tutorial only uses the Alpine Linux image, since it is a very small Linux operating system.

From : https://en.wikipedia.org/wiki/Alpine_Linux

> Because of its small size, it's heavily used in containers providing quick boot up times.

I also use the **imagePullPolicy: IfNotPresent** . The Alpine image gets downloaded from Internet only once. Thereafter it uses the locally stored copy.

From Pod spec above:

```
  volumes:
  - name: demo-volume
    emptyDir: {}
```

We define an **emptyDir** volume named **demo-volume**. The **{}** at the end means we do not supply any further requirements for the **emptyDir** .

This **emptyDir** spec makes it available to all containers in the Pod.

From Pod spec above:

```
    volumeMounts:
    - mountPath: /demo
      name: demo-volume
```

Every container in the Pod needs to specify where it wants to have the **emptyDir** mounted.

Our example mounts the **emptyDir** at the **mountPath: /demo**

The **name: demo-volume** must refer to the volume at the bottom of the Pod spec. It specifies : mount **demo-volume** at **/demo** in the container.

Create the Pod.

```
kubectl create -f myVolumes-Pod.yaml

pod/myvolumes-pod created
```

Enter the Pod. Enter commands as shown at **#** shell prompt.

```
kubectl exec myvolumes-pod -i -t -- /bin/sh

/ # pwd
/
/ # ls
bin    demo   dev    etc    home   lib    media  mnt    proc   root   run    sbin   srv    sys    tmp    usr    var
/ # ls demo/
/ # echo test > demo/textfile
/ # ls demo/
textfile
/ # cat demo/textfile
test
/ # exit
```

- ls ... we see the demo directory exists.
- echo ... we can create a text file inside this directory.
- cat ... we can display the content of our file.

Basic emptyDir example : everything works as expected.

Delete Pod.

```
kubectl delete -f myVolumes-Pod.yaml --force --grace-period=0

pod "myvolumes-pod" force deleted
```

To ensure fast action I use **--force --grace-period=0** to delete Pod immediately. **Do not use in production.** By default Pods get 30 seconds to do their shutdown routines ( after receiving **delete** command ).

## 2) Pod with 3 Containers Sharing emptyDir Use

All containers in a Pod share use of the **emptyDir** .

Each container can independently mount the **emptyDir** at the same / or different path.

Demo below shows 3 containers all mounting the one **emptyDir** at different mount paths.

```
nano myVolumes-Pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: myvolumes-pod
spec:
  containers:
  - image: alpine
    imagePullPolicy: IfNotPresent
    name: myvolumes-container-1
    
    command: ['sh', '-c', 'echo The Bench Container 1 is Running ; sleep 3600']
    
    volumeMounts:
    - mountPath: /demo1
      name: demo-volume

  - image: alpine
    imagePullPolicy: IfNotPresent
    name: myvolumes-container-2
    
    command: ['sh', '-c', 'echo The Bench Container 2 is Running ; sleep 3600']
    
    volumeMounts:
    - mountPath: /demo2
      name: demo-volume

  - image: alpine
    imagePullPolicy: IfNotPresent
    name: myvolumes-container-3
    
    command: ['sh', '-c', 'echo The Bench Container 3 is Running ; sleep 3600']
    
    volumeMounts:
    - mountPath: /demo3
      name: demo-volume

  volumes:
  - name: demo-volume
    emptyDir: {}
    
```

Note all 3 containers refer to the same **name: demo-volume**

All 3 containers mount the **emptyDir** at different mount points. This is only to make tutorial easy to understand.

( Each container is an independent isolated running Alpine instance. )

Create the Pod.

```
kubectl create -f myVolumes-Pod.yaml

pod/myvolumes-pod created
```

Enter the Pod. Enter commands as shown at **#** shell prompt.

```
 kubectl exec myvolumes-pod -c myvolumes-container-1 -i -t -- /bin/sh
/ # ls
bin    demo1  dev    etc    home   lib    media  mnt    proc   root   run    sbin   srv    sys    tmp    usr    var
/ # echo test1 > demo1/textfile1
/ # exit
```

Mount point **demo1** exists and we could create a file there.

Enter container 2 and create a file at its mount point as shown below.

```
kubectl exec myvolumes-pod -c myvolumes-container-2 -i -t -- /bin/sh

/ # ls
bin    demo2  dev    etc    home   lib    media  mnt    proc   root   run    sbin   srv    sys    tmp    usr    var
/ # ls demo2/
textfile1
/ # echo test2 > demo2/textfile2
/ # exit
```

Note **ls demo2/** shows the file container 1 created - IMPORTANT - containers in a Pod share an **emptyDir** .

Enter container 3 and create a file at its mount point as shown below.

```
kubectl exec myvolumes-pod -c myvolumes-container-3 -i -t -- /bin/sh
/ # ls
bin    demo3  dev    etc    home   lib    media  mnt    proc   root   run    sbin   srv    sys    tmp    usr    var
/ # ls demo3/
textfile1  textfile2
/ # echo test3 > demo3/textfile3
/ # ls demo3/
textfile1  textfile2  textfile3
/ # cat demo3/textfile1
test1
/ # cat demo3/textfile2
test2
/ # cat demo3/textfile3
test3
/ # exit
```

Note **ls demo3/** lists the files created by all 3 containers.

All containers in a Pod have read/write access to the same **emptyDir** - if they requested a mount point for it. Containers can access the **emptyDir** using the same or different mount points.

Demo completed. Delete Pod.

```
kubectl delete -f myVolumes-Pod.yaml --force --grace-period=0

pod "myvolumes-pod" force deleted
```

## 3) emptyDir Created in RAM

If you do not specify where to create an **emptyDir** , it gets created on the disk space of the Kubernetes node.

If you need a small scratch volume, you can define it to be created in RAM. See very last line of spec below.

Everything else is identical to previous example.

```
nano myVolumes-Pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: myvolumes-pod
spec:
  containers:
  - image: alpine
    imagePullPolicy: IfNotPresent
    name: myvolumes-container

    command: ['sh', '-c', 'echo Container 1 is Running ; sleep 3600']

    volumeMounts:
    - mountPath: /demo
      name: demo-volume
  volumes:
  - name: demo-volume
    emptyDir:
      medium: Memory      
```

Create the Pod.

```
kubectl create -f myVolumes-Pod.yaml

pod/myvolumes-pod created
```

Enter the Pod. Enter commands as shown at **#** shell prompt.

```
kubectl exec myvolumes-pod -i -t -- /bin/sh

/ # df -h
Filesystem                Size      Used Available Use% Mounted on
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                   932.3M         0    932.3M   0% /sys/fs/cgroup
tmpfs                   932.3M         0    932.3M   0% /demo
```

- df ... displays how much disk space is available
- -h ... print sizes in human readable format ( otherwise it prints size in bytes - a long unreadable string )
- -h is the short version of --human-readable

We note on last line that our **emptyDir** got mounted on tmpfs : RAM.

The default size of a RAM-based **emptyDir** is half the RAM of the node it runs on. My tiny server has 1.8 GB RAM, so 900 MB is about right.

Such massive RAM disks may be overkill for most Pods. There is functionality to specify a **sizeLimit**.

Unfortunately that does not work as expected:

From https://github.com/kubernetes/kubernetes/issues/63126#issuecomment-387817170 ( commented May 9, 2018 )

> As you discovered, the sizeLimit parameter set for emptyDir volume could not be used for creating a volume with the size. Instead what it does it that eviction manager keeps monitoring the disk space used by pod emptyDir volume and it will evict pods when the usage exceeds the limit.

Enter the Pod and use the **emptyDir** .

```
kubectl exec myvolumes-pod -i -t -- /bin/sh

/ # dd if=/dev/urandom of=/demo/largefile bs=100M count=1
0+1 records in
0+1 records out

/ # df -h /demo
Filesystem                Size      Used Available Use% Mounted on
tmpfs                   932.3M     32.0M    900.3M   3% /demo
```

**dd if=/dev/urandom of=/demo/largefile bs=100M count=1** does a diskdump of input file ( random numbers ) with a blocksize of 100M - one occurence.

Alpine dd command has limitation that it can have blocksize of 32M max.

Copy 2 100M blocks.

```
/ # dd if=/dev/urandom of=/demo/largefile bs=100M count=2
0+2 records in
0+2 records out
/ # df -h /demo
Filesystem                Size      Used Available Use% Mounted on
tmpfs                   932.3M     64.0M    868.3M   7% /demo
```

Alpine limitation: it copied 32MB twice. No problem - we can still see our **emptyDir** is in RAM ( tmpfs ).

Let's use smaller blocksizes ( 10 MB ). 10 such block equal 100 MB.

```
/ # dd if=/dev/urandom of=/demo/largefile bs=10M count=10
10+0 records in
10+0 records out
/ # df -h /demo
Filesystem                Size      Used Available Use% Mounted on
tmpfs                   932.3M    100.0M    832.3M  11% /demo
```

Success. 100 MB space used on our /demo mounted **emptyDir**.

Now use 20 times 10 MB blocks:

```
/ # dd if=/dev/urandom of=/demo/largefile bs=10M count=20
20+0 records in
20+0 records out
/ # df -h /demo
Filesystem                Size      Used Available Use% Mounted on
tmpfs                   932.3M    200.0M    732.3M  21% /demo
/ # exit
```

Success. We are now able to use our **emptyDir** mounted in RAM.

Delete Pod.

```
kubectl delete -f myVolumes-Pod.yaml --force --grace-period=0

pod "myvolumes-pod" force deleted
```

# PersistentVolume and PersistentVolumeClaim

**emptyDir** volumes get deleted when a Pod gets deleted.

If you need persistent volumes you use: **PersistentVolume** and **PersistentVolumeClaim**

This is a 3 step process:

- You or the Kubernetes administrator defines a **PersistentVolume** ( Disk space available for use )
- You define a **PersistentVolumeClaim** - you claim usage of a part of that **PersistentVolume** disk space.
- You create a Pod that refers to your **PersistentVolumeClaim**

**Step 1** : The Kubernetes administrator creates **PersistentVolume**

You need to be signed in to the Kubernetes node where you want to make available some of its disk space.

Create a directory : **/mnt/persistent-volume**

```
mkdir /mnt/persistent-volume

echo persistent data >  /mnt/persistent-volume/persistent-file
```

That **echo** step is not normally done. Only done here so you can see we access the correct volume.

```
nano myPersistent-Volume.yaml

kind: PersistentVolume
apiVersion: v1
metadata:
  name: my-persistent-volume
  labels:
    type: local
spec:
  storageClassName: pv-demo 
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/persistent-volume"
```

The **storageClassName: pv-demo** is what links **PersistentVolumeClaim** to **PersistentVolume** .

Last 2 lines : we define this disk space exists on the host at **/mnt/persistent-volume**

https://kubernetes.io/docs/concepts/storage/persistent-volumes/#class

> A PV can have a class, which is specified by setting the storageClassName attribute to the name of a StorageClass. A PV of a particular class can only be bound to PVCs requesting that class.
>
> A PV with no storageClassName has no class and can only be bound to PVCs that request no particular class. ( not done in this tutorial )

Create the **PersistentVolume** .

```
kubectl create -f myPersistent-Volume.yaml

persistentvolume/my-persistent-volume created
```

We now have 100Mi **storageClassName: pv-demo** available for use.

Ask **kubectl** what Kubernetes knows about this storage:

```
kubectl get pv  my-persistent-volume
NAME                   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
my-persistent-volume   100Mi      RWO            Retain           Available           pv-demo                 52s
```

From https://kubernetes.io/docs/concepts/storage/persistent-volumes/#retain

> The Retain reclaim policy allows for manual reclamation of the resource. When the PersistentVolumeClaim is deleted, the PersistentVolume still exists and the volume is considered "released". But it is not yet available for another claim because the previous claimant's data remains on the volume.

From https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes

> The access modes are:
>
> - ReadWriteOnce – the volume can be mounted as read-write by a single node ... RWO
> - ReadOnlyMany – the volume can be mounted read-only by many nodes ... ROX
> - ReadWriteMany – the volume can be mounted as read-write by many nodes ... RWX
>
> Abbreviations shown at end of lines above.

**Step 2** : Create a PersistentVolumeClaim

We claim usage of 10Mi of this **PersistentVolume** via a **PersistentVolumeClaim** .

```
nano myPersistent-VolumeClaim.yaml

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-persistent-volumeclaim
spec:
  storageClassName: pv-demo 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
```

Create the **PersistentVolumeClaim**.

```
kubectl create -f myPersistent-VolumeClaim.yaml

persistentvolumeclaim/my-persistent-volumeclaim created
```

Ask **kubectl** what Kubernetes knows about this storage:

```
kubectl get pvc my-persistent-volumeclaim

NAME                        STATUS   VOLUME                 CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-persistent-volumeclaim   Bound    my-persistent-volume   100Mi      RWO            pv-demo        110s
```

**Step 3** : Create a Pod that refers to your **PersistentVolumeClaim**

```
apiVersion: v1
kind: Pod
metadata:
  name: myvolumes-pod
spec:
  containers:
  - image: alpine
    imagePullPolicy: IfNotPresent
    name: myvolumes-container

    command: ['sh', '-c', 'echo Container 1 is Running ; sleep 3600']

    volumeMounts:
      - mountPath: "/my-pv-path"
        name: my-persistent-volumeclaim-name

  volumes:
    - name: my-persistent-volumeclaim-name
      persistentVolumeClaim:
       claimName: my-persistent-volumeclaim       
```

Create the Pod.

```
kubectl create -f myVolumes-Pod.yaml

pod/myvolumes-pod created
```

Enter the Pod.

```
kubectl exec myvolumes-pod -i -t -- /bin/sh


/ # ls
bin         etc         lib         mnt         proc        run         srv         tmp         var
dev         home        media       my-pv-path  root        sbin        sys         usr
/ # ls /my-pv-path/
persistent-file
/ # cat /my-pv-path/persistent-file
persistent data
/ # echo more data >> /my-pv-path/persistent-file
/ # cat /my-pv-path/persistent-file
persistent data
more data
/ # exit
```

- **ls /my-pv-path/** ... it shows the persistent-file we created on the node ( Our spec mounts the right volume at the right mount path )
- **echo** ... append one line of text to our file
- **cat** ... display our file to see line added at bottom

Run on the node:

```
cat   /mnt/persistent-volume/persistent-file

persistent data
more data
```

It shows the line we added.

Delete Pod.

```
kubectl delete -f myVolumes-Pod.yaml --force --grace-period=0

pod "myvolumes-pod" force deleted
```

Important: After we delete the Pod the **PersistentVolume** continues to exist. All data still there.

```
$ cat   /mnt/persistent-volume/persistent-file
persistent data
more data
```

Considerable theoretical detail about **PersistentVolume** is available at https://kubernetes.io/docs/concepts/storage/persistent-volumes/

You will be able to understand that better now that you have actually created and used a **PersistentVolume** .

This was just a step-by-step cut-and-paste beginner introduction to get practical experience.

## Conclusion and Cleanup

```
kubectl delete persistentvolumeclaim/my-persistent-volumeclaim

persistentvolumeclaim "my-persistent-volumeclaim" deleted
```

Run this on the node:

```
cat   /mnt/persistent-volume/persistent-file

persistent data
more data
```

Persistent data on volume still exists after **PersistentVolumeClaim** gets deleted.

**PersistentVolumeClaim** ... points ... to data on **PersistentVolume**

**PersistentVolumeClaim** IS NOT the actual data.

Delete **PersistentVolume** itself.

```
kubectl delete persistentvolume/my-persistent-volume

persistentvolume "my-persistent-volume" deleted
```

Even the **PersistentVolume** itself IS NOT the data - it just points to it.

Run this on the node:

```
cat   /mnt/persistent-volume/persistent-file

persistent data
more data
```

Data still exists. Directory still exists.

Someone can now create other **PersistentVolumeClaims** and **PersistentVolumes** to use this **PERSISTENT** data.

Kubernetes supports 27 volume types https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes

You have just experienced the simplest two local disk storage types - and only a subset of the features.

Kubernetes supports several network-attached storage volume types.
