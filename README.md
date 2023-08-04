# Table of Contents
- [Overview](#overview)
- [Install Arch](#install-arch)
- [Configure the host](#configure-the-host)
- [Set up and configure the VM](#set-up-and-configure-the-vm)
- [Managing your VM](#managing-your-vm)

## Overview

Here we are again. The last time it was NAT, and now it's the smart contract implementation putting VMs in a difficult situation.
This guide will walk you through setting up a Qemu hypervisor for Qubic on Hetzner from scratch running on Arch realtime kernel to provide BM like performance.

This process has been only tested on Hetzner AX102 (7950x3D)

## Install Arch 

Before proceeding, make sure you backed up all your data from the server as it will erase your disks.

1. Go to https://robot.hetzner.com/server and select the server you want to set up.
2. Click on Linux tab and select Arch Linux latest minimal
3. Select your public key if you have one so that you won't need to use password every time you connect via SSH.
4. Activate Linux installation
5. Now go to Reset tab and select Execute an automatic hardware reset and press Send
6. Wait until you get an email from Hetzner saying your Linux installation has been completed.

## Configure the host

SSH into the newly installed system.

1. Install the packages including the realtime kernel.
```
pacman -S --noconfirm linux-rt qemu-system-x86 libvirt virt-manager virt-install qemu-img openbsd-netcat cpupower sudo gzip dosfstools
```
3. Only run this step if the server is running on AMD 7950x3D (AX102) CPU. This will isolate the CPUs from the kernel Qubic will be using and configures 1GB pages for the Qemu process. It also disables CPU vulnerability mitigations so don't use it if you share the server with someone else.
```
sed -i 's/\(GRUB_CMDLINE_LINUX_DEFAULT="\)[^"]*"/\1mitigations=off hugepagesz=1G default_hugepagesz=1G hugepages=118 isolcpus=0-7,16-23 nohz_full=0-7,16-23"/' /etc/default/grub
```
4. Set the CPU speed scaling governor to performance mode
```
sed -i "s/#governor='\([^']*\)'/governor='performance'/" /etc/default/cpupower && systemctl enable cpupower
```
3. Update the grub config and reboot
```
grub-mkconfig -o /boot/grub/grub.cfg && reboot
```
5. If the server doesn't come back within 5 minutes, do a cold reset via Hetzner robot interface (Reset -> Execute an automatic hardware reset).
6. Now that the server is back, create a non-root user (in this example we create the computor user and add it to the libvirt group)
```
useradd -m -G libvirt computor
```
6. (optional) Allow the user to sudo without password prompt
```
echo "computor ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/custom
```
8. (optional) If you want a nice looking shell and auto-complete install fish
```
pacman -S --noconfirm fish && chsh -s /usr/bin/fish && fish
```
* to do the same for the computor user;
  ```
  su - computor
  chsh -s /usr/bin/fish && fish
  ```
## Set up and configure the VM

The following steps are based on the assumption that you are logged in as a non-root user with sudo access.

1. Create the volume image for your VM
```
qemu-img create qubic.img 12G && \
mkfs.vfat -v -F32 qubic.img && \
fatlabel qubic.img "Qubic" && \
sudo mv qubic.img /var/lib/libvirt/images/
```
2. Create a folder in /mnt where you will mount the volume image to when initially adding a files or do updates
```
sudo mkdir /mnt/qemu
```
4. Mount the image and copy the qubic binary and dependencies (NEVER MOUNT THE VOLUME WHILE THE VM IS RUNNING)
```
sudo mount /var/lib/libvirt/images/qubic.img /mnt/qemu
```
2. Download the custom EFI binary that will supercharge your smart contract processing performance
```
curl
```
1. Download the VM domain definition to your server
```
curl
```
2. Add your additional IP virtual MAC address to the domain definition (replace 00:11:22:33:44:ba with your actual virtual MAC address displayed in the IP tab for your additional IP)
```
sed -i 's/YOU_SECONDARY_MAC_ADDRESS/00:11:22:33:44:ba/g' Computor.xml
```
3. Import the VM definition
```
sudo virsh define Computor.xml
```

## Managing your VM

Install virt-manager (`sudo apt install virt-manager`) on your laptop or workstation you normally use, and;
- Connect to the host via File -> Add Connection...
- Tick Connect to remote host over SSH
- Username should be user you created during server setup
- Hostname is the primary ip address of your server
