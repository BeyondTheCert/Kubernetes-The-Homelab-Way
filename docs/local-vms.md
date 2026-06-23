Running on Local VMs (VirtualBox)

This walks through running the Kubernetes-The-Homelab-Way playbooks against VirtualBox VMs on your own PC, instead of physical hardware. Useful for testing changes, learning the stack, or just not wanting to dedicate real hardware yet.

This was tested with VirtualBox on Windows, 2 VMs (1 control plane, 1 worker), Ubuntu Server 24.04 each.

VM specs used


8GB RAM, 4 cores, 40-60GB disk per VM (generous — adjust down if your host machine is constrained; 4GB RAM / 2 cores per VM is a workable minimum)
Ubuntu Server 24.04 LTS


Networking — the part that actually matters

Use NAT Network, not Bridged Adapter, especially if your host connects via WiFi.

Bridged Adapter is the more obvious choice — it gives each VM a real IP directly on your home network, no extra configuration needed. It works fine over Ethernet. Over WiFi, it's unreliable on a number of chipsets (Realtek's WiFi 6E adapters being a known case) because the chipset doesn't reliably relay another device's MAC address over the wireless link. The VM never gets a DHCP lease — you'll see an IPv6 address show up via SLAAC but no IPv4 address at all, no matter how long you wait.

Setting up NAT Network:


File → Tools → Network Manager → NAT Networks tab → Create
Give it a name (anything — doesn't need to relate to Kubernetes)
For each VM: Settings → Network → Adapter 1 → Attached to: NAT Network → select the one you created


Boot the VMs and confirm they got an IP:

baship a

You should see an inet address in the 10.0.2.x range (VirtualBox's default NAT Network subnet) on the main interface (enp0s3 or similar).

The tradeoff: NAT Network isolates your VMs from your actual host machine by default — they can reach each other and the internet, but your Windows/Mac host can't reach into them without extra setup. If you want to SSH from your host machine directly (recommended — far less painful than working inside the VirtualBox console window), you need port forwarding.

Port forwarding for host-to-VM SSH:

File → Tools → Network Manager → NAT Networks → select your network → Port Forwarding → add a rule per VM:

NameProtocolHost PortGuest IPGuest Portssh-control-planeTCP2222<control plane's 10.0.2.x IP>22ssh-workerTCP2223<worker's 10.0.2.x IP>22

Then SSH from your host:

bashssh -p 2222 <username>@127.0.0.1
ssh -p 2223 <username>@127.0.0.1

Important distinction: those forwarded ports only work from your host machine. If you're running Ansible from inside one of the VMs (SSHed into the control plane, then running ansible-playbook from there targeting the worker), use the VMs' real internal NAT Network IPs (10.0.2.x) directly — not the forwarded ports, which only make sense from outside the NAT Network.

Setting up the control node

If you're running Ansible from inside one of the VMs rather than your actual host machine, that VM needs Ansible installed — a fresh Ubuntu Server install doesn't have it:

bashsudo apt update
sudo apt install ansible-core -y

If you're using password-based SSH auth between the VMs (rather than SSH keys), you also need sshpass:

bashsudo apt install sshpass -y

Inventory setup

bashcp inventory/hosts.yml.example inventory/hosts.yml
nano inventory/hosts.yml

Example using password auth and internal NAT Network IPs:

yamlall:
  vars:
    ansible_user: <your-username>
    ansible_ssh_pass: <password>
    ansible_become_pass: <password>
  children:
    masters:
      hosts:
        master:
          ansible_host: 10.0.2.4
          network_interface: enp0s3
    workers:
      hosts:
        worker1:
          ansible_host: 10.0.2.15
          network_interface: enp0s3

Note the ansible_become_pass — required if using password auth, since most tasks run with become: true (sudo) and Ansible needs a password for that separately from the SSH login password.

Run it

bashansible-playbook -i inventory/hosts.yml site.yml

Known friction points

Privilege escalation timeout. Under load (large Helm chart installs, image pulls), Ansible's default 12-second timeout for sudo prompts can be too short. If you hit Timeout (12s) waiting for privilege escalation prompt, increase it in ansible.cfg:

ini[privilege_escalation]
become = True
become_method = sudo
timeout = 30

Slow image pulls. VirtualBox's NAT Network has noticeably less throughput than a real LAN or cloud connection. Some Calico init containers took 20+ minutes to pull on a NAT Network in testing — not a failure, just slow. Check kubectl get pods -A and kubectl describe pod if something looks stuck; if it shows containers actively pulling or recently completed, it's just slow, not broken.

Re-running site.yml against an already-initialized cluster. If you need to re-run the full playbook after a partial failure, kubeadm init will fail with a wall of preflight errors (Port 6443 is in use, /var/lib/etcd is not empty, etc.) if the control plane already initialized successfully on a prior run. This isn't a bug — it's kubeadm correctly refusing to re-initialize over an existing cluster. Either confirm the cluster's already healthy (kubectl get nodes, kubectl get pods -A) and skip re-running entirely, or use reset-node.yml first (see the main README for correct usage) to properly tear down before retrying.
