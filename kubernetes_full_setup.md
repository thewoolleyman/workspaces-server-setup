# Kubernetes Full setup

# Table of contents

[TOC]

# Overview

- This doc contains the version of instructions for a "Full" kubernetes installation, based on https://kubernetes.io/docs/setup/production-environment/
- It is an alternative approach to the [kubernetes k3s setup](./kubernetes_k3s_setup.md), which is documented separately.

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

# Disable IPv6

THis was necessary because the initial `kubeadm init` did not properly bind to the IPv4 `0.0.0.0` interface.

- Create a sysctl config:
```
sudo tee /etc/sysctl.d/60-disable-ipv6.conf << 'EOF'
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
```
- `sudo sysctl -p /etc/sysctl.d/60-disable-ipv6.conf`
- Verify `cat /proc/sys/net/ipv6/conf/all/disable_ipv6`: Should be `1`

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
  - **_IMPORTANT NOTE!!! - If the CLI tool versions such as kubectl are incompatible with the server, you can get obscure errors such as "No route to host".
    This can happen even if the minor versions are close or perhaps even the same. If you have problems accessing the cluster via CLI tools, ensure you have
    downloaded and installed the latest (or matching) versions of the CLI tools, AND are actually using that version and not a different one on your path._**
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
- Verify `sudo ss -tulpn | grep 6443`. It should have IPv4 entry:
```
tcp   LISTEN 0      4096               *:6443             *:*    users:(("kube-apiserver",pid=12995,fd=3))
```
- **NOTE! `sudo netstat -tulpn | grep 6443` (using `netstat` instead of `ss`) will return a line with `tcp6`, even if IPv6 is disabled!**

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

### Note on redoing `kubeadmin init`

See https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#tear-down 

- Install ipvsadm: `sudo apt install ipvsadm`

Reset:

- `sudo kubeadm reset`
- `sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X`
- `sudo ipvsadm -C`

Reinstall using `kubectl init` process above.

Then after init is successful, copy over new config to `~/.kube/config` and change permissions:

- `sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config`

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

# Set up MacOS development machine to administer cluster

## Enable client connection to cluster

- On server, run `sudo cat /etc/kubernetes/admin.conf` and copy output
- Run remaining commands on dev client machine
- `mkdir -p ~/.kube`
- `vi ~/.kube/config`
- Paste the `admin.config` you copied from the server, if the file is new and empty.
- If the file is not empty (you already have other clusters), paste the following parts (carefully check for correct indentation after pasting, and consider moving `name:` to beginning of array):
    - Paste the `clusters:` entry with `name: kubernetes` entry into the `clusters:` array.
    - Paste the `contexts:` entry with `name: kubernetes-admin@kubernetes` entry into the `contexts:` array.
    - Paste the `users:` entry with `name: kubernetes-admin` entry into the `users:` array.
- Verify: `kubectl config get-contexts 'kubernetes-admin@kubernetes'`
```
...
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```
  - **_IMPORTANT NOTE!!! - If the CLI tool versions such as kubectl are incompatible with the server, you can get obscure errors such as "No route to host".
    This can happen even if the minor versions are close or perhaps even the same. If you have problems accessing the cluster via CLI tools, ensure you have
    downloaded and installed the latest (or matching) versions of the CLI tools, AND are actually using that version and not a different one on your path._**
- See current context: `kubectl config current-context`
- If it is different, switch with `kubectx` and pick context `kubernetes-admin@kubernetes`

# (supplementary/optional) Other notes/changes from debugging kubectl connection

These are some notes from debugging the `kubectl` connection from my Mac to the server (which turned out to be using a
version of kubectl that was incompatible with the server). They weren't related to the root cause, but keeping them here
for future reference.

## IPV6 error

- in `sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml` add this to kube-apiserver command:
```
- --bind-address=192.168.1.200
```


## crictl, maybe unrelated?

Get rid of warnings...

```
$ sudo crictl inspect $(sudo crictl ps | grep kube-apiserver | awk '{print $1}') | grep -A 10 Args
WARN[0000] Config "/etc/crictl.yaml" does not exist, trying next: "/usr/bin/crictl.yaml"
WARN[0000] runtime connect using default endpoints: [unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
WARN[0000] Image connect using default endpoints: [unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
WARN[0000] Config "/etc/crictl.yaml" does not exist, trying next: "/usr/bin/crictl.yaml"
WARN[0000] runtime connect using default endpoints: [unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
```

Claude suggestion:
```
$ sudo tee /etc/crictl.yaml << 'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 2
debug: false
EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 2
debug: false
```

## It works as root!

- `kubectl get nodes` works as root
- Using certs directly from curl also works from non-root:
```
curl --cert /Users/cwoolley/.kube/k3s-test-certs/client.crt --key /Users/cwoolley/.kube/k3s-test-certs/client.key -k -v -XGET  -H "User-Agent: kubectl/v1.32.0 (darwin/arm64) kubernetes/70d3cc9" -H "Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json" 'https://192.168.1.200:6443/api/v1/nodes?limit=500'
```
