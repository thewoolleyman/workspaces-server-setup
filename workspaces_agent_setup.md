# Workspaces agent setup

# Table of contents

[TOC]

# Overview

This section contains instructions on setting up a Workspaces agent on the cluster

# Requirements and assumptions

- You need a working GitLab installation with an enterprise license supporting the Workspaces feature
- You need dequate roles/permissions on the GitLab installation
- For ease of documentation, we will assume everything is created under a top-level group with a dedicated structure.
  You can change this if you want.

# Create agent

- Create a new top-level group `workspaces-dogfooding`. 
- All projects should be created in this group.
- Create a project `workspaces-agents`.
- In project, go to: Operate -> Kuberenetes clusters
- Connect a cluster
- Under "Option 2: Create and register an agent...", type `workspaces-agent`
- After the agent is created, select `Create token`
- Create a token with name `workspaces agent token`
- Copy the `Agent access token` value and copy it somewhere safe  
- Copy the helm command and run it on the server (note: if you have a real domain, replace it as the `kasAddress`):
```
helm repo add gitlab https://charts.gitlab.io
helm repo update
helm upgrade --install workspaces-agent gitlab/gitlab-agent \
    --namespace gitlab-agent-workspaces-agent \
    --create-namespace \
    --set image.tag=v17.8.0 \
    --set config.token=YOUR_TOKEN_VALUE \
    --set config.kasAddress=wss://127.0.0.1/-/kubernetes-agent/
```
- NOTE: My server is not connected to the internet, so it needs some special configs:
```
helm upgrade --install workspaces-agent gitlab/gitlab-agent \
    --namespace gitlab-agent-workspaces-agent \
    --create-namespace \
    --set image.tag=v17.8.0 \
    --set config.token=YOUR_TOKEN_VALUE \
    --set config.kasAddress=wss://192.168.1.200/-/kubernetes-agent/ \
    --set "extraArgs={--kas-tls-server-name=gitlab.mindlikewater.net,--kas-insecure-skip-tls-verify}"
```

- Verify - see logs for the agent: `kubectl logs -f -l="app.kubernetes.io/instance=workspaces-agent" -n gitlab-agent-workspaces-agent`
