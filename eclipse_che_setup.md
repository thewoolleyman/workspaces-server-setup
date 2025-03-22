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

NOTE: I end up undoing this (Docker desktop uninstall) because Che needs the VirtualBox driver, and VirtualBox requires 
KVM to be disabled, and Docker relies on KVM. I ended up installing docker engine (https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

- See https://minikube.sigs.k8s.io/docs/drivers/docker/
- Install docker via preferred method (I used Docker Desktop)

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
  
## Uninstall Docker Desktop and install Docker Engine

- Uninstall Docker Desktop (https://docs.docker.com/desktop/uninstall/):
- `sudo apt remove docker-desktop`
- `rm -r $HOME/.docker/desktop`
- `sudo rm /usr/local/bin/com.docker.cli`
- `sudo apt purge docker-desktop`
  
## Install docker engine

- See https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository
- `sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
- Verify: `sudo docker run hello-world` 

## Docker engine postinstall

- See https://docs.docker.com/engine/install/linux-postinstall/
- `sudo usermod -aG docker $USER`
- log out and back in
- run `groups`, check `docker` is listed
- Test service: `sudo systemctl stop docker` (get error about socket, not sure why, just ignored it)
- `sudo systemctl start docker`
- Verify: `docker run hello-world`
- Reset context (otherwise `minikube` delete/start complains): `docker context use default`

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
- Set up metrics server: `minikube addons enable metrics-server`
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
Eclipse Che Url        : https://192.168.59.101.nip.io/dashboard/
```
- Note the IP that is used, it will be needed below.


## View Che Dashboard

- From the server GUI: `chectl dashboard:open`
- Opens browser window, click "Advanced" to ignore cert/security/etc errors and get to login page
- Logins:
  - admin: `che@eclipse.org`/`admin`
  - user1: `user1@che`/`password` (same for `user2` through `user5`)
- Verify: Start a workspace on the Che dashboard with "Empty" workspace and default editor (VS Code)

## Set up port forwarding to access from another client on local network

NOTE: Not positive if this will always work...

- `geekom` is the linux host, `192.168.59.xxx` is the `nip.io` address of Che server, `dex` prefix is used by Che login
- Replace `xxx` with the last octet of the IP address of the Che server from the output of `chectl server:status`
- Note that port `443` must be available locally for port forwarding. You can use a different local port, but
  it will always get removed from redirect responses, so you have to manually add it and resubmit the requests.
- From other client machine set up port forwarding.
```
ssh -L 443:192.168.59.xxx:443 geekom
```
- Add to `/etc/hosts` on client:
```
127.0.0.1	localhost gitlab.local 192.168.59.xxx.nip.io dex.192.168.59.xxx.nip.io
```
- From local client visit the standard Che address: https://192.168.59.xxx.nip.io/

# Notes on Minikube

## Minikube commands quick reference

- `minikube stop`
- `minikube start --addons=ingress,dashboard --vm=true --memory=10240 --cpus=4 --disk-size=50GB --kubernetes-version=v1.23.9`
- `minikube start --v=10`

# Notes on Che

## Che commands quick reference

- Check server status: `chectl server:status`
- Set up port forward from local client: `sudo ssh -L 443:192.168.59.xxx:443 cwoolley@geekom`

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
- `k get pod $PODNAME -o jsonpath='{.spec.containers[0].lifecycle}' | jq` - Print out lifecycle hooks of first container.
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
- Do the same with YAML...
- `sudo apt install -y yq`
- `k get pod $PODNAME -o yaml | yq .spec.containers[0].lifecycle`
- Print the command with newlines:  
- `k get pod $PODNAME -o jsonpath='{.spec.containers[0].lifecycle.postStart.exec.command[2]}'`:
```{
nohup /checode/entrypoint-volume.sh > /checode/entrypoint-logs.txt 2>&1 &
} 1>/tmp/poststart-stdout.txt 2>/tmp/poststart-stderr.txt
```

### Debug a workspaces that is failing to start

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

### Access http server running from container entrypoint

NOTE: This works even though the VS Code is failing to start due to the container image at
https://gitlab.com/gitlab-org/workspaces/examples/example-sshd-http-app not having proper libraries, as was
debugged above.

- `kubectl port-forward $PODNAME 8000:8000`
- `curl http://localhost:8000` - should return the example HTTP server homepage

### Add an example postStart command to a devfile

- Add a new workspace in Che
- Import from git URL: `https://gitlab.com/gitlab-org/workspaces/examples/example-sshd-http-app.git`
- In advanced options, give path to custom devfile: `.devfile-with-poststart.yaml`, with these contents:
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
commands:
  - id: say-hello-command
    exec:
      commandLine: |-
        echo "Hello!"
      component: tooling-container
events:
  postStart:
    - say-hello-command
```  
- **IMPORTANT NOTE!!!** - Due to an apparent bug in Che, it deletes the `/examples/example-sshd-http-app.git`
  part of the path from the git URL when it appends the custom devfile URL param. You need to manually fix the git URL
  to re-add the missing parts.
- Once container starts, check the kubernetes postStart event for the tooling container:
- `k get pod $PODNAME -o jsonpath='{.spec.containers[0].lifecycle.postStart.exec.command[2]}'`:
```
{
nohup /checode/entrypoint-volume.sh > /checode/entrypoint-logs.txt 2>&1 &
echo "Hello!"
} 1>/tmp/poststart-stdout.txt 2>/tmp/poststart-stderr.txt
```
- Notice that compared to the default example above, it has inserted the devfile postStart `exec`
- Notice the usage of the curly brace shell "command block" or "command group" to redirect all output to the two files
  in `/tmp`.
- See the source of `entrypoint-volume.sh` here: https://github.com/che-incubator/che-code/blob/main/build/scripts/entrypoint-volume.sh

### Testing poststart with official devfile universal-developer-image

- A default "empty" Che workspace uses this devfile:
```
schemaVersion: 2.2.0
metadata:
  name: empty-p0bh
  namespace: user1-che
components:
  - name: universal-developer-image
    container:
      image: quay.io/devfile/universal-developer-image:ubi8-latest
```
- Container comes from here: https://quay.io/repository/devfile/universal-developer-image?tab=tags&tag=ubi8-latest
- URL for that tag leads to here: https://catalog.redhat.com/software/containers/ubi8/5c647760bed8bd28d0e38f9f?image=6722cc0038ca65309bcb0e7b
- We can use this directly by copying it to a repo: https://gitlab.com/gitlab-org/workspaces/testing/example-various-devfiles
- Start a new workspace with this repo URL and devfile (watch the bug that messes up the URL when you paste the devfile path):
  - Git URL: `https://gitlab.com/gitlab-org/workspaces/testing/example-various-devfiles.git`
  - Devfile path: `devfile-che-empty-default.yaml`
- See it start successfully  

### Testing with multiple background and non-background poststart jobs

- We are using [this devfile](https://gitlab.com/gitlab-org/workspaces/testing/example-various-devfiles/-/blob/main/devfile-che-empty-default-with-poststart-events.yaml?ref_type=heads) as an example:
```
schemaVersion: 2.2.0
components:
  - name: universal-developer-image
    container:
      image: quay.io/devfile/universal-developer-image:ubi8-latest

commands:
  - id: sleeping-background-command
    exec:
      commandLine: |-
        sh -c 'while true; do echo "sleeping from postStart at $(date +%Y-%m-%d_%H:%M:%S)" | tee -a /tmp/sleeping-from-postStart.log; sleep 1; done' &
      component: universal-developer-image
  - id: say-hello-command
    exec:
      commandLine: |-
        echo "Hello!"
      component: universal-developer-image

events:
  postStart:
    - sleeping-background-command
    - say-hello-command
```
- Start a new workspace with this repo URL and devfile (watch the bug that messes up the URL when you paste the devfile path):
  - Git URL: `https://gitlab.com/gitlab-org/workspaces/testing/example-various-devfiles.git`
  - Devfile path: `devfile-che-empty-default-with-poststart-events.yaml`
- See it start successfully and open VS Code
- In VS Code terminal, type: `tail -f /tmp/sleeping-from-postStart.log` and see that the background poststart command is running.
- In VS Code terminal, type: `head -20 /tmp/poststart-stdout.txt` and see that the `Hello!` output from non-background
  poststart command is there.
- Also see /tmp/poststart-stderr.txt` is empty.  
- In the Che dashboard, click the three dots menu next to the workspace, and select "Workspace Details". Notice there's no
  output from the postStart commands in the logs.
- In the server terminal, see both postStart commands, but notice they are in _OPPOSITE ORDER_...
- `export PODNAME=$(k get po -o name | cut -d/ -f2)`
- `k get pod $PODNAME -o jsonpath='{.spec.containers[0].lifecycle.postStart.exec.command[2]}'`
```
{
nohup /checode/entrypoint-volume.sh > /checode/entrypoint-logs.txt 2>&1 &
echo "Hello!"
sh -c 'while true; do echo "sleeping from postStart at $(date +%Y-%m-%d_%H:%M:%S)" | tee -a /tmp/sleeping-from-postStart.log; sleep 1; done' &
} 1>/tmp/poststart-stdout.txt 2>/tmp/poststart-stderr.txt
```
- Question: Why were they in opposite order? are the background commands always last?

### Testing ordering of multiple background and non-background jobs

To answer the above question "Why were they in opposite order? are the background commands always last?", I ran the following
devfile, which contains interleaved background and non-background commands: https://gitlab.com/gitlab-org/workspaces/testing/example-various-devfiles/blob/64cc785fdbc8e71ce033686ba570a03445e9a4e8/devfile-che-empty-default-with-multiple-poststart-events.yaml#L21-21

Here's the contents:

```
schemaVersion: 2.2.0
components:
  - name: universal-developer-image
    container:
      image: quay.io/devfile/universal-developer-image:ubi8-latest

commands:
  - id: sleeping1-background-command
    exec:
      commandLine: |-
        sh -c 'while true; do echo "sleeping 1 from postStart at $(date +%Y-%m-%d_%H:%M:%S)" | tee -a /tmp/sleeping-from-postStart.log; sleep 1; done' &
      component: universal-developer-image
  - id: say-hello1-command
    exec:
      commandLine: |-
        echo "Hello 1!"
      component: universal-developer-image
  - id: sleeping2-background-command
    exec:
      commandLine: |-
        sh -c 'while true; do echo "sleeping 2 from postStart at $(date +%Y-%m-%d_%H:%M:%S)" | tee -a /tmp/sleeping-from-postStart.log; sleep 1; done' &
      component: universal-developer-image
  - id: say-hello2-command
    exec:
      commandLine: |-
        echo "Hello 2!"
      component: universal-developer-image

events:
  postStart:
    - sleeping1-background-command
    - say-hello1-command
    - sleeping2-background-command
    - say-hello2-command
```

...and you can see from the built kubernetes postStart command that it _DOES_ put both of the background commands
at the end of the command list, and the two foreground ones at the beginning:

```
$ k get pod $PODNAME -o jsonpath='{.spec.containers[0].lifecycle.postStart.exec.command[2]}'
{
nohup /checode/entrypoint-volume.sh > /checode/entrypoint-logs.txt 2>&1 &
echo "Hello 1!"
echo "Hello 2!"
sh -c 'while true; do echo "sleeping 1 from postStart at $(date +%Y-%m-%d_%H:%M:%S)" | tee -a /tmp/sleeping-from-postStart.log; sleep 1; done' &
sh -c 'while true; do echo "sleeping 2 from postStart at $(date +%Y-%m-%d_%H:%M:%S)" | tee -a /tmp/sleeping-from-postStart.log; sleep 1; done' &
} 1>/tmp/poststart-stdout.txt 2>/tmp/poststart-stderr.txt
```

### Testing alphabetical ordering of multiple background and non-background jobs

There appears to be nothing in the che-server or devworkspace-operator codebases which sorts based on background vs. forground.

So, lets see if it's alphabetical based on the order of commands/events? Try this devfile: https://gitlab.com/gitlab-org/workspaces/testing/example-various-devfiles/-/blob/main/devfile-che-empty-default-with-multiple-poststart-events-ordered.yaml?ref_type=heads

```yaml
schemaVersion: 2.2.0
components:
  - name: universal-developer-image
    container:
      image: quay.io/devfile/universal-developer-image:ubi8-latest

# NOTE: This is exploring how Che and the devworkspace operator order the commands when they are inserted into the
#        kubernetes postStart lifecycle hook. It consists of interleaved foreground and background commands, but
#        with alphabetical ordering of the commandLine and ids, and differing orders of both the commands and
#        events.postStart arrays.

commands:
  - id: c-sleeping2-background-command
    exec:
      commandLine: |-
        sh -c 'while true; do echo "sleeping 2 from postStart at $(date +%Y-%m-%d_%H:%M:%S)" | tee -a /tmp/sleeping-from-postStart.log; sleep 1; done' &
      component: universal-developer-image
  - id: d-say-hello2-command
    exec:
      commandLine: |-
        echo "Hello 2!"
      component: universal-developer-image
  - id: b-say-hello1-command
    exec:
      commandLine: |-
        echo "Hello 1!"
      component: universal-developer-image
  - id: a-sleeping1-background-command
    exec:
      commandLine: |-
        sh -c 'while true; do echo "sleeping 1 from postStart at $(date +%Y-%m-%d_%H:%M:%S)" | tee -a /tmp/sleeping-from-postStart.log; sleep 1; done' &
      component: universal-developer-image

events:
  postStart:
    - d-say-hello2-command
    - c-sleeping2-background-command
    - b-say-hello1-command
    - a-sleeping1-background-command
```

And here's the resulting poststart command:

```
$ kubens admin-che
✔ Active namespace is "admin-che"
$ k get pod $(kubectl get pods -o jsonpath='{.items[*].metadata.name}') -o jsonpath='{.spec.containers[0].lifecycle.postStart.exec.command[2]}'
{
sh -c 'while true; do echo "sleeping 1 from postStart at $(date +%Y-%m-%d_%H:%M:%S)" | tee -a /tmp/sleeping-from-postStart.log; sleep 1; done' &
echo "Hello 1!"
sh -c 'while true; do echo "sleeping 2 from postStart at $(date +%Y-%m-%d_%H:%M:%S)" | tee -a /tmp/sleeping-from-postStart.log; sleep 1; done' &
echo "Hello 2!"
nohup /checode/entrypoint-volume.sh > /checode/entrypoint-logs.txt 2>&1 &
} 1>/tmp/poststart-stdout.txt 2>/tmp/poststart-stderr.txt
```

So, this shows that the commands are simply being executed in alphabetical order based on the id of the command.
