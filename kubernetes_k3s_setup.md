# Kubernetes k3s setup

## NOTE

- This doc contains the version of instructions for a "k3s" kubernetes installation.
- It is an alternative approach to the [kubernetes full setup](./kubernetes_full_setup.md), which is documented separately.

# k3s initial tools installation and configuration

- See https://github.com/k3s-io/k3s
- These docs are for k3s v1.32.0+k3s1 (Kubernetes v1.32.0)
- Unless otherwise noted, all commands are run directly on server

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
- See current context: `kubectl config current-context`
- If it is different, switch with `kubectx` and pick context `k3s`
