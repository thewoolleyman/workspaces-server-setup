# (optional) Eclipse Che setup

# Table of contents

[TOC]

## Overview

This section contains instructions on setting up and using [Eclipse Che](https://eclipse.dev/che/).

Che is the primary open-source example of using the [devfile standard](https://devfile.io/) in a Kubernetes environment,
and thus is a good comparison/contrast for the GitLab workspaces offering which also uses the devfile standard.

## Install chectl CLI on MacOs

- See https://eclipse.dev/che/docs/stable/administration-guide/installing-the-chectl-management-tool/#installing-the-chectl-management-tool-on-linux-or-macos
- `bash <(curl -sL  https://che-incubator.github.io/chectl/install.sh)`
- Verify: `chectl version`
```
chectl/7.97.0 darwin-arm64 node-v18.18.0
```

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

## Install chectl CLI on Ubuntu
