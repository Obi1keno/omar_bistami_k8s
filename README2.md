-----
PV PVC CONFIGMAP SECRET TP
-----

## CONFIGMAP

ConfigMap are basically a set of key-value pairs, that can be read as environment variables or configuration data, stored as string they can be read in serialized form.

Lets create the following directory and files in which we store colors:
```sh
cd ~ && mkdir primary
#color file for each color
echo c > primary/cyan
echo m > primary/magenta
echo y > primary/yellow
echo k > primary/black
echo "known as key" >> primary/black
tree primary
primary
├── black
├── cyan
├── magenta
└── yellow
#creating another file with our fav color
echo blue > favorite
```
Now lets create the config map
```sh
kubectl create configmap colors  \
#create from string directly
--from-literal=text=black  \
#create from file directly
--from-file=./favorite  \
#create from dir containing files
--from-file=./primary/
```

We get the cm
```sh
kubectl get configmap colors
NAME     DATA   AGE
colors   6      15s
```
```sh
kubectl get configmap -o yaml
apiVersion: v1
```
```yaml
kind: ConfigMap
metadata:
  name: colors
  namespace: default
  creationTimestamp: "2025-04-22T07:10:11Z"
  resourceVersion: "6654949"
  uid: d31d6980-f26e-4b5a-b085-1f4000cd7699
data:
  black: |
    k
    known as key
  cyan: c
  favorite: blue
  magenta: m
  text: black
  yellow: y

```

Lets use them inside a pod `simpleshell.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shell-demo
spec:
  containers:
  - name: nginx
    image: nginx
    env:
     - name: ilike
       valueFrom:
         configMapKeyRef:
           name: colors
           key: favorite
```

To test this we sh inside our pod once it is created:
```sh
#We apply the file
kubectl apply -f simpleshell.yaml
#We get the pods
kubectl get pod
NAME         READY   STATUS    RESTARTS   AGE
shell-demo   1/1     Running   0          14s
#we echo the env var
kubectl exec shell-demo -- /bin/bash -c 'echo $ilike'
kubectl delete pod shell-demo
pod "shell-demo" deleted
```

We can even include all variable inside configMap at once in env var:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shell-demo
spec:
  containers:
  - name: nginx
    image: nginx
    envFrom:
     - configMapRef:
       name: colors
```
We can get all the env variables:
```sh
kubectl exect shell-demo -- /bin/bash -c 'env'
black=k
known as key
KUBERNETES_SERVICE_PORT_HTTPS=443
cyan=c
yellow=y
KUBERNETES_SERVICE_PORT=443
HOSTNAME=shell-demo
PWD=/
PKG_RELEASE=1~bookworm
HOME=/root
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
DYNPKG_RELEASE=1~bookworm
text=black
favorite=blue
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
magenta=m
```

Configmap can also be create directly from yaml file:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fast-car
  namespace: default
data:
  car.make: Ford
  car.model: Mustang
  car.trim: Shelby
``` 
We modify our pod to use this config map as a volume:
```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
      - name: car-vol
        mountPath: /etc/cars
  volumes:
    - name: car-vol
      configMap:
        name: fast-car
```
```sh
kubectl apply -f car-map.yaml
kubectl apply -f simpleshell.yaml
```
```sh
#we exec inside and see that k8s mounted the config inside /etc/cars
kubectl exec shell-demo -- /bin/bash -c "df -ha |grep car"
/dev/root        20G  9.6G  9.7G  50% /etc/cars
#each key is a symlink to ../data/key inside the same directoy , this help ease the change of config management, so we just need to change to target:
lrwxrwxrwx 1 root root   15 Apr 22 08:10 car.trim -> ..data/car.trim
lrwxrwxrwx 1 root root   16 Apr 22 08:10 car.model -> ..data/car.model
lrwxrwxrwx 1 root root   15 Apr 22 08:10 car.make -> ..data/car.make
lrwxrwxrwx 1 root root   31 Apr 22 08:10 ..data -> ..2025_04_22_08_10_23.547447218
drwxr-xr-x 2 root root 4096 Apr 22 08:10 ..2025_04_22_08_10_23.547447218
#now lets prints one key-value:
kubectl exec shell-demo -- /bin/bash -c 'cat /etc/cars/car.trim'
Shelby
kubectl delete pod shell-demo
```

## PERSISTANT VOLUME

In This TP we will create a NFS Volume.

on CP node:
```sh
# we install needed packages
apt-get update && apt-get install -y nfs-kernel-server
# creating the shared dir
mkdir /opt/sfw
chmod 1777 /opt/sfw
# adding a file
bash -c 'echo software > /opt/sfw/hello.txt'
# we update nfs server file to add dir, you can use `snoop`to capture traffic and analyse NFS
vim /etc/exports
/opt/sfw/ *(rw,sync,no_root_squash,subtree_check)
# cause export to be re-read
exportfs -ra
```

on DP node you can validate:
```sh
# first install nfs common (not server)
apt-get install -y nfs-common
# show mounttable on cp
showmount -e cp
# mount
mount cp:/opt/sfw /mnt
# check
ls -l /mnt
total 4
-rw-r--r-- 1 root root 9 Apr 22 11:24 hello.txt
```

Now we create the pv
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvvol-1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /opt/sfw
    server: cp   #<-- Edit to match cp node
    readOnly: false
```
```sh
#create
kubectl create -f PVol.yaml
#get
kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvvol-1   1Gi        RWX            Retain           Available                          <unset>                          11s
```

## PERSISTANT VOLUME CLAIM

we create a PVC to claim the PV previously created
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-one
spec:
  accessModes:
  - ReadWriteMany
  resources:
     requests:
       storage: 200Mi # we only asked for 200Mi but we will get 1Gi because its the only availble and closet to the spec
```
```sh
# we create it
kubectl create -f pvc-one.yaml
# we check, we get 1Gi as explained before we same or more that we asked for
# wait for the pvc to be Bound
NAME      STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
pvc-one   Bound    pvvol-1   1Gi        RWX                           <unset>                 65s
```
the pv will also be bound:
```sh
# the pv is bound
kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvvol-1   1Gi        RWX            Retain           Bound    default/pvc-one                  <unset>   
```

we then create a pod to utilise this NFS
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  generation: 1
  labels:
    run: nginx
  name: nginx-nfs-omar                  #<-- Deploy name
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        volumeMounts:               #<--  VolumeMount delacration and mounting
        - name: nfs-vol-omar
          mountPath: /opt
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      volumes:                           #<-- PVC Decalaration and attachement.
      - name: nfs-vol-omar
        persistentVolumeClaim:
          claimName: pvc-one
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

We apply:
```sh
kubectl apply -f nfs-pod.yaml
# get pod
kubectl get pod
NAME                              READY   STATUS    RESTARTS   AGE
nginx-nfs-omar-58b6f86cb5-zj9hb   1/1     Running   0          15s
# describe
k describe pod nginx-nfs-omar-58b6f86cb5-zj9hb
....
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /opt from nfs-vol-omar (rw) #MOUNTED
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2vzrk (ro)
....
```

Cleaning:
```sh
kubectl delete deploy nginx-nfs-omar
kubectl delete pvc pvc-one
kubectl delete pv pvvol-1
```

We create a ResourceQuota to limit the usage of disk:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storagequota
spec:
  hard:
    persistentvolumeclaims: "10" #Limits to 10 PVC in the namespace
    requests.storage: "500Mi" #Combined Storage requests accross all PVCs in namespace limit to 500Mib 
```

We create a namespace:
```sh
vim storage-quota.yaml
kubectl create namespace small
namespace/small created
root@cp:~# kubectl describe namespaces small
Name:         small
Labels:       kubernetes.io/metadata.name=small
Annotations:  <none>
Status:       Active

No resource quota.
No LimitRange resource.
```

We create everything inside the namespace:
```sh
kubectl -n small create -f PVol.yaml
persistentvolume/pvvol-1 created
kubectl -n small create -f pvc.yaml
persistentvolumeclaim/pvc-one created
kubectl -n small create -f storage-quota.yaml
```

We describe again:
```sh
kubectl describe namespaces small
...
Resource Quotas
  Name:                   storagequota
  Resource                Used   Hard
  --------                ---    ---
  persistentvolumeclaims  1      10
  requests.storage        200Mi  500Mi

No LimitRange resource.
...
```

We re-use our old deploy and delete the namespace so we can deploy it inside small
```sh
# delete line 11 with namespace
vim nfs-pod.yaml
# create it inside small namespace
kubectl -n small create -f nfs-pod.yaml
  deployment.apps/nginx-nfs created
```
```sh
#pod created
kubectl -n small describe pod nginx-nfs-omar-58b6f86cb5-pc9fz
```
```yaml
...
    Mounts:
      /opt from nfs-vol-omar (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-n4hb5 (ro)
...
```
```sh
#Let create a 300M file inside /opt/sfw dir
#dd copy and convert rawdata 
#/dev/zero produces continuous null bytes
sudo dd if=/dev/zero of=/opt/sfw/bigfile bs=1M count=300
300+0 records in
300+0 records out
314572800 bytes (315 MB, 300 MiB) copied, 0.550368 s, 572 MB/s
#checking /opt size
136M    /opt/cni/bin
136M    /opt/cni
301M    /opt/sfw
932M    /opt
```
=> Everything went smooth since we got enough space and we respected quota.
Lets now try to see what happen if a deploy request more than the quota.
```sh
# first we delete the deploy
kubectl -n small delete deployments.apps nginx-nfs-omar
deployment.apps "nginx-nfs-omar" deleted
#The PV and PVC are sitll here, lets delete the pvc:
kubectl delete pvc pvc-one -n small
#let check the pv
kubectl describe pv pvvol-1 -n small
Status:          Released #<= Released cause pvc delete
Claim:           small/pvc-one
Reclaim Policy:  Retain #<= keep data
```
=> Dynamically provisionned storage use `ReclaimPolicy` of `StorageClass` (`Delete`,`Retain`,`Recycle`), by the default the option `Retain` is the one choosed to keep data alive for recovery.
```yaml
...
  Reclaim Policy:  Retain
...
```

Cleaning everything:
```sh
k get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   42d
root@cp:~# k get all -n small
No resources found in small namespace.
root@cp:~# k get pv
No resources found
root@cp:~# k get pvc -n small
No resources found in small namespace.
```

Now we recreate the PV with Reclaim Policy and we patch it to Delete:
```sh
grep Retain PVol.yaml
  persistentVolumeReclaimPolicy: Retain
kubectl create -f PVol.yaml
kubectl patch pv pvvol-1 -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
persistentvolume/pvvol-1 patched
kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvvol-1   1Gi        RWX            Delete           Available                          <unset>                          115m
```

Let view current quota setting
```sh
#describe ns
kubectl describe ns small
Name:         small
Labels:       kubernetes.io/metadata.name=small
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:                   storagequota
  Resource                Used  Hard
  --------                ---   ---
  persistentvolumeclaims  0     10
  requests.storage        0     500Mi

No LimitRange resource.
root@cp:~# kubectl get resourcequota
No resources found in default namespace.
#the actif resourcequota
kubectl get resourcequota -n small
NAME           AGE     REQUEST                                                   LIMIT
storagequota   6h34m   persistentvolumeclaims: 0/10, requests.storage: 0/500Mi
```

We create the pvc:
```sh
kubectl -n small create -f pvc.yaml
persistentvolumeclaim/pvc-one created
```

Now we have the 200Mi:
```sh
kubectl describe ns small
....
  Name:                   storagequota
  Resource                Used   Hard
  --------                ---    ---
  persistentvolumeclaims  1      10
  requests.storage        200Mi  500Mi
....  
```

We delete the resourcequota
```sh
kubectl -n small delete resourcequotas storagequota
resourcequota "storagequota" deleted
```

Lets edit the storage-quota.yaml and lower the capacity to 100:
```yaml
    persistentvolumeclaims: "10"
    requests.storage: "100Mi"
```
```sh
kubectl create -f storage-quota.yaml -n small
resourcequota/storagequota created
#describe ns
kubectl describe ns small
....
  persistentvolumeclaims  1      10
  requests.storage        200Mi  100Mi
```

Lets create the deployement now and **note that there is no ERRORS**
```sh
kubectl create -f nfs-pod.yaml -n small
deployment.apps/nginx-nfs-omar created
#get deploy and pod (NO ERROR)
kubectl get deploy,pod -n small
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-nfs-omar   1/1     1            1           35s

NAME                                  READY   STATUS    RESTARTS   AGE
pod/nginx-nfs-omar-58b6f86cb5-85l9p   1/1     Running   0          35s
```

```sh
kubectl -n small delete deployments.apps nginx-nfs-omar
deployment.apps "nginx-nfs-omar" deleted
kubectl -n small delete pvc pvc-one
persistentvolumeclaim "pvc-one" deleted
```
now we check the pv, we will see failed but it is because we are missing deleter volume plugin for NFS:
```sh
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM           STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvvol-1   1Gi        RWX            Delete           Failed   small/pvc-one                  <unset>                          129m
#we manually delete it
kubectl delete pv pvvol-1
persistentvolume "pvvol-1" deleted
```

applying resource limit in addition to resource quota to namespace:
```sh
kubectl -n small create -f /root/workspace/low-resource-range.yaml^C
root@cp:~# k describe ns small
...
Resource Quotas
  Name:                   storagequota
  Resource                Used  Hard
  --------                ---   ---
  persistentvolumeclaims  0     10
  requests.storage        0     100Mi

Resource Limits
 Type       Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---  ---  ---------------  -------------  -----------------------
 Container  cpu       -    -    500m             500m           -
 Container  memory    -    -    500Mi            500Mi          -
```

We create the PV:
```sh
kubectl -n small create -f PVol.yaml
Warning: spec.persistentVolumeReclaimPolicy: The Recycle reclaim policy is deprecated. Instead, the recommended approach is to use dynamic provisioning.
persistentvolume/pvvol-1 created
```

Now we cannot create the PVC because the **RESOURCE QUOTA takes effect if there is alos a LIMIT RANGE in effect**

> - **LimitRange:** <p>Set defaults and Maximums for resources at the POD/Container lvl and ensure all pod requests/limits even if not explicitly defined.</p>
> - **ResourceQuota:** <p>Enforce total usage caps for whole namespace as a whole (10CPU, 20 PODs, 10GI for all ns)</p>

>Without a LimitRange, if users don’t set resources.requests or limits, ***the ResourceQuota has nothing to count***, and is not enforced.
>> **So, LimitRange ensures that all Pods have values that ResourceQuota can track and limit.**


So now if we want to create the PVC we get:
```sh
kubectl create -f pvc.yaml -n small
Error from server (Forbidden): error when creating "pvc.yaml": persistentvolumeclaims "pvc-one" is forbidden: exceeded quota: storagequota, requested: requests.storage=200Mi, used: requests.storage=0, limited: requests.storage=100Mi
```

Lets increase the quota:
```sh
kubectl -n small edit resourcequotas
```
```yaml
spec:
  hard:
    persistentvolumeclaims: "10"
    requests.storage: 500Mi
```

And then create the deployement:
```sh
#everything working fine
kubectl -n small create -f nfs-pod.yaml
deployment.apps/nginx-nfs-omar created
```

Now if we delete the deployement, then the pvc , the pv should be available since we have recycle
```sh
kubectl -n small delete deploy nginx-nfs-omar
kubectl -n small delete pvc pvc-one
kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvvol-1   1Gi        RWX            Recycle          Available                          <unset>                          10h
#clean pv
kubectl delete pv pvvol-1
persistentvolume "pvvol-1" deleted
```

## StorageClasses in Kubernetes
**StorageClass Dynamic provisioning**

StorageClasses in Kubernetes automate storage provisioning. Users can request storage with specific properties, and Kubernetes dynamically creates PersistentVolumes (PVs) to match PersistentVolumeClaims (PVCs), simplifying management and enforcing policies.

> **Without StorageClasses:**  
> Administrators would need to manually create PVs for each PVC.

Before using `sc` we need to install the provisioner, Kubernetes lacks a built-in NFS provisioner. Deploy an external provisioner before creating an NFS StorageClass using helm:
```sh
# we add our repo
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
# update in case
helm repo update
# install by overriding some var
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=cp --set nfs.path=/opt/sfw
LAST DEPLOYED: Thu Apr 24 09:09:32 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
# check:
kubectl get sc
NAME         PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   14m
```

Lets create a pvc to utilize the SC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-one
spec:
  storageClassName: nfs-client #<= calling sc
  accessModes:
  - ReadWriteMany
  resources:
     requests:
       storage: 200Mi
```

Our pv got automatically created:

```sh
kubectl get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-bfc1445f-78e1-4ab1-bdad-2a5768522800   200Mi      RWX            Delete           Bound    default/pvc-one   nfs-client     <unset>                          3m8s

NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/pvc-one   Bound    pvc-bfc1445f-78e1-4ab1-bdad-2a5768522800   200Mi      RWX            nfs-client     <unset>                 3m8s
```

Creating a pod to use the pvc:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  containers:
  - image: nginx
    name: web-container
    volumeMounts:
    - name: nfs-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: nfs-volume
    persistentVolumeClaim:
      claimName: pvc-one
```

Lets create it and add a file into the container and check:
```sh
# creation
kubectl create -f pod-sc.yaml
pod/web-server created
# creating file
echo "Welcome to the demo of storage Class" > index.html
# copying file inside pods container
kubectl cp index.html web-server:/usr/share/nginx/html
# exec inside container to see file
kubectl exec -it web-server -- /bin/bash -c "ls -lrta /usr/share/nginx/html"
total 12
drwxr-xr-x 3 root root 4096 Apr 16 17:02 ..
drwxrwxrwx 2 root root 4096 Apr 24 10:36 .
-rw-r--r-- 1 root root   37 Apr 24 10:36 index.html
# check share to see file
ls -lrt /opt/sfw/default-pvc-one-pvc-bfc1445f-78e1-4ab1-bdad-2a5768522800/
total 4
-rw-r--r-- 1 root root 37 Apr 24 10:36 index.html
# cleaning
kubectl delete pod/web-server  pvc/pvc-one
pod "web-server" deleted
persistentvolumeclaim "pvc-one" deleted
```


