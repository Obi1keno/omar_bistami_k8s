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


# SERVICES

Kubernetes uses transient, decoupled objects. Services connect Pods or provide external access, allowing Pods to be replaced seamlessly. Labels help reconnect new Pods, and Endpoints route traffic. Google’s Extensible Service Proxy (ESP) offers more flexibility but is mainly used in Google environments.

There are multiple service types, each exposing resources internally or externally. Services can also link internal resources to external ones, like third-party databases.

The kube-proxy agent monitors the API for new services and endpoints, opens random ports, and redirects traffic to service endpoints.

Services offer automatic load-balancing based on labels, with optional session affinity. Headless services can be configured without a fixed IP or load-balancing.

Unique IPs are managed via etcd, and Services use iptables for routing, with potential for other technologies in the future.

Labels determine which Pods receive traffic from a Service. Since labels can be updated dynamically, changing them may affect which Pods are selected by the Service.

By default, Kubernetes uses a rolling deployment strategy: new Pods with updated application versions are gradually added, and traffic is automatically load balanced between old and new Pods. This ensures continuous availability during updates.

However, if different application versions are incompatible—meaning clients may have issues communicating with multiple versions—you should use more specific labels, such as including a version number.

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


# SERVICES TP

**Services** (also called *microservices*) are objects that define policies for accessing a logical set of Pods. Services are typically assigned labels, enabling persistent access to resources even when front-end or back-end containers are terminated and replaced.

- **Native applications** can use the Endpoints API to access services.
- **Non-native applications** can use a Virtual IP-based bridge to reach back-end Pods.

**Service Types:**
- **ClusterIP** (default): Exposes the service on a cluster-internal IP, making it reachable only within the cluster.
- **NodePort**: Exposes the service on each node’s IP at a static port. A ClusterIP is also automatically created.
- **LoadBalancer**: Exposes the service externally using a cloud provider’s load balancer. Both NodePort and ClusterIP are automatically created.
- **ExternalName**: Maps the service to the value of `externalName` using a CNAME DNS record.

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