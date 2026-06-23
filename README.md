Readme · MDCopyKubernetes — The Homelab Way

A companion repo to beyondthecert.dev

What This Is

When I started my first job, I walked into a world of Kubernetes operations. I understood what Kubernetes did — I had watched videos, done some studying, heard about it enough. But there is a difference between studying Kubernetes for a certification, understanding it conceptually, and actually operating a cluster. The nuances caught me off guard — not the architecture, but the day to day reality of running things.

What is a StorageClass? Why does swap being enabled break everything? Why can't PostgreSQL run on NFS? None of that was in the introductory videos.

This repo is a letter to who I was a year ago. It's for anyone new to Kubernetes or K8s operations who wants to understand not just what these tools are but why they exist and how they fit together. This is Kubernetes built the homelab way — on real physical hardware, with real operational patterns, and with the lessons learned from things actually breaking.

Who This Is For


Anyone new to Kubernetes who wants hands-on experience beyond tutorials
Engineers early in their career who want to understand K8s operations
Homelab enthusiasts who want to run a production-grade cluster at home
Anyone who has studied for the CKA and wants to go deeper into real cluster operations


What You Will Build

A bare metal Kubernetes cluster using kubeadm on physical machines. By the end you will have:


A multi-node cluster with control plane and worker nodes
Pod networking via Calico
Load balancing via MetalLB
HTTP routing via Nginx Ingress
TLS certificate management via cert-manager
GitOps via ArgoCD
Encrypted secrets via Sealed Secrets
Observability via Prometheus, Grafana, and Loki


Tested Environments

This automation has been validated against:


Bare metal — Lenovo ThinkCentre, Dell laptops, and HP EliteDesk hardware
Local VMs — VirtualBox on a personal PC (NAT Network mode recommended over Bridged if using WiFi — some chipsets don't reliably bridge VM traffic)
Cloud — AWS EC2 (t3.medium or larger; MetalLB will not function as-is since it relies on L2/ARP behavior that doesn't exist in AWS's networking model — skip it with when: false on its two tasks in 06-stack.yml, or wire up AWS's native LoadBalancer integration separately if you need one)


The core OS prep, containerd, kubeadm, and cluster bootstrap logic is the same across all three. Only networking-specific pieces (MetalLB, Bridged WiFi adapters) need environment-specific handling.

Prerequisites

Hardware


At minimum 2 machines — 1 control plane, 1 worker
Each machine: 8GB RAM minimum, 64GB storage minimum
All machines on the same local network


Software


Ubuntu 24.04 LTS installed on each node
Ansible installed on the machine you will run playbooks from — a fresh OS install typically doesn't have this; install with sudo apt install ansible-core
sshpass installed on your Ansible machine if using password-based SSH auth (not needed if using SSH keys)
SSH access from your Ansible machine to all nodes
Python 3 on all nodes (comes with Ubuntu 24.04)


Knowledge


Basic Linux command line comfort
Basic understanding of what Kubernetes is
SSH familiarity


No prior Kubernetes operations experience required. That's what this is for.

Repo Structure

Kubernetes-The-Homelab-Way/
├── inventory/
│   ├── hosts.yml.example     # copy this to hosts.yml and fill in your IPs
│   └── test-hosts.yml.example
├── group_vars/
│   ├── all.yml               # variables shared across all nodes
│   ├── masters.yml           # variables for control plane nodes
│   └── workers.yml           # variables for worker nodes
├── playbooks/
│   ├── 01-os-prep.yml        # OS configuration and swap fix
│   ├── 02-containerd.yml     # container runtime
│   ├── 03-kubeadm-tools.yml  # kubeadm, kubelet, kubectl
│   ├── 04-cluster-init.yml   # initialize the cluster
│   ├── 05-join-nodes.yml     # join worker nodes
│   ├── 06-stack.yml          # MetalLB, Nginx, cert-manager, Sealed Secrets, ArgoCD
│   ├── 07-observability.yml  # Prometheus, Grafana, Loki, Promtail
│   └── reset-node.yml        # drain and reset a node for re-joining
├── site.yml                  # runs all playbooks end to end
├── ansible.cfg
└── GLOSSARY.md

How to Use This Repo

1. Clone the repo

bashgit clone https://github.com/BeyondTheCert/Kubernetes-The-Homelab-Way.git
cd Kubernetes-The-Homelab-Way

2. Set up your inventory

bashcp inventory/hosts.yml.example inventory/hosts.yml

Open hosts.yml and replace the placeholder IPs with the actual IPs of your nodes.

3. Run the full stack

bashansible-playbook site.yml -i inventory/hosts.yml

This runs all playbooks in order — OS prep, container runtime, kubeadm tools, cluster init, node join, stack install, and observability.

Or run individual playbooks

bashansible-playbook playbooks/01-os-prep.yml -i inventory/hosts.yml

Run playbooks individually if you want to step through the process or troubleshoot a specific stage.

Playbook Descriptions

01-os-prep.yml
Prepares each node for Kubernetes. Disables swap permanently, installs nfs-common for NFS volume support, loads required kernel modules, and sets sysctl parameters for networking.

02-containerd.yml
Installs and configures containerd as the container runtime. Kubernetes needs a container runtime on every node to pull images and run containers.

03-kubeadm-tools.yml
Installs kubeadm, kubelet, and kubectl on all nodes. These are the core Kubernetes tools needed to bootstrap and operate the cluster.

04-cluster-init.yml
Initializes the cluster on the first control plane node using kubeadm init. Sets up the kubeconfig, installs Calico as the CNI, and applies the BGP interface fix for nodes also running Tailscale.

05-join-nodes.yml
Joins worker nodes to the cluster. Uses a two-play pattern — the first play retrieves the join command from the master, the second play runs it on all workers.

06-stack.yml
Installs the core platform stack via Helm in the correct order: MetalLB for load balancing, Nginx Ingress for HTTP routing, cert-manager for TLS certificates, Sealed Secrets for GitOps-safe secrets management, and ArgoCD for GitOps.

07-observability.yml
Installs the observability stack: kube-prometheus-stack for metrics and alerting, Grafana for dashboards, Loki for log aggregation, and Promtail for log collection.

reset-node.yml
Safely drains and resets a node so it can be re-joined to the cluster. Useful when replacing hardware or troubleshooting a node.

Requires two separate variables, since a node's Kubernetes-registered name and its Ansible inventory alias aren't always the same thing (they often differ on cloud providers, where Kubernetes uses the machine's auto-generated hostname):

bash ansible-playbook playbooks/reset-node.yml -i inventory/hosts.yml -e target_node_k8s_name=<kubernetes-node-name> -e target_node_inventory_host=<ansible-inventory-alias>

Find the Kubernetes node name with kubectl get nodes. Find the inventory alias in your hosts.yml (e.g. worker1, master).

A Note on the BGP Fix

If you are running Tailscale on your nodes you will need the Calico BGP fix that is built into 04-cluster-init.yml. Calico auto-detects which network interface to use for BGP peering and will pick the Tailscale virtual interface instead of your real ethernet port. This breaks pod networking across nodes.

The fix tells Calico to skip the Tailscale interface specifically (skip-interface=tailscale.*) rather than hardcoding the names of real ethernet interfaces — different hardware vendors and USB ethernet adapters all use different naming conventions, so excluding the one interface you know is wrong works on any hardware, while trying to guess the right one doesn't. See the full writeup at beyondthecert.dev.

Glossary

New to Kubernetes or K8s operations? Check GLOSSARY.md for plain English definitions of every term you will encounter in this repo — from pods and namespaces to etcd, StorageClasses, and DaemonSets.

Follow Along

This repo is the companion to the Kubernetes-The-Homelab-Way series on beyondthecert.dev. Each phase of the build has a corresponding article explaining the decisions, the incidents, and the lessons learned.
