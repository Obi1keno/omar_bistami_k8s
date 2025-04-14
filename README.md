# LFOmar
my LF project doc

----
Installaion of kube explained
----

**Using google Kubernetes Engine GKE**

```sh
$ gcloud container clusters create linuxfoundation
$ gcloud container clusters list
$ kubectl get nodes`
```

delete: 

`$ gcloud container clusters delete linuxfoundation`

**Using minikube**

```sh
$ curl -Lo minikube ht‌tps://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
$ chmod +x minikube
$ sudo mv minikube /usr/local/bin
$ minikube start
$ kubectl get nodes
```


**Using kubeadm**

```sh
$ kubeadm init
$ kubeadm join
```

upgrade using **kubeadm**

- **Plan**: Check for the latest version of Kubernetes.
- **Apply**: Upgrade the master Control Plane (CP) first.
- **Diff**: Use `apply --dry-run` to preview changes before applying.
- **Node**: Upgrade other Control Plane nodes and worker nodes.

**Pod networking** : Calico(+Canal for Flannel integration), Flannel (network policy), kube-route, Cilium.

**Installation tools** : kubeSpray (Ansible), kops (installation on cloud providers), kind (run local k8s)


----
Control plan installation
----

Installation:
```sh
apt-update && apt-upgrade -y
apt install apt-transport-https software-properties-common ca-certificates socat -y
```
```sh
swapoff -a /disable swap by default disable on cloud providers
```
```sh
modprobe overlay
modprobe br_netfilter /ensute module are loaded
```

update kernel network to allow traffic
```sh
root@cp:~# cat << EOF | tee /etc/sysctl.d/kubernetes.conf
> net.bridge.bridge-nf-call-ip6tables = 1
> net.bridge.bridge-nf-iptables = 1
> net.ipv3.ip_forward = 1
> EOF
```

to check:
```sh
root@cp: ̃# sysctl --system
```
```sh
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

```sh
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

installing containerd
```sh
apt-get update && apt-get install containerd.io -y
containerd config default | tee /etc/containerd/config.toml
sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml 
cat /etc/containerd/config.toml | grep SystemdCgroup
systemctl restart containerd
history
```

public signing key for k8s
```sh
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
This command downloads and saves a GPG key for verifying Kubernetes packages:

1. **`curl -fsSL`**: Fetches the file silently and fails gracefully if there's an error.
2. **`sudo gpg --dearmor`**: Converts the key from ASCII format to binary format.
3. **`-o /etc/apt/keyrings/kubernetes-apt-keyring.gpg`**: Saves the binary key to the specified location for use by the APT package manager.

This ensures secure package verification when installing Kubernetes.

```sh
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
installing kubeadm kubelet and kubectl
```sh
apt-get install -y kubeadm=1.31.1-1.1 kubelet=1.31.1-1.1 kubectl=1.31.1-1.1
```
hold to preven auto update
```sh
apt-mark hold kubeadm kubelet kubectl
```
get the primary interface of the cp server
```sh
ip addr show
hostname -i
```

add local dns for cp server
```sh
10.2.0.2 k8scp
10.2.0.2 cp
```
copy the config file to /root
```sh
cp /home/omarbistami/LSCourse/LFS258/SOLUTIONS/s_03/kubeadm-config.yaml /root/
```
i had to enable ip forwarding
```sh
cat /proc/sys/net/ipv4/ip_forward
vim /etc/sysctl.conf
sudo sysctl -p
```
then kubeadm init

```sh
kubeadm init --config=kubeadm-config.yaml --upload-certs --node-name=cp | tee kubeadm-init.out
```

output

```sh
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8scp:6443 --token et2s1y.8dmwmbewust6k32v \
	--discovery-token-ca-cert-hash sha256:193ebc0443f39aba7e7e84be356a5047a64fa28fd01702a52aad534c2323ab19 \
	--control-plane --certificate-key 10bd1330be0d7800b333b61e28e47eb1bca6ac279160fddbcfd54922f50de53a

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8scp:6443 --token et2s1y.8dmwmbewust6k32v \
	--discovery-token-ca-cert-hash sha256:193ebc0443f39aba7e7e84be356a5047a64fa28fd01702a52aad534c2323ab19
```

----
access cluster with non root user
----

providing a non root user to access the cluster 


```sh
mkdir -p .kube
cd .kube/
sudo cp -i /etc/kubernetes/admin.conf ./config
sudo chown $(id -u):$(id -g) config
cat config
history | cut -c 8-
```

----
Helm installation
----

installing helm


```sh
curl https://baltocdn.com/helm/signing.asc | apt-key add -

apt-get install apt-transport-https --yes
echo " deb https://baltocdn.com/helm/stable/debian/ all main" | tee /etc/apt/sources.list.d/helm-stable-debian.list

apt-get update
apt-get install helm
helm version
```


switching to root and setting the config again:

```sh
export KUBECONFIG=/etc/kubernetes/admin.conf
```

apply manifest of cilium in course :

```sh
kubectl apply -f /home/omarbistami/LSCourse/LFS258/SOLUTIONS/s_03/cilium-cni.yaml
```

setting up auto completion for bash and kubectl 


```sh
apt-get install bash-completion -y
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> .bashrc 
```

show all kubeadm init option :

```sh
kubeadm config print init-default
```


Generate join command from CP :


```sh
kubeadm token create --print-join-command
kubeadm join k8scp:6443 --token kxtgio.ztr2bsvmus9inm7p --discovery-token-ca-cert-hash sha256:193ebc0443f39aba7e7e84be356a5047a64fa28fd01702a52aad534c2323ab19
```

----
Worker Installation
----

configuring worker node:


```sh
exit
apt-get update && apt-get upgrade -y
apt install apt-transport-https software-properties-common ca-certificates tree socat -y
swapoff -a
modprobe overlay
modprobe br_netfilter

cat << EOF | tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

cat /etc/sysctl.d/kubernetes.conf 
sysctl --system
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
cat /etc/apt/keyrings/docker.gpg 

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

cat /etc/apt/sources.list.d/docker.list 
lsb_release -cs
apt-get update && apt-get install containerd.io -y
containerd config default | tee /etc/containerd/config.toml
sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml 
cat /etc/containerd/config.toml | grep SystemdCgroup
systemctl restart containerd
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-api-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
cat /etc/apt/sources.list.d/kubernetes.list 
apt-get update

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list

vim /etc/apt/sources.list.d/kubernetes.list 
apt-get install -y kubeadm=1.31.1-1.1 kubelet=1.31.1-1.1 kubectl=1.31.1-1.1
apt-mark hold kubeadm kubelet kubectl
hostname -i
vim /etc/hosts
kubeadm join k8scp:6443 --token kxtgio.ztr2bsvmus9inm7p --discovery-token-ca-cert-hash sha256:193ebc0443f39aba7e7e84be356a5047a64fa28fd01702a52aad534c2323ab19
```

untaint the control plan and all nodes to accept non-infrastructre pod (apps)

```sh
kubectl taint node --all node-role.kubernetes.io/control-plane:NoSchedule-
```

check if core dns pod are runing else we will need to restart them by delete them

```sh
k get pods --all-namespaces
```

You'll see more interface using :

```sh
ip a
```

We need to fix the runtime-endpoint (which is the gRPC Socket throughout kubernetes communicate with containerd via kubelet)

To have more info :

```sh
sudo ls /run/containerd/containerd.sock
sudo cat /etc/containerd/config.toml | grep -i "address"
address = "/run/containerd/containerd.sock"
cat /var/lib/kubelet/config.yaml | grep -i "containerRuntimeEndpoint"
```

Using the crictl which is command line for container run interface on both cp and dp:

```sh
crictl config --set runtime-endpoint=unix://run/containerd/containerd.sock
```

----
events
----


Getting events (chronological order not respected)

```sh
kubectl get events
```

----
Deploying app and exposing it
----

Deploying an app:

```sh
kubectl get deployments nginx -o yaml > first.yaml
vim first.yaml 
k delete deploy nginx
k get pod -A
k create -f first.yaml 
k get deploy nginx -o yaml > second.yaml
diff first.yaml second.yaml 
kubectl create deployment two --image=nginx --dry-run=client -o yaml
k get deploy
k describe deploy nginx
```

exposing an app as a service

```sh
kubectl expose -h
kubectl expose deployment/nginx 
```

create, apply, edit and patch manifest yaml files:
- **Create**: Used to create a new object. Typically used once.
- **Apply**: Combines the functionality of `create` and allows modifications to existing objects.
- **Edit**: Opens the object in an editor for manual modifications.
- **Patch**: Applies changes using JSON patch, merge patch, or strategic merge patch methods.

**replace --force** : force modification of fields by recreating ressource

we will expose the nginx now using service

first we add port to manifest :

```yaml
ports:
- containerPort: 80
  protocol: TCP
```

we use a kubectl replace, to recreate ressource:

```sh
kubectl replace -f first.yaml --force
```

then we expose

```sh
kubectl expose deployment/nginx
```

we check the service created:

```sh
kubectl get svc
```

the endpoint shown is managed by cilium but not the true endpoint, to check endpoint we use :

```sh
kubectl get endpoints
```

we do a curl inside the cp or dp to check if it works :

```sh
curl cluster-ip:port (10.104.228.205:80)
```

we scale to 3 replicaset :

```sh
kubectl scale deployement nginx --replicas=3
```

we delete oldetest one with kubectl delete pod to see it recreate and k8s distrubute trafic to others pods

now we expose outside the cluster:

first think we do is print env var inside pod

```sh
kubectl exec nginx-7769f8f85b-6drhz -- printenv | grep KUBERNETES
```
we then delete and expose again using type.

```sh
kubectl expose deploy nginx --type=LoadBalancer
```

now we have External IP pending and port, we can use ip of our vm to acces using a navigator

```sh
root@cp:~/workspace# k get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP        5d20h
nginx        LoadBalancer   10.109.61.198   <pending>     80:32257/TCP   15m
```

using in navigator : http://34.155.199.133:32257

if we scale to zero , not working anymore
```sh
k scale deploy nginx --replicas=0
```
if delete the deploy, the pod will be delete but svc still up and creating the deploy again, we have access again
```sh
k delete deploy nginx
```
we need to delete also the svc

```sh
k delete svc nginx
```


----
K8S Concepts
----

K8S architecture

![Kubernetes Architecture](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

The main components of Kubernetes architecture are:

- **Control Plane**:
  - API Server
  - ETCD Database
  - Kube Controller Manager
  - Kube Scheduler
  - Additional components like CoreDNS

- **Worker Nodes**:
  - Kubelet (communicates with container runtime)
  - Kube-proxy (manages networking rules)
  - Container runtime (e.g., containerd)

- **Key Concepts**:
  - Pods: The smallest deployable units in Kubernetes.
  - Services: Expose pods to other services or external traffic.
  - Namespaces: Logical partitions for resources.
  - Network Policies: Define communication rules between pods.
  - Storage: Persistent storage for stateful applications.

For cluster-wide logging, Fluentd is commonly used, and Prometheus is utilized for metrics collection.


**Fluentd** to add unified logging Layer for the cluster for cluster wide logging

For cluster wide metrics, we use **prometheus**

**Kubelet role** : communicate with container engine 

1. **Pod Management**: The Kubelet watches for Pod specifications (manifests) assigned to its node and ensures the containers in those Pods are running.
2. **Health Monitoring**: It monitors the health of the containers and restarts them if they fail, based on the Pod's restart policy.
3. **Node Communication**: The Kubelet communicates with the Kubernetes API server to report the status of the node and the Pods running on it.
4. **Volume Management**: It manages the mounting and unmounting of volumes for Pods.
5. **Container Runtime Interface (CRI)**: The Kubelet interacts with the container runtime (e.g., Docker, containerd) to manage container operations.
-report status of pod and nodes to CP

```plaintext
+-------------------+       +-------------------+
| Kubernetes Master |       | Kubernetes Node   |
|                   |       |                   |
| +---------------+ |       | +---------------+ |
| | API Server    |<--------->| Kubelet       | |
| +---------------+ |       | +---------------+ |
|                   |       | | Pod           | |
| +---------------+ |       | | +-----------+ | |
| | Scheduler     | |       | | | Container | | |
| +---------------+ |       | | +-----------+ | |
|                   |       | +---------------+ |
+-------------------+       +-------------------+
```

Also can manage topology manager for NUMA architecture.

Operators are specialized controllers or watch loops that act as agents, watchers, or informers, leveraging a downstream store with a deltaFIFO queue to manage the state of Kubernetes objects. SharedOperators, also known as informers, are designed to manage objects that are utilized by multiple components, ensuring efficient and consistent state management across the cluster.

Endpoints, namespaces, and service accounts each manage their respective resources for Pods. 
The Deployment operator oversees ReplicaSets, which in turn manage Pods running identical Pod specifications (PodSpecs) or replicas.

The Service Operator monitors the Endpoint Operator to ensure persistent IP allocation for Pods. It communicates updates through the Kubernetes API to all kube-proxy instances and the Cilium add-on across worker nodes in the cluster, facilitating seamless networking and service discovery.

Service Operator :

- Connect pods toghether
- Expose pods to internet
- Decouple settings
- Define pod access policy
### Pod: The Smallest Unit in Kubernetes

A **Pod** is the smallest deployable unit in Kubernetes, encapsulating one or more containers that share the same network namespace and storage. Key points:

- **Parallel Start**: Containers in a Pod start in parallel unless **`initContainers`** are used for initialization.
- **Single IP**: Each Pod has one IP. Containers communicate via **IPC**, **loopback**, or a **shared filesystem**.
- **Best Practice**: Typically, a Pod has one main container, but **sidecars** can provide auxiliary functions like logging or request handling.

Pods ensure shared lifecycle, network, and storage for tightly coupled containers.

----
Resource quotas
----

Resource and quotas:

```yaml
resources:
  limits:
    cpu: "1"
    memory: "4Gi"
  requests:
    cpu: "0.5"
    memory: "500Mi"
```

Another way to manager is by using ResourceQuota, allow hard and soft limit to be set in a specific namespace, manage of more resource than CPU and RAM and allow limiting several object

The scopeSelector in the quota spec is used to run a pod at a certain priority if it has the priorityClassName

InitContainer : define the first container to run before the oder container runs, for example a container that update the database.
You can also use LivenessProbes, ReadinessProbes and StatefulSets but this add a complexity levels.

LivenessProbes : check if container is running (Request an HTTP GET PROBE, or a TCP SOCKET PROBE, or Executing a command) if not running k8s will respawn it.
ReadinessProbes : ensure that app inside container can accept request using same methods as LivenessProbes (HTTP, Socket, Command), if not it will be removed from the list of endpoints for a service.

Livenessprobe HTTP example:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 3
  periodSeconds: 5
```
ReadinessProbes Socket example:

```yaml
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```
livenessProbe command example:

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 3
  periodSeconds: 5
```


InitContainers can have a different view on Storage and Security settings which can allows utilites and commands to be used , which the app containers cannot use, it also have independent security from other app containers.

```yaml
spec:
  containers:
    name: main-app
    image: databaseD
  initContainers:
    name: wait-database
    image: busybox
    command: ['sh','-c','until ls /db/dir; do sleep 5; done;']

```

----
Node Management
----

Node object define an instance a worker node.
**if the kube-api server cannot communicate with the kubelet on a node for 5min, the default NodeLease will schedule the node for deletion and the NodeStatus will change from Ready**, *pods will be evicted once a connection is established, they are no longer forcibly removed and rescheduled by the cluster.*

**All node existe in the kube-node-lease namespace**

```sh
kubectl delete node node_name
```

all pod will be evacueted , you can then use To remote cluster informations:
```sh
kubeadm reset 
```

will show all information related to the node including CPU/Memory and other ressource usage.
```sh
kubectl describe node
```

----
Networking
----

**Kind:** kubernetes in docker.
**Dind**: Docker in docker.

What we need to solve in kube networking is:

- **Container-to-Container Communication**: Solved within a Pod using `localhost` as all containers in a Pod share the same network namespace.
- **Pod-to-Pod Communication**: Achieved through virtual Ethernet (veth) pairs and bridging interfaces, allowing seamless communication across Pods.
- **Pod-to-Service Communication**: Handled by Kubernetes Services, which provide stable endpoints and load balancing for Pods.
- **External-to-Pod Communication**: Also managed by Kubernetes Services, enabling external traffic to reach Pods through mechanisms like NodePort, LoadBalancer, or Ingress.

**PodCIDR** refers to the IP address range allocated to pods on a Kubernetes node. It defines the subnet from which pod IPs are assigned, ensuring each pod on the node has a unique IP address within the cluster.

When allocating IPs in Kubernetes:

- **10.96.0.1**: This IP is always reserved for the Kubernetes API server.
- **10.96.0.10**: This IP is always reserved for CoreDNS, the cluster's DNS service.

kube-proxy, a core component of Kubernetes, is deployed on each worker node. Its primary responsibilities include:

- Maintaining network rules on each node to implement Kubernetes Services.
- Utilizing the operating system's packet filtering layer (e.g., iptables or nftables) for traffic management.
- Routing traffic between nodes within the cluster.
- Enabling service-to-node traffic remapping for seamless communication.

**kube-proxies do not communcate with each other, they all communicate with api-server who manage them**

CoreDNS handles DNS resolution within the cluster. If the IP range is `10.96.0.0/16`, CoreDNS will be accessible at `10.96.0.10` on each worker node. It enables services to be resolved using DNS names instead of IP addresses, such as `mydb.some-ns.svc.cluster.local`.


All pod will have on their /etc/resolv.conf:
```c
nameserver 10.96.0.10
```
So the pods will have a DNS server to interogate.

Network Policies are basically firewall rules on your nodes to controll trafic.

Ingress controller and Gateway API controllers manages pods that will do more complex traffic ingresses (external to cluster) to the cluster.

All containers within a Pod share the same IP address and network namespace. This shared network namespace is managed by a special "pause" container, which acts as the parent container for the Pod. The pause container is responsible for holding the network namespace and ensuring that all other containers in the Pod can communicate seamlessly using the same IP address.

![Pause Container Role](pausecontainer.png)


To communicate with each other, pods can utilize the following methods:

- **Loopback Interface**: Containers within the same pod share the same network namespace, allowing communication via `localhost`.
- **Shared Filesystem**: Pods can share common files through a shared volume mounted across containers.
- **Inter-Process Communication (IPC)**: Containers can communicate using IPC mechanisms like shared memory or semaphores.

Pods have ephemeral IP addresses, which can change over time. **Services provide a stable, fixed IP address to ensure consistent communication with Pods.**

Additionally, Services handle load balancing across Pod replicas, distributing traffic evenly to maintain performance and reliability.

Services also manage traffic both within the cluster and from external sources, ensuring seamless connectivity and accessibility.

The default service type in Kubernetes is **ClusterIP**. 
A service uses a **Selector** to identify the appropriate Pods it should route traffic to. It then uses the **TargetPort** to direct traffic to the correct port on the selected Pods.

![Kubernetes Service Types](https://kubernetes.io/images/docs/services-overview.svg)

Other service types:

![Service Types](servicetyppe.webp)

1. **ClusterIP**: The default service type, accessible only within the cluster.
2. **NodePort**: Exposes the service on a static port on each node's IP.
3. **LoadBalancer**: Integrates with cloud providers to expose the service externally via a load balancer.
4. **ExternalName**: Maps a service to an external DNS name.

Each service type is designed to handle specific use cases for internal and external communication.

When creating a Service in Kubernetes, an Endpoint (EP) resource with the same name as the Service is automatically created. This Endpoint contains a list of all the Pods that match the Service's selector. It effectively groups the Pods associated with the Service, allowing you to identify which Pods are members of the Service (e.g., Pod1, Pod2, Pod3).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

This example defines a `ClusterIP` service named `my-service` in the `default` namespace. It routes traffic on port 80 to the target port 8080 of the pods selected by the label `app: my-app`.

using selector and port target, we can specify which container to reach.

```yaml
spec:
  selector:
    app: microservice
  ports:
    - protocl: tcp
      name: nginx
      port: 3200 # This is you can choose arbitrary
      targetPort: 3000 # this need to correspond to the port inside the pod :
    - protocol: tcp
      name: nginx2
      port: 3400
      targetPort: 9000
```
Pod :

```yaml
spec:
  container:
  - name: nginx1
    image: nginx
    ports: 3000
  - name: nginx2
    image: nginx
    ports: 9000
```

### Service Type: Headless

A **headless service** is used when you need to communicate directly with specific Pods rather than relying on load balancing or random selection. This is particularly useful for stateful applications like databases (e.g., MongoDB, MySQL) where Pods are not identical and may have distinct roles (e.g., master, slave).

To create a headless service, set the `clusterIP` field to `None` in the service specification:


```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
  namespace: default
spec:
  clusterIP: None #added
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

**This configuration ensures that the service does not allocate a cluster IP and instead provides direct DNS resolution to the individual Pods.**

It will return the pod ip address instead of the service ip address.

- **clusterIP**: Default => internal service, only accessible inside the cluster and need to use ingress to get acess to outside
- **nodePort**: Create a service that has a steady (same port) on each node (worker) in the cluster : `nodeip:steady_port`, can be accessed from outside.

```yaml
spec:
  type: NodePort
  selector:
    app: microservice-one
  ports:
    - protocol: TCP
      port: 3200
      targetPort: 3000
      nodePort: 30008 (#node port range 30000-32767)
```

Services handle requests across nodes, ensuring seamless communication regardless of pod location. 

NodePort is simple and often used for testing, it is not recommended for production due to security concerns.

- **LoadBalancer**: The service becomes accessible via an external load balancer provided by the cloud platform (e.g., Google Cloud, OpenStack, AWS, Azure). This allows external traffic to reach the service seamlessly.

```yaml
spec:
  type: LoadBalancer
  selector:
    app: microservice-one
  ports:
    - protocol: TCP
      port: 3200
      targetPort: 3000
      nodePort: 30008 (#node port range 30000-32767)
```

The LoadBalancer manages traffic by directing requests to the appropriate nodes in the cluster. Instead of accessing the worker node directly using `worker-ip:worker-port`, you access the LoadBalancer, which then routes the traffic to the correct node and service.

**NodePort Range**: The NodePort service type uses a port range of `30000-32767` for external access.


LoadBalancer builds upon NodePort, which in turn extends the functionality of ClusterIP. Each type adds additional capabilities for exposing services, with LoadBalancer providing external access through a cloud provider's load balancer.

Ingress enables external access to the application with SSL/TLS encryption, while keeping the service internal for enhanced security.

```yaml
spec:
  rules:
  - host: myapp.com #routing rules
    http: #is not the http on webbrowser it is how the request is forwarded to svc.
      paths:
      - path: /
        backend:
          serviceName: myapp-internal-service
          serviceport: 8080
```

Ingress forward to ClusterIP service type (no nodePort)

First step is to install an ingressController which will do the processing of the rules, manages redirections and will be the entry point for the clusters.
In a cloud environment, the cloud provider's load balancer acts as the entry point, redirecting traffic to the ingress controller (Layer 7). 

On bare-metal infrastructure (self-managed), you must handle this functionality manually, often by configuring a load balancer or proxy server to route traffic to the ingress controller.

The most famous used nginx controll is : Nginx implementation of Ingress Controller, will run on kube-system.

Ingress includes a feature called the "default backend," which handles all incoming requests that do not match any defined backend rules. You can customize this behavior by creating your own pod to manage these unmatched requests according to your specific requirements.

you can also manager multiple path, each path will direct traffic to specif service

```yaml
spec:
  rules:
  - host: myapp.com #routing rules
    http: #is not the http on webbrowser it is how the request is forwarded to svc.
      paths:
      - path: /
        backend:
          serviceName: myapp-internal-service
          serviceport: 8080
      - path: /analytics:
        backend:
          serviceName: analtycis-service
          serviceport: 3000
      - path: /shopping
        backend:
          serviceName: shopping-service
          serviceport: 4000
```

Configuration TLS https certification:

```yaml
spec:
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-secret-tls 
  rules:
  - host: myapp.com #routing rules
    http: #is not the http on webbrowser it is how the request is forwarded to svc.
      paths:
      - path: /
        backend:
          serviceName: myapp-internal-service
          serviceport: 8080
    
```

secret:
```yaml
kind: Secret
metadata:
  name: myapp-secret-tls
  namespace: default
data:
  tls.crt: base64 encoded crt
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

### Kubernetes IP Address Allocation

- **Pod IPs**: Assigned by the network plugin (e.g., Calico, Flannel, Cilium).
- **Service IPs**: Assigned by the Kubernetes API Server.
- **Node IPs**: Assigned by the Kubelet or the Cloud Controller Manager, depending on the environment.


### CRI: Container Runtime Interface
The **Container Runtime Interface (CRI)** is the API layer that Kubernetes uses to communicate with container runtimes. It abstracts the underlying container engine, enabling Kubernetes to manage containers seamlessly, regardless of the runtime being used (e.g., containerd, CRI-O, Docker).

### CNI: Container Network Interface
The **Container Network Interface (CNI)** is a specification and a set of libraries designed to manage container networking. It provides a standard framework for writing plugins that configure network interfaces for containers. The CNI specification is language-agnostic, ensuring flexibility and compatibility across different environments and programming languages.

example of CNI:

```json
{
  "cniVersion": "0.2.0",
  "name": "mynet",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.22.0.0/16",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ] 
  }
}
```

This configuration defines a standard Linux bridge named cni0, which will give ip addresses in the subnet 10.22.0.0/16, the bridge plugin will configure the network interface in the correct namespaces to define the container network properly.

A CNI plugin can assign an IP address to a single Pod, but it does not inherently handle Pod-to-Pod communication across nodes. Kubernetes networking requires the following conditions to be met:

- All Pods must be able to communicate with each other across nodes within the cluster.
- All nodes must be able to communicate with all Pods.
- No Network Address Translation (NAT) should be required for communication.

  
All Node and Pod IPs must be routable without NAT. This can be achieved via physical network configuration, GKE (Google Kubernetes Engine), or software-defined overlays like:

- Cilium
- Flannel
- Calico

Cilium : open-source solution for networking, observability, and security in Kubernetes, built on eBPF—a Linux kernel tech that runs safe, efficient code inside the kernel:

- Networking between containers, pods, and services
- Security policies based on identity (not just IPs)
- Observability and tracing of network traffic
- Load balancing for services
- Service mesh features without sidecars (via Cilium Service Mesh)

![cilium](cilium.webp)

Both MESOS and Kubernetes share a similar architecture: a central manager exposes an API, a scheduler distributes workloads across workers, and a persistent layer stores cluster states—ETCD for Kubernetes and ZooKeeper for MESOS.

ETC Quorum or ZooKeeper Quorum, is the minmum numbers of nodes in a distrubuted system , that must agree for a decision to be valid, ensure consistency and fault tolrence by using algo like RAFT and PAXOS.

In Kubernetes etcd, the quorum plays a critical role in ensuring leader election and maintaining data consistency. To achieve consensus, a majority of nodes, calculated as (N/2) + 1, must be operational. For example, in a 5-node etcd cluster, at least 3 nodes must be available to uphold leader election and ensure consistent data replication.

----
# Node management

----
etcd management
----

Check the data directory where the snapshot will be saved. It is likely mapped to the etcd container image. You can verify this by inspecting the `etcd.yaml` file:

```sh
grep data-dir /etc/kubernetes/manifests/etcd.yaml
```

exec into the pod of etc in kube-system
```sh
kubectl -n kube-system exec -it etcd-cp -- sh
```
inside we can check help for etcdctl
```sh
etcdctl -h
```
all cert needed for command execution are inside :
```sh
cd /etc/kubernetes/pki/etcd/
echo * 
ca.crt ca.key healthcheck-client.crt healthcheck-client.key peer.crt peer.key server.crt server.key
```
since the container image of etcd is being as small as possible it is hard to work direclty inside the container, thus we will export some env variable and work as follow :

checking health :
```sh
kubectl -n kube-system exec -it etcd-cp -- sh -c "ETCDCTL_API=3 ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key etcdctl endpoint health"
```

checking quorum (we only have 1 node for the lab, normalement it should be 3 or 5 in prod)
```sh
kubectl -n kube-system exec -it etcd-cp -- sh -c "ETCDCTL_API=3 ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key etcdctl --endpoints=https://127.0.0.1:2379 member list"
```

output as a table:
```sh
kubectl -n kube-system exec -it etcd-cp -- sh -c "ETCDCTL_API=3 ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key etcdctl --endpoints=https://127.0.0.1:2379 member list -w table"
```

lets backup the database now :
```sh
kubectl -n kube-system exec -it etcd-cp -- sh -c "ETCDCTL_API=3 ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key etcdctl --endpoints=https://127.0.0.1:2379 snapshot save /var/lib/etcd/snapshot.db"

.....
{"level":"info","ts":"2025-03-20T16:14:19.022733Z","caller":"snapshot/v3_snapshot.go:88","msg":"fetched snapshot","endpoint":"https://127.0.0.1:2379","size":"4.3 MB","took":"now"}
{"level":"info","ts":"2025-03-20T16:14:19.023466Z","caller":"snapshot/v3_snapshot.go:97","msg":"saved","path":"/var/lib/etcd/snapshot.db"}
Snapshot saved at /var/lib/etcd/snapshot.db
```

let us now create a folder called backup and store the most important files to recover our cluster including the database snapshot:

```sh
mkdir etcd_backup
cp /var/lib/etcd/snapshot.db ./etcd_backup/snapshot.db-$(date +%m-%d-%y)
cp /root/kubeadm-config.yaml ./etcd_backup/
cp -r /etc/kubernetes/pki/etcd/ ./etcd_backup/

```

----
upgrade the cluster
----

**DaemonSet**: Ensures a specific pod runs on every node or selected nodes and is auto-scheduled. 

**`--ignore-daemonsets`**: Used to evict all pods except DaemonSet pods when draining a node.

**Common DaemonSet Use Cases**:
- **Monitoring**: Prometheus, Fluentd
- **Networking**: kube-proxy, Calico, Cilium
- **Storage**: Ceph, GlusterFS
- **Security & Logging**: Falco, Sysdig

Starting update, first apt update:
```sh
apt update
```

then change version on kubernetes apt list:
```sh
sed -i 's/31/32/g' /etc/apt/sources.list.d/kubernetes.list 
apt update
```

check upgradble packages:
```sh
apt list --upgradble
apt-cache madison kubeadm
```

unhold kubeadm package
```sh
apt-mark unhold kubeadm
```

install new kubeadm version
```sh
apt-get install -y kubeadm=1.32.1-1.1
```

hold again
```sh
apt-mark hold kubeadm
```

check version
```sh
kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"32", GitVersion:"v1.32.1", GitCommit:"e9c9be4007d1664e68796af02b8978640d2c1b26", GitTreeState:"clean", BuildDate:"2025-01-15T14:39:14Z", GoVersion:"go1.23.4", Compiler:"gc", Platform:"linux/amd64"}
```

drain the cp node from cluster with ignore daemonset:
```sh
kubectl drain cp --ignore-daemonsets 
node/cp cordoned
Warning: ignoring DaemonSet-managed Pods: kube-system/cilium-envoy-2xs9m, kube-system/cilium-h9s45, kube-system/kube-proxy-ff8xw
evicting pod kube-system/coredns-7c65d6cfc9-cb595
evicting pod kube-system/cilium-operator-5c7867ccd5-p2pcv
evicting pod kube-system/coredns-7c65d6cfc9-5gln7
pod/cilium-operator-5c7867ccd5-p2pcv evicted
pod/coredns-7c65d6cfc9-5gln7 evicted
pod/coredns-7c65d6cfc9-cb595 evicted
node/cp drained
```

dry run the upgrade and check availibity and suggestion (**only works on CP**) :
```sh
kubeadm upgrade plan
Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE      CURRENT   TARGET
kubelet     cp        v1.31.1   v1.32.3
kubelet     dp        v1.31.1   v1.32.3

Upgrade to the latest stable version:

COMPONENT                 NODE      CURRENT    TARGET
kube-apiserver            cp        v1.31.1    v1.32.3
kube-controller-manager   cp        v1.31.1    v1.32.3
kube-scheduler            cp        v1.31.1    v1.32.3
kube-proxy                          1.31.1     v1.32.3
CoreDNS                             v1.11.3    v1.11.3
etcd                      cp        3.5.15-0   3.5.16-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.32.3

Note: Before you can perform this upgrade, you have to update kubeadm to v1.32.3.

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
```

upgrade the cp node:
```sh
kubeadm upgrade apply v1.32.1
....
[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Renewing apiserver certificate
[upgrade/staticpods] Renewing apiserver-kubelet-client certificate
[upgrade/staticpods] Renewing front-proxy-client certificate
[upgrade/staticpods] Renewing apiserver-etcd-client certificate
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-03-21-02-06-12/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 1 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Renewing controller-manager.conf certificate
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-03-21-02-06-12/kube-controller-manager.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 1 Pods for label selector component=kube-controller-manager
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Renewing scheduler.conf certificate
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-03-21-02-06-12/kube-scheduler.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 1 Pods for label selector component=kube-scheduler
....
```

check nodes, the cp node is `Read,SchedulingDisabled` and we still have the old version since we didnt restart the daemonsets yet:
```sh
k get node
NAME   STATUS                     ROLES           AGE     VERSION
cp     Ready,SchedulingDisabled   control-plane   9d      v1.31.1
dp     Ready                      <none>          5d10h   v1.31.1
```

unmark kubelet and kubectl to upgrade them also :
```sh
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.32.1-1.1 kubectl=1.32.1-1.1
kubelet --version
kubectl version
apt-mark hold kubelet kubectl
```

restart the daemonset and kubelet
```sh
systemctl daemon-reload
systemctl restart kubelet
```

uncordon the cp and put back inside the cluster
```sh
kubectl uncordon cp
```
now check the nodes, we should have the right version now :
```sh
k get node
NAME   STATUS   ROLES           AGE     VERSION
cp     Ready    control-plane   9d      v1.32.1
dp     Ready    <none>          5d11h   v1.31.1
```


----
Updating worker
----

```sh
kubectl get ns
apt-get unhold kubeadm
apt-mark unhold kubeadm
sudo sed -i 's/31/32/g' /etc/apt/sources.list.d/kubernetes.list 
apt-get update
apt-cache madison kubeadm
apt-get update && apt-get install -y kubeadm=1.32.1-1.1
apt-mark hold kubeadm
#troubleshooting apiserver from node
kubectl get pods -n kube-system | grep apiserver
kubectl logs -n kube-system -l component=kube-apiserver
kubectl cluster-info
systemctl restart kubelet
kubectl -n kube-system get cm kubeadm-config -o yaml | grep controlPlaneEndpoint
kubeadm certs renew all
nslookup k8scp:6443
nslookup k8scp
curl -k https://k8scp:6443/healthz
# finish troubleshoting and understood plan not working on worker node
kubectl get nodes
kubeadm upgrade node
apt-mark unhold kubelet kubeadm
apt-get install -y kubelet=1.32.1-1.1 kubeadm=1.32.1-1.1
apt-mark hold kubelet kubeadm
systemctl daemon-reload
systemctl restart kubelet
kubectl get nodes
```


----
CPU and Memory Contraintes.
----

FYI : **containerd use docker.io as a registry** by default unless specified otherwise in `/etc/containerd/config.toml`

in this tp we will use 3 terminals, one for managing the hog application with stress image and 2 for top to check memory ussage with `alt+m` in cp and dp.

first we create the hog app deploy:
```sh
kubectl create deployment hog --image vish/stress
```

there is no CPU or memory limits on our POD :
```sh
k describe deploy hog
k get deployments.apps hog -o yaml
```

lets modify our deploy:
```sh
k get deployements.apps hog -o yaml > hog.yaml
```
be sure to delete unecessary fields like status or creatin stamps.

by first adding cpu and memory limit:

```yaml
    spec:
      containers:
        - image: vish/stress
          imagePullPolicy: Always
          name: stress
          resources:
            limits:
              cpu: "1"
              memory: "4Gi"
            requests:
              cpu: "0.5"
              memory: "500Mi"
```

then we replace the existing deploy:
```sh
k replace -f hog.yaml
```
Now by passing some parameter to the image by using args , we will ask stress to consume more resource:


```sh
          resources:
            limits:
              cpu: "1"
              memory: "4Gi"
            requests:
              cpu: "0.5"
              memory: "500Mi"
          args:
            - -cpus
            - "2"
            - -mem-total
            - "950Mi"
            - -mem-alloc-size
            - "100Mi"
            - -mem-alloc-sleep
            - "1s"
```

if its not working you can check k logs and adjust the indentation of args.

Now if you check top on cp and work nodes you will have more Memory and cpu consumption.


----
namespace
----

Now we will set LimitRange in a namespace.
First we create a namespace:
```sh
k create namespace low-usage-limit
k get ns
```

we create a limitrange yaml file:
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: low-resource-range
spec:
  limits:
    - default:
        memory: 500Mi
        cpu: "0.5"
      defaultRequest:
        memory: 500Mi
        cpu: "0.5"
      type: Container
```

then we apply in the ns
```sh
k -n low-usage-limit apply -f low-resource-range.yaml
k get limitranges -A
k -n low-usage-limit describe limitrange low-resource-range 
```

we then create a new hog deploy, he will unherit the limitrange automatically
```sh
k -n low-usage-limit create deploy limited-hog --image vish/stress
k get pod -A
k -n low-usage-limit get pod limited-hog -o yaml
```
we will see that it inherited the limits

we then create a copy of first hog deploy with param args and create it inside the namespace with limitrange:
```sh
cp hog.yaml hog2.yaml
```

we then change the namespace and delete any selflink found
```yaml
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  generation: 1
  labels:
    app: hog
  name: hog
  namespace: low-usage-limit
```

and we create it
```sh
k create -f hog2.yaml
k get pod -A
```
we wait for pod creation, now we have, 3 hog application, one inside default and schedulded in worker node, and 2 inside low-usage-limit but since the worker is fully used with hog1 , hog2 that uses same resource will be schedulded on CP:
```sh
NAMESPACE         NAME                               READY   STATUS              RESTARTS      AGE
default           hog-ff4f4d4fd-mq2rh                1/1     Running             0             13m
low-usage-limit   hog-ff4f4d4fd-cvbff                0/1     ContainerCreating   0             4s
low-usage-limit   limited-hog-5bc984c8bf-88ndz       1/1     Running             0             5m36s
```

If we check the system's resource usage using `top`, we will observe that both `hog1` and `hog2` are consuming the maximum allocated resources on the worker and control plane nodes. This behavior occurs because per-deployment resource settings take precedence over (have prio) the global `LimitRange` defined at the namespace level.


**=> So deploy setting > namespace settings , namespace here is like a default value, that can be overriding by deploy manifest.**


----
API Access
----

The entire architecture of kube is API-Driven, this is managed by kube-apiserver , curling the API server can expose current API groups, this groups may have multiple versions which evolve independtly and follow a domain-name format with several name reserverd such as single-word domains, the empty group and a lot of sub-domaine ending in .k8s.io.


One of the roles of kubectl is making this api calls on your behalfs and also responding to requests from kube-apiserver (HTTPverbs, GET, POST, DELETE)

You can bypass kubectl and make this calls by yourself using the right url, certificates and keys, example :

```sh
$ curl --cert userbob.pem --key userbob-key.pem --cacert /path/to/ca.pem https://k8sServer:6443/api/v1/pods
```

You can impersonate other users or groups based on RBAC settings. This is useful for debugging authorization and security policies for different users.

here are some example:

```sh
kubectl auth can-i create deploy
yes
kubectl auth can-i create deploy --as omarbistami
no
kubectl auth can-i create deploy --as omarbistami --namespace developper
no
```

### Access Control APIs for Querying Permissions

Kubernetes provides three APIs to determine who can perform specific actions and what resources can be queried:

1. **SelfSubjectAccessReview**  
  - Allows a user to review their own access permissions.
  - Useful for delegating access checks to other users.

2. **LocalSubjectAccessReview**  
  - Restricts the access review to a specific namespace.
  - Ideal for namespace-scoped permission checks.

3. **SelfSubjectRulesReview**  
  - Displays the actions a user is allowed to perform within a particular namespace.
  - Helps users understand their permissions in a given context.


**reconcile:**
```sh
kubectl auth reconcile -f <file.yaml>
```
This command checks if the permissions required to create or update the resources defined in `<file.yaml>` are in place. If no output is returned, it means the operation would be allowed.


***The default serilization for API calls must be JSON, all yaml file are convert from and to JSON.***

Kubernetes uses the `resourceVersion` value (associated with resources like Pods, Deployments, etc.) to track API updates and implement **Optimistic Concurrency Control (OCC)**. OCC is a method used to handle concurrent updates to a resource without locking it. Here's how it works:

1. **Read the Resource**: Retrieve the current state of the resource (e.g., from the database or Kubernetes API).
2. **Modify in Memory**: Make the necessary changes to the resource locally.
3. **Attempt to Save**: Try to save the updated resource back to the system.
4. **Check for Conflicts**:
  - **No Conflict**: If the `resourceVersion` matches, commit the changes successfully.
  - **Conflict Detected**: If the `resourceVersion` has changed (indicating another update occurred), the operation fails. You must retry the process by fetching the latest state of the resource.

This approach ensures consistency without requiring locks, making it efficient for distributed systems like Kubernetes.

409 CONFLICT Error: The object has been modified. Resolve by fetching the latest state (`kubectl get -o`), merging changes, and reapplying.

`resourceVersion` is derived from the `modifiedIndex` in ETCD. It is unique per namespace, kind, and server. Operations like `WATCH` and `GET` do not update it as they don't modify the object.

----
Annotation & Labels
----
Both Labels and Annotations are a key-value pair meta-data for k8s objects

Lables role is to identify resource, can help grouping, filtering and querying resource and it is used by Selectors in deploy,svc for example:

```yaml
metadata:
  labels:
    app: my-app
    env: production
```

querying:
```sh
kubectl get pods -l app=my-app
```

Annotation on the other hand, store non-identifying metadata information (description, timestamp, tracking info, etc) but can store rich info.

```yaml
metadata:
  annotations:
    description: "This pod runs my production app"
    owner: "team-devops"
```

example of usage:
```sh
kubectl annotate pods --all description="Production Pods" -n prod
kubectl annotate --overwride pod webpod description="Old Production pod" -n prod
kubectl -n prod annotate pod webpod description-
```

----
Simple POD
----
The smallest unit in Kubernetes, a Pod, typically contains one primary application container and optional supporting containers(logging for example).

example :

```yaml
apiVersion: v1
kind: pod
metadata:
  name: firstpod
spec:
  containers:
  - image: nginx
    name: stan
```

exemple of kubectl:
```sh
kubectl create -f simple.yaml
kubectl get pods
```

kubectl verbosity:
```sh
kubectl --v=10 get pods firstpod.
```

Get the cluster config view in case you want to curl the cluster from outisde:

```sh
kubectl config view
```
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://k8scp:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```

Here you can see cerfiticate and key are omitted for security reason, but if you verbose the command you will see that it gets it information from : .kube/config

- **apiVersion**: Specifies the API version for the Kubernetes server to process the data.
- **cluster**: Defines the cluster name, API server endpoint, and certificate authority for authentication.
- **context**: Simplifies access to multiple clusters and users by setting namespace, user, and cluster configurations.
- **current-context**: Indicates the active context used by `kubectl`.
- **kind**: Represents the type of Kubernetes object.
- **preferences**: Reserved for future use, such as customizing output (e.g., colorization).
- **users**: Maps nicknames to client credentials, including certificates, tokens, or user/password combinations. Can be updated using `kubectl config set-credentials`.

Namespaces refer to both the kernel feature for isolating resources and the logical segregation of API objects within Kubernetes. In Kubernetes, namespaces provide a mechanism to partition resources, enabling multiple teams or projects to share a cluster without interfering with each other. This ensures better organization, access control, and resource management.

Every API call include a namespace, else default is choosed:

https://10.128.0.3:6443/api/v1/**namespaces**/default/pods

**access policies will work on namespace boundaries.**

- **default**: The default namespace for resources unless specified otherwise.  
- **kube-node-lease**: Stores node lease information for heartbeat and node status.  
- **kube-public**: Publicly accessible, even to unauthenticated users, for general information.  
- **kube-system**: Hosts critical infrastructure pods required for Kubernetes operation.  

```sh
kubectl api-resources #will show all availble resources
kubectl api-versions #will show all availble versions
kubectl explain pod #will describe the object
kubectl get raw --raw /apis | jq .
```

these are the same:
```sh
curl --cert /tmp/client.pem --key /tmp.client-key.pem --cacert /tmp/ca.pem -v -XGET https://10.128.3:6443/api/v1/namespaces/default/pods/firstpod/log
####same as
kubeclt logs firstpod
```

### Swagger and OpenAPI

Kubernetes APIs are built using Swagger and OpenAPI specifications, ensuring consistency and compatibility.

### API Maturity

Kubernetes employs API groups and versioning (Alpha, Beta, Stable) to enable feature development without disrupting existing APIs. This approach supports multi-team development and feature iteration.

### Curling with Certificates

You can interact with the Kubernetes API using certificates from the kubeconfig file:
```sh
curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem https://k8scp:6443/api/v1/pods
```

```sh
export client=$(grep client-cert $HOME/.kube/config | cut -d" " -f 6)
export key=$(grep client-key-data $HOME/.kube/config | cut -d" " -f 6)
export auth=$(grep certificate-authority-data $HOME/.kube/config | cut -d" " -f 6)

echo $client | base64 -d - > ./client.pem
echo $key | base64 -d - > ./client-key.pem
echo $auth | base64 -d - > ./ca.pem


curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem https://k8scp:6443/api/v1/pods

```
lets try to create a pod using json file :

```sh
cat curlpod.json 
```
```json
{
    "kind": "Pod",
    "apiVersion": "v1",
    "metadata":{
        "name": "curlpod",
        "namespace": "default",
        "labels": {
            "name": "examplepod"
        }
    },
    "spec": {
        "containers": [{
            "name": "nginx",
            "image": "nginx",
            "ports": [{"containerPort": 80}]
        }]
    }
}
````
lets run this :

```sh
```sh
curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem https://k8scp:6443/api/v1/namespaces/default/pods -X POST -H 'Content-Type: application/json' -d @curlpod.json
```



You cant get more info about the api inside :
```sh
cd ./kube/cache/discovery/k8scp_6443
find .
python3 -m json.tool discovery/k8scp_6443/v1/serverresources.json | grep kind
python3 -m json.tool discovery/k8scp_6443/apps/v1/serverresources.json | grep kind
```

----
Other APIs
----

- **v1**: Stable API version.
- **Node**: A cluster machine that can be drained or undrained.
- **Service Account**: Identity for pods to interact with the Kubernetes API server.
- **Resource Quota**: Limits resources, e.g., a namespace can run only 6 pods.
- **Endpoint**: A set of IPs matching a service, managed automatically.
- **Deployment**: Manages ReplicaSets, simplifying upgrades and administration.
- **ReplicaSet**: Oversees pod lifecycle and updates.
- **Pod**: The smallest deployable unit in Kubernetes.
- **DaemonSet**: Ensures a pod runs on every node (e.g., for logging or security).
- **StatefulSet**: Manages unique pods with stable storage, network, and ordering (e.g., app-0, app-1).

----
Autoscaling
----

**Horizontal Pod Autoscaler (HPA)**: Automatically scales the number of pods in a Deployment, StatefulSet, or ReplicaSet based on metrics like CPU or memory usage. By default, it triggers scaling at 80% CPU utilization. Metrics are collected by the metrics-server every minute, and the HPA controller checks every 15 seconds. Pods are added immediately when needed, but removal waits for 300 seconds.

**Cluster Autoscaler (CA)**: Dynamically adds or removes nodes based on pod scheduling needs or low node utilization (idle for at least 10 minutes). It optimizes resource allocation and reduces costs, especially in cloud environments.

**Vertical Pod Autoscaler (VPA)**: Adjusts CPU and memory requests/limits for pods based on actual usage, unlike HPA, which scales the number of pods. VPA ensures optimal resource allocation for individual pods.


----
BATCH API
----

**Jobs**: Part of the Batch API, Jobs ensure a specified number of pods run to completion. If a pod fails, it restarts until the required completions are achieved. Key fields include `.spec.parallelism` (number of pods running concurrently) and `.spec.completions` (successful pods needed to complete the job). Defaults are set to 1 if omitted.

**Example of a Kubernetes Job**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: example-job
spec:
  template:
    spec:
      containers:
      - name: example-container
        image: busybox
        command: ["sh", "-c", "echo Hello, Kubernetes! && sleep 5"]
      restartPolicy: Never
  backoffLimit: 4
```
This Job will print "Hello, Kubernetes!" and then sleep for 5 seconds before completing.

**CronJobs**: Similar to Linux cron jobs, they schedule tasks using the same time syntax.

**Example of a Kubernetes CronJob**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: example-cronjob
spec:
  schedule: "*/5 * * * *" # Runs every 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: example-container
            image: busybox
            command: ["sh", "-c", "echo 'Hello, Kubernetes!' && sleep 10"]
          restartPolicy: OnFailure
```

This CronJob runs every 5 minutes, prints "Hello, Kubernetes!" to the logs, and then sleeps for 10 seconds before completing.

`.spec.concurrencyPolicy` controls overlapping job execution:
- **Allow**: Runs jobs concurrently.
- **Forbid**: Waits for current jobs to finish before starting new ones.
- **Replace**: Stops current jobs to start new ones immediately.

----
RBAC
----

**RBAC**: Role-Based Access Control. 

- **ClusterRole**: A ClusterRole defines a set of permissions (verbs, resources, and resource names) that apply across the entire cluster. It is not namespace-specific and is used for granting access to cluster-wide resources like nodes, persistent volumes, or custom resource definitions (CRDs). ClusterRoles can be bound to users, groups, or service accounts using a ClusterRoleBinding.

  Example:
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: cluster-admin-role
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  ```

- **Role**: A Role is similar to a ClusterRole but is namespace-specific. It defines permissions for resources within a particular namespace. Roles are bound to users, groups, or service accounts using a RoleBinding.

  Example:
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    namespace: default
    name: namespace-admin-role
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "delete", "update"]
  ```
  
- ClusterRoleBinding: A `ClusterRoleBinding` is a Kubernetes resource that grants permissions defined in a `ClusterRole` to a user, group, or service account across the entire cluster. It is used when you need to provide cluster-wide access to resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
- kind: User
  name: admin-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```
This example binds the `cluster-admin` ClusterRole to a user named `admin-user`, granting them cluster-wide administrative privileges.

- RoleBinding: A `RoleBinding` is a Kubernetes resource that grants permissions defined in a `Role` to a user, group, or service account within a specific namespace. It is used when you want to limit access to resources within a single namespace.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: namespace-admin-binding
  namespace: default
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-admin-role
  apiGroup: rbac.authorization.k8s.io
```
This example binds the `namespace-admin-role` Role to a user named `dev-user`, granting them permissions within the `default` namespace.

in this tp we use curl with a token we generate to make API request.

First we get the IP/Hostname and port of the node runing a replica of API-server in our case it is the CP:
```sh
k get pod -A -o wide | grep apiserver
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE   IP             NODE   NOMINATED NODE   READINESS GATES
kube-system   kube-apiserver-cp                  1/1     Running   0             23d   10.2.0.2       cp     <none>           <none>
```

view kube config :
```sh
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://k8scp:6443
  name: kubernetes
.....
```

Now we generate a token and stock in a var:
```sh
echo $token
eyJhbGciOiJSUzI1NiIsImtpZCI6IlZka3k4VGZ6MXdaQUlOTXIzOEN6UklZWHlVUWVjMzNQN1I1djhYZjRTYTgifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzQ0NTY4MDMwLCJpYXQiOjE3NDQ1NjQ0MzAsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiYjg4YjY2ODAtZGFhZC00NjNhLWJlMTEtZjA0MTllYzU5NGFlIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRlZmF1bHQiLCJ1aWQiOiI1MTgzMjZkNS1mZTg2LTQ4MzEtYjBlMC1jMTI2MTllNmQzYjcifX0sIm5iZiI6MTc0NDU2NDQzMCwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6ZGVmYXVsdCJ9.Uj2wIyih52wikM3Pzq5wmQShg6L5liybVF2DOHEwhiMWqtE5LmGrmZPjDyNKfnHfQxLgebsuc0imJHmscKsGft-BNI_O2Fl_XMI8mFWDIdXTsij_ieaSq5J0g-tWH-_DEIJYWsbc6_JFlDAJmGhUEENVCrmgigNdfszDd4Zyl-gP3Z0dNwxvNYWib2vMBRJch0RVVDO99ZDm2b7F8rswjzXdegY7hv7IGqE7ngvUkK5-V031WTp9SFUvs7zKxM_tDtg6DlUyj7jpjvui-ybwnNsR2ux-0CO0OPCt6LOau33BdmbnU67_sg0jJ1_A4n7-KD-BTXTAzPmUS3jliuXcOg
export token=$(kubectl create token default)
```

we curl using this token
```sh
curl https://k8scp:6443/apis --header "Authorization: Bearer $token" -k
```
```json
{
  "kind": "APIGroupList",
  "apiVersion": "v1",
  "groups": [
    {
      "name": "apiregistration.k8s.io",
      "versions": [
        {
          "groupVersion": "apiregistration.k8s.io/v1",
          "version": "v1"
        }
      ],
```

now we targed version 1 (api instead of apis)
```sh
curl https://k8scp:6443/api/v1 --header "Authorization: Bearer $token" -k
```
```json
{
  "kind": "APIResourceList",
  "groupVersion": "v1",
  "resources": [
    {
      "name": "bindings",
      "singularName": "binding",
      "namespaced": true,
      "kind": "Binding",
      "verbs": [
        "create"
      ]
    },
    {
      "name": "componentstatuses",
      "singularName": "componentstatus",
      "namespaced": false,
      "kind": "ComponentStatus",
      "verbs": [
        "get",
        "list"
      ],
      "shortNames": [
        "cs"
      ]
    },
    {
      "name": "configmaps",
      "singularName": "configmap",
      "namespaced": true,
      "kind": "ConfigMap",
      "verbs": [
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "shortNames": [
        "cm"
      ],
      "storageVersionHash": "qFsyl6wFWjQ="
    },
```

now lets try to get namespace and see what we get:
```sh
curl https://k8scp:6443/api/v1/namespaces --header "Authorization: Bearer $token" -k
```
```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "namespaces is forbidden: User \"system:serviceaccount:default:default\" cannot list resource \"namespaces\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "namespaces"
  },
  "code": 403
}
```

As you can we do not have the authorization to manage this. (missing RBAC to list namespace).

