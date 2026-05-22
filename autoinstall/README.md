# Ubuntu Autoinstall

Hands-free Ubuntu 24.04 Server installation for bare metal nodes.

## What Gets Automated
- Locale, keyboard, timezone (UTC by default)
- Network configuration (DHCP)
- User account creation (k8s-admin)
- SSH server enabled
- Swap disabled permanently
- sudo without password for k8s-admin
- Required packages installed

## What Prompts You
- Disk selection — you choose which drive to install on

## Before You Use This
Change the timezone in user-data to your local timezone.
Generate a new hashed password:
```bash
openssl passwd -6 yournewpassword
```
Replace the password field in user-data with the output.

## How to Use

### 1. Download Ubuntu 24.04 Server ISO
https://ubuntu.com/download/server

### 2. Flash ISO to USB
Download Balena Etcher: https://etcher.balena.io
Flash the Ubuntu 24.04 Server ISO to your USB drive.

### 3. Add autoinstall files to USB
After flashing, copy both files to the root of the USB boot partition:
- user-data
- meta-data

### 4. Boot target machine from USB
Enter BIOS/UEFI (F2, F12, or Del on boot).
Set USB as first boot device. Save and reboot.

### 5. Select your disk
Installer prompts for disk selection. Choose your internal SSD/NVMe.

### 6. Walk away
Machine reboots automatically when complete.

### 7. Set hostname after install
```bash
sudo hostnamectl set-hostname master
```
Then proceed with the Ansible playbooks.

## Default Credentials
- Username: k8s-admin
- Password: password1 (change this)
