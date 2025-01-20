# Kubernetes k3s setup

# Table of contents

[TOC]

# Overview

- This doc contains the version of instructions for a "k3s" kubernetes installation.
- It is an alternative approach to the [kubernetes full setup](./kubernetes_full_setup.md), which is documented separately.

# k3s initial tools installation and configuration

- See https://github.com/k3s-io/k3s
- These docs are for k3s v1.32.0+k3s1 (Kubernetes v1.32.0)
- Unless otherwise noted, all commands are run directly on server

## Disable traefik in k3s config file prior to install

NOTE: I assume this works, I forgot to do it on my initial install, I've only done it via a re-install as documented below.

- See https://docs.k3s.io/installation/configuration#configuration-file
- `sudo mkdir -p /etc/rancher/k3s`
- `sudo vi /etc/rancher/k3s/config.yaml`, add the following lines:
```
disable:
  - traefik
```

## Download and install k3s

- See https://docs.k3s.io/quick-start
- `curl -sfL https://get.k3s.io | sh -`
- Output:
```
[INFO]  Finding release for channel stable
[INFO]  Using v1.31.4+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.31.4+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.31.4+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Skipping installation of SELinux RPM
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
```

## Enable server non-root user to access cluster

- See https://docs.k3s.io/cluster-access
- Run the following as non-root user (`cwoolley`)
- `sudo chmod a+r /etc/rancher/k3s/k3s.yaml` (not concerned about opening up read access, it's a local server)
- `vi ~/.bashrc` and add the following lines:
```
# Kubernetes / k3s

export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```
- `source ~/.bashrc`
- Verify: `kubectl get nodes`
```
NAME        STATUS   ROLES                  AGE   VERSION
poweredge   Ready    control-plane,master   13h   v1.31.4+k3s1
```

## (optional) Reinstalling/reconfiguring k3s

You can reinstall k3s to change configuration options.

You can do this by reinstalling with the correct options. This works for any configuration changes (AFAIK?).

### Changing port

Here's an example where I changed the server port from default 6443 to 7443 (note: not fully tested, I reverted back to 6443).

- See https://docs.k3s.io/installation/configuration#configuration-file
- `sudo vi /etc/rancher/k3s/config.yaml`, add the following line:
```
https-listen-port: 7443
```
- re-run install: `curl -sfL https://get.k3s.io | sh -`
- restart the service: `sudo systemctl restart k3s`
- Verify (in new terminal): `watch sudo systemctl status k3s`
- Update any `kubectl` configs that may exist (e.g. on MacOS client) to point to new port

### Changing default to not install traefik ingress

Here's an example where I changed the default ingress to not use traefik because I forgot to do it on initial install.

- See:
  - https://docs.k3s.io/installation/configuration#configuration-file
  - https://docs.k3s.io/installation/packaged-components
  - https://qdnqn.com/k3s-remove-traefik/
- Run `sudo file /var/lib/rancher/k3s/server/manifests/traefik.yaml` to verify it exists.
- `kubectl get pods --namespace kube-system | grep traefik`
```
helm-install-traefik-crd-jjsf2            0/1     Completed   0               3m15s
helm-install-traefik-x6fbr                0/1     Completed   1               3m15s
svclb-traefik-9f2713f2-lhb5w              2/2     Running     0               3m12s
traefik-57b79cf995-pgxhd                  1/1     Running     0               3m12s
```  
- `sudo vi /etc/rancher/k3s/config.yaml`, add the following lines:
```
disable:
  - traefik
```
- re-run install: `curl -sfL https://get.k3s.io | sh -`
- restart the service: `sudo systemctl restart k3s`
- Verify: `sudo file /var/lib/rancher/k3s/server/manifests/traefik.yaml` to verify it no longer exists.
  (NOTE: it might take a few seconds after the service restart to actually delete the file!)
- Verify: `kubectl get pods --namespace kube-system | grep traefik` is empty

## Run an image to verify setup

- `kubectl run nginx --image=nginx`
- Verify with `kubectl get pods`
- Wait for `STATUS` to be `Running`
- Cleanup: `kubectl delete pod nginx`

# Set up MacOS development machine to administer cluster

## Enable client connection to cluster

- On server, run `sudo cat /etc/rancher/k3s/k3s.yaml` and copy output
- Run remaining commands on dev client machine
- `mkdir -p ~/.kube`
- `vi ~/.kube/config`
- Paste the `admin.config` you copied from the server, if the file is new and empty.
- If the file is not empty (you already have other clusters), paste the following parts (carefully check for correct indentation after pasting, and consider moving `name:` to beginning of array):
    - Paste the `clusters:` entry with `name: default` entry into the `clusters:` array. Change name from `default` to `k3s`
    - Paste the `contexts:` entry with `name: default` entry into the `contexts:` array. Change `cluster` and `name` to `k3s`, and `user` to `k3s-admin`
    - Paste the `users:` entry with `name: default` entry into the `users:` array. Change `name` to `k3s-admin`
- Verify: `kubectl config get-contexts 'k3s'`
```
...
CURRENT   NAME   CLUSTER   AUTHINFO    NAMESPACE
          k3s    k3s       k3s-admin
```
  - **_IMPORTANT NOTE!!! - If the CLI tool versions such as kubectl are incompatible with the server, you can get obscure errors such as "No route to host".
    This can happen even if the minor versions are close or perhaps even the same. If you have problems accessing the cluster via CLI tools, ensure you have
    downloaded and installed the latest (or matching) versions of the CLI tools, AND are actually using that version and not a different one on your path._**
- See current context: `kubectl config current-context`
- If it is different, switch with `kubectx` and pick context `k3s`
