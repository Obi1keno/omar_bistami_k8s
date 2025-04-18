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
