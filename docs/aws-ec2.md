Running on AWS EC2

This walks through running the Kubernetes-The-Homelab-Way playbooks against AWS EC2 instances instead of physical hardware or local VMs.

This is a portability test, not an EKS alternative. If you actually want production Kubernetes on AWS, use EKS — it manages the control plane for you. This proves the same automation that builds a cluster on hardware in your closet also builds one on a generic cloud VM, with no cloud-specific magic required. Using EKS here would test nothing, since EKS abstracts away the exact layer (kubeadm init, certs, etcd, CNI) these playbooks are built to handle.

Tested with 2 t3.medium instances (1 control plane, 1 worker), Ubuntu Server 24.04.

Prerequisites


An AWS account with EC2 access
A key pair (.pem file). AWS only lets you download the private key once, at creation time — save it somewhere safe immediately.


Launching instances

Configuration:


AMI: Ubuntu Server 24.04 LTS (HVM), SSD Volume Type
Instance type: t3.medium (2 vCPU, 4 GiB RAM) — the free-tier t3.micro (1GB RAM) is too small for kubeadm to run reliably
Number of instances: 2
Storage: bump up from the 8 GiB default to at least 20-30 GiB
Auto-assign public IP: enabled


Cost: roughly $0.0416/hour per instance. Two instances for a few hours of testing costs well under $1 — just remember to terminate (not stop) the instances when done, since stopped instances still incur EBS storage charges.

Security group — fix this before doing anything else

AWS's default security group only allows inbound SSH (22), HTTP (80), and HTTPS (443) from the internet. It does not allow your two instances to talk to each other on the ports Kubernetes actually needs — 6443 (API server), 2379-2380 (etcd), 10250-10259 (kubelet/scheduler/controller-manager), plus CNI overlay traffic.

If you skip this step, the worker join will fail with a connection timeout trying to reach the control plane's API server — confirmed in testing.

Fix it proactively, before launching SSH or cloning anything:

EC2 Console → Security Groups → select the group your instances are using → Edit inbound rules → Add rule:


Type: All traffic
Source: Anywhere-IPv4 (0.0.0.0/0)


This is broader than ideal — it also allows inbound traffic from the public internet on all ports, not just between your two instances. The more precise approach is sourcing the rule from the security group itself (self-referencing), so only members of that group can talk to each other. In testing, the self-reference autocomplete field in the AWS console didn't reliably populate — if you hit the same issue, Anywhere-IPv4 is an acceptable fallback for a short-lived test, but don't leave it that way on anything long-running.

SSH into both instances

bashssh -i "/path/to/your-key.pem" ubuntu@<control-plane-public-ip>
ssh -i "/path/to/your-key.pem" ubuntu@<worker-public-ip>

ubuntu is the default login user on Canonical's official Ubuntu AMI.

On the worker, confirm its private IP:

baship a

Note the inet address on ens5 (or similar) — you'll need it for the inventory.

Clone the repo and set up inventory (on the control-plane instance)

bashgit clone https://github.com/BeyondTheCert/Kubernetes-The-Homelab-Way.git
cd Kubernetes-The-Homelab-Way/08-ansible
cp inventory/hosts.yml.example inventory/hosts.yml
nano inventory/hosts.yml

yamlall:
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/your-key.pem
  children:
    masters:
      hosts:
        master:
          ansible_host: <control-plane-private-ip>
          network_interface: ens5
    workers:
      hosts:
        worker1:
          ansible_host: <worker-private-ip>
          network_interface: ens5

Use private IPs (172.31.x.x), not public ones. Both instances are in the same VPC by default, so they reach each other directly over AWS's internal network — faster, free (no data transfer charges), and works regardless of the public-facing security group rules.

Get the SSH key onto the control-plane instance

Since Ansible runs from the control-plane instance and needs to SSH into the worker, the private key itself needs to exist there too — not just on your local machine.

From your local machine:

bashscp -i "/path/to/your-key.pem" "/path/to/your-key.pem" ubuntu@<control-plane-public-ip>:~/your-key.pem

On the control-plane instance:

bashchmod 600 ~/your-key.pem

SSH refuses to use a private key with overly permissive file permissions — scp doesn't preserve the restrictive permissions a key needs, so this step is required every time.

Install Ansible on the control-plane instance

A fresh EC2 Ubuntu instance doesn't have it:

bashsudo apt update
sudo apt install ansible-core -y

(sshpass is not needed here since EC2 uses key-based auth, not passwords.)

Skip MetalLB

MetalLB relies on Layer 2/ARP-based IP announcement to assign LoadBalancer-type Service IPs — a mechanism that doesn't function in AWS's networking model. AWS's actual equivalent is an Elastic Load Balancer, provisioned via the cloud-provider-aws controller — wiring that up is outside the scope of this automation.

Before running site.yml, add when: false to both MetalLB tasks in playbooks/06-stack.yml:

yaml    - name: Add metallb chart repo
      kubernetes.core.helm_repository:
        name: metallb
        repo_url: "https://metallb.github.io/metallb"
      when: false
    - name: Deploy latest version of Metallb chart inside metallb namespace (and create it)
      kubernetes.core.helm:
        name: metallb
        chart_ref: metallb/metallb
        release_namespace: metallb
        kubeconfig: /etc/kubernetes/admin.conf
        create_namespace: true
        values:
          frr-k8s:
            prometheus:
              serviceMonitor:
                enabled: false
      when: false

Both tasks will show skipping: [master] in the playbook output — that's expected and correct.

Run it

bashansible-playbook -i inventory/hosts.yml site.yml

Known friction points

Privilege escalation timeout. Same as the local VM environment — under load, Ansible's default 12-second sudo prompt timeout can be too short. Increase it in ansible.cfg before running:

ini[defaults]
host_key_checking = False
inventory = inventory/hosts.yml
timeout = 30

[privilege_escalation]
become = True
become_method = sudo
timeout = 30

Re-running site.yml after a partial failure. If the control plane already initialized successfully and you re-run the whole playbook, kubeadm init will fail with preflight errors since the ports/files/directories it expects to be empty are already in use by the running cluster. If you only need to resume past a single failed task, use --start-at-task="<task name>" instead of re-running everything:

bashansible-playbook -i inventory/hosts.yml site.yml --start-at-task="Add sealed-secrets chart repo"

Be aware this skips any variables registered by earlier tasks — if a later task depends on a register:'d variable from a task before your resume point (for example, the Tailscale detection check used by the Calico BGP fix), you may need to skip past that dependent task too, or run it manually.

Result

Both nodes reached Ready. Full stack — Calico, Nginx Ingress, cert-manager, Sealed Secrets, ArgoCD, Prometheus, Grafana, Loki, Promtail — deployed successfully with MetalLB skipped. loki-chunks-cache and loki-results-cache pods may sit in Pending/ContainerCreating in a test environment without persistent storage configured — expected, not a failure.

Don't forget to terminate the instances when you're done.
