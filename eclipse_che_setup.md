# (optional) Eclipse Che setup

# Table of contents

[TOC]

# Overview

This section contains instructions on setting up and using [Eclipse Che](https://eclipse.dev/che/).

Che is the primary open-source example of using the [devfile standard](https://devfile.io/) in a Kubernetes environment,
and thus is a good comparison/contrast for the GitLab workspaces offering which also uses the devfile standard.

# Install chectl CLI on MacOs

- See https://eclipse.dev/che/docs/stable/administration-guide/installing-the-chectl-management-tool/#installing-the-chectl-management-tool-on-linux-or-macos
- `bash <(curl -sL  https://che-incubator.github.io/chectl/install.sh)`
- Verify: `chectl version`
```
chectl/7.97.0 darwin-arm64 node-v18.18.0
```

# Install Che on local MacOS

## Install Minikube

- See https://eclipse.dev/che/docs/stable/administration-guide/installing-che-on-minikube/
- Follow https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download to install and start
```
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube
minikube start --v=10
```
- Add to kubectl context: `minikube update-context`
- Verify: `kubectx`
- Verify: `kubectl get nodes`
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   58s   v1.32.0
```
- Set up metrics server: `minikube addons enable metrics-server`
- view `minikube dashboard`

## Reinstall minikube with configuration required for Che

- See: https://eclipse.dev/che/docs/stable/administration-guide/installing-che-on-minikube/
- `minikube delete`
- `minikube start --addons=ingress,dashboard --vm=true --memory=10240 --cpus=4 --disk-size=50GB --kubernetes-version=v1.23.9`
- Add to kubectl context: `minikube update-context`
- For some reason this wanted to use the `parallels` driver instead of `docker` as above? I let it...

## Install Che on minikube

- `chectl server:deploy --platform minikube`
- Verify: `chectl server:status`
```
Eclipse Che Version    : 7.97.0
Eclipse Che Url        : https://10.211.55.6.nip.io/dashboard/
```

## View Che Dashboard

- `chectl dashboard:open`
- Opens browser window, click "Advanced" to ignore cert/security/etc errors and get to login page
- Logins:
    - admin: `che@eclipse.org`/`admin`
    - user1: `user1@che`/`password` (same for `user2` through `user5`)

## Try to start a workspace

PROBLEM: Any workspace gives the following error:

```
Error creating DevWorkspace deployment: Init Container che-code-injector has state CrashLoopBackOff
```

This was difficult to debug, because Che deletes the pod immediately after it fails, so it was hard to get logs.

I was eventually able to get the logs this way:

```
kubectl logs -n user1-che $(kubectl get events -n user1-che --sort-by=.lastTimestamp | grep 'Created pod:' | tail -1 | cut -d: -f2 | tr -d ' ') -c che-code-injector --previous
```

This showed the following error:

```
exec /entrypoint-init-container.sh: exec format error
```

So, it was an AMD64 vs ARM64 issue. Rather than try to fix this, I decided to just reinstall Che on a native linux
machine instead.

# Install Che on native linux machine

## Install cubectx (kubens)

- `sudo apt install kubectx`

## Install Minikube

- See https://eclipse.dev/che/docs/stable/administration-guide/installing-che-on-minikube/
- Follow https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fdebian+package
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

## Install docker driver for minikube

NOTE: I end up undoing this because Che needs the VirtualBox driver, and VirtualBox requires KVM to be disabled,
and Docker relies on KVM.

- See https://minikube.sigs.k8s.io/docs/drivers/docker/
- Install docker via preferred method

## Start minikube with docker driver

- `minikube start --driver=docker`

## Set up alias for minikube kubectl

NOTE: I ended up undoing this, because the `chectl` Che install didn't like an alias for `kubectl`, it wanted a native
executable.

- `vi ~/.bash_aliases`
- Add `alias kubectl="minikube kubectl --"`

## Finish minikube setup

- Add to kubectl context: `minikube update-context`
- Verify: `minikube kubectl get nodes`
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   58s   v1.32.0
```
- Set up metrics server: `minikube addons enable metrics-server`
- view `minikube dashboard` from browser (using native UI or NoMachine)

## Install virtualbox

Unfortunately, Che doesn't work with Docker driver. So, we install the virtualbox driver

- https://minikube.sigs.k8s.io/docs/drivers/virtualbox/
- Install virtualbox via preferred method
- Verify virtualbox starts and runs

## Disable KVM (it conflicts with VirtualBox)

- Disable KVM modules
```
echo "blacklist kvm_intel" | sudo tee -a /etc/modprobe.d/blacklist-kvm.conf
echo "blacklist kvm" | sudo tee -a /etc/modprobe.d/blacklist-kvm.conf
```
- Reboot
- Verify: `lsmod | grep kvm` (should not see any modules)

## Install a native kubectl on linux

- The approach to alias `kubectl` to `minikube kubectl` is not working for the Che install, so we need to install a native kubectl
- See https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
- `vi ~/.bash_aliases` and comment out the `alias kubectl="minikube kubectl --"` line  
- `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"`
- `sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl`
- Add bash alias `k` for `kubectl` for convenience

## Reinstall minikube with configuration required for Che

- See: https://eclipse.dev/che/docs/stable/administration-guide/installing-che-on-minikube/
- `minikube delete`
- `minikube start --addons=ingress,dashboard --vm=true --memory=10240 --cpus=4 --disk-size=50GB --kubernetes-version=v1.23.9`
  - This should automatically select the virtualbox driver
- Verify minikube still running: `kubectl get nodes`

# Install chectl CLI on Linux

- See https://eclipse.dev/che/docs/stable/administration-guide/installing-the-chectl-management-tool/#installing-the-chectl-management-tool-on-linux-or-macos
- `bash <(curl -sL  https://che-incubator.github.io/chectl/install.sh)`
- Verify: `chectl version`
```
chectl/7.97.0 linux-x64 node-v18.18.0
```

## Install Che on minikube

- `chectl server:deploy --platform minikube`
- Verify: `chectl server:status`
```
Eclipse Che Version    : 7.97.0
Eclipse Che Url        : https://192.168.59.100.nip.io/dashboard/
```

## View Che Dashboard

- From the server GUI: `chectl dashboard:open`
- Opens browser window, click "Advanced" to ignore cert/security/etc errors and get to login page
- Logins:
  - admin: `che@eclipse.org`/`admin`
  - user1: `user1@che`/`password` (same for `user2` through `user5`)
- Verify: Start a workspace on the Che dashboard with "Empty" workspace and default editor (VS Code)

## Set up port forwarding to access from another client on local network

NOTE: Not positive if this will always work...

- `geekom` is the linux host, `192.168.59.100` is the `nip.io` address of Che server, `dex` prefix is used by Che login
- Note that port `443` must be available locally for port forwarding. You can use a different local port, but
  it will always get removed from redirect responses, so you have to manually add it and resubmit the requests.
- From other client machine set up port forwarding.
```
ssh -L 443:192.168.59.100:443 geekom
```
- Add to `/etc/hosts` on client:
```
127.0.0.1	localhost gitlab.local 192.168.59.100.nip.io dex.192.168.59.100.nip.io
```
- From local client visit the standard Che address: https://192.168.59.100.nip.io/

# Notes on Che

# Che commands quick reference

- Check server status: `chectl server:status`
- Set up port forward from local client: `sudo ssh -L 443:192.168.59.100:443 cwoolley@geekom`

## Inspecting Che's init container

- pull it
```
docker pull quay.io/che-incubator/che-code:7.97.0
```
- Check out the entrypoint script
```
docker create --name che-code quay.io/che-incubator/che-code:7.97.0
docker cp che-code:/entrypoint-init-container.sh .
docker rm che-code
cat entrypoint-init-container.sh
```

Here's the output:

```
#!/bin/sh
#
# Copyright (c) 2021 Red Hat, Inc.
# This program and the accompanying materials are made
# available under the terms of the Eclipse Public License 2.0
# which is available at https://www.eclipse.org/legal/epl-2.0/
#
# SPDX-License-Identifier: EPL-2.0
#
# Contributors:
#   Red Hat, Inc. - initial API and implementation
#
# Copy checode stuff to the shared volume
cp -r /checode-* /checode/
# Copy machine-exec as well
mkdir -p /checode/bin
cp /bin/machine-exec /checode/bin/
# Copy entrypoint
cp /entrypoint-volume.sh /checode/
# Copy remote configuration
mkdir -p /checode/remote
cp -r /remote /checode
echo "listing all files copied"
ls -la /checode
```

Sample output:

```
listing all files copied
total 28
drwxrwxrwx 6 root root 4096 Jan 20 10:59 .
drwxr-xr-x 1 root root 4096 Jan 20 10:59 ..
drwxr-xr-x 2 1234 root 4096 Jan 20 10:59 bin
drwxr-xr-x 4 1234 root 4096 Jan 20 10:59 checode-linux-libc
drwxr-xr-x 8 1234 root 4096 Jan 20 10:59 checode-linux-musl
-rwxr-xr-x 1 1234 root 3547 Jan 20 10:59 entrypoint-volume.sh
drwxr-xr-x 3 1234 root 4096 Jan 20 10:59 remote
```

## Starting a sample repo with a devfile pointing to a container with a daemon entrypoint

- Use https://gitlab.com/gitlab-org/workspaces/examples/example-sshd-http-app/
- Start it in Che, by pointing to the repo (have to kill any existing workspaces, default is only 1)
- `kubens user1-che`
- `k get po -o yaml`

```
      lifecycle:
        postStart:
          exec:
            command:
            - /bin/sh
            - -c
            - |
              {
              nohup /checode/entrypoint-volume.sh > /checode/entrypoint-logs.txt 2>&1 &
              } 1>/tmp/poststart-stdout.txt 2>/tmp/poststart-stderr.txt
```

## Get information on a running Che workspace

### Get general info on pod

- Ensure a workspace is running and has been recently (re)started
- `kubens user1-che` - set namespace to workspace user's namespace
- `k get pod` - get pods
- `export PODNAME=$(k get po -o name | cut -d/ -f2)` - store podname in env var for easy use in other commands
  (assumes that there's just one pod)
- `k get pod $PODNAME -o yaml` - print entire pod as YAML output

### Get lifecycle hook info on the workspace

- `k explain pod.spec.containers.lifecycle` - example of explaining part of the schema
- `k get pod $PODNAME -o jsonpath='{.spec.containers[0].lifecycle}' | jq` - Print out lifecycle hooks of first container:
- Default postStart entry for a Che workspace, using a devfile with no `postStart` hooks:
```
{
  "postStart": {
    "exec": {
      "command": [
        "/bin/sh",
        "-c",
        "{\nnohup /checode/entrypoint-volume.sh > /checode/entrypoint-logs.txt 2>&1 &\n} 1>/tmp/poststart-stdout.txt 2>/tmp/poststart-stderr.txt\n"
      ]
    }
  }
}
```

### Get info on a workspaces that is failing to start - example 1

Scenario: 

- Using a workspace with a daemon as the entrypoint (https://gitlab.com/gitlab-org/workspaces/examples/example-sshd-http-app)
- Che UI hung on "Waiting for workspace to start"

Debugging:

- `k get po` - verify that all main and init containers are running
- `k get events --sort-by='.lastTimestamp'` - get events, verify no errors
- `k get po -o yaml` - See all info for pod
- `kubectl get pod -o yaml | grep-A 1  DEVFILE` - see locations of devfiles in mounted volume
- `kubectl exec $PODNAME -c tooling-container -- cat /devworkspace-metadata/original.devworkspace.yaml` - print the original devfile
- Note the parts Che added to the original devfile in the repo. Che's mounted version:
```
attributes:
  controller.devfile.io/devworkspace-config:
    name: devworkspace-config
    namespace: eclipse-che
  controller.devfile.io/storage-type: per-user
  dw.metadata.annotations:
    che.eclipse.org/devfile-source: |
      scm:
        repo: https://gitlab.com/gitlab-org/workspaces/examples/example-sshd-http-app.git
        fileName: .devfile.yaml
      factory:
        params: >-
          url=https://gitlab.com/gitlab-org/workspaces/examples/example-sshd-http-app.git
components:
- attributes:
    gl/inject-editor: true
  container:
    endpoints:
    - exposure: public
      name: http-8000
      protocol: http
      targetPort: 8000
    image: registry.gitlab.com/gitlab-org/workspaces/examples/example-sshd-http-app:latest
    sourceMapping: /projects
  name: tooling-container
projects:
- git:
    remotes:
      origin: https://gitlab.com/gitlab-org/workspaces/examples/example-sshd-http-app.git
  name: example-sshd-http-app
```
- original devfile version from repo:
```
schemaVersion: 2.2.0
components:
  - name: tooling-container
    attributes:
      gl/inject-editor: true
    container:
      # NOTE: THIS IMAGE EXISTS ONLY FOR DEMO PURPOSES AND WILL NOT BE MAINTAINED
      image: registry.gitlab.com/gitlab-org/workspaces/examples/example-sshd-http-app:latest
      endpoints:
        - name: http-8000
          targetPort: 8000
```
- `kubectl exec $PODNAME -c tooling-container -- cat /checode/entrypoint-logs.txt` - cat entrypoint logs
- Determined error - container was missing some required libraries:
```
...
[INFO] Using linux-libc ubi8-based assembly...
[INFO] Node.js dir for running VS Code: /checode/checode-linux-libc/ubi8
/checode/checode-linux-libc/ubi8/node: error while loading shared libraries: libbrotlidec.so.1: cannot open shared object file: No such file or directory
```
