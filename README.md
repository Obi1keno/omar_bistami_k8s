# LFOmar

my LF project doc

----

Installaion of kube explained
----

**Using google Kubernetes Engine GKE**

```sh
gcloud container clusters create linuxfoundation
gcloud container clusters list
kubectl get nodes`
```

delete:

`$ gcloud container clusters delete linuxfoundation`

**Using minikube**

```sh
curl -Lo minikube ht‌tps://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin
minikube start
kubectl get nodes
```

**Using kubeadm**

```sh
kubeadm init
kubeadm join
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

using in navigator : <http://34.155.199.133:32257>

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

CPU and Memory Contraintes
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
kind: Deployment # kind specifies the type of Kubernetes resource, such as Deployment, Service, Pod, etc.
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
curl --cert userbob.pem --key userbob-key.pem --cacert /path/to/ca.pem https://k8sServer:6443/api/v1/pods
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

<https://10.128.0.3:6443/api/v1/**namespaces**/default/pods>

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

We can also interact with the Kubernetes API using a proxy. The proxy can be initiated from a node or within a pod. When running within a pod, this is often achieved through the use of a sidecar container, which acts as an intermediary to facilitate API communication. This approach is particularly useful for debugging or securely accessing the API server without exposing it directly.

```sh
k proxy -h
k proxy --api-prefix=/ &
[1] 3820489
Starting to serve on 127.0.0.1:8001
```

curl through proxy

```sh
curl http://127.0.0.1:8001/api
```

curl namespaces

```sh
curl http://127.0.0.1:8001/api/v1/namespaces
```

----

Jobs example
----

we will create a simple job:

cp /home/omarbistami/LFCourse/LFS258/SOLUTIONS/s_06/job.yaml .

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sleepy
spec:
  template:
    spec:
      containers:
      - name: resting
        image: busybox
        command: ["/bin/sleep"]
        args: ["3"]
      restartPolicy: Never
```

```sh
kubectl create -f job.yaml
kubectl get job
kubectl describe job sleepy
kubectl delete job sleepy
```

now we modify the job and add completion param.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sleepy
spec:
  completions: 5
  template:
    spec:
      containers:
      - name: resting
        image: busybox
        command: ["/bin/sleep"]
        args: ["3"]
      restartPolicy: Never
```

```sh
k get job
NAME     STATUS    COMPLETIONS   DURATION   AGE
sleepy   Running   4/5           31s        31s

k get pod
NAME           READY   STATUS      RESTARTS   AGE
sleepy-2jch6   0/1     Completed   0          34s
sleepy-96glk   0/1     Completed   0          26s
sleepy-rsjvg   0/1     Completed   0          19s
sleepy-t2679   1/1     Running     0          5s
sleepy-x2bjl   0/1     Completed   0          12s

k delete pod sleepy
```

we can also add parallelism :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sleepy
spec:
  completions: 5
  parallelism: 2
  template:
    spec:
      containers:
      - name: resting
        image: busybox
        command: ["/bin/sleep"]
        args: ["3"]
      restartPolicy: Never
```

jobs will run pods in parallel :

```sh
NAME           READY   STATUS      RESTARTS   AGE
sleepy-6grs2   0/1     Completed   0          12s
sleepy-lbh45   1/1     Running     0          4s
sleepy-qc2xn   1/1     Running     0          4s
sleepy-wdh9m   0/1     Completed   0          12s
```

We now add activeDeadlineSeconds which determine the time limit for the job and all his completions to finish, else k8s, will stop the job and the rest of the completions and will show job as failed:

```yaml
spec:
  completions: 5
  parallelism: 2
  activeDeadlineSeconds: 15
  template:
    spec:
      containers:
        - name: resting
          image: busybox
          command: ["/bin/sleep"]
          args: ["5"]
      restartPolicy: Never
```

after creation the job its stops at 2/5 because we exceeded the time :

```sh
k get job
NAME     STATUS    COMPLETIONS   DURATION   AGE
sleepy   Running   2/5           14s        14s
```

we can see more info :

```sh
k get job sleepy -o yaml
.....
status:
  conditions:
  - lastProbeTime: "2025-04-15T20:02:37Z"
    lastTransitionTime: "2025-04-15T20:02:37Z"
    message: Job was active longer than specified deadline
    reason: DeadlineExceeded
    status: "True"
    type: FailureTarget
  - lastProbeTime: "2025-04-15T20:02:40Z"
    lastTransitionTime: "2025-04-15T20:02:40Z"
    message: Job was active longer than specified deadline
    reason: DeadlineExceeded
    status: "True"
    type: Failed
  failed: 2
  ready: 0
  startTime: "2025-04-15T20:02:22Z"
  succeeded: 2

k describe job sleepy
.....
 Type     Reason            Age    From            Message
  ----     ------            ----   ----            -------
  Normal   SuccessfulCreate  2m17s  job-controller  Created pod: sleepy-p57nt
  Normal   SuccessfulCreate  2m17s  job-controller  Created pod: sleepy-bstls
  Normal   SuccessfulCreate  2m7s   job-controller  Created pod: sleepy-p8tm8
  Normal   SuccessfulCreate  2m6s   job-controller  Created pod: sleepy-rbwd8
  Normal   SuccessfulDelete  2m2s   job-controller  Deleted pod: sleepy-p8tm8
  Normal   SuccessfulDelete  2m2s   job-controller  Deleted pod: sleepy-rbwd8
  Warning  DeadlineExceeded  119s   job-controller  Job was active longer than specified deadline
```

now we will create a cronjob whom will use linux style cronjob syntaxe and each time he need to run will create a job

```yaml
kind: CronJob
metadata:
  name: sleepy
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: resting
            image: busybox
            command: ["/bin/sleep"]
            args: ["5"]
          restartPolicy: Never
```

```sh
k get cronjob
NAME     SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
sleepy   */2 * * * *   <none>     False     0        19s             4m20s

k get job
NAME              STATUS     COMPLETIONS   DURATION   AGE
sleepy-29079776   Complete   1/1           9s         4m45s
sleepy-29079778   Complete   1/1           9s         2m45s
sleepy-29079780   Complete   1/1           9s         45s
```

we then add `activeDeadlineSeconds: 10` and set the sleep to 30 to check the fail

```yaml
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      activeDeadlineSeconds: 10
      template:
        spec:
          containers:
            - name: resting
              image: busybox
              command: ["/bin/sleep"]
              args: ["30"]
          restartPolicy: Never
```

```sh
k get cronjob
NAME     SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
sleepy   */2 * * * *   <none>     False     0        <none>          7s

k get job
NAME              STATUS   COMPLETIONS   DURATION   AGE
sleepy-29079834   Failed   0/1           106s       106s

k get cronjob
NAME     SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
sleepy   */2 * * * *   <none>     False     1        33s             6m50s

k describe sleepy-29079834
...
Events:
  Type     Reason            Age   From            Message
  ----     ------            ----  ----            -------
  Normal   SuccessfulCreate  74s   job-controller  Created pod: sleepy-29079834-cvnsw
  Normal   SuccessfulDelete  64s   job-controller  Deleted pod: sleepy-29079834-cvnsw
  Warning  DeadlineExceeded  41s   job-controller  Job was active longer than specified deadline
```

----
DEPLOYEMENT
____

Deployments are YAML files that define the deployment of pods. They include configurations for controllers that manage ReplicaSets, which in turn manage the pods.

Deployments also manage pod updates, either through a block update (replacing all pods at once) or a rolling update (replacing pods one by one). This allows for rollbacks, as older ReplicaSets are retained.

Labels are used to target specific resources. For example, a 'PROD' label can be used to target all resources in the production environment.

deployement YAML/Manifest composition:

```yaml
# Note: `apiVersion: v1` is used for the list object, while `apps/v1` is used for individual deployment objects.
apiVersion: v1
kind: List
items:
- apiVersion: apps/v1
  kind: Deployment
```

- `apiVersion: v1` specifies the API version for the list object, which is used to group multiple Kubernetes resources.
- `kind: List` indicates that this YAML file contains a collection of Kubernetes objects.
- `items` is the key that holds the list of Kubernetes objects, such as deployments, services, and other resources.
- `apiVersion: apps/v1` is the stable API version for managing deployment objects in Kubernetes.
- `kind: Deployment` specifies that the object is a deployment.

This structure allows you to define and manage multiple resources in a single YAML file.

in the next par we have the metadata:

```yaml
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1" # Tracks the revision history of the deployment
  creationTimestamp: 2024-10-21T13:57:07Z # Indicates when the deployment was created
  generation: 1 # Represents the generation of the deployment, incremented with changes
  labels:
    app: dev-web # Identifies the deployment with a key-value pair for selection
  name: dev-web # Specifies the unique name of the deployment
  namespace: default # Defines the namespace where the deployment resides
  resourceVersion: "774003" # Tracks the version of the deployment for concurrency control
  uid: d52d3a63-e656-11e7-9319-42010a800003 # Unique identifier for the deployment
```

```yaml
spec:                                   # delecation that the following item will configure the object to be created
  replicas: 1                           # Number of desired pod replicas
  selector:                             # Determines which pods are managed by this deployment.
    matchLabels:
      app: dev-web                      # Selects pods with this label
  strategy:                             # Defines how updates are performed (e.g., rolling updates).    
    type: RollingUpdate                 # Use rolling updates for deployments
    rollingUpdate:
      maxSurge: 25%                     # Up to 25% more pods than desired during update => if i have 10 => will have 12 during update
      maxUnavailable: 25%               # Up to 25% of pods can be unavailable during update
  revisionHistoryLimit: 10              # Number of old ReplicaSets to retain for rollback 
  progressDeadlineSeconds: 600          # Time (in seconds) to wait for progress before marking deployment as failed
```

**Key fields explained:**

- `revisionHistoryLimit`: Limits how many previous ReplicaSets are kept for rollback.
- `progressDeadlineSeconds`: Fails the deployment if progress stalls for this duration.

```yaml
template:                                   # Pod template for the deployment that will be passed to replicaset to deploy the container.
  metadata:
    creationTimestamp: null                  # No creation timestamp set for the pod template
    labels:
      app: dev-web                          # Label to identify pods created by this template
  spec:
    containers:                             # following items are for container
    - image: nginx:1.17.7-alpine            # Container image to use (nginx version 1.17.7-alpine)
      imagePullPolicy: IfNotPresent         # Pull image only if not already present on the node
      name: dev-web                        # Name of the container
      resources: {}                        # No resource requests or limits specified
      terminationMessagePath: /dev/termination-log   # Path for container termination messages
      terminationMessagePolicy: File        # Store termination message as a file
    dnsPolicy: ClusterFirst                 # Use cluster DNS for pod name resolution (Determines if DNS queries should go to coredns which is the ClusterDNS or, if set to Default, use the node's DNS resolution configuration.)
    restartPolicy: Always                   # Always restart containers if they exit
    schedulerName: default-scheduler        # Use the default Kubernetes scheduler
    securityContext: {}                     # No specific security context set, Flexible setting to pass one or more security settings, such as SELinux context, AppArmor values, users and UIDs for the containers to use.
    terminationGracePeriodSeconds: 30       # Wait 30 seconds before forcefully terminating the pod
```

The Status Section:

```yaml
status:
  availableReplicas: 2           # Number of replicas (pods) that are available to serve requests (i.e., ready and minimum availability requirements met).
  conditions:
    - lastTransitionTime: "2024-10-21T13:57:07Z"  # The last time the condition transitioned from one status to another.
      lastUpdateTime: "2024-10-21T13:57:07Z"      # The last time this condition was updated.
      message: Deployment has minimum availability.  # Human-readable message indicating details about the condition.
      reason: MinimumReplicasAvailable           # Brief reason for the condition's last transition.
      status: "True"                            # Status of the condition (True, False, Unknown).
      type: Available                           # Type of condition: Available means the deployment has enough available replicas.
    - lastTransitionTime: "2024-10-29T06:00:24Z"
      lastUpdateTime: "2024-10-29T06:00:33Z"
      message: ReplicaSet "test-5f6778868d" has successfully progressed. # Indicates the new ReplicaSet is now serving traffic.
      reason: NewReplicaSetAvailable            # Reason for the condition's last transition.
      status: "True"
      type: Progressing                        # Type: Progressing means the deployment is updating pods as expected.
  observedGeneration: 2                         # Most recent generation observed by the deployment controller (matches .metadata.generation).
  readyReplicas: 2                             # Number of pods ready to serve requests (passed readiness checks).
  replicas: 2                                  # Total number of desired pod replicas as specified in the deployment spec.
  updatedReplicas: 2                           # Number of replicas updated to the latest spec (i.e., running the new pod template).
```

**Key fields explained:**

- `availableReplicas`: Pods available to serve requests (minimum availability met).
- `readyReplicas`: Pods that have passed readiness checks and are ready to receive traffic.
- `replicas`: Desired number of pod replicas as defined in the deployment spec.
- `updatedReplicas`: Pods running the latest deployment spec (after an update/rollout).
- `observedGeneration`: Tracks which version of the deployment spec the status reflects.
- `conditions`: List of status conditions for the deployment, such as `Available` and `Progressing`, each with timestamps, reasons, and messages for troubleshooting and monitoring rollout progress.
This status section provides a quick overview of your deployment's rollout progress, availability, and health.

The API server allow for some configuration to be updated quickly, you can scale your deployment easily with:

```sh
kubectl scale deploy/dev-web --replicas=4
```

But they are some immutable values, that need to edit the object for update, for example, to update the container image (e.g., change the nginx version), edit the deployment:

```sh
kubectl edit deployment dev-web
```

Then modify the image field:

```yaml
containers:
- image: nginx:1.8  # Update to desired version
  name: dev-web
```

Kubernetes will automatically perform a rolling update when the image changes.

Deployement Rollbacks:

When a Deployment is updated, Kubernetes retains the ReplicaSets from previous versions. This enables easy rollback to an earlier revision. Rollbacks work by scaling up the desired ReplicaSet and scaling down the current one. The number of retained previous ReplicaSets can be configured using the `revisionHistoryLimit` field in the Deployment spec.

```sh
# Create a new deployment named 'ghost' using the 'ghost' container image
kubectl create deploy ghost --image=ghost

# Annotate the deployment with a change-cause for better tracking and auditability
kubectl annotate deployment/ghost kubernetes.io/change-cause="kubectl create deploy ghost --image=ghost"

# Retrieve the full YAML definition of the 'ghost' deployment for inspection or documentation
kubectl get deployment ghost -o yaml
```

```yaml
deployment.kubernetes.io/revision: "1" 
kubernetes.io/change-cause: kubectl create deploy ghost --image=ghost
```

Lets set the wrong image version:

```sh
kubectl set image deployement/ghost ghost=ghost:09 --all
kubectl get pods
NAME                    READY  STATUS            RESTARTS  AGE
ghost-2141819201-tcths  0/1    ImagePullBackOff  0         1m​
```

To rollback the change :

```sh
kubectl rollout undo deployement/ghost
kubectl get pods
NAME                    READY  STATUS   RESTARTS  AGE
ghost-3378155678-eq5i6  1/1    Running  0         7s
```

To rollout to a specific previous version, you can use : `--to-revision=2` or directly edit the deployement manifest.

To pause then resume a deployement:

```sh
kubectl rollout pause deployement/ghost
kubectl rollout resume deployement/ghost
```

Please note that you can still do a rolling update on ReplicationControllers with the kubectl rolling-update command, but this is done on the client side. Hence, if you close your client, the rolling update will stop:

### Example

Suppose you have a ReplicationController named `my-app` running version 1 of your app. You want to update it to version 2.

You would run:

```sh
kubectl rolling-update my-app --image=my-app:v2
```

- As long as this command is running in your terminal, Kubernetes will gradually replace old pods with new ones.
- If you close your terminal or lose connection, the update will stop partway through, and you’ll need to restart the process.

**Key point:** Unlike newer Deployment objects (which handle rolling updates on the server side), ReplicationController rolling updates depend on your local `kubectl` session staying active.

-----

DaemonSets
-----

A DaemonSet ensures that exactly one pod runs on each node in the cluster, typically using the same container image. When new nodes join, the DaemonSet automatically schedules a pod on them; when nodes are removed, their pods are cleaned up. This is ideal for deploying system-level services like logging or monitoring agents across all nodes without manual intervention.

There are ways of effecting the kube-scheduler such that some nodes will not run a DaemonSet.

-----

LABELS
-----

Labels are part of metadata and are not standalone API objects. They enable selection and grouping of Kubernetes resources using key-value pairs, regardless of the object type.

As of API version apps/v1, a Deployment's label selector is immutable after it gets created.  
If you need to change the label selector, you must delete the existing Deployment and create a new one with the desired label selector.

When creating a deployement, k8s by default adds :

```yaml
    labels:
        pod-template-hash: "3378155678"
        run: ghost ....
```

to view the labels

```sh
kubectl get pods -l run=ghost
NAME                    READY  STATUS   RESTARTS  AGE
ghost-3378155678-eq5i6  1/1    Running  0         10m
```

```sh
kubectl get pods -L run
NAME                    READY  STATUS   RESTARTS  AGE  RUN
ghost-3378155678-eq5i6  1/1    Running  0         10m  ghost
nginx-3771699605-4v27e  1/1    Running  1         1h   nginx
```

In addition to defining labels on pod and deployement templates, you can also add them on the fly:

```sh
kubectl label pods ghost-3378155678-eq5i6 foo=bar
```

```sh
kubectl get pods --show-labels
NAME                    READY  STATUS   RESTARTS  AGE  LABELS
ghost-3378155678-eq5i6  1/1    Running  0         11m  foo=bar, pod-template-hash=3378155678,run=ghost
```

You can force scheduling **a pod on a specific node** by using `nodeSelector` in the pod definition. The `nodeSelector` matches the specified key-value pair with the labels on nodes, ensuring the pod is scheduled only on nodes that have the matching labels:

```yaml
spec:
    containers:
    - image: nginx
    nodeSelector:
        disktype: ssd
```

-----

REPLICASETS TP
-----

```yaml
First we create a replicaset :
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-one
spec:
  replicas: 2
  selector:
    matchLabels:
      system: ReplicaOne
  template:
    metadata:
      labels:
        system: ReplicaOne
    spec:
      containers:
      - name: nginx
        image: nginx:1.22.1
        ports:
        - containerPort: 80
```

This will spawn 2 pods:

```sh
kubectl get rs

NAME     DESIRED   CURRENT   READY   AGE
rs-one   2         2         0       3s
```sh
kubectl describe rs rs-one

NAME           READY   STATUS    RESTARTS   AGE
rs-one-6kx4x   1/1     Running   0          50s
rs-one-ljkh9   1/1     Running   0          50s
```

```sh
kubectl describe rs rs-one
Name:         rs-one
Namespace:    default
Selector:     system=ReplicaOne
Labels:       <none>
Annotations:  <none>
Replicas:     2 current / 2 desired
Pods Status:  2 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  system=ReplicaOne
  Containers:
   nginx:
    Image:         nginx:1.22.1
    Port:          80/TCP
    Host Port:     0/TCP
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  22s   replicaset-controller  Created pod: rs-one-ljkh9
  Normal  SuccessfulCreate  22s   replicaset-controller  Created pod: rs-one-6kx4x
```

We then delete the rs but separte the pod from it using `--cascade=orphan`

```sh
kubectl delete rs rs-one --cascade=orphan
replicaset.apps "rs-one" deleted
```

```sh
kubectl get rs
No resources found in default namespace.
```

```sh
kubectl get pod
AME           READY   STATUS    RESTARTS   AGE
rs-one-ddj78   1/1     Running   0          30s
rs-one-qr6sv   1/1     Running   0          30s
```

if we recreate the rs, the pods will be automatically attached to it since they use the Selectors:

```sh
kubectl edit pod rs-one-ddj78 -o yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2025-04-17T15:29:30Z"
  generateName: rs-one-
  labels:
    system: ReplicaOne #once we recreate the rs, pod will attach using this
```

now lets edit the labet in one pod :

```sh
kubectl edit pod rs-one-ddj78 -o yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2025-04-17T15:29:30Z"
  generateName: rs-one-
  labels:
    system: IsolatedPod #once we recreate the rs, pod will attach using this
```

the rs will create another pod since now we only have one with `system: ReplicaOne`

```sh
kubectl get pod -L system
NAME           READY   STATUS    RESTARTS   AGE     SYSTEM
rs-one-ddj78   1/1     Running   0          5m45s   IsolatedPod
rs-one-qr6sv   1/1     Running   0          5m45s   ReplicaOne
rs-one-zvkdh   1/1     Running   0          29s     ReplicaOne
```

if we delete the rs, the isolated one stays:

```sh
kubectl delete pod rs-one-ddj78
pod "rs-one-ddj78" deleted
```

```sh
kubectl get pod
NAME           READY   STATUS    RESTARTS   AGE
rs-one-ddj78   1/1     Running   0          6m52s
```

```sh
kubectl delete pod rs-one-ddj78
pod "rs-one-ddj78" deleted
```

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

-----

## Helm and Kustomize

Helm is a package manager for Kubernetes, bundling manifests and deployment logic into reusable, versioned charts. Charts simplify application installation, upgrades, and rollbacks, and support environment-specific customization via values files or CLI. Helm integrates with CI/CD pipelines and has a large ecosystem of community-maintained charts.

A typical Helm chart structure:

```
├── Chart.yaml        # Chart metadata (name, version, etc.)
├── values.yaml       # Configurable values for templating
├── templates/        # Kubernetes resource templates using Go templating.
│   ├── deployment.yaml
│   ├── svc.yaml
│   ├── configmap.yaml
│   ├── secrets.yaml
│   ├── pvc.yaml
│   ├── NOTES.txt
│   └── _helpers.tpl
└── README.md
```

**Templates**

Templates are resource manifests that use Go templating syntax. Variables defined in `values.yaml` are injected into the template when a release is created.

In the MariaDB example below: the database password is stored inside a Kubernetes Secret, while the database configuration is stored in a ConfigMap. Labels are defined in the Secret metadata using the chart name, release name, and other values.

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: {{ template "fullname" . }}
    labels:
        app: {{ template "fullname" . }}
        chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
        release: "{{ .Release.Name }}"
        heritage: "{{ .Release.Service }}"
type: Opaque
data:
    mariadb-root-password: {{ default "" .Values.mariadbRootPassword | b64enc | quote }}
    mariadb-password: {{ default "" .Values.mariadbPassword | b64enc | quote }}
```

**Chart Repositories and Hub**

Repositories are simple HTTP servers that contain an index file and a tarball of all the Charts present: <https://artifacthub.io/> that you can search using the `helm search hub`

```sh
helm search hub redis
```

you can use `helm repo` to interact with Repositories :

```sh
# Add the Bitnami Helm repository
helm repo add bitnami https://charts.bitnami.com/bitnami
```

```sh
# List all configured Helm repositories
helm repo list

NAME      URL
bitnami   https://charts.bitnami.com/bitnami
```

once the repository is availble you can search it:

```sh
helm search repo bitnami
```

**Deploying a Chart**

To deploy a Helm chart, review its README for required resources (like PVs for PVCs), then install with `helm install`.

```sh
helm fetch bitnami/apache --untar
cd apache/
ls

Chart.lock  Chart.yaml  README.md  charts  ci  files  templates  values.schema.json  values.yaml
helm install anotherweb
```

You will be able to list the release, delete it, even upgrade it and roll back.
Also review the deployment output carefully. It provides access details and highlights missing cluster resources, making it a key source for troubleshooting.

**Kustomize**
Kustomize simplifies Kubernetes configuration by letting you define base resources and apply overlays for environment-specific changes, like image tags or labels. This modular approach avoids duplicating YAML and complex templates.

Kustomize dont rely on templating like helm to inject values, its uses startegic merging and patches to adjusts configurations.

Kustomize is now a built-in feature in `kubectl` you can directly call it using for example

```sh
# output the full rendered manifests to stdout — it doesn't apply them.
kubectl kustomize dir

# Apply the generated manifests to the Kubernetes cluster
kubectl apply -f dir
```

The `kustomization.yaml` file is central to Kustomize, defining resources and how to modify them.

Organize files into bases and overlays to reduce repetition and maintain consistency. Bases hold shared configurations, while overlays adjust specifics like replica counts or image tags for different environments.

Use patches for precise changes, such as adding environment variables, and transformers for broader modifications, like applying common labels.

Kustomize also generates resources like ConfigMaps and Secrets dynamically, ideal for creating them from literals or files during deployment.

**Kustomize installation**

```sh
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
cp kustomize /usr/local/bin/
kustomize -h
```

A kustomize directory example:

```sh
kustomize-example/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patch-replicas.yaml
    └── prod/
        ├── kustomization.yaml
        └── patch-image.yaml    
```

`kustomization.yaml` is a key file in Kustomize for managing Kubernetes manifests. It defines resources, applies environment-specific changes, adds prefixes/suffixes, labels, annotations, generates ConfigMaps/Secrets, and applies patches to modify configurations without duplicating files.

-----
HELM KUSTOMIZE TP

Tiller was the server-side component of Helm v2(usually in kube-system namespace). but it was removed due to security issues.
Now Helm v3+ communicate with k8s directly using kubeconfig.

Installing Helm:

```sh
wget https://get.helm.sh/helm-v3.15.2-linux-amd64.tar.gz
tar -xvf helm-v3.15.2-linux-amd64.tar.gz
sudo cp linux-amd64/helm /usr/local/bin/helm
```

get some good starting charts in : <https://github.com/helm/charts/tree/master/stable>

You can get for a list of available databases in the repo using:

```sh
helm search hub database 
URL                                                CHART VERSION             APP VERSION                DESCRIPTION                                       
https://artifacthub.io/packages/helm/osc/database  0.12.0                    0.1.0                      OSC database service Helm Chart                   
https://artifacthub.io/packages/helm/mongodb-he... 1.13.0                                               MongoDB Kubernetes Enterprise Database.           
https://artifacthub.io/packages/helm/lsst-sqre/... 2.1.0                     1.0.0                      Archival database of alerts sent through the al...
.....
```

You can also add external repo often by searching <https://artificathub.io/>:

```sh
#adding ealenn repo
helm repo add ealenn https://ealenn.github.io/charts

helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ealenn" chart repository
...Successfully got an update from the "cilium" chart repository
Update Complete. ⎈Happy Helming!⎈
```

we will now install the tester echo-server in the ealenn repo:

```sh
helm upgrade -i tester ealenn/echo-server --debug

history.go:56: [debug] getting history for release tester
Release "tester" does not exist. Installing it now.
install.go:173: [debug] Original chart version: ""
install.go:190: [debug] CHART PATH: /home/student/.cache/helm/repository/echo-server-0.5.0.tgz
client.go:122: [debug] creating 4 resource(s)
NAME: tester
```

View the chart history (using -a shows all even the failed):

```sh
helm list
NAME   NAMESPACE REVISION UPDATED                                 STATUS   CHART             APP VERSION
tester default   1        2025-04-19 16:36:19.006451824 +0000 UTC deployed echo-server-0.5.0 0.6.0
```

Delete the tester chart:

```sh
helm uninstall tester
release "tester" uninstalled
```

all resources are deleted and helm list shows empty:

```sh
helm list -a
NAME NAMESPACE REVISION UPDATED STATUS CHART APP VERSION
```

The chart is still here compressed:

```sh
find $HOME -name *echo*
/root/.cache/helm/repository/echo-server-0.5.0.tgz
```

lets untar it and check the directory structure:

```sh
cd /root/.cache/helm/repository/ ; tar -xvf echo-server-*
echo-server/Chart.yaml
echo-server/values.yaml
echo-server/templates/_helpers.tpl
echo-server/templates/configmap.yaml
echo-server/templates/deployment.yaml
echo-server/templates/ingress.yaml
echo-server/templates/service.yaml
echo-server/templates/serviceaccount.yaml
echo-server/templates/tests/test-connection.yaml
echo-server/.helmignore
echo-server/README.md
echo-server/README.md.gotmpl
```

Instead of installing directly a chart, you can dll it and modify it before, lets first add a repo

```sh
#adding bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
#updating local repo to take in consideration
helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ealenn" chart repository
...Successfully got an update from the "cilium" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈

#downloading bitnami apache chart locally:
helm fetch bitnami/apache --untar
cd apache
ll
-rw-r--r--  1 root root   376 Apr 19 17:12 .helmignore
-rw-r--r--  1 root root   226 Apr 19 17:12 Chart.lock
-rw-r--r--  1 root root  1100 Apr 19 17:12 Chart.yaml
-rw-r--r--  1 root root 67848 Apr 19 17:12 README.md
drwxr-xr-x  3 root root  4096 Apr 19 17:12 charts/
drwxr-xr-x  3 root root  4096 Apr 19 17:12 files/
drwxr-xr-x  2 root root  4096 Apr 19 17:12 templates/
-rw-r--r--  1 root root  1345 Apr 19 17:12 values.schema.json
-rw-r--r--  1 root root 34035 Apr 19 17:12 values.yaml
```

Now once we are inside the chart dir we can install by providing a name

```sh
helm install omarweb .
NAME: omarweb
LAST DEPLOYED: Sat Apr 19 17:23:28 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: apache
CHART VERSION: 11.3.5
APP VERSION: 2.4.63

** Please be patient while the chart is being deployed **

1. Get the Apache URL by running:

** Please ensure an external IP is associated to the omarweb-apache service before proceeding **
** Watch the status using: kubectl get svc --namespace default -w omarweb-apache **

  export SERVICE_IP=$(kubectl get svc --namespace default omarweb-apache --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
  echo URL            : http://$SERVICE_IP/

WARNING: You did not provide a custom web application. Apache will be deployed with a default page. Check the README section "Deploying your custom web application" in https://github.com/bitnami/charts/blob/main/bitnami/apache/README.md#deploying-a-custom-web-application.

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
```

uninstall:

```sh
#unsintalling omarweb
helm uninstall omarweb
release "omarweb" uninstalled
```

**Kustomize TP**
In order to work with kustomize, we need to create the dir tree

```sh
#creating the directories
mkdir -p myapp/base myapp/overlays/dev myapp/overlays/prod
tree myapp
myapp
├── base
└── overlays
    ├── dev
    └── prod

#we copy the files from the TP to work with them
cp /home/omarbistami/LFCourse/LFS258/SOLUTIONS/s_08/*.yaml-base myapp/base/
cp /home/omarbistami/LFCourse/LFS258/SOLUTIONS/s_08/*.yaml-dev myapp/overlays/dev
cp /home/omarbistami/LFCourse/LFS258/SOLUTIONS/s_08/*.yaml-prod myapp/overlays/prod

#we rename them
cd $HOME/myapp/base && for file in *.yaml-base; do mv "$file" "${file/-base/}"; done
cd $HOME/myapp/overlays/dev && for file in *.yaml-dev; do mv "$file" "${file/-dev/}"; done
cd $HOME/myapp/overlays/prod && for file in *.yaml-prod; do mv "$file" "${file/-prod/}"; done


#final result
tree myapp
myapp
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── dev
    │   ├── deployment-patch.yaml
    │   ├── kustomization.yaml
    │   └── service-patch.yaml
    └── prod
        ├── deployment-patch.yaml
        ├── kustomization.yaml
        └── service-patch.yaml
```

The `kustomize.yaml` file defines the resource to be deployed:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namePrefix: lf-
resources:
- deployment.yaml
- service.yaml
labels:
- includeSelectors: true
  pairs:
    company: linux-foundation
```

Using `kubectl kustomize myapp/base` we can view the full configuration:

```sh
kubectl kustomize myapp/base/
```

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: myapp
    company: linux-foundation
  name: lf-myapp
spec:
  ports:
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: myapp
    company: linux-foundation
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: myapp
    company: linux-foundation
  name: lf-myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      company: linux-foundation
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: myapp
        company: linux-foundation
    spec:
      containers:
      - image: nginx
        name: nginx
```

Adding the dev overlay:

```sh
kubectl kustomize myapp/overlays/dev
```

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: myapp
    company: linux-foundation
  name: lf-myapp
spec:
  ports:
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: myapp
    company: linux-foundation
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: myapp
    company: linux-foundation
  name: lf-myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      company: linux-foundation
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: myapp
        company: linux-foundation
    spec:
      containers:
      - image: nginx
        name: nginx
```

and same for `kubectl kustomize myapp/overlays/prod`

if you want to check the diff between prod and dev:

```sh
kubectl kustomize myapp/overlays/prod > omar_kuztomise_output.yaml
kubectl kustomize myapp/overlays/dev > omar_kuztomise1_output.yaml
diff omar_kuztomise_output.yaml omar_kuztomise1_output.yaml
18c18
<   type: LoadBalancer
---
>   type: NodePort
29c29
<   replicas: 3
---
>   replicas: 2
43c43
<       - image: nginx:1.22
---
>       - image: nginx
```

Now we can apply:

```sh
kubectl apply -k myapp/base
service/lf-myapp created
deployment.apps/lf-myapp created
```

```sh
#we get all resource created with label company=linux-foundation(key value)
kubectl get all -l company=linux-foundation
NAME                            READY   STATUS    RESTARTS   AGE
pod/lf-myapp-5b68c7d779-7q48s   1/1     Running   0          3m51s
pod/lf-myapp-5b68c7d779-p9kkv   1/1     Running   0          3m51s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/lf-myapp   ClusterIP   10.106.24.113   <none>        80/TCP    3m51s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/lf-myapp   2/2     2            2           3m51s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/lf-myapp-5b68c7d779   2         2         2       3m51s
```

we now patch with dev configuration:

```sh
kubectl apply -k myapp/overlays/dev
```

we check:

```sh
kubectl get all -l company=linux-foundation
#configuration has been updated
NAME                            READY   STATUS    RESTARTS   AGE
pod/lf-myapp-5b68c7d779-7q48s   1/1     Running   0          8m25s
pod/lf-myapp-5b68c7d779-p9kkv   1/1     Running   0          8m25s

NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/lf-myapp   NodePort   10.106.24.113   <none>        80:32547/TCP   8m25s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/lf-myapp   2/2     2            2           8m25s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/lf-myapp-5b68c7d779   2         2         2       8m25s
```

Now we clean and delete all resource:

```sh
kubectl delete -k myapp/overlays/dev
service "lf-myapp" deleted
deployment.apps "lf-myapp" deleted
```

we check:

```sh
kubectl get all -l company=linux-foundation
No resources found in default namespace.
```

-----

## Kubernetes Storage: Volumes, Persistent Volumes, Secrets, and ConfigMaps

Kubernetes provides a flexible storage system for managing data, configuration, and secrets. This guide covers the main storage concepts, types, and usage patterns.

---

### 1. Volumes

A **volume** in Kubernetes is a directory, possibly pre-populated, mounted into one or more containers in a Pod. Volumes persist for the Pod’s lifetime, surviving container restarts.

#### Common Volume Types

| Type         | Description                                                                                  |
|--------------|---------------------------------------------------------------------------------------------|
| `emptyDir`   | Temporary storage, deleted when Pod ends                                                    |
| `hostPath`   | Mounts a file/directory from the host node                                                  |
| `configMap`  | Mounts configuration data as files                                                          |
| `secret`     | Mounts sensitive data (e.g., passwords, tokens)                                             |
| `downwardAPI`| Exposes Pod info (labels, annotations)                                                      |
| `projected`  | Combines several volume sources into one                                                    |
| `ephemeral`  | Creates a PVC automatically per Pod (short-lived)                                           |

#### Example: Shared `emptyDir` Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: shared-volume-pod
spec:
    containers:
    - name: alpha
        image: busybox
        volumeMounts:
        - mountPath: /data/alpha
            name: shared-vol
    - name: beta
        image: busybox
        volumeMounts:
        - mountPath: /data/beta
            name: shared-vol
    volumes:
    - name: shared-vol
        emptyDir: {}
```

Both containers can read/write to the same volume. **Note:** Without proper locking, concurrent writes may cause data corruption.

---

### 2. Persistent Volumes (PV) and PersistentVolumeClaims (PVC)

**Persistent Volumes (PV)** are cluster resources abstracting storage from the underlying backend. **PersistentVolumeClaims (PVC)** are requests for storage by users or Pods.

#### Lifecycle Phases

- **Pending**: PVC is waiting for a matching PV.
- **Bound**: PVC is bound to a PV.
- **Released**: PVC deleted, PV not yet reclaimed.
- **Lost**: PV deleted or unavailable.

#### Access Modes

| Mode            | Description                                              |
|-----------------|---------------------------------------------------------|
| ReadWriteOnce   | Mounted read-write by a single node                     |
| ReadOnlyMany    | Mounted read-only by many nodes                         |
| ReadWriteMany   | Mounted read-write by many nodes                        |

![Access Modes](persistent-volume-access-modes.png)

#### Example: Static PV and PVC

```yaml
# Persistent Volume
apiVersion: v1
kind: PersistentVolume
metadata:
    name: pv-local
spec:
    capacity:
        storage: 10Gi
    accessModes:
        - ReadWriteOnce
    hostPath:
        path: "/mnt/data"
---
# Persistent Volume Claim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: myclaim
spec:
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
            storage: 8Gi
```

#### Using PVC in a Pod

```yaml
spec:
    containers:
    - name: app
        image: nginx
        volumeMounts:
        - name: data
            mountPath: /data
    volumes:
    - name: data
        persistentVolumeClaim:
            claimName: myclaim
```

#### Dynamic Provisioning with StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
    name: fast
provisioner: kubernetes.io/gce-pd
parameters:
    type: pd-ssd
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: dynamic-claim
spec:
    storageClassName: fast
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
            storage: 1Gi
```

**Reclaim Policies:**  
- `Retain`: Keeps data for manual recovery  
- `Delete`: Removes storage and object  
- `Recycle`: (Deprecated) Cleans and reuses storage

---

### 3. Container Storage Interface (CSI)

CSI is the modern, extensible way to integrate external storage providers. It replaces in-tree plugins and supports dynamic provisioning.

**Examples:**  
- `pd.csi.storage.gke.io` (Google PD)  
- `ebs.csi.aws.com` (AWS EBS)  
- `disk.csi.azure.com` (Azure Disk)  
- `rook-ceph.rbd.csi.ceph.com` (Ceph)

---

### 4. Secrets

**Secrets** store sensitive data (e.g., passwords, tokens) securely.

#### Creating and Using Secrets

```sh
kubectl create secret generic mysql --from-literal=password=root
```

**As Environment Variable:**

```yaml
env:
- name: MYSQL_ROOT_PASSWORD
    valueFrom:
        secretKeyRef:
            name: mysql
            key: password
```

**As Volume:**

```yaml
volumes:
- name: mysql-secret
    secret:
        secretName: mysql
volumeMounts:
- mountPath: /mysqlpassword
    name: mysql-secret
```

**Encryption:**  
Enable encryption at rest by configuring `EncryptionConfiguration` and updating kube-apiserver.

---

### 5. ConfigMaps

**ConfigMaps** store non-sensitive configuration data as key-value pairs or files.

#### Creating and Using ConfigMaps

```sh
kubectl create configmap app-config --from-literal=APP_ENV=production
```

**As Environment Variable:**

```yaml
envFrom:
- configMapRef:
        name: app-config
```

**As Volume:**

```yaml
volumes:
- name: config-volume
    configMap:
        name: app-config
volumeMounts:
- mountPath: /etc/config
    name: config-volume
```

**Mounting Specific Keys:**

```yaml
volumes:
- name: config-volume
    configMap:
        name: app-config
        items:
        - key: app.properties
            path: custom-name.properties
```

---

### 6. Useful Commands

```sh
kubectl get pv
kubectl get pvc
kubectl get sc
kubectl get secrets
kubectl get configmap
```

---

### 7. Visual Overview

![Kubernetes Pod Volumes](njibb9wicexq-KubernetesPodVolumes.png)
![StorageK8S](pvpvcStorageClass.gif)

---

**Summary:**  
- Use Volumes for ephemeral or Pod-scoped data.
- Use PV/PVC for persistent, cluster-scoped storage.
- Use StorageClass and CSI for dynamic, cloud-native storage.
- Use Secrets and ConfigMaps for managing sensitive and configuration data, respectively.

-----

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

___

# SERVICES

Kubernetes uses transient, decoupled objects. Services connect Pods or provide external access, allowing Pods to be replaced seamlessly. Labels help reconnect new Pods, and Endpoints route traffic. Google’s Extensible Service Proxy (ESP) offers more flexibility but is mainly used in Google environments.

There are multiple service types, each exposing resources internally or externally. Services can also link internal resources to external ones, like third-party databases.

The kube-proxy agent monitors the API for new services and endpoints, opens random ports, and redirects traffic to service endpoints.

Services offer automatic load-balancing based on labels, with optional session affinity. Headless services can be configured without a fixed IP or load-balancing.

Unique IPs are managed via etcd, and Services use iptables for routing, with potential for other technologies in the future.

Labels determine which Pods receive traffic from a Service. Since labels can be updated dynamically, changing them may affect which Pods are selected by the Service.

By default, Kubernetes uses a rolling deployment strategy: new Pods with updated application versions are gradually added, and traffic is automatically load balanced between old and new Pods. This ensures continuous availability during updates.

However, if different application versions are incompatible—meaning clients may have issues communicating with multiple versions—you should use more specific labels, such as including a version number.

| Access Method                 | Source of Access         | Public? | Compatible Service Type(s)         | Requires Cloud LB? | Use Case                               |
|------------------------------|--------------------------|---------|-------------------------------------|---------------------|----------------------------------------|
| `CLUSTER-IP`                 | Inside cluster only       | ❌      | `ClusterIP` (default)               | ❌                  | Internal-only communication            |
| `EXTERNAL-IP`                | External (internet)       | ✅      | `LoadBalancer`                      | ✅                  | Public access via cloud LB             |
| `NODE-INTERNAL-IP + NodePort`| Internal network (VPC)    | ❌      | `NodePort`, `LoadBalancer`         | ❌                  | Private/bastion access                 |
| `NODE-EXTERNAL-IP + NodePort`| Public access via Node    | ✅      | `NodePort`, `LoadBalancer`         | ❌                  | Manual public access, dev/test setup   |

**Basic Steps to access a new service**

```sh
# expose
$ kubectl expose deployment/nginx --port=80 --type=NodePort
# check
$ kubectl get svc
NAME        TYPE       CLUSTER-IP  EXTERNAL-IP  PORT(S)        AGE
kubernetes  ClusterIP  10.0.0.1    <none>       443/TCP        18h
nginx       NodePort   10.0.0.112  <none>       80:31230/TCP   5s
#get
$ kubectl get svc nginx -o yaml
apiVersion: v1
kind: Service
...
spec:
    clusterIP: 10.0.0.112
    ports:
    - nodePort: 31230
...
```
>>Open browser http://Public-IP:31230. (nodeport type => ip of the machine/node)

`kubectl expose` creates a Service for the nginx Deployment, exposing port 80 by default and assigning a random NodePort. You can set `port` and `targetPort` to control traffic routing; `targetPort` can also reference a named port in the Pod spec. `kubectl get svc` lists all Services, including the new nginx Service (default type: ClusterIP). ClusterIP and NodePort ranges are configurable. Services can target resources in other namespaces or external endpoints.

### Kubernetes Service Types

#### ClusterIP (default): **Exposes the service on an internal IP address within the cluster**
- Accessible only from within the cluster.
- The range of ClusterIP addresses is defined via an API server startup option.

#### NodePort: **Exposes the service on each Node’s IP at a static port.**
- Useful for debugging or when a fixed port is needed (e.g., opening a firewall).
- The NodePort range is set in the cluster configuration.
- Traffic to `<NodeIP>:<NodePort>` is forwarded to the service.

#### LoadBalancer: **Integrates with cloud providers to provision an external load balancer.**
- Automatically assigns a public IP address to the service.
- Distributes incoming traffic across the Pods in the deployment.
- Supported by cloud providers like GKE, AWS, and via plugins for private clouds (e.g., CloudStack, OpenStack).
- Even without a cloud provider, the service is made available to public traffic.

#### ExternalName: **Maps the service to an external DNS name.**
- Does not use selectors, ports, or endpoints.
- Returns a CNAME record redirecting traffic at the DNS level, not via proxying.
- Useful for referencing external services not yet migrated to the cluster.


> The `kubectl proxy` command starts a local HTTP proxy server, allowing you to securely access Kubernetes ClusterIP services from your local machine. This is especially helpful for troubleshooting, development, or accessing the Kubernetes API without exposing services externally.

Kubernetes Services are essential components thatexpose pods to network traffic, they build upon each other to provide connectivity:

- **Service Controller**: Runs inside the `kube-controller-manager` and manages Service resources by communicating with the `kube-apiserver`.
- **Network Plugin** (e.g., Cilium): Configures the underlying network and enforces network policies.
- **kube-proxy**: Runs on every node, setting up network rules (using `iptables` or `ipvs`) to route traffic to the correct pods.
- **Endpoint Controller**: Monitors pod IP changes and updates Service endpoints accordingly to keep a fixed ip for service. (Ephemeral IP in pod created/destroyed > fixed ip for service)


- **ClusterIP** service assigns a stable internal IP address within the Kubernetes cluster, routing traffic to the appropriate pods. This service type only handles traffic originating from within the cluster.

- **NodePort** service builds on ClusterIP by exposing the service on a static port (typically in the 30000–32767 range) on each node’s IP. When a NodePort is created, Kubernetes first provisions a ClusterIP, then allocates a port and configures firewall rules so that traffic to this port on any node is forwarded to the ClusterIP, and subsequently to the pods.

- **LoadBalancer** service further extends NodePort by requesting an external load balancer from the underlying cloud provider. Kubernetes creates a NodePort and makes an asynchronous request for a load balancer. If the infrastructure supports it (e.g., on public clouds), an external load balancer is provisioned and traffic is routed to the NodePort. Otherwise, the service remains in a Pending state.

- **Ingress controller** is a specialized pod that manages external access to services, typically via HTTP/HTTPS. It listens on a high port and routes incoming requests to services based on URL paths or hostnames. Ingress controllers are not built-in services, but are commonly deployed to centralize and manage external traffic to multiple services.


![Service Relationships](Service_Relationships.png)
![ServiceRelationships](servicerelationship.jpg)

Controllers in `kube-controller-manager` send API requests to `kube-apiserver`, which are handled by the network plugin (e.g., `cilium-controller`) and node agents (e.g., `cilium-node`). `kube-proxy` manages local firewall rules using `iptables`, `IPVS`, or `userspace`, as set by its configuration.

![kubeproxyiptable](kubeproxyiptable.png)

The diagram shows a multi-container pod with two services receiving traffic via the pod’s IP. An ingress controller forwards traffic to a service, which can be ClusterIP, NodePort, or LoadBalancer.

![Cluster network diagram](clusternetwork1.png)

## Using a Local Proxy for Kubernetes Development

When developing applications or services on Kubernetes, you can use a local proxy to easily interact with the Kubernetes API and access ClusterIP services from your local machine. The `kubectl proxy` command starts a proxy server that listens on a configurable IP address and port (default: `127.0.0.1:8001`). By default, this command captures your shell unless run in the background.

### Starting the Proxy

```sh
kubectl proxy

Starting to serve on 127.0.0.1:8001
```

### Accessing Services via the Proxy

With the proxy running, you can access Kubernetes API resources and services using URLs like:

- **Access a service named `ghost` in the `default` namespace:**
    ```
    http://localhost:8001/api/v1/namespaces/default/services/ghost
    ```

- **If the service port has a name (e.g., `http`):**
    ```
    http://localhost:8001/api/v1/namespaces/default/services/ghost:http
    ```
*reminder*    
| Feature          | Forward Proxy                    | Reverse Proxy                   |
|------------------|----------------------------------|---------------------------------|
| Position         | In front of clients              | In front of servers             |
| Hides            | Clients (users)                  | Servers (infrastructure)        |
| Common Use Cases | Anonymity, filtering, bypass     | Load balancing, security        |
| Example Tool     | Squid, SOCKS5                    | NGINX, Envoy, HAProxy           |

> This approach is useful for testing, debugging, or interacting with services that are not exposed outside the cluster.

## DNS in Kubernetes

Since version 1.13, Kubernetes uses CoreDNS as the default DNS provider for DNS management within the cluster.

When a CoreDNS container starts, it launches a server for each configured DNS zone. Each server can load one or more plugin chains, Clients typically access DNS services through the `kube-dns` service.

CoreDNS includes around thirty built-in plugins that cover most common DNS functionalities, Common plugins provide:

- Metrics for Prometheus monitoring
- Error logging
- Health checks and reporting
- TLS support for securing DNS and gRPC servers

To verify DNS setup in your cluster, run a pod with networking tools and exec into it for DNS lookups.

Use tools like `nslookup`, `dig`, and `nc` for troubleshooting. In Kubernetes, check service labels, selectors, and review `/etc/resolv.conf` and any Network Policies or firewall rules.

___

# SERVICES TP

**Services** (also called *microservices*) are objects that define policies for accessing a logical set of Pods. Services are typically assigned labels, enabling persistent access to resources even when front-end or back-end containers are terminated and replaced.

- **Native applications** can use the Endpoints API to access services.
- **Non-native applications** can use a Virtual IP-based bridge to reach back-end Pods.

**Service Types:**
- **ClusterIP** (default): Exposes the service on a cluster-internal IP, making it reachable only within the cluster.
- **NodePort**: Exposes the service on each node’s IP at a static port. A ClusterIP is also automatically created.
- **LoadBalancer**: Exposes the service externally using a cloud provider’s load balancer. Both NodePort and ClusterIP are automatically created.
- **ExternalName**: Maps the service to the value of `externalName` using a CNAME DNS record.

| Access Method                 | Source of Access         | Public? | Compatible Service Type(s)         | Requires Cloud LB? | Use Case                               |
|------------------------------|--------------------------|---------|-------------------------------------|---------------------|----------------------------------------|
| `CLUSTER-IP`                 | Inside cluster only       | ❌      | `ClusterIP` (default)               | ❌                  | Internal-only communication            |
| `EXTERNAL-IP`                | External (internet)       | ✅      | `LoadBalancer`                      | ✅                  | Public access via cloud LB             |
| `NODE-INTERNAL-IP + NodePort`| Internal network (VPC)    | ❌      | `NodePort`, `LoadBalancer`         | ❌                  | Private/bastion access                 |
| `NODE-EXTERNAL-IP + NodePort`| Public access via Node    | ✅      | `NodePort`, `LoadBalancer`         | ❌                  | Manual public access, dev/test setup   |

Services enable decoupling, allowing any agent or object to be replaced without interrupting client access to back-end applications.


In the first TP we deploy two nginx server:
```sh
cat nginx-one.yaml
```
```yaml
# YAML versioned schema
apiVersion: apps/v1
# This file defines a Kubernetes Deployment resource
kind: Deployment
metadata:
  name: nginx-one
  labels:
    # Label to categorize this deployment within the namespace
    system: secondary
    # Namespace where the deployment and its resources will be created
  namespace: accounting
spec:
  selector:
    matchLabels:
      # The deployment will manage pods with this label
      system: secondary
    # Number of pod replicas to run
  replicas: 2
  template:
    metadata:
      labels:
        # Labels provide metadata for identifying and organizing resources.
        # They are useful for grouping, selecting, and filtering objects.
        system: secondary
    spec:
      # Specification for the containers that will run in the pod
      containers:
      - image: nginx:1.20.1 # Docker image to deploy
        imagePullPolicy: Always # Always pull the latest image version
        name: nginx # Unique name for this container within the pod
          # Ports to expose from the container
        ports:
        - containerPort: 8080
          protocol: TCP
            # Node affinity: schedule pods on nodes with matching labels
      nodeSelector:
        system: secondOne
```

Lets test our formating:
```sh
kubectl apply --dry-run=client -f nginx-one.yaml
deployment.apps/nginx-one created (dry run)
```

Lets get the labels we have in each node and create our ns
```sh
kubectl get nodes --show-labels
NAME   STATUS   ROLES           AGE   VERSION   LABELS
cp     Ready    control-plane   44d   v1.32.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=cp,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
dp     Ready    <none>          40d   v1.32.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=dp,kubernetes.io/os=linux
#creating ns accounting
kubectl create ns accounting
namespace/accounting created
```

Lets now apply our deploy
```sh
kubectl apply -f nginx-one.yaml
deployment.apps/nginx-one created
```

Checking:
```sh
# getting deploy
kubectl get deploy -n accounting
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-one   0/2     2            0           2m29s
# getting pod, they are on pending
kubectl get pod -n accounting
NAME                         READY   STATUS    RESTARTS   AGE
nginx-one-7cf678fc5b-49sdj   0/1     Pending   0          2m36s
nginx-one-7cf678fc5b-lmwf4   0/1     Pending   0          2m36s
# cause of node selector as shown by describe
 kubectl -n accounting describe pod nginx-one-7cf678fc5b-49sdj
......
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  2m58s  default-scheduler  0/2 nodes are available: 2 node(s) didn't match Pod's node affinity/selector. preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling.
```

Lets add label to our DP (worker node):
```sh
#labeling node
kubectl label nodes dp system=secondOne
node/dp labeled
#now everything work fines
kubectl -n accounting get deploy,pod
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-one   2/2     2            2           5m43s

NAME                             READY   STATUS    RESTARTS   AGE
pod/nginx-one-7cf678fc5b-49sdj   1/1     Running   0          5m43s
pod/nginx-one-7cf678fc5b-lmwf4   1/1     Running   0          5m43s
#getting pod by using label
kubectl get pod -A -l system=secondary
NAMESPACE    NAME                         READY   STATUS    RESTARTS   AGE
accounting   nginx-one-7cf678fc5b-49sdj   1/1     Running   0          9m15s
accounting   nginx-one-7cf678fc5b-lmwf4   1/1     Running   0          9m15s
```

Lets expose our deployement:
```sh
kubectl -n accounting expose deployment nginx-one
service/nginx-one exposed
kubectl -n accounting get ep nginx-one
NAME        ENDPOINTS                               AGE
nginx-one   192.168.1.176:8080,192.168.1.193:8080   14s
```

Since we expose port 8080, we should curl now:
```sh
curl 192.168.1.176:8080
curl: (7) Failed to connect to 192.168.1.176 port 8080: Connection refused
```
> it fails because even if we exposed 8080, the application within listen on 80 by default (nginx), so if we curl to 80:
```sh
#it works fine:
 curl 192.168.1.176:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

Lets modify our deployement:
```sh
#delete
kubectl -n accounting delete deployments.apps nginx-one
deployment.apps "nginx-one" deleted
#modify our deploy
cp:~# edit nginx-one.yaml
```
```yaml
        ports:
        - containerPort: 8080
          protocol: TCP
```
```sh
#create again
kubectl -n accounting create -f nginx-one.yaml
deployment.apps/nginx-one created
#check pod/deploy
kubectl -n accounting get deploy,pod
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-one   2/2     2            2           60s

NAME                             READY   STATUS    RESTARTS   AGE
pod/nginx-one-6b9c6fbb7b-frpxt   1/1     Running   0          60s
pod/nginx-one-6b9c6fbb7b-tckln   1/1     Running   0          60s
#check ep
NAME        ENDPOINTS                               AGE
nginx-one   192.168.1.139:8080,192.168.1.147:8080   6m34s
#test curl
#work on ip directly
curl 192.168.1.139

<title>Welcome to nginx!</title>
#works on 80
curl 192.168.1.139:80

<title>Welcome to nginx!</title>
#not work on 8080
root@cp:~# curl 192.168.1.139:8080

curl: (7) Failed to connect to 192.168.1.139 port 8080: Connection refused
```

In a previous exercise, we deployed a LoadBalancer(at start of course), which automatically created both a ClusterIP and a NodePort. In this exercise, we will focus on deploying a NodePort service directly. While containers are accessible from within the cluster, a NodePort allows you to expose services to external traffic by mapping a port on each node to your service. One reason to use a NodePort instead of a LoadBalancer is that LoadBalancers typically provision external resources from cloud providers, such as GKE or AWS, which may not always be necessary or cost-effective.

**NodePort vs LoadBalancer (K8s)**

- **NodePort**
  - Exposes service on each node's IP and a fixed port.
  - Access via `<NodeIP>:<NodePort>`.
  - No cloud needed, good for dev/testing.

- **LoadBalancer**
  - Provisions an external IP via cloud provider.
  - Easiest for public access.
  - Requires cloud, good for production.

in a previous step we were able to view the nginx page using the internal Pod IP address. Now expose the deployment
using the --type=NodePort and give it a name.its possible to pass the port as well, which could help with opening ports in the firewall instead of getting random one by K8S:

```sh
#exposing the service
kubectl -n accounting expose deployment nginx-one --type=NodePort --name=service-lab
service/service-lab exposed
#check if everything created
kubectl -n accounting get all

NAME                             READY   STATUS    RESTARTS   AGE
pod/nginx-one-6b9c6fbb7b-5l5tz   1/1     Running   0          15m
pod/nginx-one-6b9c6fbb7b-hkvg5   1/1     Running   0          15m

NAME                  TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/service-lab   NodePort   10.110.157.41   <none>        80:30131/TCP   14m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-one   2/2     2            2           15m

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-one-6b9c6fbb7b   2         2         2       15m
kubectl -n accounting describe services

Name:                     service-lab
Namespace:                accounting
Labels:                   system=secondary
Annotations:              <none>
Selector:                 system=secondary
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.110.157.41
IPs:                      10.110.157.41
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30131/TCP
Endpoints:                192.168.1.79:80,192.168.1.147:80
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:                   <none>
```

First we test with internal ip address
#getting hostname
kubectl cluster-info
```sh
Kubernetes control plane is running at https://k8scp:6443
CoreDNS is running at https://k8scp:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
#curling
curl http://k8scp:30131
```
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```
```sh
#Lets get the external ip
curl ifconfig.io
34.155.199.133
#you can curl or use on browser
curl http://34.155.199.133:30131/
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

## CoreDNS

First we will create a pod with ubuntu to do some DNS troubleshoting using dig

```yaml
#nettool.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu
spec:
  containers:
  - name: ubuntu
    image: ubuntu:latest
    command: [ "sleep" ]
    args: [ "infinity" ]
``` 
```sh
#creating the pod
kubectl create -f nettool.yaml
#exec inside the pod
kubectl exec -it ubuntu -- /bin/bash
``` 

**Inside the POD ubuntu**
```sh
##Inside Ubuntu##
root@ubuntu:/# apt-get update ; apt-get install curl dnsutils -y
root@ubuntu:/# dig

; <<>> DiG 9.16.1-Ubuntu <<>>
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3394
;; flags: qr rd ra; QUERY: 1, ANSWER: 13, AUTHORITY: 0, ADDITIONAL: 1
<output_omitted>
;; Query time: 4 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Thu Aug 27 22:06:18 CDT 2024
;; MSG SIZE rcvd: 431
``
``` sh
#we cat resolv.conf
#which will indicate nameservers and default domains to search
#if no using a Fully Qualified Distinguished Name (FQDN)
root@ubuntu:/# cat /etc/resolv.conf

nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
c.endless-station-188822.internal google.internal
options ndots:5
#Use the dig command to view more information about the DNS server. the -x argument to get the
#FQDN using the IP we know. Notice the domain name, which uses ".kube-system.svc.cluster.local.",
#to match the pod namespaces instead of default. Also note the name, kube-dns, is the name of a service
#not a pod
root@ubuntu:/# dig @10.96.0.10 -x 10.96.0.10

...
;; QUESTION SECTION:
;10.0.96.10.in-addr.arpa. IN PTR
;; ANSWER SECTION:
10.0.96.10.in-addr.arpa.
30 IN PTR kube-dns.kube-system.svc.cluster.local.↪→
;; Query time: 0 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Thu Aug 27 23:39:14 CDT 2024
;; MSG SIZE rcvd: 139
```
```sh
# lets curl the service-lab we previously created using the FQDN:
root@ubuntu:/# curl service-lab.accounting.svc.cluster.local.

© Copyright The Linux Foundation 2025: DO NOT COPY OR DISTRIBUTE
10.4. LABS 3
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```
```sh
#If we curl directly the service name , we will get an error since we are looking inside default namespace
curl service-lab

curl: (6) Could not resolve host: service-lab
#adding the namespace, it works!
root@ubuntu:/# curl service-lab.accounting

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<output_omitted>
root@ubuntu:/# exit
```
**Outside the POD ubuntu**

Lets now check the CoreDNS pod and svc:

```sh
#in the kube-system namespace the kube-dns server has the DNS ServerIP and exposed DNS default ports:
kubectl -n kube-system get svc

NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kube-dns ClusterIP 10.96.0.10 <none> 53/UDP,53/TCP,9153/TCP 42h
# lets check the svc in yaml and precisly the Selector:
kubectl -n kube-system get svc kube-dns -o yaml
```
```yaml
...
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: CoreDNS
...
  selector:
    k8s-app: kube-dns
    sessionAffinity: Non
    type: ClusterIP
...
```
```sh
#all pods inside cluster that share the label k8s-app
kubectl get pod -l k8s-app --all-namespaces

NAMESPACE NAME READY STATUS RESTARTS AGE
kube-system cilium-5tv9d 1/1 Running 0 136m
kube-system cilium-gzdk6 1/1 Running 0 54m
kube-system coredns-5d78c9869d-44qvq 1/1 Running 0 31m
kube-system coredns-5d78c9869d-j6tqx 1/1 Running 0 31m
kube-system kube-proxy-lpsmq 1/1 Running 0 35m
kube-system kube-proxy-pvl8w 1/1 Running 0 34m
#let check the coreDNS pod:
kubectl -n kube-system get pod coredns-f9fd979d6-4dxpl -o yaml
```
```yaml
...
spec:
containers:
- args:
- -conf
- /etc/coredns/Corefile
image: k8s.gcr.io/coredns:1.7.0
...
volumeMounts:
- mountPath: /etc/coredns
name: config-volume
readOnly: true
...
volumes:
- configMap:
defaultMode: 420
items:
- key: Corefile
path: Corefile
name: coredns
name: config-volume
...
```
```sh
#Viewing the config map inside kube-system 
kubectl -n kube-system get configmaps
#The coredns config
kubectl -n kube-system get configmaps coredns -o yaml
```
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {                           # Listen on port 53 for all DNS queries (root zone)
      errors                         # Log DNS errors
      health {                       # Health check endpoint
        lameduck 5s                  # Wait 5s before shutdown during termination
      }
      ready                          # Add /ready endpoint for Kubernetes readiness probes
      kubernetes cluster.local in-addr.arpa ip6.arpa {  # Handle cluster DNS for services and pods
        pods insecure                # Allow insecure pod DNS (no validation)
        fallthrough in-addr.arpa ip6.arpa  # If query not found, pass to next handler
        ttl 30  # Set DNS record Time To Live to 30 seconds
      }
      prometheus :9153               # Expose Prometheus metrics on port 9153
      forward . /etc/resolv.conf {   # Forward unmatched queries to DNS servers in /etc/resolv.conf
        max_concurrent 1000          # Allow up to 1000 concurrent DNS queries
      }
      cache 30                       # Cache DNS responses for 30 seconds
      loop                           # Detect and prevent DNS forwarding loops
      reload                         # Auto-reload CoreDNS if config (Corefile) changes
      loadbalance                    # Randomize DNS response order (load balancing)
    }
```
```sh
#Lets backup this config and modify it
kubectl -n kube-system get configmaps coredns -o yaml > coredns-backup.yaml
kubectl -n kube-system edit configmaps coredns
```
```yaml
apiVersion: v1
kind: ConfigMap
data:
  Corefile: |
    .:53 {
      rewrite name regex (.*)\.test\.io {1}.default.svc.cluster.local #<--Add this line, explanation below
.....
    }
```
>if a client tries to resolve something like myapp.test.io,
>➔ CoreDNS rewrites it internally to myapp.default.svc.cluster.local (the real Kubernetes service name).
```sh
#we delete all pod for reacreation and taking in the new config
kubectl -n kube-system delete pod coredns-f9fd979d6-s4j98 coredns-f9fd979d6-xlpzf

pod "coredns-f9fd979d6-s4j98" deleted
pod "coredns-f9fd979d6-xlpzf" deleted
#we create a deployement nginx
kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
#and expose it as ClusterIp + port 80
kubectl expose deployment nginx --type=ClusterIP --port=80
service/nginx expose
kubectl get svc
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kubernetes ClusterIP 10.96.0.1 <none> 443/TCP 3d15h
nginx ClusterIP 10.104.248.141 <none> 80/TCP 7s
```

**Inside the POD ubuntu Again**
```sh
#execing
kubectl exec -it ubuntu -- /bin/bash
#inside ubuntu pod, not that the service name become part of the FQDN inside the default namespace
root@ubuntu:/# dig -x 10.104.248.141

...
;; QUESTION SECTION:
;141.248.104.10.in-addr.arpa. IN PTR
;; ANSWER SECTION:
141.248.104.10.in-addr.arpa.
30 IN PTR nginx.default.svc.cluster.local.↪→
...
```
```sh
#Lets reverse lookup the and get the IP with the FQDN
root@ubuntu:/# dig nginx.default.svc.cluster.local.

....
;; QUESTION SECTION:
;nginx.default.svc.cluster.local. IN A
;; ANSWER SECTION:
nginx.default.svc.cluster.local. 30 IN A 10.104.248.141
....
```
Lets test the rewrite rule
```sh
#Lets test the rewrite rule of test.io , it should resolve the IP
root@ubuntu:/# dig nginx.test.io
....
;; QUESTION SECTION:
;nginx.test.io. IN A
;; ANSWER SECTION:
nginx.default.svc.cluster.local. 30 IN A 10.104.248.141
....
#exiting the container ubuntu
root@ubuntu:/# exit
```
**Outside the POD ubuntu Again**
```sh
Lets edit the config again
#editing the config
kubectl -n kube-system edit configmaps coredns
#adding this part:
```
```yaml
apiVersion: v1
kind: ConfigMap
data:
  Corefile: |
    .:53 {
        rewrite stop { #<-- Edit this and following two lines
          name regex (.*)\.test\.io {1}.default.svc.cluster.local
          answer name (.*)\.default\.svc\.cluster\.local {1}.test.io
    }
    errors
    health {
    ....
```
### CoreDNS Rewrite Explanation

- **`rewrite stop`**  
  Tells CoreDNS to **stop processing further rewrite rules** if this one matches.
- **`rewrite name regex (.*)\.test\.io {1}.default.svc.cluster.local`**  
  resolve something like `myapp.test.io`,  
  CoreDNS rewrites it internally to `myapp.default.svc.cluster.local` (the real Kubernetes service name).

- **`answer name (.*)\.default\.svc\.cluster\.local {1}.test.io`**  
  When returning DNS responses, CoreDNS **rewrites the answer back** from the internal service name (`.default.svc.cluster.local`)  
  to the **friendly name** (`.test.io`).
✅ Incoming request ➔ rewrite to service
✅ DNS response ➔ rewrite back to friendly domain

Applying modification:

```sh
#lets delete the pod for this to be taking in consideration
kubectl -n kube-system delete pod coredns-f9fd979d6-fv9qn coredns-f9fd979d6-lnxn5

pod "coredns-f9fd979d6-fv9qn" deleted
pod "coredns-f9fd979d6-lnxn5" deleted
```

**Inside the POD ubuntu Again**
```sh
#execing inside ubuntu pod again
kubectl exec -it ubuntu -- /bin/bash
#inside the pod
root@ubuntu:/# dig nginx.test.io

;; QUESTION SECTION:
;nginx.test.io. IN A
;; ANSWER SECTION:
nginx.test.io. 30 IN A 10.104.248.141
```
**Outside the POD ubuntu Again**
Deleting the nettool for resource:
```sh
kubectl delete -f nettool.yaml
```


**Using Labels to manage resources**
```sh
#get pod with label system=secondary
kubectl get pods -l system=secondary --all-namespaces

NAMESPACE    NAME                         READY   STATUS    RESTARTS   AGE
accounting   nginx-one-6b9c6fbb7b-5l5tz   1/1     Running   0          25h
accounting   nginx-one-6b9c6fbb7b-hkvg5   1/1     Running   0          25h
#delete pod with label system=secondary
kubectl delete pods -l system=secondary --all-namespaces

pod "nginx-one-6b9c6fbb7b-5l5tz" deleted
pod "nginx-one-6b9c6fbb7b-hkvg5" deleted

#new pod created since deployement controller/replicas controller still here
kubectl get pod -n accounting
NAME                         READY   STATUS    RESTARTS   AGE
nginx-one-6b9c6fbb7b-76sdx   1/1     Running   0          16s
nginx-one-6b9c6fbb7b-9httj   1/1     Running   0          16s

#deleting deployement
kubectl -n accounting delete deploy -l system=secondary

#show node labels
kubectl get nodes --show-labels
NAME   STATUS   ROLES           AGE   VERSION   LABELS
cp     Ready    control-plane   46d   v1.32.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=cp,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
dp     Ready    <none>          42d   v1.32.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=dp,kubernetes.io/os=linux,system=secondOne

#remove labels from dp
kubectl label nodes dp system-
node/dp unlabeled
```

___

# Ingress

Ingress Controllers and Rules let you efficiently expose multiple services outside a Kubernetes cluster by routing traffic based on host or path, centralizing access instead of using many LoadBalancer services. Unlike most controllers, Ingress Controllers run separately from kube-controller-manager and can be deployed with different configurations. Popular options include nginx, Traefik, and Envoy. Ingress Rules, created via kubectl, configure these controllers to route external traffic to internal ClusterIP services.

An Ingress Controller is a daemon running in a Pod that monitors the `/ingresses` endpoint on the Kubernetes API server (under the `networking.k8s.io/v1` group) for new Ingress resources. When a new Ingress is created, the controller applies its configured rules to route inbound connections—typically HTTP traffic—to the appropriate Kubernetes Service.

You can deploy multiple Ingress Controllers within a cluster. To direct traffic to a specific controller, use annotations on your Ingress resources. If no matching annotation is present, all controllers will attempt to handle the Ingress, which may lead to conflicts.

![ingress1111.png](ingress1111.png)

## Deploying the NGINX Ingress Controller

Deploying an NGINX Ingress Controller is straightforward using the provided YAML files available in the [ingress-nginx/docs/deploy](https://github.com/kubernetes/ingress-nginx/tree/main/docs/deploy) GitHub repository.

### Configuration Requirements

To ensure proper deployment, consider the following configuration options:

- **Easy integration with RBAC**
- Uses the annotation:  
    ```yaml
    kubernetes.io/ingress.class: "nginx"
    ```
- For L7 traffic, configure the `proxy-real-ip-cidr` option to trust proxy IP ranges and correctly identify client IPs.
- Bypasses `kube-proxy` *(traffic routing to pods is handled directly by the Ingress Controller)* to enable session affinity ensuring that Ensures that requests from the same client are consistently routed to the same pod
- Does not use `conntrack` entries for iptables DNAT (***avoids** kernel connection tracking to improve performance and scalability in high-traffic environments, It is commonly used for NAT (Network Address Translation)*)
- For TLS, the `host` field must be defined

Customization is possible via a **ConfigMap**, **Annotations**, or, for advanced scenarios, a **custom template**.


## Google Load Balancer Controller (GLBC)

The **Google Load Balancer Controller (GLBC)** is a Kubernetes component that automates the management of Google Cloud Load Balancers. It works by interpreting Kubernetes Ingress resources and creating the necessary Google Cloud resources to route traffic to your application.

### Key Features:
- **Ingress Resource Management**: GLBC monitors Kubernetes Ingress objects and ensures that the corresponding Google Cloud Load Balancer is created and configured.
- **Multi-Pool Path**: Traffic is routed through a series of objects, referred to as a "pool," to ensure connectivity. The path includes:
  - **Global Forwarding Rule**: Directs traffic to the appropriate target.
  - **Target HTTP Proxy**: Handles HTTP/HTTPS requests.
  - **URL Map**: Maps incoming requests to backend services based on URL paths.
  - **Backend Service**: Manages backend instances and health checks.
  - **Instance Group**: A group of virtual machine instances that serve the traffic.

#### Deployment Steps:
1. **Create the GLBC Controller**: Deploy the GLBC Controller in your Kubernetes cluster.
2. **Set Up Resources**: Create a ReplicationController with a single replica, three services for the application Pod, and an Ingress with two hostnames and three endpoints for each service.
3. **Monitor Quotas**: Be aware that several objects are created for each service, and quotas are not evaluated prior to creation.

#### Notes:
- The GLBC Controller must be started first to ensure proper resource creation.
- Each pool regularly checks the next hop to maintain connectivity.
- TLS Ingress currently supports only port 443 and assumes TLS termination. It does not support SNI and uses the first certificate in the TLS secret.

## Ingress API Resources

Ingress objects are now part of the `networking.k8s.io` API group and are still considered beta. Below is an example of a typical Ingress resource you can apply to your Kubernetes cluster:


```yaml
apiVersion: networking.k8s.io/v1 # API version for the Ingress resource
kind: Ingress              # Resource type
metadata:
  name: ghost              # Name of the Ingress object
spec:
  rules:
  - host: ghost.192.168.99.100.nip.io # Hostname for routing
      http:
        paths:
        - path: /               # URL path to match
          pathType: ImplementationSpecific # Path matching type, *ImplementationSpecific* means that the interpretation of the path matching is determined by the specific Ingress controller being used not k8s
          backend:
            service:
            name: ghost         # Backend service name
            port:
              number: 2368       # Backend service port
```


You can manage Ingress resources just like other Kubernetes objects (pods, deployments, services, etc.) using the following commands:

```sh
# List all ingress resources
kubectl get ingress

# Delete a specific ingress resource
kubectl delete ingress <ingress_name>

# Edit an existing ingress resource
kubectl edit ingress <ingress_name>
```

## Deploying an Ingress Controller

Deploy an Ingress Controller by applying its deployment YAML with:


```bash
kubectl create -f backend.yaml
```

This command will create a set of pods managed by a replication controller, along with some internal services:

```bash
kubectl get pods,rc,svc
```
output:
```sh
NAME                               READY   STATUS    RESTARTS   AGE
po/default-http-backend-xvep8      1/1     Running   0          4m
po/nginx-ingress-controller-fkshm  1/1     Running   0          4m

NAME                     DESIRED   CURRENT   READY   AGE
rc/default-http-backend  1         1         1       4m

NAME                      CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
svc/default-http-backend  10.0.0.212    <none>        80/TCP     4m
svc/kubernetes            10.0.0.1      <none>        443/TCP    77d
```

- The `default-http-backend` pod and service handle unmapped HTTP requests.
- The `nginx-ingress-controller` pod runs the Ingress Controller.
- The replication controller (`rc`) ensures the backend pod is always running.

For more details, refer to the [official Kubernetes Ingress documentation](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

## Creating an Ingress Rule

To quickly expose your application using Ingress, follow these steps:

1. **Deploy Ghost and Expose with a ClusterIP Service:**

    ```sh
    kubectl run ghost --image=ghost
    kubectl expose deployment ghost --port=2368
    ```

    This creates a Ghost deployment and exposes it internally within the cluster.

2. **Create an Ingress Resource:**

    ```yaml
    apiVersion: networking.k8s.io/v1 # Specifies the API version for the Ingress resource.
    kind: Ingress # Declares the resource type as Ingress.
    metadata:
      name: ghost-ingress # The name of the Ingress resource.
    spec:
      rules:
        - host: ghost.192.168.99.100.nip.io # Defines the hostname for routing traffic.
          http:
            paths:
              - path: / # Specifies the URL path to match incoming requests.
                pathType: ImplementationSpecific # Allows the Ingress controller to determine how to interpret the path.
                backend:
                  service:
                    name: ghost # The name of the service to route traffic to.
                    port:
                      number: 2368 # The port on the service to forward traffic to.
    ```

With the deployment, service, and Ingress in place, you should be able to access the Ghost application from outside the cluster using the specified host.

To expose multiple services with one Ingress, define several rules in the same manifest. Each rule matches a host and routes traffic to the correct backend service.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: example-ingress
spec:
    rules:
        # Rule 1: Routes traffic for ghost.192.168.99.100.nip.io to the 'external' service on port 80
        - host: ghost.192.168.99.100.nip.io #specifies the domain name for which the rule applies.
            http:
                paths: #section defines how requests are routed to backend services.
                - path: /
                    pathType: Prefix
                    backend: # specifies the target Kubernetes service and port.
                        service:
                            name: external
                            port:
                                number: 80
        # Rule 2: Routes traffic for nginx.192.168.99.100.nip.io to the 'internal' service on port 8080
        - host: nginx.192.168.99.100.nip.io #specifies the domain name for which the rule applies.
            http:
                paths: #section defines how requests are routed to backend services.
                - path: /
                    pathType: Prefix
                    backend: # specifies the target Kubernetes service and port.
                        service:
                            name: internal
                            port:
                                number: 8080
```

## SideCar

A **sidecar container** is a helper container that runs alongside the main container in the same Pod to provide extra features like logging, monitoring, proxying, or configuration.

A **sidecar proxy** is a lightweight proxy running next to an application container, handling its network traffic to enable routing, security, and observability without changing the app code.


![sidecar.png](sidecar.png)

## Service Mesh

A **Service Mesh** is a dedicated infrastructure layer that manages service-to-service communication in a microservices architecture. It provides features like traffic management, security, observability, and reliability without requiring changes to the application code.

#### Key Features:
- **Traffic Control**: Load balancing, shaping, and retries.
- **Security**: Mutual TLS (mTLS) for secure service communication.
- **Observability**: Monitoring, logging, and tracing.
- **Resilience**: Circuit breaking and rate limiting.


![Service Mesh Architecture](servicemesh2.png)

## Intelligent Connected Proxies

For advanced networking needs—such as service discovery, rate limiting, traffic management, security policies, and detailed observability—you may want to implement a **service mesh**. Service meshes provide a dedicated infrastructure layer for handling service-to-service communication in a secure, reliable, and observable way.

A service mesh typically consists of **edge** and **sidecar (embedded)** proxies that intercept and manage all network traffic between microservices. These proxies operate under the direction of a **control plane**, which distributes configuration and enforces policies across the mesh.

Popular service mesh solutions include:

![istioarch.png](istioarch.png)

### Envoy

**Envoy** is a high-performance, open-source proxy designed for cloud-native applications. Its modular and extensible architecture makes it a popular choice as the data plane for many service meshes. Envoy supports advanced features such as dynamic service discovery, load balancing, TLS termination, observability, and traffic shadowing.

### Istio

**Istio** builds on Envoy by providing a robust control plane that manages traffic routing, security, telemetry, and policy enforcement. Istio is platform-agnostic and integrates seamlessly with Kubernetes, making it a flexible and feature-rich option for managing microservices at scale.

### Linkerd

**linkerd** is a lightweight, ultrafast service mesh focused on simplicity and performance. It is easy to deploy and operate, with a minimal resource footprint. linkerd provides essential service mesh features such as mTLS, traffic shifting, and observability, making it ideal for teams seeking a straightforward solution.

## Ingress Limitations

Kubernetes Ingress supports basic HTTP(S) routing, but lacks native features like TCP/UDP routing, gRPC, header-based routing, and canary deployments. Advanced needs rely on vendor-specific annotations, causing inconsistent and non-portable configurations.

Access control is limited—there’s no built-in way to delegate granular responsibilities within an Ingress, increasing the risk of misconfiguration in multi-tenant setups.

Because the Ingress spec is minimal and feature-frozen, controllers add features differently, leading to vendor lock-in. Kubernetes now focuses on the **Gateway API** for advanced traffic management, so Ingress may not meet future requirements.


# Gateway API

The Gateway API, part of the `gateway.networking.k8s.io/v1` group, introduces a modern, extensible approach to Kubernetes networking. It defines three stable resource kinds:

- **GatewayClass**: Specifies a class of gateways with shared configuration, managed by a controller that implements the class.
- **Gateway**: Represents a traffic-handling infrastructure instance, such as a cloud load balancer.
- **HTTPRoute**: Describes HTTP-specific routing rules, mapping traffic from Gateway listeners to backend network endpoints (typically Kubernetes Services).

## Key Advantages

Gateway API was designed to overcome the limitations of the Ingress resource. It offers:

- **Native support for L4/L7 protocols**
- **Advanced routing**: Header and query-based routing, traffic splitting, mirroring, and more
- **First-class features**: Capabilities that previously required custom annotations or workarounds in Ingress

## Extensibility and Portability

Gateway API is highly extensible via well-defined mechanisms, such as custom route filters, policies, and new route types, all while maintaining API consistency. This allows the API to evolve with new requirements and ensures portability across implementations (NGINX, HAProxy, Istio, etc.) with minimal configuration changes.

## Separation of Concerns

By separating Gateways from Routes, the API enables safe multi-tenant usage:

- **Platform administrators** control entry points and security.
- **Developers** manage routing rules independently.

This separation reduces conflicts, aligns with enterprise team structures, and effectively provides a built-in RBAC model for networking.

## Security Improvements

Gateway API codifies configuration previously handled via annotations, providing a vendor-neutral language for ingress and traffic policy. It introduces security enhancements such as:

- **Cross-namespace permission checks** (Route binding and ReferenceGrants)
- **Consistent policy enforcement**

These features give administrators confidence to delegate route control without compromising security.
## Gateway API vs Ingress API

| Aspect        | Ingress API                              | Gateway API                                                      |
|---------------|------------------------------------------|------------------------------------------------------------------|
| **Purpose**   | Basic HTTP routing into Kubernetes       | Advanced, flexible traffic routing                               |
| **Maturity**  | Older, simpler                           | Newer, designed to fix Ingress limits                            |
| **Flexibility** | Limited (only HTTP/S)                  | Supports HTTP, TCP, UDP, and mTLS                                |
| **Extensibility** | Hard to extend                       | Built to be extendable (CRDs like GatewayClass, HTTPRoute)       |
| **Use Case**  | Simple websites, small apps              | Large platforms, multi-tenant apps, complex traffic needs        |

> ** Super short summary:**  
> Ingress = basic router (HTTP/HTTPS only).  
> Gateway API = advanced, modular, and future-proof traffic control (for big/complex setups).

![diffapigateway-ingress.png](diffapigateway-ingress.png)

## GatewayClass

A **GatewayClass** is a fundamental resource in the Kubernetes Gateway API, providing a standardized way to define and manage gateways across a cluster. Unlike the traditional Ingress resource, the Gateway API introduces a modular and extensible approach to network traffic management, with GatewayClass serving as the foundation for this design.

GatewayClass is a **cluster-scoped** resource that acts as a template or blueprint for a category of gateways. It defines shared properties and behaviors that any Gateway referencing it will inherit. Typically, cluster administrators or infrastructure teams create and maintain GatewayClass resources to enforce organizational standards and best practices for gateway deployments.

The most important field in a GatewayClass is `spec.controllerName`. This field specifies the controller (for example, an NGINX-based controller, Istio, etc) responsible for managing the lifecycle and configuration of all Gateway resources that reference this GatewayClass.

**Example:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
    name: example-class
spec:
    controllerName: example.com/gateway-controller
```

> **Note:** You can find a list of supported Gateway controllers in the [Kubernetes Gateway API documentation](https://gateway-api.sigs.k8s.io/implementations/).

## Gateway in Kubernetes

A **Gateway** in Kubernetes is a resource that manages inbound and outbound network traffic for services within a cluster. It defines how external clients connect to internal services by specifying protocol, port, and security settings. Gateways are part of the Gateway API, which provides a more flexible and expressive way to configure networking compared to traditional Ingress resources.

**Key Features:**
- **Separation of Concerns:** Gateways reference a `GatewayClass`, allowing infrastructure teams to manage networking implementations while application teams define routing.
- **Protocol Support:** Supports HTTP, HTTPS, TCP, and more, enabling advanced routing and security features such as TLS termination.
- **Cross-Namespace Routing:** Enables secure and controlled routing across namespaces.
- **Extensibility:** Integrates with service meshes and custom controllers for advanced traffic management.

**Example Gateway Resource:**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
    name: example-gateway
spec:
    gatewayClassName: example-class
    listeners:
        - name: http
            protocol: HTTP
            port: 80
            allowedRoutes:
                namespaces:
                    from: All
```

## HTTPRoute

An **HTTPRoute** is a Kubernetes resource that controls how HTTP traffic is routed within a cluster, allowing flexible rules based on hostnames, paths, headers, and more.

By linking HTTPRoutes to one or more Gateways, Kubernetes separates infrastructure management from application-level routing. This separation enhances flexibility, scalability, and maintainability across your applications.

### Example HTTPRoute Manifest

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
    name: example-httproute
spec:
    parentRefs:
        - name: example-gateway
    hostnames:
        - "www.example.com"
    rules:
        - matches:
                - path:
                        type: PathPrefix
                        value: /login
            backendRefs:
                - name: example-svc
                    port: 8080
```

## who manages what

| Resource      | Managed by      | Why                                                                                   |
|---------------|----------------|---------------------------------------------------------------------------------------|
| GatewayClass  | Administrators  | Defines the infrastructure (e.g., what controller is used: Envoy, NGINX, etc.). It’s cluster-wide. |
| Gateway       | Administrators  | Allocates real network resources (IPs, ports, certificates). It’s like "setting up the router."    |
| HTTPRoute (or TCPRoute, etc.) | Developers      | Defines application-specific routing (paths, hosts, services). Focused on app logic, not infra. |

![gateway-api-resources.png](gateway-api-resources.png)

![apigateawayarch.webp](apigateawayarch.webp)

# **INGRESS GATEWAY SERVICEMESH TP**
## **SERVICE MESH**

If you need to expose many services or low-numbered ports outside your cluster, use an **Ingress controller** (e.g., NGINX, GCE). For advanced features and metrics, consider a **service mesh** like Istio, Linkerd, or Contour.

- **Ingress controllers:** Manage external access to services.
- **Service meshes:** Add traffic management, security, and observability.

Lets us install **linkerd**
```sh
#Downloading the CLI
curl -sL run.linkerd.io/install-edge | sh

Downloading linkerd2-cli-edge-25.4.4-linux-amd64...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 74.9M  100 74.9M    0     0  79.9M      0 --:--:-- --:--:-- --:--:-- 79.9M
Download complete!

Validating checksum...
Checksum valid.

Linkerd edge-25.4.4 was successfully installed 🎉


Add the linkerd CLI to your path with:

  export PATH=$PATH:/root/.linkerd2/bin

Now run:

  # install the GatewayAPI CRDs
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

  linkerd check --pre                         # validate that Linkerd can be installed
  linkerd install --crds | kubectl apply -f - # install the Linkerd CRDs
  linkerd install | kubectl apply -f -        # install the control plane into the 'linkerd' namespace
  linkerd check                               # validate everything worked!

You can also obtain observability features by installing the viz extension:

  linkerd viz install | kubectl apply -f -  # install the viz extension into the 'linkerd-viz' namespace
  linkerd viz check                         # validate the extension works!
  linkerd viz dashboard                     # launch the dashboard

Looking for more? Visit https://linkerd.io/2/tasks
```
```sh
#Setting env var for bin and doing a first check
export PATH=$PATH:/root/.linkerd2/bin
#into bash also
echo "export PATH=$PATH:/root/.linkerd2/bin" >> $HOME/.bashrc
#first check
linkerd check --pre
kubernetes-api
--------------
√ can initialize the client
√ can query the Kubernetes API

kubernetes-version
------------------
√ is running the minimum Kubernetes API version

pre-kubernetes-setup
--------------------
√ control plane namespace does not already exist
√ can create non-namespaced resources
√ can create ServiceAccounts
√ can create Services
√ can create Deployments
√ can create CronJobs
√ can create ConfigMaps
√ can create Secrets
√ can read Secrets
√ can read extension-apiserver-authentication configmap
√ no clock skew detected

linkerd-version
---------------
√ can determine the latest version
√ cli is up-to-date

Status check results are √
```

> **The Gateway API is not part of the Kubernetes core (like Ingress), but rather a separately maintained SIG-Network project that can be installed on any Kubernetes version starting from v1.19 via CRDs.**
>> Lets install it

```sh
#first we check if we have the crds for gateway api
kubectl get crds | grep gateway.networking.k8s.io
#we get the GATEWAY API crds from k8s git and apply them
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
```

Lets now install linkerd crds
```sh
#installing linkerd crds
linkerd install --crds | kubectl apply -f -
Rendering Linkerd CRDs...
Next, run `linkerd install | kubectl apply -f -` to install the control plane.

customresourcedefinition.apiextensions.k8s.io/authorizationpolicies.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/egressnetworks.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/httplocalratelimitpolicies.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/meshtlsauthentications.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/networkauthentications.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/serverauthorizations.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/servers.policy.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/serviceprofiles.linkerd.io created
customresourcedefinition.apiextensions.k8s.io/externalworkloads.workload.linkerd.io created
```

Lets now install linkerd
```sh
#install linkerd
linkerd install | kubectl apply -f -
namespace/linkerd created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-identity created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-identity created
serviceaccount/linkerd-identity created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-destination created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-destination created
serviceaccount/linkerd-destination created
secret/linkerd-sp-validator-k8s-tls created
validatingwebhookconfiguration.admissionregistration.k8s.io/linkerd-sp-validator-webhook-config created
secret/linkerd-policy-validator-k8s-tls created
validatingwebhookconfiguration.admissionregistration.k8s.io/linkerd-policy-validator-webhook-config created
clusterrole.rbac.authorization.k8s.io/linkerd-policy created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-destination-policy created
role.rbac.authorization.k8s.io/remote-discovery created
rolebinding.rbac.authorization.k8s.io/linkerd-destination-remote-discovery created
role.rbac.authorization.k8s.io/linkerd-heartbeat created
rolebinding.rbac.authorization.k8s.io/linkerd-heartbeat created
clusterrole.rbac.authorization.k8s.io/linkerd-heartbeat created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-heartbeat created
serviceaccount/linkerd-heartbeat created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-proxy-injector created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-proxy-injector created
serviceaccount/linkerd-proxy-injector created
secret/linkerd-proxy-injector-k8s-tls created
mutatingwebhookconfiguration.admissionregistration.k8s.io/linkerd-proxy-injector-webhook-config created
configmap/linkerd-config created
role.rbac.authorization.k8s.io/ext-namespace-metadata-linkerd-config created
secret/linkerd-identity-issuer created
configmap/linkerd-identity-trust-roots created
service/linkerd-identity created
service/linkerd-identity-headless created
deployment.apps/linkerd-identity created
service/linkerd-dst created
service/linkerd-dst-headless created
service/linkerd-sp-validator created
service/linkerd-policy created
service/linkerd-policy-validator created
deployment.apps/linkerd-destination created
cronjob.batch/linkerd-heartbeat created
deployment.apps/linkerd-proxy-injector created
service/linkerd-proxy-injector created
secret/linkerd-config-overrides created
```

Lets do a check post install
```sh
linkerd check
kubernetes-api
--------------
√ can initialize the client
√ can query the Kubernetes API

kubernetes-version
------------------
√ is running the minimum Kubernetes API version

linkerd-existence
-----------------
√ 'linkerd-config' config map exists
√ heartbeat ServiceAccount exist
√ control plane replica sets are ready
√ no unschedulable pods
√ control plane pods are ready
√ cluster networks contains all node podCIDRs
√ cluster networks contains all pods
√ cluster networks contains all services

linkerd-config
--------------
√ control plane Namespace exists
√ control plane ClusterRoles exist
√ control plane ClusterRoleBindings exist
√ control plane ServiceAccounts exist
√ control plane CustomResourceDefinitions exist
√ control plane MutatingWebhookConfigurations exist
√ control plane ValidatingWebhookConfigurations exist
√ proxy-init container runs as root user if docker container runtime is used

linkerd-identity
----------------
√ certificate config is valid
√ trust anchors are using supported crypto algorithm
√ trust anchors are within their validity period
√ trust anchors are valid for at least 60 days
√ issuer cert is using supported crypto algorithm
√ issuer cert is within its validity period
√ issuer cert is valid for at least 60 days
√ issuer cert is issued by the trust anchor

linkerd-webhooks-and-apisvc-tls
-------------------------------
√ proxy-injector webhook has valid cert
√ proxy-injector cert is valid for at least 60 days
√ sp-validator webhook has valid cert
√ sp-validator cert is valid for at least 60 days
√ policy-validator webhook has valid cert
√ policy-validator cert is valid for at least 60 days

linkerd-version
---------------
√ can determine the latest version
√ cli is up-to-date

control-plane-version
---------------------
√ can retrieve the control plane version
√ control plane is up-to-date
√ control plane and cli versions match

linkerd-control-plane-proxy
---------------------------
√ control plane proxies are healthy
√ control plane proxies are up-to-date
√ control plane proxies and cli versions match

linkerd-extension-checks
------------------------
√ namespace configuration for extensions

Status check results are √
```

Lets now install linkerd viz (Grafana/Prometheus)
```sh
#Install linkerd viz
linkerd viz install | kubectl apply -f -
namespace/linkerd-viz created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-viz-metrics-api created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-metrics-api created
serviceaccount/metrics-api created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-viz-prometheus created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-prometheus created
serviceaccount/prometheus created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-viz-tap created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-viz-tap-admin created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-tap created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-tap-auth-delegator created
serviceaccount/tap created
rolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-tap-auth-reader created
secret/tap-k8s-tls created
apiservice.apiregistration.k8s.io/v1alpha1.tap.linkerd.io created
role.rbac.authorization.k8s.io/web created
rolebinding.rbac.authorization.k8s.io/web created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-viz-web-check created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-web-check created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-web-admin created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-viz-web-api created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-viz-web-api created
serviceaccount/web created
service/metrics-api created
deployment.apps/metrics-api created
server.policy.linkerd.io/metrics-api created
authorizationpolicy.policy.linkerd.io/metrics-api created
meshtlsauthentication.policy.linkerd.io/metrics-api-web created
networkauthentication.policy.linkerd.io/kubelet created
configmap/prometheus-config created
service/prometheus created
deployment.apps/prometheus created
server.policy.linkerd.io/prometheus-admin created
authorizationpolicy.policy.linkerd.io/prometheus-admin created
service/tap created
deployment.apps/tap created
server.policy.linkerd.io/tap-api created
authorizationpolicy.policy.linkerd.io/tap created
clusterrole.rbac.authorization.k8s.io/linkerd-tap-injector created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-tap-injector created
serviceaccount/tap-injector created
secret/tap-injector-k8s-tls created
mutatingwebhookconfiguration.admissionregistration.k8s.io/linkerd-tap-injector-webhook-config created
service/tap-injector created
deployment.apps/tap-injector created
server.policy.linkerd.io/tap-injector-webhook created
authorizationpolicy.policy.linkerd.io/tap-injector created
networkauthentication.policy.linkerd.io/kube-api-server created
service/web created
deployment.apps/web created
serviceprofile.linkerd.io/metrics-api.linkerd-viz.svc.cluster.local created
serviceprofile.linkerd.io/prometheus.linkerd-viz.svc.cluster.local created
#checking
linkerd viz check
linkerd-viz
-----------
√ linkerd-viz Namespace exists
√ can initialize the client
√ linkerd-viz ClusterRoles exist
√ linkerd-viz ClusterRoleBindings exist
√ tap API server has valid cert
√ tap API server cert is valid for at least 60 days
√ tap API service is running
√ linkerd-viz pods are injected
√ viz extension pods are running
√ viz extension proxies are healthy
√ viz extension proxies are up-to-date
√ viz extension proxies and cli versions match
√ prometheus is installed and configured correctly
√ viz extension self-check

Status check results are √
```

Lets run linkerd viz in background
```sh
#Runing linkerd viz in background
linkerd viz dashboard &
[1] 1256334
#output
Linkerd dashboard available at:
http://localhost:50750
Grafana dashboard available at:
http://localhost:50750/grafana
Opening Linkerd dashboard in the default browser
Failed to open Linkerd dashboard automatically
Visit http://localhost:50750 in your browser to view the dashboard
```

The GUI is only available on localhost by default. To allow external access, edit the service and deployment, and clear the value for `-enforced-host` (around line 59).
```sh
#before any change , backup
kubectl -n linkerd-viz get deployments.apps web -o yaml > linkerd-viz-deploy.yaml
#lets edit now and comment the enforced-host to localhost
kubectl -n linkerd-viz edit deployments.apps web
```
```yaml
...
    spec:
      automountServiceAccountToken: false
      containers:
      - args:
        - -linkerd-metrics-api-addr=metrics-api.linkerd-viz.svc.cluster.local:8085
        - -cluster-domain=cluster.local
        - -controller-namespace=linkerd
        - -log-level=info
        - -log-format=plain
        - -enable-pprof=false
        - -enforced-host=^(localhost......  #<--- comment this line
...
```

Now we need to change the service from ClusterIp to NodePort
```sh
#backupfirst
kubectl -n linkerd-viz get svc web -o yaml > linkerd-viz-svc.yaml
#lets edit the svc
kubectl -n linkerd-viz edit svc web
#lets change the type of the service and add a nodeport
```
```yaml
spec:
  clusterIP: 10.105.225.142 #**The main internal IP used by other pods to access the service.**
  clusterIPs: #**A list of internal IPs (used for dual-stack IPv4/IPv6 support).** added in new versions
  - 10.105.225.142
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http
    nodePort: 31500 # <--- added node port we choose our self
    port: 8084
    protocol: TCP
    targetPort: 8084
  - name: admin-http
    nodePort: 31787 # <--- We let the k8s choose randomly this
    port: 9994
    protocol: TCP
    targetPort: 9994
  selector:
    component: web
    linkerd.io/extension: viz
  sessionAffinity: None
  type: NodePort #<--changed to NodePort
status:
  loadBalancer: {}
```

To be sure deployement modifications have been taking in
```sh
#let restart the deploy
kubectl -n linkerd-viz rollout restart deployment web
deployment.apps/web restarted
# kubectl -n linkerd-viz get pod
NAME                           READY   STATUS    RESTARTS      AGE
metrics-api-86bff98cbc-mhqrw   2/2     Running   0             47m
prometheus-7ff979ffd9-6hnvj    2/2     Running   0             47m
tap-5b55c9bb76-6c8r5           2/2     Running   1 (46m ago)   47m
tap-injector-f67944698-kvfhr   2/2     Running   0             47m
web-755b796446-jvpjn           2/2     Running   0             6s
```

Now lets get our ip and try to access it
```
curl ifconfig.io
34.155.199.133
```

using : http://34.155.199.133:31500/namespaces
We can access :
![linkerd.png](linkerd.png)

To enable Linkerd to monitor a Kubernetes object, you need to add a specific annotation. The `linkerd inject` command automates this process. You can generate the YAML manifest, pipe it through `linkerd inject`, and then apply it with `kubectl`. You might see an error indicating the object was created, but this is expected and the process will still succeed, lets do this by recreating the `nginx-one` deployment.
```sh
#First we check if our namespace is still here
kubectl get ns accounting
NAME         STATUS   AGE
accounting   Active   3d20h
#We then relabel the node
kubectl label node dp system=secondOne
node/dp labeled
#valide the correct container port 80 not 8080
vim nginx-one.yaml
```
```yaml
       ports:
        - containerPort: 80
          protocol: TCP
```
```sh
#we apply
kubectl -n accounting apply -f nginx-one.yaml
deployment.apps/nginx-one created
```

Now we inject the annotation using `linkerd inject` command
```sh
#injecting linkerd through pipe
kubectl -n accounting get deployments.apps nginx-one -o yaml | linkerd inject - | kubectl apply -f -

deployment "nginx-one" injected
deployment.apps/nginx-one configured
```

Now if we check the ui, we will the see that accounting and his pods are meshed:

![linkerd2.png](linkerd2.png)
![linkerd3.png](linkerd3.png)

Lets generate some traffic and watch the RPS (request per second)
```sh
#first get the ip from svc
kubectl -n accounting get svc
NAME          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service-lab   NodePort   10.110.157.41   <none>        80:30131/TCP   3d14h
```

Before:
![linkerd4.png](linkerd4.png)

```sh
#then run some curl on it
for i in {1..150}; do curl -s http://10.110.157.41 > /dev/null; done
```
After:
![linkerd5.png](linkerd5.png)

Lets scale and get traffic for all pods:
```sh
#scaling to 5 replicas
kubectl -n accounting scale deployment nginx-one --replicas=5
deployment.apps/nginx-one scaled
#lets curl
for i in {1..300}; do curl -s http://10.110.157.41 > /dev/null; done
```

![linkerd6.png](linkerd6.png)
![linkerd7.png](linkerd7.png)


## **Ingres Controller TP**

First we create 2 deploy and open them as follow:
```sh
#create first deploy and expose port 80
kubectl create deployment web-one --image=nginx --port=80
deployment.apps/web-one created
#create second deploy and open also port 80
kubectl create deployment web-two --image=nginx --port=80
deployment.apps/web-two created
#then we expose svc as clusterIP and keep 80==>80
kubectl expose deployment web-one --port=80 --target-port=80 --type=ClusterIP
service/web-one exposed
kubectl expose deployment web-two --port=80 --target-port=80 --type=ClusterIP
service/web-two exposed
#we get to check
kubectl get svc,deploy
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/web-one      ClusterIP   10.107.243.59   <none>        80/TCP    13s
service/web-two      ClusterIP   10.99.245.196   <none>        80/TCP    7s

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web-one                           1/1     1            1           78s
deployment.apps/web-two                           1/1     1            1           73s
#we curl
curl 10.107.243.59
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
curl 10.107.243.59:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
 curl 10.99.245.196
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

Now we use HELM to **install an ingress controller** we will install NGINX the most famous one
```sh
#first a search
helm search hub ingress | grep -i nginx
...
https://artifacthub.io/packages/helm/ingress-ng...      4.12.1          1.12.1                                  Ingress controller for Kubernetes using NGINX a...
https://artifacthub.io/packages/helm/ingress-ng...      4.1.0           1.2.0                                   Ingress controller for Kubernetes using NGINX a...
https://artifacthub.io/packages/helm/nginx-ingr...      4.0.13          1.1.0                                   Ingress controller for Kubernetes using NGINX a...
https://artifacthub.io/packages/helm/nginx-ingr...      2.1.0           5.0.0                                   NGINX Ingress Controller
https://artifacthub.io/packages/helm/wenerme/in...      4.12.1          1.12.1                                  Ingress controller for Kubernetes using NGINX a...
https://artifacthub.io/packages/helm/api/ingres...      3.29.1          0.45.0                                  Ingress controller for Kubernetes using NGINX a...
https://artifacthub.io/packages/helm/kubeblocks...      4.12.1          1.12.1                                  Ingress controller for Kubernetes using NGINX a...
https://artifacthub.io/packages/helm/softonic/i...      4.9.1           1.9.6                                   Ingress controller for Kubernetes using NGINX a...
https://artifacthub.io/packages/helm/kubebb/ing...      4.7.0           1.8.0                                   Ingress controller for Kubernetes using NGINX a...
https://artifacthub.io/packages/helm/mxytest/in...      4.12.1          1.12.1                                  Ingress controller for Kubernetes using NGINX a...
https://artifacthub.io/packages/helm/gpg-dev/in...      4.9.0           1.9.5                                   Ingress controller for Kubernetes using NGINX a...
...
#we add the repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
"ingress-nginx" has been added to your repositories
#we update repo to take in consideraton the new repo added
helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "nfs-subdir-external-provisioner" chart repository
...Successfully got an update from the "ealenn" chart repository
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "cilium" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
#we fetch the repo locally
helm fetch ingress-nginx/ingress-nginx --untar
#we modify the value file to modifiy the installation mode from Deploy to Daemonset
vim ingress-nginx/value.yaml
```
```yaml
 kind: DaemonSet #changed from Deployement
  # -- Annotations to be added to the controller Deployment or DaemonSet
```
```sh
#installing
cd ingress-nginx
helm install myingress .
NAME: myingress
LAST DEPLOYED: Tue Apr 29 21:14:42 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the load balancer IP to be available.
You can watch the status by running 'kubectl get service --namespace default myingress-ingress-nginx-controller --output wide --watch'

An example Ingress that makes use of the controller:
```
```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls
```
```sh
If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:
```
```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```
```sh
#check if nginx service is up and runing
kubectl get service --namespace default myingress-ingress-nginx-controller --output wide --watch
NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE   SELECTOR
myingress-ingress-nginx-controller   LoadBalancer   10.99.124.35   <pending>     80:32720/TCP,443:31053/TCP   39s   app.kubernetes.io/component=controller,app.kubernetes.io/instance=myingress,app.kubernetes.io/name=ingress-nginx
#check if nginx pods runing
kubectl get pods --all-namespaces -o wide | grep nginx
default       myingress-ingress-nginx-controller-mn52x          1/1     Running   0             79s     192.168.1.138   dp     <none>           <none>
default       myingress-ingress-nginx-controller-zm7qz          1/1     Running   0             80s     192.168.0.77    cp     <none>           <none>
```
We then create the following ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-test
  namespace: default
  annotations:
    # By default, NGINX load balances across pod IPs.
    # With this annotation, traffic is sent to the service for Kubernetes to handle load balancing.
    nginx.ingress.kubernetes.io/service-upstream: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: www.external.com
      http:
        paths:
        - path: /
          # pathType: ImplementationSpecific lets NGINX decide path matching.
          # e.g., /api may match /api/, /api/v1, etc.
          pathType: ImplementationSpecific
          backend:
            service:
              name: web-one
              port:
                number: 80
```
```sh
#dry run to check syntaxe
kubectl apply --dry-run=client --validate=true -f ingress.yaml
ingress.networking.k8s.io/ingress-test created (dry run)
#create
kubectl apply -f ingress.yaml
ingress.networking.k8s.io/ingress-test created
#get
 kubectl get ingress
NAME           CLASS   HOSTS              ADDRESS   PORTS   AGE
ingress-test   nginx   www.external.com             80      25s
```

Lets check if our nginx is working
```sh
#we get the ingress controller with owide
kubectl get pod -o wide |grep ingress
myingress-ingress-nginx-controller-mn52x          1/1     Running   0          10h     192.168.1.138   dp     <none>           <none>
myingress-ingress-nginx-controller-zm7qz          1/1     Running   0          10h     192.168.0.77    cp     <none>           <none>
#we will curl one of them but it wont work cause we need to passe a header and spoof the hostname
curl 192.168.1.138
```
```html
<html>
<head><title>404 Not Found</title></head>
<body>
```
```sh
#lets check using ingress service (not admission)
kubectl get svc -o wide | grep ingress
myingress-ingress-nginx-controller             LoadBalancer   10.99.124.35    <pending>     80:32720/TCP,443:31053/TCP   10h     app.kubernetes.io/component=controller,app.kubernetes.io/instance=myingress,app.kubernetes.io/name=ingress-nginx
myingress-ingress-nginx-controller-admission   ClusterIP      10.102.59.239   <none>        443/TCP                      10h     app.kubernetes.io/component=controller,app.kubernetes.io/instance=myingress,app.kubernetes.io/name=ingress-nginx
```
Even if we curl the controller service we still need to add header, else we get 404:
```sh
curl 10.99.124.35
```
```html
<html>
<head><title>404 Not Found</title></head>
```

To simulate accessing a specific domain while using an IP address, you can use the `-H` option to add a custom `Host` header to your HTTP request. The `Host` header is required in every HTTP/1.1 request and tells the server which hostname you want to reach. For example:

```
GET / HTTP/1.1
Host: www.example.com
```

This is important because many servers host multiple domains (virtual hosts) on the same IP address. By specifying the `Host` header (e.g., with `-H "Host: www.example.com"`), you can make the server respond as if you accessed it via the actual domain name, even when connecting directly to its IP address.

```sh
#spoofing www.external.com on the svc ip 10.99.124.35
 curl -H "Host: www.external.com" http://10.99.124.35
 ```
 ```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```


## Admission Controllers in Kubernetes

Admission controllers are plugins that intercept requests to the Kubernetes API server **after authentication/authorization but before objects are persisted to etcd**. They can validate, mutate, or even deny requests.

### Types of Admission Webhooks

- **Validating Admission Webhooks**
    - Can reject objects if they fail custom logic.
    - *Example*: Prevent creating pods without resource limits.

- **Mutating Admission Webhooks**
    - Can modify objects before they’re saved.
    - *Example*: Inject sidecars (like Linkerd or Istio), add default labels, etc.

**Differences Between Mutating and Validating Webhooks**

| Feature           | Mutating Webhooks                                         | Validating Webhooks                                         |
|-------------------|----------------------------------------------------------|-------------------------------------------------------------|
| Purpose           | Modify the resource (mutate) before storing it.           | Validate the resource and decide if it should be accepted.   |
| Actions           | Can change resource fields (e.g., add annotations).       | Cannot modify the resource, only approve/reject it.          |
| Triggering Event  | Triggered on CREATE, UPDATE, and DELETE.                  | Triggered on CREATE, UPDATE, and DELETE.                     |
| Response          | Modifies the resource and sends back an updated version.  | Sends a pass/fail response based on the resource’s validity. |
| Example Use Case  | Injecting sidecar containers, adding default labels.      | Enforcing naming conventions or mandatory labels/annotations.|

### Common Use Cases
- Enforce security (e.g., block privileged containers)
- Inject env vars or init containers
- Auto-label or annotate resources
- Integrate with Gatekeeper, Kyverno, Linkerd, Istio

---

## Example: ValidatingWebhookConfiguration

```yaml
apiVersion: admissionregistration.k8s.io/v1  # API version for admission webhook configurations
kind: ValidatingWebhookConfiguration         # Specifies this is a Validating webhook (used for rejecting invalid resources)
metadata:
    name: validate-pods                        # Name of the webhook configuration object
webhooks:
    - name: example.webhook.k8s.io             # Unique name for the webhook, must be a DNS-compliant name
        rules:
            - apiGroups: [""]                      # Target the core API group (e.g., for pods, services, etc.)
                apiVersions: ["v1"]                  # Target version of the API; pods are v1 in the core group
                operations: ["CREATE"]               # Hook only triggers on CREATE operations
                resources: ["pods"]                  # Only apply this webhook to Pod resources
        clientConfig:
            service:
                name: webhook-service                # Kubernetes service name hosting the webhook
                namespace: webhook-namespace         # Namespace where the webhook service is running
                path: "/validate"                    # URL path that will handle validation requests
            caBundle: <base64-encoded-CA>          # Base64-encoded certificate authority to trust the webhook server
        failurePolicy: Fail                      # Optional - Fail the request if the webhook is unavailable
        admissionReviewVersions: ["v1"]          # AdmissionReview API versions supported by the webhook
```

### Kubernetes Validating Webhook Workflow

1. **Create ValidatingWebhookConfiguration**  
    Define a webhook with rules for validating resources (e.g., Pods, Deployments) in the cluster.

2. **Trigger Admission Control**  
    When a resource is created or updated, the Kubernetes API triggers the webhook validation process.

3. **Send Resource to Webhook**  
    The resource (e.g., Pod) is sent to the webhook endpoint for validation.

4. **Webhook Validation Logic**  
    The webhook server validates the resource based on custom rules (e.g., naming conventions, required labels).

5. **Admission Review Response**  
    The webhook responds with either approval (`allowed: true`) or rejection (`allowed: false`) along with an error message if applicable.

6. **Kubernetes API Server Action**  
    Based on the webhook response, the API server either proceeds with or rejects the resource creation/update.

7. **Monitor and Adjust**  
    Admins monitor webhook behavior and can adjust rules as needed.


>In our case we have the **K8S API Server**(Admission Control) -> sees **Ingress** *Create* -> send to -> **NGINX Admission WebHook** *ValidatingWebhookConfigurationbhook* -> Send to **Controller Admission Service** *myingress-ingress-nginx-controller-admission* -> send it to **NGINX Controller POD** on /webhook *myingress-ingress-nginx-controller-mn52x* for validation -> send back to SVC to WEBHOOK > Tells API Server its OK!

Get the nginx ValidationWebHook
```sh
kubectl get ValidatingWebhookConfiguration myingress-ingress-nginx-admission -o yaml
```
```yaml
apiVersion: admissionregistration.k8s.io/v1 # Specifies the API version for the ValidatingWebhookConfiguration resource.
kind: ValidatingWebhookConfiguration # Defines the type of Kubernetes resource (ValidatingWebhookConfiguration in this case).
metadata: # Metadata section contains information about the resource.
...

webhooks: # List of webhook configurations for validating admission requests.
- admissionReviewVersions: # Specifies the API versions supported for admission review requests and responses.
  - v1 # Indicates that version "v1" of admission review is supported.
  clientConfig: # Configuration for connecting to the webhook server.
    caBundle: .... # Base64-encoded CA certificate used to verify the webhook server's TLS certificate.
    service:
      name: myingress-ingress-nginx-controller-admission # Name of the Service that hosts the webhook server.
      namespace: default # Namespace where the Service is located.
      path: /networking/v1/ingresses # Path on the webhook server to send admission requests.
      port: 443 # Port on which the webhook server is listening.
  failurePolicy: Fail # Specifies the policy for handling webhook failures (Fail means reject the request if the webhook fails).
  matchPolicy: Equivalent # Specifies how the rules are matched (Equivalent means match resources with the same semantic meaning).
  name: validate.nginx.ingress.kubernetes.io # Name of the webhook configuration.
  namespaceSelector: {} # Empty selector means the webhook applies to all namespaces.
  objectSelector: {} # Empty selector means the webhook applies to all objects.
  rules: # Defines the rules for when the webhook is triggered.
  - apiGroups:
    - networking.k8s.io # Applies to the "networking.k8s.io" API group.
    apiVersions:
    - v1 # Applies to version "v1" of the API.
    operations:
    - CREATE # Trigger the webhook on CREATE operations.
    - UPDATE # Trigger the webhook on UPDATE operations.
    resources:
    - ingresses # Applies to "ingresses" resources.
    scope: '*' # Specifies the scope of the resources (e.g., cluster-wide or namespace-specific). '*' means all scopes.
  sideEffects: None # Indicates that the webhook has no side effects.
  timeoutSeconds: 10 # Specifies the timeout for the webhook call in seconds.
```
And the service that process the request and direct it to the pod:
```sh
kubectl get svc myingress-ingress-nginx-controller-admission -o yaml
```
```yaml
apiVersion: v1  # Specifies the API version for the resource (v1 for core Kubernetes objects like Service).
kind: Service  # Defines the type of Kubernetes resource (Service in this case).
metadata:  # Metadata section contains information about the resource.
  #....
  name: myingress-ingress-nginx-controller-admission  # Name of the Service resource.
  namespace: default  # Namespace where the Service is deployed.
  resourceVersion: "8162693"  # Internal version of the resource for tracking changes.
  uid: 69a6c3ba-1800-4d23-b1bf-b0b60fb3fa0b  # Unique identifier for the resource.
  #....
spec:  # Specification of the Service.
  clusterIP: 10.102.59.239  # Internal IP address assigned to the Service.
  clusterIPs:  # List of internal IPs assigned to the Service (for multi-IP families).
  - 10.102.59.239  # IPv4 address of the Service.
  internalTrafficPolicy: Cluster  # Determines how traffic is routed within the cluster.
  ipFamilies:  # Specifies the IP families supported by the Service.
  - IPv4  # Indicates the Service uses IPv4 addresses.
  ipFamilyPolicy: SingleStack  # Specifies that the Service uses a single IP family (IPv4).
  ports:  # Defines the ports exposed by the Service.
  - appProtocol: https  # Application protocol used by the port (HTTPS in this case).
    name: https-webhook  # Name of the port (used for identification).
    port: 443  # Port number exposed by the Service.
    protocol: TCP  # Protocol used by the port (TCP in this case).
    targetPort: webhook  # Target port on the pods to which traffic is forwarded.
  selector:  # Selector to identify the pods this Service routes traffic to.
    app.kubernetes.io/component: controller  # Matches pods with this label.
    app.kubernetes.io/instance: myingress  # Matches pods with this label.
    app.kubernetes.io/name: ingress-nginx  # Matches pods with this label.
  sessionAffinity: None  # Specifies whether session affinity is enabled (None means disabled).
  type: ClusterIP  # Type of Service (ClusterIP exposes the Service only within the cluster).
status:  # Current status of the Service.
  loadBalancer: {}  # Empty because this is not a LoadBalancer type Service.
```
Inside the pod the request is managed by :
```sh
kubectl get pod myingress-ingress-nginx-controller-mn52x -o yaml
```
```yaml
...
 ports:
    - containerPort: 80
      name: http
      protocol: TCP
    - containerPort: 443
      name: https
      protocol: TCP
    - containerPort: 8443 
      name: webhook # <--- manage the validation
      protocol: TCP
    readinessProbe:
      failureThreshold: 3
      httpGet:
        path: /healthz
        port: 10254
        scheme: HTTP
...
```

**LINKERD onto NGINX Controller**
Let us inject Linkerd annotation into nginx controller ingress 
```sh
kubectl get ds myingress-ingress-nginx-controller -o yaml | linkerd inject --ingress - | kubectl apply -f -

daemonset "myingress-ingress-nginx-controller" injected

Warning: resource daemonsets/myingress-ingress-nginx-controller is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
daemonset.apps/myingress-ingress-nginx-controller configured
```

#our pod have restarted and took in the configuration:
```sh
kubectl get pod | grep ingress
myingress-ingress-nginx-controller-mt2dt          2/2     Running   0          92s
myingress-ingress-nginx-controller-x8pgg          2/2     Running   0          62s
```

On linkerd interface 
![linkerd8.png](linkerd8.png)

Lets curl and sees what happen in Linkerd
```sh
for i in {1..250}; do curl -H "Host: www.external.com" http://10.99.124.35; done
```
![linkerd9.png](linkerd9.png)

At this stage, we’ll add one more server as an example—after that, the same process can be repeated as many times as needed to add additional servers.

Lets go inside web-two pod and modify the index HTML page
```sh
kubectl get pod | grep web-
web-one-8d48dcf57-dm2rr                           1/1     Running   0          25h
web-two-64558f4b75-qfzq8                          1/1     Running   0          25h
kubectl exec -it web-two-64558f4b75-qfzq8 -- /bin/bash
root@web-two-64558f4b75-qfzq8:/# apt-get update
root@web-two-64558f4b75-qfzq8:/# apt-get install -y vim
root@web-two-64558f4b75-qfzq8:/# vim /usr/share/nginx/html/index.html
```
```html
<!DOCTYPE html>
<html>
<head>
<title>INTERNAL WELCOME PAGE to OmarCluster</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>INTERNAL WELCOME PAGE to OmarCluster</h1>
```
```sh
root@web-two-64558f4b75-qfzq8:/# exit
#lets add the new server in our ingress with a new url
kubectl edit ingress ingress-test
```
```yaml
...
spec:
  ingressClassName: nginx
  rules:
  - host: internal.org
    http:
      paths:
      - backend:
          service:
            name: web-two
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
  - host: www.external.com
    http:
      paths:
      - backend:
          service:
            name: web-one
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
status:
  loadBalancer: {}
...
```

Lets curl it
```sh
for i in {1..50}; do curl -H "Host: internal.org" http://10.99.124.35; done
```
On linkerd
![linkerd10.png](linkerd10.png)
![linkerd11.png](linkerd11.png)

## GATEWAY API TP

### Install Gateway API for NGINX
As explained before, GATEWAY API is not part of k8s. Before using linkerd, we have already installed the GATEWAY CRDs custom resources definitions using:
```sh
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml

customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created
```
- Source: Kubernetes SIGs (official)
- Purpose: Standard Gateway API CRDs for any implementation
- Version: Manually selected (e.g., v1.3.0)

Now we will use the NGINX GATEWAY Custom tailord API
```sh
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.6.2" | kubectl apply -f -

customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io configured
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io configured
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io configured
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io configured
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io configured
```
- Source: NGINX Gateway Fabric
- Purpose: CRDs tailored for NGINX Gateway Fabric (v1.6.2)
- Usage: Ensures compatibility with NGINX Gateway Fabric

> Installing NGINX Gateway API CRDs after the official ones will override them. The CRDs (e.g., gateways.gateway.networking.k8s.io) are updated in place—no duplication occurs, but some fields may be replaced or updated, but do be carreful, it all depend on what controller you want to install

### install NGINX Gateway Fabric CRDs

**After install Gateway API tailord for NGINX we install NGINX Gateway Fabric CRDs**
```sh
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.1/deploy/crds.yaml
customresourcedefinition.apiextensions.k8s.io/clientsettingspolicies.gateway.nginx.org created
customresourcedefinition.apiextensions.k8s.io/nginxgateways.gateway.nginx.org created
customresourcedefinition.apiextensions.k8s.io/nginxproxies.gateway.nginx.org created
customresourcedefinition.apiextensions.k8s.io/observabilitypolicies.gateway.nginx.org created
customresourcedefinition.apiextensions.k8s.io/snippetsfilters.gateway.nginx.org created
customresourcedefinition.apiextensions.k8s.io/upstreamsettingspolicies.gateway.nginx.org created
```

### install NGINX Gateway Fabric Controller

**After we install NGINX Gateway Fabric CRDs we install the NGINX Gateway Fabric**
```sh
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.1/deploy/default/deploy.yaml
namespace/nginx-gateway created
serviceaccount/nginx-gateway created
clusterrole.rbac.authorization.k8s.io/nginx-gateway created
clusterrolebinding.rbac.authorization.k8s.io/nginx-gateway created
configmap/nginx-includes-bootstrap created
service/nginx-gateway created
deployment.apps/nginx-gateway created
gatewayclass.gateway.networking.k8s.io/nginx created
nginxgateway.gateway.nginx.org/nginx-gateway-config created
```

Verify the NGINX Gateway Fabric is runing:
```sh
kubectl get all -n nginx-gateway
NAME                                READY   STATUS    RESTARTS   AGE
pod/nginx-gateway-96f76cdcf-tlgmc   2/2     Running   0          44s

NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/nginx-gateway   LoadBalancer   10.104.234.221   <pending>     80:31192/TCP,443:31607/TCP   44s

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-gateway   1/1     1            1           44s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-gateway-96f76cdcf   1         1         1       44s
```

The NGINX Gateway Fabric service is initially of type `LoadBalancer`. While the external IP is pending, Kubernetes allocates two random ports on every node in the cluster. To access the NGINX Gateway Fabric, use the IP address of any cluster node along with these allocated ports.

For simplicity, you can change the service type to `NodePort`:

```sh
kubectl patch service/nginx-gateway -n nginx-gateway -p '{"spec": {"type": "NodePort"}}'
kubectl get service/nginx-gateway -n nginx-gateway
```

Deploy an application and expose it with a Kubernetes service named `books`. Next, define two Gateway API resources: a `Gateway` and an `HTTPRoute`. These resources will work together to route all HTTP requests targeting the hostname `shop.example.com` to the `books` service.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: books
spec:
  replicas: 2
  selector:
    matchLabels:
      app: books
  template:
    metadata:
      labels:
        app: books
    spec:
      containers:
      - name: book
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: books
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: books
```

dryrun on both client and server:
```sh
kubectl  apply -f books.yaml --dry-run=client
deployment.apps/books created (dry run)
service/books created (dry run)
kubectl  apply -f books.yaml --dry-run=server
deployment.apps/books created (server dry run)
service/books created (server dry run)
```
then create:
```sh
kubectl apply -f books.yaml
deployment.apps/books created
service/books created
kubectl get svc/books deploy/books

NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/books   ClusterIP   10.101.113.112   <none>        80/TCP    2m57s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/books   2/2     2            2           2m57s
```

To direct traffic to the books application, we'll set up a Gateway and an HTTPRoute. The Gateway acts as the entry point for HTTP traffic into the cluster. Specifically, we'll configure a shop Gateway to listen on port 80, allowing HTTP requests to reach the cluster and be routed to the books application.

```yaml
apiVersion: gateway.networking.k8s.io/v1 # API version of the resource.
kind: Gateway # Type of resource.
metadata: # Metadata info.
  name: shop # Gateway name.
spec: # Gateway spec.
  gatewayClassName: nginx # Gateway class.
  listeners: # Listener config.
  - name: http # Listener name.
    port: 80 # Listener port.
    protocol: HTTP # Listener protocol.
```
```sh
#create
kubectl apply -f gateway.yaml
gateway.gateway.networking.k8s.io/shop created
#check
kubectl get Gateway
NAME   CLASS   ADDRESS   PROGRAMMED   AGE
shop   nginx             True         11s
```

To route HTTP traffic from the gateway to the books service, we create an `HTTPRoute` resource and attach it to the gateway. It should define a routing rule that directs all traffic with the hostname `shop.example.com` from the gateway to the books service.
NGINX Gateway Fabric manages both the shop gateway and the books `HTTPRoute`. It automatically configures its data plane (NGINX) to route all HTTP requests for `shop.example.com` to the pods targeted by the books service.


```yaml
apiVersion: gateway.networking.k8s.io/v1 # API version of the resource.
kind: HTTPRoute # Type of resource.
metadata: # Metadata info.
  name: books # Name of the HTTPRoute.
spec: # Specification of the HTTPRoute.
  parentRefs: # References to parent gateways.
  - name: shop # Name of the parent gateway.
  hostnames: # Hostname for the route.
  - "shop.example.com" # Hostname value.
  rules: # Routing rules.
  - matches: # Match conditions.
    - path: # Path matching config.
        type: PathPrefix # Match paths with the given prefix.
        value: / # Path prefix to match.
    backendRefs: # Backend references.
    - name: books # Name of the backend service.
      port: 80 # Port of the backend service.
```
```sh
#create
kubectl apply -f httproute.yaml
httproute.gateway.networking.k8s.io/books created
#check
kubectl get HTTPRoute
NAME    HOSTNAMES              AGE
books   ["shop.example.com"]   10s
```

The Gateway API and HTTPRoute have been deployed successfully. To verify the configuration, lets send a request to the Node IP and node port of the NGINX gateway Fabric. 
First we send a request to the root path `/`. Since the `shop` HTTPRoute is configured to route all traffic—regardless of the path—to the `books` application, any requests (including those to `/`) should be handled by the `books` pods.

```bash
curl --resolve shop.example.com:31192:10.2.0.2 http://shop.example.com:31192/
```
- `curl`: Command-line tool to transfer data from or to a server.
- `--resolve shop.example.com:31192:10.2.0.2`: Instructs `curl` to resolve `shop.example.com` on port `31192` to the IP address `10.2.0.2`, bypassing DNS.
- `http://shop.example.com:31192/`: The URL to request. The hostname remains `shop.example.com`, but the connection is made to `10.2.0.2:31192`.

```html
<!--output-->
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

> useful for testing or debugging, especially when directing traffic to a specific server (such as in a staging environment) without modifying your system's DNS settings.

Requests to hostnames other than “shop.example.com” should not be routed to the books application, since the books HTTPRoute only matches requests with the shop.example.com” hostname. To verify this, send a request to the hostname “test.example.com”:
```bash
curl --resolve test.example.com:31192:10.2.0.2 http://test.example.com:31192/
```bash
```
```html
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

# SCHEDULING

## kube-scheduler

As Kubernetes deployments grow in size and complexity, effective scheduling becomes increasingly important. The **kube-scheduler** is responsible for deciding which nodes will run each Pod, using topology-aware *(take into account the physical or logical layout (topology))* algorithms to optimize placement.

Users can assign priorities to Pods, allowing higher-priority Pods to evict lower-priority ones during resource shortages. This ensures critical workloads are scheduled under heavy load.

The scheduler maintains awareness of all nodes in the cluster, filtering and scoring them to select the most suitable node for each Pod. Once a decision is made, the Pod specification is sent to the kubelet on the chosen node for creation.

Scheduling decisions can be influenced using labels and selectors on nodes and Pods. Features such as **affinity**, **taints**, **tolerations**, and **pod bindings** allow fine-grained control over where Pods are scheduled. For example, tolerations let a Pod be scheduled on a node with a taint that would otherwise prevent scheduling.

Not all scheduling constraints are strict. Affinity rules may prefer nodes but allow others if needed. Some settings, like `requiredDuringScheduling` and `requiredDuringExecution`, can trigger Pod eviction if conditions change.

For advanced use cases, you can implement and deploy a custom scheduler tailored to your cluster's specific needs.

## Node Selection in kube-scheduler

The kube-scheduler selects a Node for a Pod through two main stages: **Filtering** and **Scoring**.

![Filtering+Scoring.png](Filtering+Scoring.png)

![scheduler-worlkflow.gif](scheduler-worlkflow.gif)

### Filtering

During the Filtering stage, the scheduler evaluates all available Nodes to determine which ones can accommodate the Pod's requirements. For example, the `PodFitsResources` filter checks if a Node has **enough resources** ***(CPU, memory, etc.)*** to meet the Pod’s requests and limits. Only Nodes that pass all filters are included in the candidate list. If no Nodes are suitable, the list stay empty > the Pod remains unscheduled.

### Scoring

In the Scoring stage, the scheduler assigns a score to each candidate Node based on various criteria defined in the scheduler’s configuration. The Node with the highest score is selected as the placement for the Pod.

> **Note:** You can customize the filtering and scoring behavior of the scheduler using **scheduling configuration profiles.**

You can customize the kube-scheduler by providing a configuration file and specifying its path as a command-line argument. The scheduler supports **scheduling profiles**, which let you configure the various stages of the scheduling process. Each stage, known as an **extension point**, can be extended or modified using plugins.

Scheduling profiles define how plugins interact with these extension points, allowing you to tailor the scheduler's behavior to your needs. There are twelve extension points in total, each representing a stage in the scheduling workflow where plugins can influence the scheduler's decisions.

## Extension Points

Kubernetes scheduling supports several extension points, each allowing plugins to customize the scheduling process:

1. **queueSort**  
   Provides an ordering function to sort pending Pods in the scheduling queue. Only one queueSort plugin can be enabled at a time.  
2. **preFilter**  
   Pre-processes or checks information about a Pod or the cluster before filtering. Can mark a Pod as unschedulable.  
3. **filter**  
   Equivalent to Predicates in a scheduling policy. Filters out nodes that cannot run the Pod, called in the configured order. If no nodes pass all filters, the Pod is marked as unschedulable.  
4. **postFilter**  
   Invoked in order when no feasible nodes are found for a Pod. If any postFilter plugin marks the Pod as schedulable, the remaining plugins are skipped.  
5. **preScore**  
   An informational extension point for performing work before scoring.  
6. **score**  
   Assigns a score to each node that passed filtering. The scheduler selects the node with the highest weighted score sum.  
7. **reserve**  
   Notifies plugins when resources are reserved for a Pod. Also implements an Unreserve call if a failure occurs during or after reservation.  
8. **permit**  
   Can prevent or delay the binding of a Pod.  
9. **preBind**  
   Performs any required work before a Pod is bound.  
10. **bind**  
    Binds a Pod to a Node. Called in order; once a plugin completes the binding, the rest are skipped. At least one bind plugin is required.  
11. **postBind**  
    An informational extension point called after a Pod has been bound.  
12. **multiPoint**  
    A configuration-only field that enables or disables plugins for all applicable extension points simultaneously.

## Scheduling Plugins

The following plugins are enabled by default and implement one or more scheduling extension points in Kubernetes:

- **ImageLocality**  
    Favors nodes that already have the container images required by the Pod.  
    _Extension points:_ `score`

- **TaintToleration**  
    Handles taints and tolerations for Pods and nodes.  
    _Extension points:_ `filter`, `preScore`, `score`

- **NodeName**  
    Ensures the Pod's specified node name matches the current node.  
    _Extension points:_ `filter`

- **NodePorts**  
    Checks if a node has available ports for the Pod's requested ports.  
    _Extension points:_ `preFilter`, `filter`

- **NodeAffinity**  
    Implements node selectors and node affinity rules.  
    _Extension points:_ `filter`, `score`

- **PodTopologySpread**  
    Distributes Pods across topology domains to improve availability.  
    _Extension points:_ `preFilter`, `filter`, `preScore`, `score`

- **NodeUnschedulable**  
    Filters out nodes marked as unschedulable (`.spec.unschedulable: true`).  
    _Extension points:_ `filter`

- **NodeResourcesFit**  
    Checks if the node has sufficient resources for the Pod's requirements.

- **NodeResourcesBalancedAllocation**  
    Prefers nodes that result in more balanced resource usage when scheduling the Pod.  
    _Extension points:_ `score`

For more details, see the [Kubernetes scheduling plugins documentation](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins).

## Multiple Scheduler Profiles

You can configure `kube-scheduler` to run with multiple scheduling profiles. Each profile must have a unique scheduler name and can use a different set of plugins.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
    - schedulerName: default-scheduler
    - schedulerName: custom-scheduler
        plugins:
            preFilter:
                disabled:
                    - name: '*'
            filter:
                disabled:
                    - name: '*'
            postFilter:
                disabled:
                    - name: '*'
```

In this example, the scheduler runs with two profiles:

- **default-scheduler**: Uses the default set of plugins.
- **custom-scheduler**: Disables all preFilter, filter, and postFilter plugins.

To use a specific profile, set the corresponding scheduler name in your Pod specification under `.spec.schedulerName`:

```yaml
spec:
    schedulerName: custom-scheduler
```

**Notes:**
- By default, a profile named `default-scheduler` is created with the standard plugins.
- Each profile must have a unique `schedulerName`.
- If `.spec.schedulerName` is not set in a Pod, the `kube-apiserver` assigns `default-scheduler`. Ensure a profile with this name exists to schedule such Pods.

## Pod Specification

Most scheduling decisions in Kubernetes are defined within the Pod specification (`PodSpec`). The PodSpec includes several key fields that influence how and where Pods are scheduled:

- **nodeName**
- **nodeSelector**
- **affinity**
- **schedulerName**
- **tolerations**

### nodeName and nodeSelector

- `nodeName`: Directly assigns a Pod to a specific node by name.
- `nodeSelector`: Assigns a Pod to nodes that match specific label criteria, allowing selection of a group of nodes.

### Affinity and Anti-Affinity

- **Affinity**: Specifies rules about which nodes a Pod should (or should not) be scheduled on, based on labels. Affinity can be required or preferred.
- **Anti-affinity**: Prevents Pods from being scheduled on certain nodes, often to avoid co-locating specific workloads.

If a preferred rule is used, the scheduler will try to honor it, but may schedule the Pod elsewhere if no matching nodes are available.

### Taints and Tolerations

- **Taints**: Mark nodes so that Pods are not scheduled onto them unless they tolerate the taint
- **Tolerations**: Allow Pods to be scheduled onto tainted nodes if they meet the toleration criteria.

### schedulerName

If the default scheduler does not meet your requirements, you can deploy a custom scheduler. By setting the `schedulerName` field in the PodSpec, you can specify which scheduler should be used for that Pod.

## NodeSelector vs Affinity Taints

![NodeSelector_Affinity_Tains.jpg](NodeSelector_Affinity_Tains.jpg)

## Specifying the Node Label

The `nodeSelector` field in a Pod specification allows you to schedule Pods onto specific nodes by matching one or more key-value pairs against node labels.

```yaml
spec:
    containers:
        - name: redis
            image: redis
    nodeSelector:
        net: fast
```

This Pod will only run on nodes labeled `net=fast` (just an example). If no such node exists, the Pod stays `Pending`. For more flexible scheduling, use node affinity.

## Scheduler Profiles

You can also configure the scheduler using scheduling profiles, they let you customize the scheduler by specifying which plugins run at various extension points during the scheduling process.

An extension point is a specific stage in the scheduler’s workflow where plugins can be invoked to influence scheduling decisions. The twelve extension points are:

- `queueSort`
- `preFilter`
- `filter`
- `postFilter`
- `preScore`
- `score`
- `reserve`
- `permit`
- `preBind`
- `bind`
- `postBind`
- `multiPoint`

Many plugins—some default, others optional—affect how the scheduler selects nodes. See the [Kubernetes documentation](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins) for a full list.

Common scheduler plugins and their extension points:

- **ImageLocality**: Favors nodes with required images (`score`).
- **TaintToleration**: Handles taints/tolerations (`filter`, `preScore`, `score`).
- **NodeName**: Matches Pod node name (`filter`).
- **NodePorts**: Checks node port availability (`preFilter`, `filter`).
- **NodeAffinity**: Implements node selectors/affinity (`filter`, `score`).
- **PodTopologySpread**: Enforces topology spread (`preFilter`, `filter`, `preScore`, `score`).
- **NodeUnschedulable**: Skips unschedulable nodes (`filter`).
- **NodeResourcesFit**: Checks node resources (`preFilter`, `filter`, `score`).
- **NodeResourcesBalancedAllocation**: Balances resource usage (`score`).
- **VolumeBinding**: Handles volume binding (`preFilter`, `filter`, `reserve`, `preBind`, `score`).
- **VolumeRestrictions/Zone/NodeVolumeLimits/EBSLimits/GCEPDLimits/AzureDiskLimits**: Enforce volume constraints (`filter`).
- **InterPodAffinity**: Enforces inter-Pod affinity/anti-affinity (`preFilter`, `filter`, `preScore`, `score`).
- **PrioritySort**: Default queue sorting (`queueSort`).
- **DefaultBinder**: Default binding (`bind`).
- **DefaultPreemption**: Default preemption (`postFilter`).

> Some plugins (e.g., `VolumeBinding` with StorageCapacityScoring) require specific features enabled.

A scheduler can support multiple profiles simultaneously, which can eliminate the need to run multiple schedulers. Each `PodSpec` can specify which profile to use; if not specified, the `default-scheduler` profile is used.

## Pod Affinity Rules

![Podaffinity.png](Podaffinity.png)

Affinity places Pods that communicate or share data on the same node, while anti-affinity spreads them across nodes for fault tolerance. These rules use Pod labels and operators (`In`, `NotIn`, `Exists`, `DoesNotExist`) to guide scheduling. In large clusters, evaluating these rules can affect scheduler performance.

### requiredDuringSchedulingIgnoredDuringExecution

When you use `requiredDuringSchedulingIgnoredDuringExecution`, the scheduler enforces a hard rule: the Pod will only be scheduled on a node if the specified condition is true. If the condition later becomes false, the Pod continues running on that node.

### preferredDuringSchedulingIgnoredDuringExecution

With `preferredDuringSchedulingIgnoredDuringExecution`, the scheduler tries to place the Pod on a node that matches the preference, but if none are available, it will still schedule the Pod elsewhere. This is a soft rule, expressing a preference rather than a strict requirement.

### podAffinity

`podAffinity` tells the scheduler to try to co-locate Pods on the same node, based on matching labels.

### podAntiAffinity

`podAntiAffinity` instructs the scheduler to spread Pods across different nodes, avoiding co-location.

## Pod Affinity Example

In this example of `affinity` and `podAffinity` settings in Pod specification, the Pod will only be scheduled onto a node that is already running another Pod with a specific label. The requirement is enforced at scheduling time, but if the label is ***later removed***, the Pod will ***not be evicted***.

```yaml
spec: # The specification of the Pod
  affinity: # Affinity rules for scheduling the Pod
    podAffinity: # Specifies Pod affinity rules
      requiredDuringSchedulingIgnoredDuringExecution: # These rules must be met during scheduling but are ignored during execution
      - labelSelector: # Selector to match labels on other Pods
          matchExpressions: # List of label matching expressions
          - key: security # The label key to match
            operator: In # The operator to use for matching (e.g., In, NotIn, Exists, etc.)
            values: # The list of values to match for the key
            - S1 # The value of the label key that must be present
```
**Explanation:**  
The Pod is scheduled only if another Pod with the label `security: S1` exists on the node; otherwise, it stays `Pending`.

## Using `podAntiAffinity`

**anti-affinity rules** with `podAntiAffinity` in Kubernetes is used to prevent pods from running on the same node as others pods with specific labels, helping to spread workloads and avoid single points of failure.

tells the scheduler to *prefer* avoiding nodes that are running pods labeled with `security: S2`:

```yaml
podAntiAffinity: # Defines rules for avoiding scheduling pods on the same node as other pods with specific labels
  preferredDuringSchedulingIgnoredDuringExecution: # Indicates a preference for scheduling but not a strict requirement
  - weight: 100 # Assigns a weight to this preference; higher weight means higher priority
    podAffinityTerm: # Specifies the conditions for pod anti-affinity
      labelSelector: # Defines the label-based selection criteria
        matchExpressions: # Specifies a list of label matching rules
        - key: security # The label key to match
          operator: In # The operator to use for matching; "In" means the value must be in the specified list
          values: # The list of acceptable values for the label
          - S2 # The specific value of the "security" label to match
```
**Explanation**
Here we tell the scheduler to *prefer* avoiding nodes that are running pods labeled with `security: S2`

- **`preferredDuringSchedulingIgnoredDuringExecution`**: Scheduler prefers to avoid matching nodes but may still schedule there if needed.

- **`weight`**: Assigns a priority (1–100). Higher weights mean the scheduler will give more importance to avoiding nodes that match the condition. the scheduler will prioritize nodes with the lowest combined weight. 

For instance, if one node has a pod with a podAntiAffinity rule weighted at 100, and another node has a pod with a podAntiAffinity rule weighted at 50, the scheduler will prefer the node with the lower combined weight (in this case, the node with weight 50). This is because the scheduler tries to minimize the total weight, so nodes with lower anti-affinity weights are more likely to be selected for scheduling.

![antiaffinitywieght.png](antiaffinitywieght.png)
![weight_explained.png](weight_explained.png)


> **Tip:** Use multiple weighted anti-affinity rules to fine-tune pod placement. The scheduler prefers nodes with the lowest combined weight, balancing workloads without strict constraints.

## Node Affinity Rules

Node affinity allows you to control Pod scheduling based on node labels, rather than the presence or absence of other Pods (as with Pod affinity/anti-affinity). This approach is similar to using `nodeSelector`, but is more flexible and is expected to eventually replace `nodeSelector`.

### Supported Operators

- `In`
- `NotIn`
- `Exists`
- `DoesNotExist`

### Types of Node Affinity

- **`requiredDuringSchedulingIgnoredDuringExecution`**: Hard requirement for scheduling. The Pod will only be scheduled on nodes that meet the criteria.
- **`preferredDuringSchedulingIgnoredDuringExecution`**: Soft preference. The scheduler will try to place the Pod on a matching node, but will still schedule it elsewhere if necessary.
- **Planned for future:** `requiredDuringSchedulingRequiredDuringExecution`.

> **Note:** Until `nodeSelector` is fully deprecated, both `nodeSelector` and `requiredDuringSchedulingIgnoredDuringExecution` rules must be satisfied for a Pod to be scheduled.

### Node Affinity Example

```yaml
spec: # Specifies the configuration for the pod or workload
  affinity: # Defines rules for scheduling pods based on affinity or anti-affinity
    nodeAffinity: # Specifies rules for scheduling pods to specific nodes
      preferredDuringSchedulingIgnoredDuringExecution: # Indicates a preference for scheduling but not a strict requirement
      - weight: 1 # Assigns a low priority to this preference; higher weights mean higher priority
        preference: # Specifies the conditions for the preferred node
          matchExpressions: # Defines label-based selection criteria
          - key: diskspeed # The label key to match on the node
            operator: In # The operator to use for matching; "In" means the value must be in the specified list
            values: # The list of acceptable values for the label
            - quick # One of the acceptable values for the "diskspeed" label
            - fast # Another acceptable value for the "diskspeed" label
```

This configuration sets a preferred node affinity for pods. It increases the likelihood that the pod will be scheduled on nodes labeled with `diskspeed: quick` or `diskspeed: fast`. However, if no such nodes are available, the pod can still be scheduled on other nodes. The `weight` determines the preference strength for matching nodes, **weight** refers to a numerical value assigned to a node (or a rule, option, etc.):

- **Higher weight**: The node is more likely to be selected or matched.
- **Lower weight**: The node is less likely to be chosen.

**Analogy:** for your favorite snack. If you give "chocolate" a weight of 10 and "chips" a weight of 2, chocolate is much more likely to win.

![node_affinity.png](node_affinity.png)

## Taints

A node with a **taint** repels Pods that lack a matching toleration. Taints are specified as `key=value:effect`, where both key and value are defined by the administrator.
If a Pod does not have a **toleration** for a node's taint, the scheduler will avoid placing the Pod on that node.

### Taint Scheduling Effects

Kubernetes defines three main effects for handling Pod scheduling with taints and tolerations:

#### 1. `NoSchedule`
- **Description:** The scheduler will not place a Pod on this node unless the Pod has a matching toleration.
- **Note:** Existing Pods already running on the node are not affected, regardless of their tolerations.

#### 2. `PreferNoSchedule`
- **Description:** The scheduler will try to avoid placing a Pod on this node unless there are no other suitable (untainted) nodes available.
- **Note:** Existing Pods are not affected.

#### 3. `NoExecute`
- **Description:** 
    - Existing Pods without a matching toleration will be evicted from the node.
    - New Pods without a matching toleration will not be scheduled on the node.
- **Special Case:** 
    - If a Pod has a `tolerationSeconds` value set, it will remain on the node for that duration before being evicted.
    - For certain node issues, the kubelet may automatically add a 300-second toleration to prevent unnecessary evictions.

![taint_effect.png](taint_effect.png)

> If a node has multiple taints, the scheduler ignores any taints that have matching tolerations on a pod. Only taints without corresponding tolerations will impact pod scheduling.

**Note:** The use of `TaintBasedEvictions` is still an **alpha feature** in Kubernetes.

When a node has resource pressure, the kubelet adds taints (like `MemoryPressure`, `DiskPressure`, or `PIDPressure`). With `TaintBasedEvictions` enabled, only pods with matching tolerations stay; others are evicted. This allows for controlled, gradual evictions.

**Example:**
```yaml
# Node tainted by kubelet due to memory pressure
kubectl describe node <node-name>
# Output includes:
Taints: node.kubernetes.io/memory-pressure:NoSchedule
```

## Tolerations

Tolerations allow Pods to be scheduled onto nodes with specific taints. This mechanism helps control which Pods can run on certain nodes, ensuring that only Pods with matching tolerations are scheduled on tainted nodes.

### How Tolerations Work

- **Taints** are applied to nodes to repel Pods that do not have matching tolerations.
- **Tolerations** are applied to Pods, allowing them to be scheduled on nodes with matching taints.

### Toleration Operators

- The `operator` field in a toleration can be set to `Equal` (default) or `Exists`.
    - `Equal` requires both `key` and `value` to match the taint.
    - `Exists` only requires the `key` to match; if the key is empty, it matches all taints.
- If only `key` and `operator` are specified (without `effect`), the toleration matches all effects for that key.

### Example

```yaml
tolerations:
- key: "server"
    operator: "Equal"
    value: "ap-east"
    effect: "NoExecute"
    tolerationSeconds: 3600
```

In this example, the Pod will tolerate a taint with:

- `key`: `server`
- `value`: `ap-east`
- `effect`: `NoExecute`

The taint on the node :

```yaml
kind: Node # Indicates that this is a Node resource
metadata: # Metadata about the node
  name: node1 # The name of the node
spec: # Specifies the configuration for the node
  taints: # Defines taints applied to the node
  - key: "server" # The key for the taint
    value: "ap-east" # The value for the taint
    effect: "NoExecute"  # The effect of the taint; "NoSchedule" means pods without a matching toleration won't be scheduled on this node
```
The Pod will remain running on the tainted node for up to **3600 seconds** after the taint is applied. After this period, the Pod will be evicted from the node.

> **Note:** Tolerations do not guarantee scheduling on tainted nodes; they only allow it if other scheduling requirements are met.

![example_toleration1.png](example_toleration1.png)

![TolerationExample.png](TolerationExample.png)

## Custom Scheduler

Kubernetes provides default scheduling mechanisms such as **affinity**, **taints**, and **policies**. However, if these are not flexible enough for your requirements, you can implement your own custom scheduler.

> **Note:** Developing a custom scheduler is an advanced topic and is outside the scope of this course. To get started, you can explore the [Kubernetes Scheduler source code](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler) on GitHub.

### How Scheduling Works

- If a **Pod** specification does **not** declare a scheduler, the **default scheduler** is used.
- If a Pod specifies a custom scheduler and that scheduler is not running, the Pod will remain in the `Pending` state indefinitely.

The outcome of scheduling is a **binding** that assigns a Pod to a specific Node. A binding is a Kubernetes API object in the `api/v1` group.  
You can manually schedule a Pod by creating a binding, even if no scheduler is running.

> **Tip:** It is possible to run multiple schedulers in a cluster at the same time.

### Viewing Scheduler Events

To view scheduler-related events and other cluster information, use:

```sh
kubectl get events
```

## Summary video

[Watch on YouTube](https://youtu.be/rX4v_L0k4Hc?si=qIRFyE78HMQ6npgq)
<iframe width="560" height="315" src="https://www.youtube.com/embed/rX4v_L0k4Hc?si=qIRFyE78HMQ6npgq" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

# SCHEDULING TP

## Assign Pods Using Labels

First lets get the number of pod and image running inside each of our nodes:

```sh
#Nodes
NAME   STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
cp     Ready    control-plane   51d   v1.32.1   10.2.0.2      <none>        Ubuntu 20.04.6 LTS   5.15.0-1075-gcp   containerd://1.7.25
dp     Ready    <none>          47d   v1.32.1   10.2.0.3      <none>        Ubuntu 20.04.6 LTS   5.15.0-1075-gcp   containerd://1.7.25
#get pods and pip WC after
kubectl get pods --all-namespaces -o wide | grep cp
default         books-5bcf5ccfd7-bz8pq                            1/1     Running   0              28h     192.168.0.90    cp     <none>           <none>
default         myingress-ingress-nginx-controller-mt2dt          2/2     Running   0              2d5h    192.168.0.19    cp     <none>           <none>
default         web-one-8d48dcf57-dm2rr                           1/1     Running   0              3d      192.168.0.26    cp     <none>           <none>
default         web-two-64558f4b75-qfzq8                          1/1     Running   0              3d      192.168.0.185   cp     <none>           <none>
kube-system     cilium-envoy-2xs9m                                1/1     Running   0              48d     10.2.0.2        cp     <none>           <none>
kube-system     cilium-h9s45                                      1/1     Running   0              48d     10.2.0.2        cp     <none>           <none>
kube-system     cilium-operator-5c7867ccd5-24jvr                  1/1     Running   2 (42d ago)    42d     10.2.0.2        cp     <none>           <none>
kube-system     coredns-7c65d6cfc9-fpkpr                          1/1     Running   0              5d23h   192.168.0.102   cp     <none>           <none>
kube-system     coredns-7c65d6cfc9-fxt75                          1/1     Running   0              5d23h   192.168.0.164   cp     <none>           <none>
kube-system     etcd-cp                                           1/1     Running   1 (42d ago)    42d     10.2.0.2        cp     <none>           <none>
kube-system     kube-apiserver-cp                                 1/1     Running   0              42d     10.2.0.2        cp     <none>           <none>
kube-system     kube-controller-manager-cp                        1/1     Running   0              42d     10.2.0.2        cp     <none>           <none>
kube-system     kube-proxy-hx2xw                                  1/1     Running   0              42d     10.2.0.2        cp     <none>           <none>
kube-system     kube-scheduler-cp                                 1/1     Running   0              42d     10.2.0.2        cp     <none>           <none>
linkerd-viz     tap-5b55c9bb76-6c8r5                              2/2     Running   1 (3d5h ago)   3d5h    192.168.0.46    cp     <none>           <none>
linkerd-viz     web-755b796446-jvpjn                              2/2     Running   0              3d4h    192.168.0.32    cp     <none>           <none>
nginx-gateway   nginx-gateway-96f76cdcf-tlgmc                     2/2     Running   0              41h     192.168.0.29    cp     <none>           <none>
#wc
kubectl get pods --all-namespaces -o wide | grep cp  | wc -l
17
#get image with ps
sudo crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD
72617e47cc168       a830707172e80       29 hours ago        Running             book                      0                   8ae35a1c95d17       books-5bcf5ccfd7-bz8pq
08feca0eb98de       6d3c4dc55e9d2       41 hours ago        Running             nginx                     0                   ee41586606f4c       nginx-gateway-96f76cdcf-tlgmc
8bd5d261da549       8c4cd00e239ee       41 hours ago        Running             nginx-gateway             0                   ee41586606f4c       nginx-gateway-96f76cdcf-tlgmc
2d693a4b30c7e       78e25eaa557d4       2 days ago          Running             controller                0                   8eba0d0087211       myingress-ingress-nginx-controller-mt2dt
6ebdffc27c291       f941e69465d6c       2 days ago          Running             linkerd-proxy             0                   8eba0d0087211       myingress-ingress-nginx-controller-mt2dt
498c7597339b0       a830707172e80       3 days ago          Running             nginx                     0                   3cccc66ef5bfa       web-two-64558f4b75-qfzq8
64e153def1539       a830707172e80       3 days ago          Running             nginx                     0                   4263cd8a9aa45       web-one-8d48dcf57-dm2rr
eeafb5b3e0072       b7af74c87ced5       3 days ago          Running             web                       0                   ed69d330ded9d       web-755b796446-jvpjn
4ca36bd0e167c       f941e69465d6c       3 days ago          Running             linkerd-proxy             0                   ed69d330ded9d       web-755b796446-jvpjn
b8eed1bd0178d       dfe7111f42831       3 days ago          Running             tap                       1                   9dc3d141950b1       tap-5b55c9bb76-6c8r5
76d7a82b1ae6a       f941e69465d6c       3 days ago          Running             linkerd-proxy             0                   9dc3d141950b1       tap-5b55c9bb76-6c8r5
150957e81001f       c69fa2e9cbf5f       5 days ago          Running             coredns                   0                   7c58d4f788f12       coredns-7c65d6cfc9-fpkpr
125a06e5cf4c2       c69fa2e9cbf5f       5 days ago          Running             coredns                   0                   b6d040162b6cd       coredns-7c65d6cfc9-fxt75
3a075cbcc0f15       d3cf9fb7f3cba       6 weeks ago         Running             cilium-operator           2                   49aad977c3a77       cilium-operator-5c7867ccd5-24jvr
60fecaa370ad5       a9e7e6b294baf       6 weeks ago         Running             etcd                      1                   c6bddf9d47237       etcd-cp
814ea9005b0aa       019ee182b58e2       6 weeks ago         Running             kube-controller-manager   0                   6fd7c005c4bd6       kube-controller-manager-cp
150a2d0bba8e7       95c0bda56fc4d       6 weeks ago         Running             kube-apiserver            0                   700ba10975ae6       kube-apiserver-cp
43617b6aefa37       2b0d6572d062c       6 weeks ago         Running             kube-scheduler            0                   487c3ba52a291       kube-scheduler-cp
e273739e9cfd4       e29f9c7391fd9       6 weeks ago         Running             kube-proxy                0                   d19da242923d0       kube-proxy-hx2xw
9240a61216a33       b9d596d6e2d4f       6 weeks ago         Running             cilium-envoy              0                   74fb629846b25       cilium-envoy-2xs9m
d9acf630d32d5       119e6111c0e41       6 weeks ago         Running             cilium-agent              0                   e50989e224e8b       cilium-h9s45
#wc it
crictl ps | wc -l
22
#on dp and with only WC we have
crictl ps | wc -l
34
kubectl get pods --all-namespaces -o wide | grep dp  | wc -l
19
```
> the number of image is more than the pod cause some pods use 2 images

**>> if you have issue using `crictl`, validate that you have the correct `runtime-endpoint: "unix:///run/containerd/containerd.sock"` in `"/etc/crictl.yaml"`**

```sh
Lets label nodes by giving the `CP` > VIP Hardware, and the `DP` > Others
#labeling CP
kubectl label node cp status=vip
node/cp labeled
#labeling DP
kubectl label node dp status=other
node/dp labeled
#check if label applied
kubectl get nodes --show-labels
NAME   STATUS   ROLES           AGE   VERSION   LABELS
cp     Ready    control-plane   51d   v1.32.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=cp,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=,status=vip
dp     Ready    <none>          47d   v1.32.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=dp,kubernetes.io/os=linux,status=other,system=secondOne
```

Let us create a pod with 4 containers and use `nodeSelector`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vip
spec:
  containers:
  - name: vip1
    image: busybox
    args:
    - sleep
    - "1000000"
  - name: vip2
    image: busybox
    args:
    - sleep
    - "1000000"
  - name: vip3
    image: busybox
    args:
    - sleep
    - "1000000"
  - name: vip4
    image: busybox
    args:
    - sleep
    - "1000000"
  nodeSelector:
    status: vip
```
```sh
#after apply dry run and applying the file
kubectl apply -f vip.yaml --dry-run=client
pod/vip created (dry run)
kubectl apply -f vip.yaml --dry-run=server
pod/vip created (server dry run)
kubectl apply -f vip.yaml
pod/vip created
#get
kubectl get pod -o wide

NAME                                              READY   STATUS    RESTARTS   AGE    IP              NODE   NOMINATED NODE
vip                                               4/4     Running   0          116m   192.168.0.114   cp     <none>
#crictl to check number of image, we now have 26 instead of 22
crictl ps | wc -l
26
```

Lets now delete the pod and delete the nodeSelector part and create the pod again
```sh
kubectl delete pod vip
pod "vip" deleted
vim vip.yaml
```
```yaml
#  nodeSelector: 
#    status: vip
```
Apply and check where the pod are runing:
```sh
kubectl apply -f vip.yaml
kubectl get pod
vip                                               4/4     Running   0          53s
#on CP we have now 22 back to the original
crictl ps | wc -l
22
#on worker/DP is another story we now have the 4 image added to DP
root@dp:~# crictl ps | wc -l
38
```

Lets now create a new pod called other by copying the old one and uncommenting the nodeSelector part :
```sh
#copy
cp vip.yaml other.yaml
#sed to swap vip with other
sed -i s/vip/other/g other.yaml
#uncomment
vim other.yaml
#apply
kubectl apply -f other.yaml
pod/other created
#check
NAME                                              READY   STATUS    RESTARTS   AGE
other                                             4/4     Running   0          56s
vip                                               4/4     Running   0          7m30s
# on the worker/DP machine we now have the pods: going from 38 to 42
root@dp:~# crictl ps | wc -l
42
```

Clean everything
```sh
kubectl delete pod vip other
pod "vip" deleted
pod "other" deleted
```

# Using Taints to Control Pod Deployment

Use taints to control where Pods run. Taints can limit or prevent Pods from running on certain nodes. We'll use three taints to manage Pod placement.

first we create following deployement:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taint-deployment
spec:
  replicas: 8
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20.1
        ports:
        - containerPort: 80
```
After we create it we check deployment and pod :
```sh
#create
kubectl apply -f taint.yaml
deployment.apps/taint-deployment created
#get deploy and wait for all replicas to deploy
kubectl get deployments.apps
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
taint-deployment                  4/8     8            4           9s
#get pod owide
kubectl get pod -o wide
NAME                                              READY   STATUS    RESTARTS   AGE     IP              NODE   NOMINATED NODE   READINESS GATES
taint-deployment-787bd7bfb4-7sw4l                 1/1     Running   0          5m      192.168.1.191   dp     <none>           <none>
taint-deployment-787bd7bfb4-blmc6                 1/1     Running   0          5m      192.168.0.103   cp     <none>           <none>
taint-deployment-787bd7bfb4-cv99d                 1/1     Running   0          5m      192.168.1.149   dp     <none>           <none>
taint-deployment-787bd7bfb4-hlmdl                 1/1     Running   0          5m      192.168.0.200   cp     <none>           <none>
taint-deployment-787bd7bfb4-nrdk9                 1/1     Running   0          5m      192.168.1.91    dp     <none>           <none>
taint-deployment-787bd7bfb4-qwxjz                 1/1     Running   0          5m      192.168.1.123   dp     <none>           <none>
taint-deployment-787bd7bfb4-r54hg                 1/1     Running   0          5m      192.168.0.212   cp     <none>           <none>
taint-deployment-787bd7bfb4-rgjp4                 1/1     Running   0          5m      192.168.0.50    cp     <none>           <none>
#inside cp we crictl :
crictl ps | grep -e nginx -e IMAGE
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD
3b4476abd0a4c       c8d03f6b8b915       7 minutes ago       Running             nginx                     0                   d6c43c01d2a8e       taint-deployment-787bd7bfb4-r54hg
a4e44c27c124e       c8d03f6b8b915       7 minutes ago       Running             nginx                     0                   fab51c1610e2e       taint-deployment-787bd7bfb4-blmc6
8dbe02c314d99       c8d03f6b8b915       7 minutes ago       Running             nginx                     0                   3eab204217a1e       taint-deployment-787bd7bfb4-hlmdl
d61713cd927e7       c8d03f6b8b915       7 minutes ago       Running             nginx                     0                   9a5e21f010de8       taint-deployment-787bd7bfb4-rgjp4
08feca0eb98de       6d3c4dc55e9d2       2 days ago          Running             nginx                     0                   ee41586606f4c       nginx-gateway-96f76cdcf-tlgmc
8bd5d261da549       8c4cd00e239ee       2 days ago          Running             nginx-gateway             0                   ee41586606f4c       nginx-gateway-96f76cdcf-tlgmc
2d693a4b30c7e       78e25eaa557d4       2 days ago          Running             controller                0                   8eba0d0087211       myingress-ingress-nginx-controller-mt2dt
6ebdffc27c291       f941e69465d6c       2 days ago          Running             linkerd-proxy             0                   8eba0d0087211       myingress-ingress-nginx-controller-mt2dt
498c7597339b0       a830707172e80       3 days ago          Running             nginx                     0                   3cccc66ef5bfa       web-two-64558f4b75-qfzq8
64e153def1539       a830707172e80       3 days ago          Running             nginx                     0                   4263cd8a9aa45       web-one-8d48dcf57-dm2rr
#get wc -l in the CP
crictl ps | wc -l
26
#on the DP/worker
crictl ps | wc -l
38
```
We'll use taints to control where new containers are deployed. NoSchedule and PreferNoSchedule affect only new containers, while NoExecute also evicts running ones. Taint the secondary node with a key like "bubba", verify the taint, then redeploy to see the effect.
We delete, add taint and try again

```sh
#First we taint the node
kubectl taint node dp bubba=value:PreferNoSchedule
node/dp tainted
#check
kubectl describe node dp | grep -A5 Taint
Taints:             bubba=value:PreferNoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  dp
  AcquireTime:     <unset>
  RenewTime:       Fri, 02 May 2025 22:44:12 +0000
#we create pods again using our deploy
kubectl apply -f taint.yaml
deployment.apps/taint-deployment created
#check if deploy completed
kubectl get deployments.apps taint-deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
taint-deployment   8/8     8            8           17s
#all pod runing in CP now
kubectl get pod -o wide | grep taint
taint-deployment-787bd7bfb4-2wccj                 1/1     Running   0          35s     192.168.0.145   cp     <none>           <none>
taint-deployment-787bd7bfb4-frnvx                 1/1     Running   0          35s     192.168.0.240   cp     <none>           <none>
taint-deployment-787bd7bfb4-jpz7b                 1/1     Running   0          35s     192.168.0.188   cp     <none>           <none>
taint-deployment-787bd7bfb4-jssrc                 1/1     Running   0          35s     192.168.0.215   cp     <none>           <none>
taint-deployment-787bd7bfb4-kr2h4                 1/1     Running   0          35s     192.168.0.1     cp     <none>           <none>
taint-deployment-787bd7bfb4-n45zs                 1/1     Running   0          35s     192.168.0.121   cp     <none>           <none>
taint-deployment-787bd7bfb4-phff7                 1/1     Running   0          35s     192.168.0.175   cp     <none>           <none>
taint-deployment-787bd7bfb4-tznkf                 1/1     Running   0          35s     192.168.0.165   cp     <none>           <none>
#on CP went from 22 to 30:
root@cp ~ # crictl ps | wc -l
30
#on worker stayed 34:
root@dp:~# crictl ps | wc -l
34
```

Lets delete the deployment taint and other, then untaint the node
```sh
kubectl taint node dp bubba-
node/dp untainted
kubectl describe nodes | grep Taint
Taints:             <none>
Taints:             <none>
```

Now we taint with NoSchedule
```sh
#Tainting
kubectl taint node dp bubba=value:NoSchedule
node/dp tainted
#The we create our deployement again and check node placement
kubectl apply -f taint.yaml
deployment.apps/taint-deployment created
#placement
#on CP from 22 to 30 +8
root@cp ~ # crictl ps | wc -l
30
#on Worker, as you can see , working didnt change from 34
root@dp:~# crictl ps | wc -l
34
```
Lets now try with NoExecute
```sh
#first delete the deployement and untaint
kubectl delete -f taint.yaml
deployment.apps "taint-deployment" deleted
#check the images
root@cp ~ # crictl ps | wc -l
22
#delete the taint
root@cp ~ # kubectl taint node dp bubba-
node/dp untainted
#deploy again pod and then we taint to check if they get evicted
kubectl create -f taint.yaml
deployment.apps/taint-deployment created
#now on CP we have 22 > 26 +4
root@cp ~ # crictl ps | wc -l
26
#on DP we went from 34 to 38 +4
root@dp:~# crictl ps | wc -l
38
#now lets taint and check again
kubectl taint node dp bubba=value:NoExecute
node/dp tainted
#alsmot all pod went to CP:
root@cp ~ # crictl ps | wc -l
40
#and in DP only important and Daemonset pod stayed
root@dp:~# crictl ps | wc -l
5
#checking each node and his placement:
kubectl get pod --all-namespaces -owide | grep -e NAME -e dp

NAMESPACE       NAME                                              READY   STATUS    RESTARTS       AGE     IP              NODE     NOMINATED NODE   READINESS GATES
kube-system     cilium-envoy-k2qdh                                1/1     Running   0              48d     10.2.0.3        dp       <none>           <none>
kube-system     cilium-operator-5c7867ccd5-jghsj                  1/1     Running   0              42d     10.2.0.3        dp       <none>           <none>
kube-system     cilium-xlngl                                      1/1     Running   0              48d     10.2.0.3        dp       <none>           <none>
kube-system     kube-proxy-kv9ww                                  1/1     Running   0              43d     10.2.0.3        dp       <none>           <none>
kubectl get pod --all-namespaces -owide | grep -e NAME -e cp

NAMESPACE       NAME                                              READY   STATUS    RESTARTS       AGE     IP              NODE     NOMINATED NODE   READINESS GATES
default         books-5bcf5ccfd7-7zwm7                            1/1     Running   0              5m10s   192.168.0.115   cp       <none>           <none>
default         books-5bcf5ccfd7-bz8pq                            1/1     Running   0              2d      192.168.0.90    cp       <none>           <none>
default         myingress-ingress-nginx-controller-mt2dt          2/2     Running   0              3d1h    192.168.0.19    cp       <none>           <none>
default         nfs-subdir-external-provisioner-d4c865f58-4vf5f   1/1     Running   0              5m10s   192.168.0.107   cp       <none>           <none>
default         nginx-5869d7778c-p79gx                            1/1     Running   0              5m8s    192.168.0.239   cp       <none>           <none>
default         taint-deployment-787bd7bfb4-2ngcd                 1/1     Running   0              15m     192.168.0.253   cp       <none>           <none>
default         taint-deployment-787bd7bfb4-77h89                 1/1     Running   0              5m11s   192.168.0.143   cp       <none>           <none>
default         taint-deployment-787bd7bfb4-7jkfx                 1/1     Running   0              15m     192.168.0.60    cp       <none>           <none>
default         taint-deployment-787bd7bfb4-fdlqp                 1/1     Running   0              5m9s    192.168.0.204   cp       <none>           <none>
default         taint-deployment-787bd7bfb4-glkx5                 1/1     Running   0              15m     192.168.0.154   cp       <none>           <none>
default         taint-deployment-787bd7bfb4-klrn2                 1/1     Running   0              5m9s    192.168.0.44    cp       <none>           <none>
default         taint-deployment-787bd7bfb4-lvwm4                 1/1     Running   0              5m10s   192.168.0.231   cp       <none>           <none>
default         taint-deployment-787bd7bfb4-r4sjj                 1/1     Running   0              15m     192.168.0.238   cp       <none>           <none>
default         web-one-8d48dcf57-dm2rr                           1/1     Running   0              3d20h   192.168.0.26    cp       <none>           <none>
default         web-two-64558f4b75-qfzq8                          1/1     Running   0              3d20h   192.168.0.185   cp       <none>           <none>
kube-system     cilium-envoy-2xs9m                                1/1     Running   0              49d     10.2.0.2        cp       <none>           <none>
kube-system     cilium-h9s45                                      1/1     Running   0              49d     10.2.0.2        cp       <none>           <none>
kube-system     cilium-operator-5c7867ccd5-24jvr                  1/1     Running   2 (43d ago)    43d     10.2.0.2        cp       <none>           <none>
kube-system     coredns-7c65d6cfc9-fpkpr                          1/1     Running   0              6d19h   192.168.0.102   cp       <none>           <none>
kube-system     coredns-7c65d6cfc9-fxt75                          1/1     Running   0              6d19h   192.168.0.164   cp       <none>           <none>
kube-system     etcd-cp                                           1/1     Running   1 (43d ago)    43d     10.2.0.2        cp       <none>           <none>
kube-system     kube-apiserver-cp                                 1/1     Running   0              43d     10.2.0.2        cp       <none>           <none>
kube-system     kube-controller-manager-cp                        1/1     Running   0              43d     10.2.0.2        cp       <none>           <none>
kube-system     kube-proxy-hx2xw                                  1/1     Running   0              43d     10.2.0.2        cp       <none>           <none>
kube-system     kube-scheduler-cp                                 1/1     Running   0              43d     10.2.0.2        cp       <none>           <none>
linkerd-viz     metrics-api-86bff98cbc-qljxb                      1/1     Running   0              5m9s    192.168.0.205   cp       <none>           <none>
linkerd-viz     prometheus-7ff979ffd9-5zkf6                       2/2     Running   0              5m10s   192.168.0.61    cp       <none>           <none>
linkerd-viz     tap-5b55c9bb76-6c8r5                              2/2     Running   1 (4d1h ago)   4d1h    192.168.0.46    cp       <none>           <none>
linkerd-viz     tap-injector-f67944698-mbbmh                      1/1     Running   0              5m9s    192.168.0.192   cp       <none>           <none>
linkerd-viz     web-755b796446-jvpjn                              2/2     Running   0              4d      192.168.0.32    cp       <none>           <none>
linkerd         linkerd-destination-5477b7b488-vhjrt              4/4     Running   0              5m10s   192.168.0.252   cp       <none>           <none>
linkerd         linkerd-identity-868c8bbdc4-msk6c                 2/2     Running   0              5m9s    192.168.0.96    cp       <none>           <none>
linkerd         linkerd-proxy-injector-9d4bbf576-9fpc8            2/2     Running   0              5m10s   192.168.0.41    cp       <none>           <none>
nginx-gateway   nginx-gateway-96f76cdcf-tlgmc                     2/2     Running   0              2d12h   192.168.0.29    cp       <none>           <none>

#Lets now untaint and check if all container returned
kubectl taint node dp bubba-
node/dp untainted
#on CP
root@cp ~ # crictl ps | wc -l
45
#on DP , we see that not all container got back
root@dp:~# crictl ps | wc -l
13
```

# LOGGING and TROUBLESHOOTING

## OverView

Kubernetes clusters rely on API communication and are particularly affected by network issues. When troubleshooting, use standard Linux tools and procedures. If the affected Pod lacks a shell (like `bash`), deploy a similar Pod with a shell, such as `busybox`. Begin by examining DNS configuration files and using utilities like `dig`. For deeper analysis, you might need to install tools like `tcpdump`.

Monitoring is essential for managing large workloads. It involves collecting key metrics—CPU, memory, disk, and network usage—on nodes and applications. Kubernetes provides this via the Metrics Server, which exposes a standard API (e.g., `/apis/metrics.k8s.io/`) for use by components like autoscalers.

Centralized logging across all nodes is not built into Kubernetes by default. Tools like Fluentd can aggregate logs into a unified logging layer, making it easier to visualize and search for issues—especially when network troubleshooting does not reveal the root cause. Fluentd can be downloaded from its official website.

For a comprehensive solution that combines logging, monitoring, and alerting, consider Prometheus, a CNCF project. Prometheus provides a time-series database and integrates with Grafana for visualization and dashboards.

## Basic Troubleshooting Steps

When troubleshooting Kubernetes issues, start with the most obvious causes and proceed methodically. Follow these steps to efficiently diagnose and resolve problems:

1. **Check Command Line Errors**  
    Investigate any errors returned by CLI tools such as `kubectl`. These often provide direct clues about the root cause.

2. **Inspect Pod State and Logs**  
    - Deploy a test Pod if needed:
      ```sh
      kubectl create deploy busybox --image=busybox -- sleep 3600
      ```
    - Access the Pod shell for interactive troubleshooting:
      ```sh
      kubectl exec -ti <busybox_pod> -- /bin/sh
      ```
    - View container logs:
      ```sh
      kubectl logs <pod-name>
      ```
    If logs are missing, consider adding a sidecar container for enhanced logging capabilities.

3. **Verify Networking and DNS**  
    Use standard Linux tools within the Pod shell to check DNS resolution, firewall rules, and general connectivity.


 **DNS Resolution**

| Tool       | Example Command                            | Description                            |
|------------|---------------------------------------------|----------------------------------------|
| `nslookup` | `nslookup kubernetes.default`               | Simple DNS query tool                  |
| `dig`      | `dig kubernetes.default.svc.cluster.local`  | Detailed DNS lookups                   |
| `getent`   | `getent hosts kubernetes.default`           | Uses system's name service resolution  |
| `host`     | `host google.com`                           | DNS lookup tool                        |

---

**Network Connectivity**

| Tool       | Example Command                     | Description                               |
|------------|--------------------------------------|-------------------------------------------|
| `ping`     | `ping 8.8.8.8`                       | Check basic reachability                  |
| `curl`     | `curl http://service-name:port`      | HTTP(S) requests to endpoints             |
| `wget`     | `wget http://example.com`            | HTTP requests (alternative to curl)       |
| `telnet`   | `telnet service-name 80`             | Check if a TCP port is reachable          |
| `nc`       | `nc -zv service-name 80`             | Netcat: check for open ports              |
| `traceroute` | `traceroute 8.8.8.8`               | Trace route to a remote IP                |
| `ip`       | `ip a`, `ip r`, `ip link`            | Check IP addresses, routes, interfaces    |
| `ss`       | `ss -lntp`                           | List listening sockets and connections    |

---

**Firewall & Network Policy Inspection**

| Tool       | Example Command                 | Description                               |
|------------|----------------------------------|-------------------------------------------|
| `iptables` | `iptables -L -n -v`              | View firewall rules (if available)        |
| `nft`      | `nft list ruleset`               | View nftables rules (modern alternative)  |

> ⚠️ **Note**: Most containers don't include `iptables` or `nft`. Firewall rules usually reside on the **host**, not inside Pods.

---

**Recommended Debug Image**

For full tooling, run a debug container like this:

```bash
kubectl run -it --rm debug --image=nicolaka/netshoot -- bash
```

4. **Review Security Settings**  
    - Ensure RBAC permissions are correctly configured.
    - Check SELinux and AppArmor profiles, especially for network-centric applications.

5. **Enable Auditing**  
    Kubernetes supports auditing on the kube-apiserver, providing visibility into API actions after requests are accepted.

6. **Check Node Health and Resources**  
    - Review node logs for errors.
    - Confirm sufficient CPU, memory, and disk resources are available.

7. **Investigate Control Plane and Inter-Node Issues**  
    - Examine logs for control plane components.
    - Look for Pods stuck in pending or error states.
    - Check for inter-node network issues, DNS problems, and firewall misconfigurations.

### Summary Checklist

- [ ] Errors from the command line
- [ ] Pod logs and state
- [ ] Pod shell access for DNS/network troubleshooting
- [ ] Node logs and resource allocation
- [ ] Security settings (RBAC, SELinux, AppArmor)
- [ ] API calls and auditing
- [ ] Inter-node networking, DNS, and firewall
- [ ] Control plane server/controller health

By following this structured approach, you can systematically identify and resolve issues in your Kubernetes environment.

## Ephemeral Containers

Introduced in Kubernetes 1.16, ephemeral containers allow you to add a temporary container to a running pod without restarting or recreating it. This is especially useful for debugging intermittent or hard-to-reproduce issues, as you can inject troubleshooting tools directly into the live environment.

Ephemeral containers are currently an alpha feature, They have some limitations: they are not restarted automatically if they exit, and they cannot request ports or certain resources.

Unlike regular containers, ephemeral containers are added through the `ephemeralcontainers` API endpoint, not by editing the pod specification. Therefore, you cannot use `kubectl edit` to add them.

For debugging, you might use `kubectl attach` to connect to an existing process in a container, which can be preferable to `kubectl exec` if you want to avoid starting a new process. The effectiveness of this depends on the process you attach to.

To add an ephemeral container and attach to it, you can use:

```sh
kubectl debug buggypod --image=debian --attach
```

## Cluster Startup Sequence

When using **kubeadm** to build your cluster, the startup process is managed by **systemd**. To check the status and configuration of the kubelet service, run:

```sh
systemctl status kubelet.service
```

The kubelet service uses the configuration file:

- `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`

Within the kubelet's main configuration file (`/var/lib/kubelet/config.yaml`), you'll find several important settings. One key setting is `staticPodPath`, which specifies the directory where kubelet looks for static pod manifests:

- `staticPodPath: /etc/kubernetes/manifests/`

Any YAML file placed in this directory will be automatically started as a static pod by the kubelet, bypassing the scheduler. This is useful for troubleshooting or ensuring critical components always run.

By default, the following static pod manifests are present, and kubelet will create these core cluster components:

- `kube-apiserver`
- `etcd`
- `kube-controller-manager`
- `kube-scheduler`

Once these base pods are running, the **kube-controller-manager** uses etcd data to launch the remaining configured objects and controllers, completing the cluster initialization.

## Monitoring

Monitoring involves collecting metrics from both infrastructure and applications to ensure system health and performance.

The deprecated Heapster has been replaced by the integrated Metrics Server in Kubernetes. Once installed and configured, the Metrics Server exposes a standard API that agents and tools can use to access resource usage data. It also supports custom metrics, enabling autoscalers to make informed scaling decisions based on application-specific data.

Prometheus, a Cloud Native Computing Foundation (CNCF) project, is widely used for monitoring Kubernetes clusters. It scrapes resource usage metrics from Kubernetes objects cluster-wide and provides client libraries for instrumenting application code to collect detailed application-level metrics.

Other CNCF projects enhance observability in distributed systems. OpenTelemetry enables developers to add instrumentation to their code, while Jaeger is used for tracing and analyzing telemetry data. Kubernetes also offers an alpha feature called API Server Tracing, which leverages OpenTelemetry for deeper insights. For more information, refer to the Kubernetes documentation.

## Using krew

Throughout this course, we've used the `kubectl` command to interact with Kubernetes. While the basic commands are powerful, you can extend `kubectl`'s functionality even further by using plugins to help you manage Kubernetes objects and components more efficiently.

> **Note:**  
> At the time of writing, plugins cannot overwrite existing `kubectl` commands or add sub-commands to them.

When using a plugin, options such as `--namespace` or `--container` must come **after** the plugin command. For example:

```sh
kubectl sniff bigpod-abcd-123 -c mainapp -n accounting
```

Plugins can be distributed in various ways. The recommended approach is to use **krew**, the `kubectl` plugin manager. Krew provides cross-platform packaging and a curated plugin index, making it easy to discover and install new plugins.

To install krew, follow the instructions in the [krew GitHub repository](https://github.com/kubernetes-sigs/krew/):

First run the following script:
```sh
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
#add to PATH
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
echo "export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"" >> $HOME/.bashrc
#test
kubectl krew
krew is the kubectl plugin manager.
You can invoke krew through kubectl: "kubectl krew [command]..."

Usage:
  kubectl krew [command]

Available Commands:
  help        Help about any command
  index       Manage custom plugin indexes
...
#Update the local copy of the plugin index
kubectl krew update
#Lets install some plugin
kubectl krew install ns
kubectl krew install tree
kubectl krew install get-all
#to use a plugin:
#example of tree:
kubectl tree deploy web-one
NAMESPACE  NAME                                          READY  REASON  AGE
default    Deployment/web-one                            -              4d
default    └─ReplicaSet/web-one-8d48dcf57                -              4d
default      └─Pod/web-one-8d48dcf57-dm2rr               True           4d
default        └─CiliumEndpoint/web-one-8d48dcf57-dm2rr  -              4d
#example of get-all:
kubectl get-all
W0503 14:26:43.699538 1774996 warnings.go:70] v1 ComponentStatus is deprecated in v1.19+
NAME                                                                                                        NAMESPACE        AGE
componentstatus/scheduler                                                                                                    <unknown>
componentstatus/controller-manager                                                                                           <unknown>
componentstatus/etcd-0                                                                                                       <unknown>
configmap/kube-root-ca.crt                                                                                  accounting       8d
configmap/colors                                                                                            default          11d
configmap/fast-car                                                                                          default          11d
...
#to upgrade
kubectl krew upgrade
#to uninstall
kubectl krew uninstall access-matrix
#acess help
kubectl krew help
#to see all installed plugins
kubectl plugin list
The following compatible plugins are available:

/root/.krew/bin/kubectl-get_all
/root/.krew/bin/kubectl-krew
/root/.krew/bin/kubectl-ns
/root/.krew/bin/kubectl-tree
```

## Sniffing traffic

When troubleshooting network issues in a Kubernetes cluster, keep in mind that most cluster network traffic is encrypted. This can make diagnosing problems more challenging. The `sniff` plugin helps by allowing you to capture and analyze traffic directly from within a pod. To use this tool, you'll need to have Wireshark installed and be able to export graphical displays.

By default, the `sniff` command attaches to the first container it finds in the specified pod. If you want to target a specific container, use the `-c` option.
First we install sniff

Lets **install sniff**
```sh 
kubectl krew install sniff
Updated the local copy of plugin index.
Installing plugin: sniff
Installed plugin: sniff
\
 | Use this plugin:
 |      kubectl sniff
 | Documentation:
 |      https://github.com/eldadru/ksniff
 | Caveats:
 | \
 |  | This plugin needs the following programs:
 |  | * wireshark (optional, used for live capture)
 | /
/
WARNING: You installed plugin "sniff" from the krew-index plugin repository.
   These plugins are not audited for security by the Krew maintainers.
   Run them at your own risk.
```

Then we **install Wireshark**
```sh
apt install wireshark -y
```

We need a pod with tcpdump:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netshoot
  labels:
    app: netshoot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: netshoot
  template:
    metadata:
      labels:
        app: netshoot
    spec:
      containers:
      - name: netshoot
        image: nicolaka/netshoot:latest
        imagePullPolicy: Always
        command:
          - "sleep"
          - "3600"  # Keep the pod alive for 1 hour (change as needed)
```
#then we run this:
```yaml
kubectl sniff netshoot-64576c6b7d-996gm -c netshoot -n default
INFO[0000] using tcpdump path at: '/root/.krew/store/sniff/v1.6.2/static-tcpdump'
INFO[0000] sniffing method: upload static tcpdump
INFO[0000] sniffing on pod: 'netshoot-64576c6b7d-996gm' [namespace: 'default', container: 'netshoot', filter: '', interface: 'any']
INFO[0000] uploading static tcpdump binary from: '/root/.krew/store/sniff/v1.6.2/static-tcpdump' to: '/tmp/static-tcpdump'
INFO[0000] uploading file: '/root/.krew/store/sniff/v1.6.2/static-tcpdump' to '/tmp/static-tcpdump' on container: 'netshoot'
INFO[0000] executing command: '[/bin/sh -c test -f /tmp/static-tcpdump]' on container: 'netshoot', pod: 'netshoot-64576c6b7d-996gm', namespace: 'default'
INFO[0000] command: '[/bin/sh -c test -f /tmp/static-tcpdump]' executing successfully exitCode: '0', stdErr :''
INFO[0000] file found: ''
INFO[0000] file was already found on remote pod
INFO[0000] tcpdump uploaded successfully
INFO[0000] spawning wireshark!
INFO[0000] start sniffing on remote container
INFO[0000] executing command: '[/tmp/static-tcpdump -i any -U -w - ]' on container: 'netshoot', pod: 'netshoot-64576c6b7d-996gm', namespace: 'default'
INFO[0000] starting sniffer cleanup
INFO[0000] sniffer cleanup completed successfully
Error: signal: aborted (core dumped)
```

>> WE NEED TO CHECK LATER WHY NOT WORKING

## Logging Tools

Logging is a crucial aspect of IT operations, closely related to monitoring. There are numerous tools available to help you collect, aggregate, and analyze logs effectively.

A typical logging workflow involves collecting logs locally, aggregating them, and then ingesting them into a search engine. These logs are then visualized through dashboards that support advanced search syntax. Among the various logging solutions, the **Elasticsearch, Logstash, and Kibana (ELK) Stack** is widely adopted.

In Kubernetes environments:

- The **kubelet** writes container logs to local files using the Docker logging driver.
- You can retrieve these logs using the `kubectl logs` command.

**Logging Node Level**

Container runtimes manage application logs, standardizing them via the CRI logging format for kubelet integration. By default, logs from one terminated container are kept after a restart, but all logs are removed if a pod is evicted. Access logs using `kubectl logs`.

![logging-node-level.png](logging-node-level.png)

**Cluster-Level logging architectures**

- Using a node logging agent

Use a node-level logging agent (often as a DaemonSet) on each node to collect and forward container logs. This approach requires no changes to applications and aggregates logs from stdout and stderr for all containers on the node.

![logging-with-node-agent.png](logging-with-node-agent.png)

**Using a sidecar container with the logging agent**

- Streaming sidecar container

Sidecar containers should write logs to their own stdout and stderr. This lets kubelet and logging agents collect logs easily, even if your app doesn't support stdout/stderr. Redirecting logs is simple and enables tools like `kubectl logs` to work. 

![logging-with-streaming-sidecar.png](logging-with-streaming-sidecar.png)

- Sidecar container with a logging agent

If the node-level logging agent isn't flexible enough, use a sidecar container with a custom logging agent for your app.

Note:
Sidecar logging agents can consume more resources and their logs aren't accessible via kubectl logs.

![logging-with-sidecar-agent.png](logging-with-sidecar-agent.png)

- Exposing logs directly from the application

Cluster-logging that exposes or pushes logs directly from every application is outside the scope of Kubernetes

![logging-from-application.png](logging-from-application.png)

For cluster-wide log aggregation, **Fluentd** is a popular choice. Fluentd is part of the [Cloud Native Computing Foundation (CNCF)](https://www.cncf.io/), and it pairs well with [Prometheus](https://prometheus.io/) for comprehensive monitoring and logging.


### Fluentd in Kubernetes

Setting up Fluentd for Kubernetes logging is an excellent way to learn about **DaemonSets**. Fluentd agents are deployed on each node via a DaemonSet, aggregate logs from all containers, and forward them to an Elasticsearch instance. These logs can then be visualized using a Kibana dashboard.

```mermaid
graph LR
    A[Container Logs] --> B[Fluentd DaemonSet]
    B --> C[Elasticsearch]
    C --> D[Kibana Dashboard]
```

## TroubleShooting

### Cluster

- **List Nodes**: Check if all nodes are registered and in the `Ready` state.

```sh
#check if ready
kubectl get nodes
```

- **Cluster Info Dump**: Collect detailed cluster information

```sh
#dump cluster info
#full dump that you can direct into a file
kubectl cluster-info dump > output_dump
#grep on errors
kubectl cluster-info dump | grep -i "error"
#grep on kubelet
kubectl cluster-info dump | grep -A 5 "kubelet"
#focus only on kube-system
kubectl cluster-info dump --namespaces=kube-system
#dump on dir
kubectl cluster-info dump --output-directory=dump
root@cp ~ # tree dump/
dump/
├── default
│   ├── books-5bcf5ccfd7-7zwm7
│   │   └── logs.txt
│   ├── books-5bcf5ccfd7-bz8pq
│   │   └── logs.txt
│   ├── daemonsets.json
....
├── kube-system
│   ├── cilium-envoy-2xs9m
│   │   └── logs.txt
│   ├── cilium-envoy-k2qdh
│   │   └── logs.txt
│   ├── cilium-h9s45
│   │   └── logs.txt
│   ├── cilium-operator-5c7867ccd5-24jvr
│   │   └── logs.txt
....
```

- **investigate node issues**: (NotReady status or unreachable nodes), use `kubectl describe node` or `kubectl get node -o yaml` for details. Look for NotReady events and note that pods are evicted after five minutes of NotReady status.

```sh
#look for events
kubectl describe node kube-worker-1
#get node info
kubectl get node kube-worker-1 -o yaml
```

#### Logs

##### Control Plane Logs
- `/var/log/kube-apiserver.log` — API Server logs
- `/var/log/kube-scheduler.log` — Scheduler logs
- `/var/log/kube-controller-manager.log` — Controller Manager logs

##### Worker Node Logs
- `/var/log/kubelet.log` — Kubelet logs
- `/var/log/kube-proxy.log` — Kube-proxy logs

#### Common Issues

- **Node Not Ready:**  
    Use `kubectl describe node` to check events and conditions.

- **Pod Scheduling Issues:**  
    Verify node taints, tolerations, and resource availability.


#### Useful Tools

- **Node Problem Detector:** Monitors node health.
- **crictl:** Debugs container runtime issues.
- **kubectl debug:** Enables interactive node debugging.

#### Kubelet Service
```sh
#get kubelet status
sudo systemctl status kubelet
#get kubelet logs
sudo journalctl -u kubelet -xe
#restart kubelet
sudo systemctl restart kubelet
```

#### ContainerD Service
```sh
#get containerD status
sudo systemctl status containerd
# or for Docker:
sudo systemctl status docker
#check logs
sudo journalctl -u containerd -xe
```

#### Check node memory and CPU and logs
```sh
df -h        # Disk usage
free -m      # Memory usage
top or htop  # CPU load
dmesg   #View kernel ring buffer messages(Hardware,Driver,KernelPanic,IO,OOM,Boot)
sudo journalctl -xe #node logs
```

#### Check Network and DNS
```sh
ping 8.8.8.8
ping <control-plane-ip>
nslookup kubernetes.default
```

#### Drain node restart it and join again
```sh
#cordon node
kubectl cordon <node-name>
#drain it
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
#uncordon and let it join again
kubectl uncordon <node-name>
```


#### Mitigations

##### 1. Automatic VM Restart

- **Action:** Use the IaaS provider's automatic VM restarting feature for IaaS VMs.
    - **Mitigates:** 
        - Apiserver VM shutdown or crashing
        - Supporting services VM shutdown or crashes

##### 2. Reliable Storage

- **Action:** Use IaaS provider's reliable storage (e.g., GCE PD or AWS EBS volume) for VMs running apiserver and etcd.
    - **Mitigates:** 
        - Apiserver backing storage loss

##### 3. High-Availability Configuration

- **Action:** Configure high-availability (HA) for control plane components.
    - **Mitigates:** 
        - Control plane node shutdown
        - Control plane component (scheduler, API server, controller-manager) crashes
        - API server backing storage (etcd data directory) loss (assumes HA etcd configuration)
    - **Benefit:** Tolerates one or more simultaneous node or component failures.

##### 4. Regular Snapshots

- **Action:** Snapshot apiserver persistent disks (PDs) or EBS volumes periodically.
    - **Mitigates:** 
        - Apiserver backing storage loss
        - Some cases of operator error
        - Some Kubernetes software faults

##### 5. Replication Controllers and Services

- **Action:** Use replication controllers and services in front of pods.
    - **Mitigates:** 
        - Node shutdown
        - Kubelet software faults

##### 6. Resilient Application Design

- **Action:** Design applications (containers) to tolerate unexpected restarts.
    - **Mitigates:** 
        - Node shutdown
        - Kubelet software faults

### Debug POD

#### Key Commands
- **Check Pod State**: View the current state and recent events of a Pod.
  ```bash
  kubectl describe pods <pod-name>
  ```
- **List Pods by Selector**: Verify Pods matching a specific label.
  ```bash
  kubectl get pods --selector=name=nginx,type=frontend
  ```
- **Validate Pod YAML**: Ensure your Pod configuration is correct.
  ```bash
  kubectl apply --validate -f mypod.yaml
  ```
- **Compare Pod YAML**: Compare the Pod on the apiserver with your local YAML.
  ```bash
  kubectl get pods/mypod -o yaml > mypod-on-apiserver.yaml
  ```

#### Common Pod Issues
- **Pod Stuck in Pending**:
  - Insufficient resources: Adjust resource requests or add nodes.
  - `hostPort` conflicts: Avoid using `hostPort` unless necessary.
- **Pod Stuck in Waiting**:
  - Image pull issues: Verify image name, registry, and manual pull.
- **Pod Stuck in Terminating**:
  - Finalizer issues: Check for admission webhooks preventing deletion.
- **Pod Crashing or Unhealthy**:
  - Debug running Pods using logs and events.
- **Pod Not Behaving as Expected**:
  - Validate YAML and check for typos or incorrect nesting.

#### Debugging Services
- **Missing Endpoints**:
  - `kubectl get endpointslices -l kubernetes.io/service-name=${SERVICE_NAME}`
  - Verify Pods match the Service's label selector.
  - Ensure `containerPort` matches the Service's `targetPort`.
- **Network Traffic Not Forwarded**:
  - Check kube-proxy and iptables rules.

### Debugging Running Pods in Kubernetes

#### Prerequisites
- Ensure your Pod is already scheduled and running.(refer to the Debugging pod section.)
- Have a shell inside the pods Node and be able to use `kubectl`

#### Debugging first Steps

##### 1. Fetch Pod Details
Use `kubectl describe pod` to get detailed information about a Pod, including events and status.
```bash
kubectl describe pod <pod-name>
```
For YAML output with more metadata:
```bash
kubectl get pod <pod-name> -o yaml
```

- The container state is one of *Waiting*, *Running* or *Terminated*
- Ready tells you if container passed its last readiness probe(if no readiness Prob configured, by default he is Ready)
- Restart Count tells you number of restart, to detect crash loops in container with restart policy always
- In log of recent event , "From Indicates the component logging the evens.

**Pending Pods**

Most common scenario is requesting more resources than free in node, describe the pod and check the events, then you can scale down or add node/resource to cluster, or tune in your pod.

##### 2. Examine Pod Logs
Check the logs of the affected container:
```bash
kubectl logs <pod-name> -c <container-name>
```
For previous container crash logs:
```bash
kubectl logs --previous <pod-name> -c <container-name>
```

##### 3. Events
Get k8s event for default on namespace in contexte
```bash
kubectl get events
```
get specific namespace events
```bash
kubectl get events --namespace=my_namespace
```

##### 4. Debug with Container Exec
Execute commands inside a running container:
```bash
kubectl exec -it <pod-name> -c <container-name> -- <command> <ARG1> <ARG2> ... <ARGN>
```
Example:
```bash
#check cassandra logs
kubectl exec cassandra -- cat /var/log/cassandra/system.log
#run a shell using -it
kubectl exec -it cassandra -- sh
```

#### Debugging with ephemeral debug container (Stable)

Ephemeral containers help troubleshoot when `kubectl exec` isn't enough, such as with crashed containers or minimal images lacking debugging tools (***Distroless*** minimal image with no package manager, shell or OS utilities).

##### 1. Debugging Using ephemeral containers
Lets create this pod, with pause image::
```bash
kubectl run ephemeral-demo --image=registry.k8s.io/pause:3.1 --restart=Never
```

We cannot exec this image:
```bash
kubectl exec -it ephemeral-demo -- sh
OCI runtime exec failed: exec failed: container_linux.go:346: starting container process caused "exec: \"sh\": executable file not found in $PATH": unknown
```

Lets add a debugging container witgh -i (--interactive) to attach to the console of the Ephemeral storage
```bash
#This will add new busybox container and attach to it
#--target param, targets the process namespaces of another container (check if the CRI support it)
kubectl debug -it ephemeral-demo --image=busybox:1.28 --target=ephemeral-demo
Defaulting debug container name to debugger-8xzrl.
If you don't see a command prompt, try pressing enter.
/ #
```

You can check the state of newly create ephemeral container using `kubectl describe`
```yaml
...
Ephemeral Containers:
  debugger-8xzrl:
    Container ID:   docker://b888f9adfd15bd5739fefaa39e1df4dd3c617b9902082b1cfdc29c4028ffb2eb
    Image:          busybox
    Image ID:       docker-pullable://busybox@sha256:1828edd60c5efd34b2bf5dd3282ec0cc04d47b2ff9caa0b6d4f07a21d1c08084
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 12 Feb 2020 14:25:42 +0100
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:         <none>
...
```
---
- **Share process namespace between containers inside pod**

**Process namespace sharing** lets containers in a pod see each other's processes. Use it for sidecar containers (like log handlers) or debugging containers without built-in tools:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true #<=Shared Process Enabled
  containers:
  - name: nginx #<= First container: Nginx 
    image: nginx
  - name: shell #<= Second container: Shell
    image: busybox:1.28
    command: ["sleep", "3600"]
    securityContext:
      capabilities:
        add:
        - SYS_PTRACE
    stdin: true
    tty: true
```
Lets attach
```bash
#Create
kubectl apply -f https://k8s.io/examples/pods/share-process-namespace.yaml
kubectl exec -it nginx -c shell -- /bin/sh
#Run this inside the "shell" container
ps ax
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   14 101       0:00 nginx: worker process
   15 root      0:00 sh
   21 root      0:00 ps ax
```
inside `shell` container i can do action on `nginx` container process:
```bash
# run this inside the "shell" container
kill -HUP 8   # change "8" to match the PID of the nginx leader process
ps ax
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   15 root      0:00 sh
   22 101       0:00 nginx: worker process #restarted
   23 root      0:00 ps ax
```
```bash
You can even access file system from `shell` container to nginx `container`
# run this inside the "shell" container
# change "8" to the PID of the Nginx process, if necessary
head /proc/8/root/etc/nginx/nginx.conf

```

Pods can share a process namespace, but this has implications:

- **PID 1 Issue:** Containers expecting PID 1 (like those using systemd) may not work as expected. In shared namespaces, `kill -HUP 1` signals the pod sandbox, not the container.
- **Process Visibility:** All processes (and their details in `/proc`) are visible to containers in the pod, including sensitive info passed as arguments or environment variables.
- **Filesystem Access:** Containers can access each other's filesystems via `/proc/$pid/root`, making secrets only as secure as filesystem permissions.

---

##### 2. Debugging Using a copy of the pod

Troubleshooting Pods can be hard if the container lacks a shell or crashes on startup. In these cases, use `kubectl debug` to create a modified copy of the Pod for debugging.

###### 2.1. Copying a Pod while adding a new container

Add a new container to your Pod if you need extra troubleshooting tools not available in your current container image.

We create a pod using `busybox`, but we need tools not included in `busybox`:
```bash
kubectl run myapp --image=busybox:1.28 --restart=Never -- sleep 1d
```

Lets create a copy of `myapp` named `myapp-debug` that add a ubuntu container:
```bash
kubectl debug myapp -it --image=ubuntu --share-processes --copy-to=myapp-debug
Defaulting debug container name to debugger-w7xmf.
If you don't see a command prompt, try pressing enter.
root@myapp-debug:/#
```

- `kubectl debug` auto-generates a container name unless you use `--container`.
- Use `-i` to attach to the new container (default). To avoid attaching, add `--attach=false`.
- If disconnected, reattach with `kubectl attach`.
- `--share-processes` lets containers in the Pod see each other's processes.

###### 2.2. Copying a Pod while changing its command

You may need to change a container's command, for example to debug or fix crashes.
Lets simulate a crashing app using `kubectl run` that exit immediatly
```bash
kubectl run --image=busybox:1.28 myapp -- false
#When using describe:
kubectl describe pod myapp
Containers:
  myapp:
    Image:         busybox
    ...
    Args:
      false
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
```

We can now use `kubectl debug`to create a copy of this pod with command change to interactive shell:
```bash
kubectl debug myapp -it --copy-to=myapp-debug --container=myapp -- sh
If you don't see a command prompt, try pressing enter.
/ #
```

- Use `--container <name>` with `kubectl debug` to change a specific container's command; otherwise, a new container is created.
- The `-i` flag attaches to the container by default. Use `--attach=false` to prevent this. Reattach with `kubectl attach` if disconnected.

###### 2.3. Copying a Pod while changing container images

Sometimes you may need to swap a Pod's image for one with debugging tools.
Lets create a pod using `kubectl run`:
```bash
kubectl run myapp --image=busybox:1.28 --restart=Never -- sleep 1d
```
Now using `kubectl debug` to make copy and change the container image to `ubuntu`
```bash
kubectl debug myapp --copy-to=myapp-debug --set-image=*=ubuntu
```
`--set-image` uses `container_name=image` syntax like `kubectl set image`. Use `*=ubuntu` to set all containers' images to Ubuntu.
```sh
kubectl set image deployment/myapp *=ubuntu
```


##### 3. Debugging via a shell on the node:

If none of these approaches work, you can find the Node on which the Pod is running and create a Pod running on the Node. 
Lets use `kubectl debug` to create an interactive on a Node
```bash
kubectl debug node/my_node -it --image=ubuntu

Creating debugging pod node-debugger-mynode-pdx84 with container debugger on node mynode.
If you don't see a command prompt, try pressing enter.
root@ek8s:/#
```

- `kubectl debug` auto-generates the Pod name from the Node name.
- The Node's root filesystem mounts at `/host`.
- The container shares host IPC, network, and PID namespaces, but isn't privileged—some actions (like `chroot /host`) may fail.
- For a privileged pod, create one manually or use `--profile=sysadmin`.

##### 5. Debugging pode or node while applying profile

Use `kubectl debug` to apply profiles when debugging nodes or Pods. Profiles set properties like `securityContext` for different scenarios. There are two profile types: **static** and **custom**.

###### 5.1 applying a Static profile

These are a set of predefined properties that we apply using `--profile` flag

| Profile    | Description                                                              |
|------------|-------------------------------------------------------------------------|
| legacy     | A set of properties backwards compatibility with 1.22 behavior           |
| general    | A reasonable set of generic properties for each debugging journey        |
| baseline   | A set of properties compatible with PodSecurityStandard baseline policy  |
| restricted | A set of properties compatible with PodSecurityStandard restricted policy|
| netadmin   | A set of properties including Network Administrator privileges           |
| sysadmin   | A set of properties including System Administrator (root) privileges     |

> Note if you dont specify `--profile` the `legacy` will be used by defautl (This will depricated)

Lets create a pod named `myapp`
```bash
kubectl run myapp --image=busybox:1.28 --restart=Never -- sleep 1d
```
Debug the Pod with an ephemeral container. For privileged access, use the `sysadmin` profile:
```bash
kubectl debug -it myapp --image=busybox:1.28 --target=myapp --profile=sysadmin
Targeting container "myapp". If you don't see processes from this container it may be because the container runtime doesn't support this feature.
Defaulting debug container name to debugger-6kg4x.
If you don't see a command prompt, try pressing enter.
/ #
```
Testing:
```bash
#check the capabilities of the ephemeral container process ($$) by running command:
grep Cap /proc/$$/status
...
CapPrm:	000001ffffffffff
CapEff:	000001ffffffffff
...
#We can also validate that eph-container was created as priviliged
kubectl get pod myapp -o jsonpath='{.spec.ephemeralContainers[0].securityContext}'
{"privileged":true}
```

###### 5.2 applying a Customer profile (stable)

- You can use a custom profile (YAML or JSON) with the `--custom` flag to modify the container spec for debugging.  
- You cannot change the container's name, image, command, lifecycle, or volumeDevices, nor the Pod spec.

Lets create a pod
```bash
kubectl run myapp --image=busybox:1.28 --restart=Never -- sleep 1d
```
And create a customer yaml file
```yaml
env:
- name: ENV_VAR_1
  value: value_1
- name: ENV_VAR_2
  value: value_2
securityContext:
  capabilities:
    add:
    - NET_ADMIN
    - SYS_TIME
```

Then run the debug with the customer profile
```bash
kubectl debug -it myapp --image=busybox:1.28 --target=myapp --profil=general --custom=custom-profile.yaml
```
check
```bash
kubectl get pod myapp -o jsonpath='{.spec.ephemeralContainers[0].env}'
```
```json
[{"name":"ENV_VAR_1","value":"value_1"},{"name":"ENV_VAR_2","value":"value_2"}]
```
```bash
kubectl get pod myapp -o jsonpath='{.spec.ephemeralContainers[0].securityContext}'
```
```json
{"capabilities":{"add":["NET_ADMIN","SYS_TIME"]}}
```

### Debugging Kubernetes Services

While Debugging services you should ask following question

1. **Does the Service exist?**
2. **Are there any Network Policy Ingress rules affecting the target Pods?**
3. **Does the Service work by DNS name?**
    - Does any Service work by DNS name?
4. **Does the Service work by IP?**
5. **Is the Service defined correctly?**
6. **Does the Service have any EndpointSlices?**
7. **Are the Pods working?**
8. **Is the kube-proxy working?**
    - Is kube-proxy running?
    - Is kube-proxy proxying?
    - *Edge case*: A Pod fails to reach itself via the Service IP.

Most of the time to start debugging, you need to check what a Pod sees in the cluster by run an interactive busybox Pod:

```bash
kubectl run -it --rm --restart=Never busybox --image=registry.k8s.io/busybox -- sh
```

Lets considere we have the following Deployement runing image serve_hostname with 3 replicas:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hostnames
  name: hostnames
spec:
  selector:
    matchLabels:
      app: hostnames
  replicas: 3  #3 replicas
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: registry.k8s.io/serve_hostname #serve_hostname is a container that serves its hostname via HTTP on port 9376. For any  other app, use the port the  app Pods are listening on.
```

To confirm the pods are serving we can get the list of POD IP Addresses using :
```bash
kubectl get pods -l app=hostnames -o go-template='{{range .items}}{{.status.podIP}}{{"\n"}}{{end}}'

10.244.0.5
10.244.0.6
10.244.0.7
#pinging them -qO- quiet mode + output to stdout (-=> means stdout)
#using the port of serve_hostname : 9376
for ep in 10.244.0.5:9376 10.244.0.6:9376 10.244.0.7:9376; do
    wget -qO- $ep
done
```

#### 1. Does the Service exit ?

Most of the time the service does not exist, check that with a get:
```bash
kubectl get svc hostnames
No resources found.
Error from server (NotFound): services "hostnames" not found
```
You need to expose it
```bash
#--port on service, --target-port on pod
kubectl expose deployment hostnames --port=80 --target-port=9376
service/hostnames exposed
```

Read it back
```bash
kubectl get svc hostnames
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
hostnames   ClusterIP   10.0.1.175   <none>        80/TCP    5s
```

#### 2. Any Network Policy Ingress rules affecting the pods ?

Check the network policies :
```bash
kubectl get networkpolicies --all-namespaces
kubectl get netpol -A #short
```

#### 3. Does the Service work by DNS Name?

Exec pod in the same namespace:
```bash
nslookup hostnames
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      hostnames
Address 1: 10.0.1.175 hostnames.default.svc.cluster.local
```

if previous fail :
```bash
#.default = namespace, to replace if needed
nslookup hostnames.default
```

if previous fail, try fully-qualified name:
```bash
nslookup hostnames.default.svc.cluster.local
```

You can also from the a node in the cluster
```bash
#k8s dns server : 10.0.0.10
nslookup hostnames.default.svc.cluster.local 10.0.0.10

Server:         10.0.0.10
Address:        10.0.0.10#53

Name:   hostnames.default.svc.cluster.local
Address: 10.0.1.175
```

if only fully-qualified name work but not relative one, check your:

```bash
cat /etc/resolv.conf
```

Should be something like:
```bash
nameserver 10.0.0.10
search default.svc.cluster.local svc.cluster.local cluster.local example.com
options ndots:5
```
- **The `nameserver`** line should point to your cluster's DNS service (set with `--cluster-dns`). 
- **The `search`** line must include the correct suffixes to resolve service names, typically ending with `cluster.local` (set with `--cluster-domain`). Adjust these if your cluster uses different settings. 
- The **`options` line** should set `ndots` to at least 5 so DNS search paths work as expected.

#### 4. Does any Service work by DNS Name ?

If all previous fail, check the k8s Master Service from within a Pod:
```bash
nslookup kubernetes.default

Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local
```
> if this fail also , there is an issue with kube-proxy

#### 5. Does the service work by IP ?

If DNS works, lets check IP, from a pod within the cluster, access service IP from `kubectl get`
```bash
for i in $(seq 1 3); do 
    wget -qO- 10.0.1.175:80
done

hostnames-632524106-bbpiw
hostnames-632524106-ly40y
hostnames-632524106-tlaok
```

Check service definition :
```bash
kubectl get service hostnames -o json/yaml
```

- Is the Service port in `spec.ports[]`?
- Is `targetPort` correct for your Pods?
- Is the port a number (`9376`) or a string (`"9376"`)?
- For named ports, do Pods expose the same name?
- Is the protocol correct?

#### 6. Does the service have any EndpointSlices:

Check the endpoint:
```bash
#Endpoints should have pod ip addresses
kubectl get endpointslices -l k8s.io/service-name=hostnames

NAME              ADDRESSTYPE   PORTS   ENDPOINTS
hostnames-ytpni   IPv4          9376    10.244.0.5,10.244.0.6,10.244.0.7
```
#### 7. Are the pods behind service working ?

Double check that pod are running and `wget`the ips in the `endpoint` which are the pod IPs

#### 8. is kube-proxy running

If you've reached this point, your Service is running, EndpointSlices exist, and Pods are serving traffic. Now, the Service proxy (usually `kube-proxy`) is the likely issue. 

##### 8.1. Is kube-proxy running

On your nodes, check the process :
```bash
ps auxw | grep kube-proxy

root  4194  0.4  0.1 101864 17696 ?    Sl Jul04  25:43 /usr/local/bin/kube-proxy --master=https://kubernetes-master --kubeconfig=/var/lib/kube-proxy/kubeconfig --v=2
```
Check the logs:

```bash
#or use journactl
cat /var/log/kube-proxy.log,
```

**Kube-Proxy Mode**

Check the `kube-proxy` mode (`iptables` or `ipvs`)

- **Iptables mode** 

```bash
iptables-save | grep hostnames
-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376
-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376
-A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR
```

- Each Service port has:
    - 1 rule in `KUBE-SERVICES`
    - 1 `KUBE-SVC-<hash>` chain
- Each Pod endpoint has:
    - A few rules in its `KUBE-SVC-<hash>` chain
    - 1 `KUBE-SEP-<hash>` chain with a few rules

> Rules may vary depending on your configuration (e.g., node-ports, load-balancers).

- **ipvs mode**

```bash
ipvsadm -ln
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
...
TCP  10.0.1.175:80 rr
  -> 10.244.0.5:9376               Masq    1      0          0
  -> 10.244.0.6:9376               Masq    1      0          0
  -> 10.244.0.7:9376               Masq    1      0          0
...
```

For each port of every **Service**, including any **NodePorts**, **external IPs**, and **load-balancer IPs**, `kube-proxy` creates a *virtual server*. 
For each Pod endpoint, it creates corresponding *real servers*: 

- **Service hostname:** `10.0.1.175:80`
- **Endpoints:**
    - `10.244.0.5:9376`
    - `10.244.0.6:9376`
    - `10.244.0.7:9376`

This means that traffic to the service IP and port is load-balanced across all listed endpoints.

##### 8.2. is Kube-Proxy Proxying ?

If above work, using a node, try accessing your service:
```bash
curl 10.0.1.175:80
hostnames-632524106-bbpiw
```

If fail, check kube proxy logs:
Setting endpoints for default/hostnames:default to [10.244.0.5:9376 10.244.0.6:9376 10.244.0.7:9376]

If you dont see this, restart kube-proxy on node, you can restart it on al nodes using:
```bash
kubectl rollout restart daemonset kube-proxy -n kube-system
```

##### 8.3. Edge Case

## Edge Case: Pod Fails to Reach Itself via Service IP

This can occur if the network isn't configured for "hairpin" traffic, especially with kube-proxy in iptables mode and bridge networking. The Kubelet's `--hairpin-mode` flag should be set to `hairpin-veth` or `promiscuous-bridge`.

### Quick Troubleshooting

**Check Kubelet Hairpin Mode**
```sh
ps auxw | grep kubelet | grep hairpin-mode
```
Look for `--hairpin-mode=hairpin-veth` or `--hairpin-mode=promiscuous-bridge`.

**Verify Effective Hairpin Mode**
- Check kubelet logs for lines like:
```sh
 grep hairpin /var/log/kubelet.log
```
Example log:
```
Hairpin mode set to "promiscuous-bridge"
```

**Validate Permissions**
- For `hairpin-veth`:
```sh
for intf in /sys/devices/virtual/net/cbr0/brif/*; do cat $intf/hairpin_mode; done
```
Output should be all `1`.
- For `promiscuous-bridge`:
```sh
ifconfig cbr0 | grep PROMISC
```
Output should include `PROMISC`.

### Debugging a statefulset
```bash
Same as for debugging pod, use kubectl to investigate, look for unknown or terminating pod:

kubectl get pods -l app.kubernetes.io/name=MyApp
```

### Debugging Init Containers

Check init container status:
```bash
#get
kubectl get pod <pod-name>
#here 1/2 means 1 init has completed successully 
NAME         READY     STATUS     RESTARTS   AGE
<pod-name>   0/1       Init:1/2   0          7s
```

Describe the pod for more analyses 
```bash
kubectl describe pod <pod-name>
```
```yaml
#here we have 2 init container
Init Containers:
  <init-container-1>:
    Container ID:    ...
    ...
    State:           Terminated
      Reason:        Completed
      Exit Code:     0
      Started:       ...
      Finished:      ...
    Ready:           True
    Restart Count:   0
    ...
  <init-container-2>:
    Container ID:    ...
    ...
    State:           Waiting
      Reason:        CrashLoopBackOff
    Last State:      Terminated
      Reason:        Error
      Exit Code:     1
      Started:       ...
      Finished:      ...
    Ready:           False
    Restart Count:   3
    ...
```

For quick acccess:
```bash
kubectl get pod nginx --template '{{.status.initContainerStatuses}}'
```
Access logs:
```bash
kubectl logs <pod-name> -c <init-container-2>
```

**Understanding Pod Status**

the Pod status can provide valuable insights. 
If the status begins with `Init:`, it summarizes the execution state of the Init Containers. 

| Status                  | Meaning                                                      |
|-------------------------|--------------------------------------------------------------|
| `Init:N/M`              | The Pod has **M** Init Containers; **N** have completed.     |
| `Init:Error`            | An Init Container has failed to execute.                     |
| `Init:CrashLoopBackOff` | An Init Container has failed repeatedly.                     |
| `Pending`               | The Pod has not yet begun executing Init Containers.         |
| `PodInitializing`       | The Pod is initializing (may be running Init Containers).    |
| `Running`               | All Init Containers have completed; main containers running. |



### Determine the reason for Pod failure

#### 1. Writing and reading a termination message

Lets create this pod
```yaml 
apiVersion: v1
kind: Pod
metadata:
  name: termination-demo
spec:
  containers:
  - name: termination-demo-container
    image: debian
    command: ["/bin/sh"]
    args: ["-c", "sleep 10 && echo Sleep expired > /dev/termination-log"]
```
```bash
kubectl apply -f https://k8s.io/examples/debug/termination.yaml
```

This container sleep for 10sec then write Sleep expired to the `/dev/termination-log`:
```bash
kubectl get pod termination-demo --output=yaml
```
```yaml
apiVersion: v1
kind: Pod
...
    lastState:
      terminated:
        containerID: ...
        exitCode: 0
        finishedAt: ...
        message: |
          Sleep expired          
        ...
```
```sh
#filter on terminated message
kubectl get pod termination-demo -o go-template="{{range .status.containerStatuses}}{{.lastState.terminated.message}}{{end}}"
#if you have multiple containers in one pod and want to discover which is failling
kubectl get pod multi-container-pod -o go-template='{{range .status.containerStatuses}}{{printf "%s:\n%s\n\n" .name .lastState.terminated.message}}{{end}}'
#printf ".name:(container name) back to line .LastState.Terminated.message(message) 2x backline
```
#### 2. Customizing the termination message

Kubernetes retrieves termination messages from the file specified in the `terminationMessagePath` field (default: `/dev/termination-log`). You can customize this path per container, but not after the Pod is launched.

- **Purpose:** Brief final status (e.g., assertion failure), limited to Per container: 4096 bytes  

**Example:**

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: msg-path-demo
spec:
    containers:
    - name: msg-path-demo-container
        image: debian
        terminationMessagePath: "/tmp/my-log"
```

You can also set `terminationMessagePolicy`:
- `"File"` (default): Use only the file.
- `"FallbackToLogsOnError"`: Use last 2048 bytes or 80 lines of logs if the file is empty and the container failed.


## Trouble Shooting Summary Image

![troubleshooting_k8s.png](troubleshooting k8s.png)

