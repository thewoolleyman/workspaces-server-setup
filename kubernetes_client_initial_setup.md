# Kubernetes client initial Setup

# Table of contents

[TOC]

# Overview

This doc contains instructions for installing basic dependencies of a MacOS client machine to administer a kubernetes cluster.

The remaining client setup will be done after the kubernetes cluster setup is completed, in the corresponding section
"Set up MacOS development machine to administer cluster", under the separate docs for
[kubernetes full setup](./kubernetes_full_setup.md) and [kubernetes k3s setup](./kubernetes_k3s_setup.md).

# Install basic kubernetes tools on MacOS client machine

## Install kubernetes tools

Run all these commands on the main development machine, i.e. the client normally used to administer the cluster.

- `brew install kubernetes-cli` - kubernetes cli tools
    - Verify with `kubectl version`
```
Client Version: v1.32.0
Kustomize Version: v5.5.0
```
- **_IMPORTANT NOTE!!! - If the CLI tool versions such as kubectl are incompatible with the server, you can get obscure errors such as "No route to host".
  This can happen even if the minor versions are close or perhaps even the same. If you have problems accessing the cluster via CLI tools, ensure you have
  downloaded and installed the latest (or matching) versions of the CLI tools, AND are actually using that version and not a different one on your path._**
- `brew install k9s`:  text-based kubernetes management interface
    - Verify with `k9s` (`ctrl-C` to quit)
- `brew install kubectx`: gives `kubens` command line to change kubernetes namespaces
    - Verify with `kubens --help`
