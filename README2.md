```sh
```
----
DEPLOYMENT TP
----

A Deployment is a high-level Kubernetes object for managing Pods and ReplicaSets. It allows declarative updates, scaling, and rolling updates to ensure the desired number of pods are running. Deployments make it easy to roll out new versions and maintain application availability with zero downtime.

we create a deployement using imperative method:
```sh
kubectl create deploy webserver --image nginx:1.22.1 --replicas=2 --dry-run=client -o yaml | tee dep.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: webserver
  name: webserver
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webserver
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: webserver
    spec:
      containers:
      - image: nginx:1.22.1
        name: nginx
        resources: {}
status: {}
```

we create the deploy:
```sh
kubectl create -f dep.yaml
deployment.apps/webserver created
```
```sh
kubectl get deployments.apps
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
webserver   2/2     2            2           9s
```
```sh
kubectl get pod
NAME                         READY   STATUS    RESTARTS   AGE
webserver-764f6ffcc6-mv2cf   1/1     Running   0          16s
webserver-764f6ffcc6-s8wth   1/1     Running   0          16s
```

we check the image used :
```sh
kubectl describe pod webserver-764f6ffcc6-s8wth | grep Image:
    Image:          nginx:1.22.1
```

Now we will use rollout and rollback in deployement to update and rollback.
First we view the current strategy setting:
```sh
kubectl get deployments.apps webserver -o yaml | grep -A 4 strategy
```
```yaml
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
```

we edit the file to:
```sh
kubectl get deployments.apps webserver
```
```yaml
  strategy:
    #rollingUpdate:
    #  maxSurge: 25%
    #  maxUnavailable: 25%
    type: Recreate
```

To update the Deployment to use a newer version of the nginx image, run:

```sh
kubectl set image deployment/webserver nginx=nginx:1.23.1-alpine --record
```

> **Note:** The `--record` flag, which was previously used to record the command in the resource's annotation, is now deprecated and no longer needed.

check the modification:
```sh
root@cp:~# kubectl get pod
NAME                         READY   STATUS    RESTARTS   AGE
webserver-66455f7f85-ks2l4   1/1     Running   0          3m9s
webserver-66455f7f85-mccfm   1/1     Running   0          3m9s
```
```sh
root@cp:~# kubectl describe po webserver-66455f7f85-ks2l4 | grep Image:
    Image:          nginx:1.23.1-alpine
```

View the history of changes:

```sh
kubectl rollout history deploy webserver

REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployments webserver nginx=nginx:1.23.1-alpine --record=true
```
View the settings:
```sh
kubectl rollout history deploy webserver --revision=1
deployment.apps/webserver with revision #1
Pod Template:
  Labels:       app=webserver
        pod-template-hash=764f6ffcc6
  Containers:
   nginx:
    Image:      nginx:1.22.1
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
  Node-Selectors:       <none>
  Tolerations:  <none>
```
```sh
kubectl rollout history deploy webserver --revision=2
deployment.apps/webserver with revision #2
Pod Template:
  Labels:       app=webserver
        pod-template-hash=66455f7f85
  Annotations:  kubernetes.io/change-cause: kubectl set image deployments webserver nginx=nginx:1.23.1-alpine --record=true
  Containers:
   nginx:
    Image:      nginx:1.23.1-alpine
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
  Node-Selectors:       <none>
  Tolerations:  <none>
```

Now lets use rollout undo to change the deploy back to previous version:
```sh
kubectl rollout undo deployment webserver
deployment.apps/webserver rolled back
```
```sh
kubectl get po
NAME                         READY   STATUS    RESTARTS   AGE
webserver-764f6ffcc6-pkl59   1/1     Running   0          6s
webserver-764f6ffcc6-ppqbx   1/1     Running   0          6s
```

```sh
kubectl describe pod webserver-764f6ffcc6-pkl59 | grep Image:
    Image:          nginx:1.22.1
```

Lets now try the `RollingUpdate` Strategy, we create from the dep.yaml we created previously:
```sh
kubectl apply -f dep.yaml
```
```yaml
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
```

Everything is created
```sh
k get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/webserver-764f6ffcc6-2r5xr   1/1     Running   0          35s
pod/webserver-764f6ffcc6-v8xwv   1/1     Running   0          35s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/webserver   2/2     2            2           35s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/webserver-764f6ffcc6   2         2         2       35s
```
We change the image:
```sh
kubectl set image deployement webserver nginx=nginx:1.23.1-alpine
```
The rolling update will gradually update pod:
```sh
k get pod
NAME                         READY   STATUS              RESTARTS   AGE
webserver-66455f7f85-blhlj   1/1     Running             0          41s
webserver-66455f7f85-nkwcl   1/1     Running             0          39s
webserver-764f6ffcc6-tdrgg   0/1     ContainerCreating   0          2s
```

Here we can also use `kubectl rollout history` and `kubectl rollout undo` 
```sh
kubectl rollout history deployment webserver # show history
kubectl rollout undo deployment webserver # rollback to revision n-1
```

-----
DAEMONSETS
-----

A DaemonSet ensures a pod runs on every node in a Kubernetes cluster, unlike a Deployment which manages a set number of pods regardless of node distribution.
DaemonSets are useful for tasks like metrics collection or logging on all nodes, especially in dynamic or large clusters. 
Since Kubernetes v1.12, you can configure which nodes run DaemonSet pods, enabling more flexible and complex deployments.

```yaml
kind: DaemonSet
metadata:
  name: ds-one
spec:
  selector:
    matchLabels:
      system: DaemonSetOne
  template:
    metadata:
      labels:
        system: DaemonSetOne
    spec:
      containers:
      - name: nginx
        image: nginx:1.22.1
        ports:
        - containerPort: 80
```
```sh
kubectl create -f ds.yaml
daemonset.apps/ds-one created
```
```sh
kubectl get ds
NAME     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ds-one   2         2         2       2            2           <none>          27s
```
```sh
k get pod -owide
NAME           READY   STATUS    RESTARTS   AGE   IP              NODE   NOMINATED NODE   READINESS GATES
ds-one-56txc   1/1     Running   0          78s   192.168.0.167   cp     <none>           <none>
ds-one-h9pdh   1/1     Running   0          78s   192.168.1.44    dp     <none>           <none>
```

Micro-services let you upgrade containers without downtime. We'll use the `OnDelete`: new Pods are NOT automatically created when you update the Pod template.
Instead, you must manually delete the old Pods.

Lets get the updateStrategy:
```sh
kubectl get ds ds-one -o yaml | grep -A 4 Strategy
```
```yaml
  updateStrategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
```
Lets change it to OnDelete:
```sh
kubectl edit ds ds-one
```
```yaml
  updateStrategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: OnDelete
```

we will then update the nginx image version
```sh
kubectl set image ds ds-one nginx=nginx:1.26-alpine --record
#Flag --record has been deprecated, --record will be removed in the future
daemonset.apps/ds-one image updated
```

pod will not update automatically:
```sh
kubectl get pod
NAME           READY   STATUS    RESTARTS   AGE
ds-one-56txc   1/1     Running   0          167m
ds-one-h9pdh   1/1     Running   0          167m
```
```sh
kubectl describe po ds-one-56txc | grep Image:
    Image:          nginx:1.22.1
```

we delete one pod:
```sh
kubectl delete po ds-one-56txc
pod "ds-one-56txc" deleted
```
new pod created with newer version:
```sh
kubectl get pod
NAME           READY   STATUS              RESTARTS   AGE
ds-one-dx9d9   0/1     ContainerCreating   0          2s
ds-one-h9pdh   1/1     Running             0          171m
```
new pod have the right image:
```sh
kubectl describe po ds-one-dx9d9 | grep Image:
    Image:          nginx:1.26-alpine
kubectl describe po ds-one-h9pdh | grep Image:
    Image:          nginx:1.22.1
```

We can also use rollout here to check old version or rollback
```sh
kubectl rollout history ds ds-one
daemonset.apps/ds-one
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image ds ds-one nginx=nginx:1.26-alpine --record=true
```
```sh
kubectl rollout history ds ds-one --revision=1
...
    Image:      nginx:1.22.1
...
```
```sh
kubectl rollout history ds ds-one --revision=2
...
   Image:      nginx:1.26-alpine
...
```

We rollout undo to rollback to specific revision:
```sh
kubectl rollout undo ds ds-one --to-revision=1
```

but we need to delete the pod for it to be recreated with the rollback:
```sh
kubectl describe pod ds-one-dx9d9 | grep Image:
    Image:          nginx:1.26-alpine
```
```sh
kubectl delete pod ds-one-dx9d9
pod "ds-one-dx9d9" deleted
```
```sh
kubectl get pod
NAME           READY   STATUS    RESTARTS   AGE
ds-one-h9pdh   1/1     Running   0          3h49m
ds-one-vshq5   1/1     Running   0          4s
```

now we have both image with the right version
```sh
kubectl describe pod ds-one-vshq5 | grep Image:
    Image:          nginx:1.22.1
kubectl describe pod ds-one-h9pdh | grep Image:
    Image:          nginx:1.22.1
```

Now if we change to by creating a new ds2 and changing name to ds-two:

```sh
kubectl get ds ds-one -o yaml > ds2.yaml
```
```yaml
  updateStrategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
```
```sh
kubectl create -f ds2.yaml
daemonset.apps/ds-two created
```
```sh
kubectl get pod
NAME           READY   STATUS    RESTARTS   AGE
ds-one-h9pdh   1/1     Running   0          5h42m
ds-one-vshq5   1/1     Running   0          113m
ds-two-jqkdk   1/1     Running   0          24s
ds-two-x7z97   1/1     Running   0          24s
```
```sh
kubectl describe pod ds-two-jqkdk | grep Image:
    Image:          nginx:1.22.1
```

now we have the roling update after editing the ds:
```sh
kubectl edit ds ds-two
```
```yaml
.....
- image: nginx:1.26-alpine
.....
```

Rolling update:
```sh
k get pod
NAME           READY   STATUS              RESTARTS   AGE
ds-one-h9pdh   1/1     Running             0          5h44m
ds-one-vshq5   1/1     Running             0          115m
ds-two-4fx7z   0/1     ContainerCreating   0          0s
ds-two-jqkdk   1/1     Running             0          3m3s
```
```sh
kubectl get ds ds-two
NAME     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ds-two   2         2         1       1            1           <none>          4m25s
```

to check rollout status:
```sh
kubectl rollout status ds ds-two
Waiting for daemon set "ds-two" rollout to finish: 1 out of 2 new pods have been updated...
```
if success:
```sh
kubectl rollout status ds ds-two
daemon set "ds-two" successfully rolled out
```

clean all:
```sh
kubectl delete ds ds-one ds-two
daemonset.apps "ds-one" deleted
daemonset.apps "ds-two" deleted
```