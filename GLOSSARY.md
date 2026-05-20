# Kubernetes Homelab Glossary

> Part of [Kubernetes-The-Homelab-Way](https://github.com/BeyondTheCert/Kubernetes-The-Homelab-Way) · [beyondthecert.dev](https://beyondthecert.dev)
>
> A living document — updated as the stack evolves.

---

## How It All Fits Together

Before jumping into individual terms — here's the big picture. A Kubernetes cluster has two layers: the control plane and the worker nodes. The control plane makes decisions. The worker nodes run the actual workloads. etcd stores the state of everything. kubelet on each node takes instructions from the control plane and executes them. Pods run on workers. Everything communicates through the API server. Once you have that mental model, the individual pieces start making a lot more sense.

---

## Foundational Concepts

### Kubernetes
Kubernetes is an open source container orchestrator. The word orchestrator sounds fancy but the job is pretty straightforward — if you have thousands of containers running across different machines, Kubernetes manages all of that for you. If a container dies it brings it back. If traffic spikes it scales up. Back in the day you would have to do all of that manually. Kubernetes takes care of it.

### Bare Metal Cluster
A bare metal cluster is exactly what the name says — a cluster built entirely with physical machines. Actual hardware. A laptop, a mini PC, a NAS for storage. No virtual machines in between. You own the hardware, you own the cluster. This is different from a cloud cluster where the provider handles the underlying machines. Bare metal gives you full control but also full responsibility — if a node goes down, you're the one who has to go fix it.

### Node
A node is a machine in the cluster — physical or virtual. Think of it as a computer. It could be a ThinkPad, a ThinkCentre, a cloud VM, whatever. Kubernetes runs workloads across a pool of nodes. There are two types: control plane nodes which are the brain of the cluster, and worker nodes which are where your actual workloads run.

### Pod
A pod is the smallest unit in Kubernetes. A pod is not a container — it encapsulates a container image. When the container runtime pulls an image from a registry it creates a pod. Think of the pod as the wrapper and the container as what's inside. Most of the time one pod runs one container, but a pod can run multiple containers that need to work closely together.

### Namespace
A namespace is a way to group resources inside a cluster. Think of a house with many rooms — each room belongs to a different person or team. The `immich` namespace holds everything related to Immich. The `monitoring` namespace holds Prometheus and Grafana. Each namespace is its own section of the cluster. Resources in different namespaces don't interfere with each other, and you can set permissions and resource limits per namespace.

### Container
A container is a packaged unit of software. It bundles your application code along with everything it needs to run — dependencies, libraries, binaries. The container runs the same way regardless of what machine it's on because everything it needs is inside it. Think of it as a self-contained box. You carry that box to any machine and it just works. Docker and containerd are examples of container runtimes that manage containers on each node.

### Control Plane
The control plane is the brain of the cluster. It makes all the operational decisions — which pod goes on which node, how many replicas should be running, what the desired state of the cluster looks like. The control plane is made up of several components: the API server which is the front door to the cluster, etcd which is the database, the scheduler which decides where pods go, and the controller manager which makes sure the actual state matches what you asked for. In a proper high availability setup you run the control plane across at least 3 nodes so if one goes down the others keep running.

### kubectl
kubectl is the command line tool for Kubernetes. It's how you interact with the cluster — deploy applications, check pod status, view logs, drain nodes, everything. Think of it as the terminal for your cluster. Most of what you do day to day as a Kubernetes operator goes through kubectl.

### kubeconfig
A kubeconfig is a file that lets you interact with a cluster. It holds a few key things — the address of the cluster, the credentials to authenticate, and which context you're working in. When you run kubectl commands, it reads from the kubeconfig to know which cluster to talk to and how to authenticate. On most systems it lives at `~/.kube/config`. If you manage multiple clusters you can switch between them by changing the active context in the kubeconfig.

---

## Cluster Components

### kubeadm
kubeadm is a tool that bootstraps a Kubernetes cluster. Instead of manually installing and configuring every control plane component, kubeadm automates all of that. You run `kubeadm init` on your first master node and it brings up a working cluster. You run `kubeadm join` on worker nodes and they join the cluster. It handles certificates, kubeconfigs, and component configuration so you don't have to.

### kubelet
kubelet is an agent that runs on every single node. It's the middleman between the node and the API server. When the control plane decides a pod should run on a node, kubelet is what actually makes that happen — pull the image, start the container, report back. If a container crashes kubelet detects it and restarts it. If kubelet itself stops running, the node goes NotReady and the cluster treats it as unavailable. This is why swap being enabled breaks things — kubelet checks for it on startup and refuses to run.

### etcd
etcd is the database of the cluster. It stores the entire state of Kubernetes — every pod, every deployment, every secret, every node registration. When you run `kubectl get pods` the API server reads from etcd. When you create a deployment the API server writes to etcd. If etcd goes down the cluster can't make decisions. If etcd data is lost without a backup, the cluster state is gone. Backing up etcd regularly is one of the most important operational habits you can build.

### CNI (Container Network Interface)
CNI stands for Container Network Interface. Kubernetes doesn't handle pod networking itself — it defines a standard and lets you plug in whatever networking solution you want. The CNI is what allows pods to communicate with each other across nodes. Without a CNI installed pods can't talk to each other at all. Calico, Flannel, and Cilium are all examples of CNI plugins. You pick one when you bootstrap the cluster.

### Calico
Calico is a CNI plugin — one specific implementation of the Container Network Interface. It handles pod networking using BGP to advertise pod IP ranges across nodes. Each node manages a subnet of pod IPs and Calico makes sure every node knows how to route traffic to pods on other nodes. One important gotcha if you're also running Tailscale: Calico auto-detects which network interface to use for BGP peering and may pick the Tailscale virtual interface instead of your real ethernet port. That breaks pod networking. The fix is explicitly telling Calico which interfaces to use.

### BGP (Border Gateway Protocol)
BGP is the protocol Calico uses to share pod routing information between nodes. Each node advertises its pod IP range to other nodes so traffic knows where to go. The reason this matters for homelab clusters running Tailscale is that Tailscale creates a virtual network interface on each machine. Calico sees that interface and may try to use it for BGP peering instead of the real ethernet interface. BGP over Tailscale doesn't work — Tailscale is an application-layer VPN that doesn't carry routing protocols. The result is pods can't reach each other across nodes.

### MetalLB
MetalLB is a load balancer for bare metal clusters. In cloud environments when you create a LoadBalancer service the cloud provider automatically provisions a load balancer and gives it an IP. On bare metal there's no cloud provider to do that. MetalLB fills that gap — it watches for LoadBalancer services and assigns them IPs from a pool you configure. This is how services running in the cluster get reachable IP addresses on your local network.

### Ingress Controller
An Ingress controller is the HTTP routing layer of the cluster. MetalLB gives services IP addresses. The Ingress controller routes HTTP and HTTPS traffic to the right service based on hostname and path. Both `grafana.yourdomain.com` and `immich.yourdomain.com` can share the same IP — the Ingress controller looks at the host header and sends traffic to the right place. Nginx Ingress is a common choice for bare metal clusters.

### PersistentVolume (PV) and PersistentVolumeClaim (PVC)
Pods are ephemeral — when a pod restarts any data written to its local filesystem is gone. PersistentVolumes solve this. A PersistentVolume is the actual storage resource — a disk, an NFS share, a Longhorn block device. A PersistentVolumeClaim is a pod's request for storage — "I need 10GB with these access modes." Kubernetes matches the PVC to an available PV and the pod mounts it. When the pod restarts the data is still there because the storage lives outside the pod. Stateless workloads don't need PVs. Stateful workloads like databases do.

### StorageClass
A StorageClass defines how storage gets provisioned. Instead of manually creating a PV for every pod that needs storage, a StorageClass automates it. When a PVC requests storage from a specific StorageClass, a provisioner creates the PV automatically. An NFS-backed StorageClass creates a subdirectory on the NFS share. A Longhorn StorageClass carves out block storage from node disks. You can have multiple StorageClasses for different storage types — fast block storage for databases, NFS for file storage like photos.

### Helm
Helm is the package manager for Kubernetes. Think of it like apt on Ubuntu or yum on Red Hat — but for Kubernetes applications. Instead of managing dozens of individual YAML files for a complex app, Helm bundles everything into a chart. You run `helm install` and it deploys the whole thing with one command. Charts are configurable through values so you can customize without editing the underlying templates. Helm also tracks what's installed so upgrades and uninstalls are clean.

### ArgoCD and GitOps
GitOps is an approach where Git is the source of truth for your cluster state. Instead of running `kubectl apply` manually, you commit changes to a Git repository and automation applies them. ArgoCD is a GitOps tool that watches a Git repo and continuously syncs the cluster to match what's there. If someone manually changes something in the cluster that doesn't match Git, ArgoCD detects the drift and corrects it. Your cluster state becomes version controlled, auditable, and recoverable.

### Sealed Secrets
Regular Kubernetes Secrets are only base64 encoded — not encrypted. Committing one to a public Git repo exposes the sensitive value to anyone who clones it. Sealed Secrets fixes this. It encrypts the secret using a key that only your cluster has. The encrypted SealedSecret can be safely committed to Git. When ArgoCD syncs it, the Sealed Secrets controller decrypts it and creates a regular Kubernetes Secret. Nobody who clones your repo can read the value without access to your cluster.

### cert-manager
cert-manager automates TLS certificate lifecycle management. Without it you manually generate certificates, track expiry dates, renew them, and update deployments. cert-manager watches for Certificate resources and handles all of that — requesting, storing, renewing, rotating. It works with Let's Encrypt for public certificates or can issue self-signed certificates for internal services.

### Longhorn
Longhorn is a distributed block storage system for Kubernetes. It pools the local disks across your cluster nodes into a replicated storage layer. When a pod requests a PV from Longhorn, it carves out block storage from the node disks and presents it as a real disk to the pod. Unlike NFS which is a network filesystem, Longhorn provides proper block storage with reliable file locking — important for databases like PostgreSQL that need it to function correctly.

### NFS (Network File System)
NFS is a protocol that lets machines share directories over a network. One machine — in this case a NAS — exports a directory. Other machines mount that directory as if it were a local folder. In Kubernetes, NFS shares become PersistentVolumes. It's great for file-based workloads like photo storage where multiple pods might need to read and write the same files at the same time. It's not great for databases — PostgreSQL and similar tools need block storage with proper file locking, which NFS doesn't reliably provide over a network.

### DaemonSet
A DaemonSet ensures one copy of a pod runs on every node in the cluster. When a new node joins, the DaemonSet automatically schedules a pod on it. DaemonSets are used for things that need to run everywhere — network plugins like Calico, log collectors like Promtail, monitoring agents. When you drain a node you have to explicitly ignore DaemonSet pods because they're supposed to be there and can't be evicted normally.

### Taint and Toleration
A taint is a label you put on a node that says "don't schedule pods here unless they explicitly allow it." If a node has a taint that says `red`, only pods that have a toleration for `red` can run on it. Think of a taint as a restriction and a toleration as the exception that overrides it. Control plane nodes have a taint by default so regular workloads don't get scheduled there. If you want a pod to run on a tainted node — like an etcd backup job that needs to run on master — you add a toleration to the pod spec.

### Node Drain
Draining a node is the process of safely removing all workloads from it before taking it offline. When you drain a node Kubernetes gracefully evicts all pods and reschedules them on other nodes. The node is then marked as unschedulable so nothing new gets placed on it. This is the right way to take a node offline for maintenance — patching the OS, rebooting, hardware work. Draining first means your workloads keep running on other nodes while you work on the drained one.

### Static Pod
A static pod is managed directly by kubelet on a specific node, not by the Kubernetes API server. The control plane components themselves — the API server, etcd, scheduler, controller manager — all run as static pods. Their manifests live in `/etc/kubernetes/manifests` on the master node. kubelet watches that directory and starts those pods automatically. This is why the control plane can come up even before the API server is fully running — kubelet doesn't need the API server to start static pods.

### Swap
Swap is a Linux feature that uses disk space as overflow memory. When RAM fills up the OS moves data from RAM to a swap file on disk. Kubernetes requires swap to be disabled because it manages container memory precisely. If swap is enabled a container that exceeds its memory limit might start using swap instead of being killed and restarted as Kubernetes expects. kubelet checks for swap on startup and refuses to run if it finds it. On Ubuntu swap is enabled by default — you have to explicitly disable it and delete the swap file or it comes back after a reboot.

---

*This glossary is a living document. Terms will be added as the stack evolves.*
