# workspaces-server-setup

# Table of contents

[TOC]

# What is this project?

[Chad's](https://gitlab.com/cwoolley-gitlab) notes and supporting files for setting up [GitLab Remote Development Workspaces](https://docs.gitlab.com/ee/user/workspace/) from scratch.

For now, it only contains notes on setting up my home bare-metal server.

# Overview

The approach I am taking is simple and manual. Automation is not a priority for now. For now, the goal is to ensure I document every step in detail for future reference (and potential automation).

Some of this is very specific to my particular server, home network setup, and ISP, as well as the versions of OS/software used.

# Server hardware info

- https://www.theserverstore.com/Dell-PowerEdge-R630-36-CORE-Virtualization-Server-192GB-3x-800GB-SSD-H730p_p_988.html
- https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://i.dell.com/sites/doccontent/shared-content/data-sheets/en/documents/dell-poweredge-r630-spec-sheet.pdf&ved=2ahUKEwiOmZmr_bCEAxW3FjQIHexrAh8QFnoECCYQAQ&usg=AOvVaw0GT_z0LYLwK9Rt-9iKSgBz
- https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://i.dell.com/sites/doccontent/shared-content/data-sheets/en/Documents/Dell-PowerEdge-R630-Technical-Guide-v1-6.pdf&ved=2ahUKEwiOmZmr_bCEAxW3FjQIHexrAh8QFnoECCgQAQ&usg=AOvVaw0UcLDArCKOOhUeVQuWJ_oZ
- https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://www.dell.com/support/manuals/en-us/poweredge-r630-dsms/r630_om_pub/dell-poweredge-r630-system-overview%3Fguid%3Dguid-a743b271-3784-4599-8c90-0fbca035aeb0%26lang%3Den-us&ved=2ahUKEwiO2JzZ_rCEAxXVHjQIHdIyBgwQFnoECA4QAQ&usg=AOvVaw14gsicB1V6nroCDHgMTogF
- https://www.dell.com/support/manuals/en-us/poweredge-r630-dsms/r630_om_pub/dell-poweredge-r630-system-overview?guid=guid-a743b271-3784-4599-8c90-0fbca035aeb0&lang=en-us

# Network info

## Basic network and machine IP info

- Home routers and subnets:
    - External modem/router subnet: `10.0.0.0` (XFinity model CGM4981COM modem/router - admin webpage http://10.0.0.1/)
    - Internal router subnet (all home machines): `192.168.1.0` (TP-Link Archer - admin webpage http://192.168.1.1/webpages/login.html)
    - All admin passwords in 1password
- Main development machine `192.168.1.111`
- `poweredge` server: `192.168.1.200`, connected to `eno1`, MAC `14:18:77:57:75:36`
- Router DHCP configured to assign static IPs based on MAC addresses

## Dynamic DNS Raspberry PI setup

- Raspberry PI Zero 2 W
- DNS provider namecheap.com
    - https://www.namecheap.com/support/knowledgebase/article.aspx/595/11/how-do-i-enable-dynamic-dns-for-a-domain/
    - https://www.namecheap.com/support/knowledgebase/article.aspx/5/11/are-there-any-alternate-dynamic-dns-clients/
- TODO: finish setup, test, and document    

## Main development machine setup

- Add `192.168.1.200 poweredge` to `/etc/hosts`
- `brew install k9s` (text-based kubernetes management interface)

# Install Ubuntu Desktop LTS

## Why Ubuntu desktop instead of server?

Because:

- I'm not an expert in Linux (even though I've installed and used it since it came out, including for a couple of years as my main development OS), and sometimes things are easier to do or debug with a GUI.
- Other people who may try to follow these instructions on their own may benefit from having access to a GUI
- Ubuntu desktop is (I assume) more likely to come with drivers/etc for my hardware, so I don't have to waste time futzing with those myself (been there, done that, lost countless hours of my life to it)
- I can use other useful GUI tools if desired, e.g. Rancher Desktop.

If this were a production server, that would be a different story.

## Download Ubuntu, burn to installation media, and start install

- https://ubuntu.com/download/desktop
- Follow instructions to burn to DVD or USB drive (using balenaEtcher - MacOS Sequoia and Windows 11 don't seem to even recognize the Ubuntu ISO image). I used a WD 120GB USB portable drive.
- Plug in USB to server
- F11 to get boot menu in BIOS
- Boot to USB

## Ubuntu initial install

Basic steps like language, keyboard layout, etc are omitted. All defaults are assumed, only changes from defaults are documented.

- Installer asked to auto-update, I did it, then re-launched from desktop
- Type of installation: Interactive
- Applications: Extended selection
- Optimise your computer: Check "Install third-party software..."
- Disk setup: Manual installation (I had issues with grub customization and boot-from-lan when choosing automatic disk setup on a previous install)
    - Delete any existing partitions on existing drive (`sda`)
    - Pick `sda` for bootloader installation (creates ~1GB FAT32 partition)
    - Create 1000GB ext4 partition with mount point of `/` (leaving ~900 GB free for future partitions)
- Create account
    - Username: `cwoolley` (same as my main dev laptop, so I don't have to specify username for SSH, just host. If your username is different you can configure it in your ssh config)
    - Computer name: `poweredge`
    - So, wherever it says `ssh to server`, I'm typing `ssh poweredge`
- Wait for install to finish, then restart, and wait for reboot and welcome screen
- Take all defaults and finish
- From "Show Apps", run software updater and do all updates

# Basic application setup

## General notes

- Use Firefox from Ubuntu GUI as browser

## No-password sudo

- `sudo visudo`
- Change `%sudo ...` line to `%sudo ALL=(ALL) NOPASSWD: ALL`

## Package installation

- `sudo apt update`
- `sudo apt install -y net-tools` - for `ifconfig` and others

## sshd server setup

- `sudo apt update`
- `sudo apt install -y openssh-server`
- `sudo systemctl status ssh`

### Add public ssh key

- From main development machine, copy public key: `cat ~/.ssh/id_rsa.pub | pbcopy`
- ssh to server
- `sudo vi ~/.ssh/authorized_keys`
- paste public key and save

## Nomachine NX

Allows access to server GUI from main development machine.

- Download
    - https://www.nomachine.com
    - download free edition
    - Linux DEB amd64
- Double click downloaded .deb, open with App Center, install
- On main development machine, install nomachine client, and access via hostname (TODO: document config)

# Kubernetes initial tools installation and configuration

- See https://kubernetes.io/docs/setup/production-environment/
- These docs are for Kubernetes v1.32
- Unless otherwise noted, all commands are run directly on server

## Disable swap on kubernetes host server

This is required for kubernetes hosts (https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#swap-configuration)

- `sudo vi /etc/fstab`
- comment (prepend `#`) or delete `/swap.img` line
- `sudo reboot`
- `free -h` and verify swap line shows all zeros

## Install kubeadm

- Following https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/
- `nc 127.0.0.1 6443 -v` to check port (should be connection refused now, because we haven't set anything up yet. Don't know why Kubernetes docs have it in this order)

## Set up containerd

Set up `containerd` - https://kubernetes.io/docs/setup/production-environment/container-runtimes/

### Enable IPv4 packet forwarding

- Enable IPv4 packet forwarding:
```
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```
- Test with `sysctl net.ipv4.ip_forward`

### Install `containerd`:

- See https://github.com/containerd/containerd/blob/main/docs/getting-started.md for reference.
- Note that this points you to https://docs.docker.com/engine/install/debian/ for the `apt-get` installation, but we don't actually follow these, because we don't want all of docker. Just `containerd`, and that doesn't even need the docker apt repo added.
- `sudo apt update`
- `sudo apt -y install containerd`
- Verify: `containerd --version`:  `containerd github.com/containerd/containerd/v2 v2.0.0 207ad711eabd375a01713109a8a197d197ff6542`

### Configure `systemd` cgroup driver:

- See https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-drivers and https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd
- `sudo mkdir -p /etc/containerd`
- Make default config file `sudo containerd config default | sudo tee /etc/containerd/config.toml`
- `sudo vi /etc/containerd/config.toml`
- Search for (`/` in vim): `plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options` section
- Add `SystemdCgroup = true` to that section
- Restart: `sudo systemctl restart containerd`
- Verify:
    - `sudo systemctl status containerd`
    - `sudo journalctl -u containerd --no-pager -n 50`

## Install `nerdctl`:

- See https://github.com/containerd/nerdctl
- `cd ~/Downloads`
- `wget https://github.com/containerd/nerdctl/releases/download/v2.0.2/nerdctl-full-2.0.2-linux-amd64.tar.gz`
- `sudo tar Cxzvvf /usr/local nerdctl-full-2.0.2-linux-amd64.tar.gz`
- Verify:
    - `nerdctl --version`: `nerdctl version 2.0.2`
    - `sudo nerdctl --debug run ruby:alpine ruby --version` (check for successful output of ruby version)
- Clean up any running containers: `sudo nerdctl rm $(sudo nerdctl ps -aq)`    

## Install kubeadm, kubelet, and kubectl

- See https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ and https://kubernetes.io/docs/tasks/tools/
- `sudo apt update`
- `sudo apt install -y apt-transport-https ca-certificates curl gpg` (command from docs, only `apt-transport-https` was actually newly installed)
- `curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg`
- `echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list`
- `sudo apt update`
- `sudo apt install -y kubelet kubeadm kubectl`
- `sudo apt-mark hold kubelet kubeadm kubectl`
- Verify `kubectl version`:
```
Client Version: v1.32.0
Kustomize Version: v5.5.0
```
- Verify `kubeadm version -o yaml`
```
clientVersion:
  buildDate: "2024-12-11T18:04:20Z"
  compiler: gc
  gitCommit: 70d3cc986aa8221cd1dfb1121852688902d3bf53
  gitTreeState: clean
  gitVersion: v1.32.0
  goVersion: go1.23.3
  major: "1"
  minor: "32"
  platform: linux/amd64
```
- Verify `kubelet --version`
```
Kubernetes v1.32.0
```
- Enable kubelet service: `sudo systemctl enable --now kubelet`. From docs: "_The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do._"

# Kubernetes cluster setup

See https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

- Verify usable IP via `ip route show`: `192.168.1.0/24 dev eno1 proto kernel scope link src 192.168.1.200 metric 100`

## Init kubernetes cluster

- See https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#more-information and https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/

### Run `kubeadm init`

- `sudo kubeadm init`
- It should work, and end with output like this:
```
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

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.200:6443 --token TOKEN \
	--discovery-token-ca-cert-hash sha256:f7475baf329ec7d4eaff11ecfa750adfeb9e3498cbfe83bc7d9c8f93802dfa74
```

### Setup to start using cluster

(using commands from `kubeadm init` output above)

- `mkdir -p $HOME/.kube`
- `sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`
- `sudo chown $(id -u):$(id -g) $HOME/.kube/config`
- Verify `kubectl get nodes`
```
NAME        STATUS   ROLES           AGE     VERSION
poweredge   Ready    control-plane   9m23s   v1.32.0
```
- Verify `kubectl get pods -n kube-system`
```
NAME                                READY   STATUS    RESTARTS   AGE
coredns-668d6bf9bc-khnsf            1/1     Running   0          9m58s
coredns-668d6bf9bc-lwhw2            1/1     Running   0          9m58s
etcd-poweredge                      1/1     Running   0          10m
kube-apiserver-poweredge            1/1     Running   0          10m
kube-controller-manager-poweredge   1/1     Running   0          10m
kube-proxy-xv7np                    1/1     Running   0          9m58s
kube-scheduler-poweredge            1/1     Running   0          10m
```
- Verify `kubectl cluster-info`:
```
Kubernetes control plane is running at https://192.168.1.200:6443
CoreDNS is running at https://192.168.1.200:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## Set up Pod network add-on

- See https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network and https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy
- Use Calico: https://www.tigera.io/project-calico/, github https://github.com/projectcalico/calico
- `3.29.1` is latest current release
- `kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/calico.yaml`
- Verify `kubectl get pods -n kube-system -l k8s-app=calico-node`
```
NAME                READY   STATUS    RESTARTS   AGE
calico-node-hxpm5   1/1     Running   0          2m16s
```
- Verify `kubectl get pods -n kube-system -l k8s-app=calico-kube-controllers`
```
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-5745477d4d-bbx2d   1/1     Running   0          2m48s
```
- Verify `kubectl get pods -n kube-system | grep calico`
```
calico-kube-controllers-5745477d4d-bbx2d   1/1     Running   0          3m14s
calico-node-hxpm5                          1/1     Running   0          3m14s
```

## Allow cluster to schedule pods on control plane node

- See https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#control-plane-node-isolation
- `kubectl taint nodes --all node-role.kubernetes.io/control-plane-`
- Look for output `node/poweredge untainted`
- Verify `kubectl describe node | grep Taints`
```
Taints:             <none>
```

## Run an image to verify setup

- `kubectl run nginx --image=nginx`
- Verify with `kubectl get pods`
- Wait for `STATUS` to be `Running`
- Cleanup: `kubectl delete pod nginx`
