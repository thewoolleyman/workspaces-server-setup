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
- Create an agent config file at `.gitlab/agents/workspaces-agent/config.yaml`:
```
remote_development:
  dns_zone: "workspaces.gitlab.example.com"
  network_policy:
    enabled: true
    egress:
      - allow: "0.0.0.0/0"
        except:
          - "10.0.0.0/8"
          - "172.16.0.0/12"
          - "192.168.0.0/16"

```  
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
    --set "extraArgs={--kas-tls-server-name=gitlab.example.com,--kas-insecure-skip-tls-verify}"
```

- Verify - see logs for the agent: `kubectl logs -f -l="app.kubernetes.io/instance=workspaces-agent" -n gitlab-agent-workspaces-agent`

# Setup GitLab Workspaces proxy

- See https://docs.gitlab.com/17.8/ee/user/workspace/set_up_workspaces_proxy.html

## Set up required DNS records

- Add DNS records for workspaces proxy domain and wildcard domain:
  - `workspaces.gitlab.example.com`
    - Type: CNAME Record
    - Host: `workspaces.gitlab`
    - Value: `gitlab.example.com`
  - `*.workspaces.gitlab.example.com`
    - Type: CNAME Record
    - Host: `*.workspaces.gitlab`
    - Value: `gitlab.example.com`

## Allow both gitlab and workspaces proxy ingress to share port 443 on server

We need to change the GitLab default port from `443` to `11443`, and set up  an nginx-based hostname-based proxy to route
incoming traffic to either gitlab or the workspaces proxy based on the hostname.

## Change gitlab port from 443 default to 11443

- Edit configuration file:
  - `sudo vi /etc/gitlab/gitlab.rb`
  - Uncomment and modify the to use non-default port: `nginx['listen_port'] = 11443`
- Reconfigure GitLab: `sudo gitlab-ctl reconfigure`
- Restart GitLab services:`sudo gitlab-ctl restart`
- Verify : `sudo netstat -tlpn | grep 11443`
```
tcp        0      0 0.0.0.0:11443           0.0.0.0:*               LISTEN      255259/nginx: maste
```
- Test it from a client with local port forwarding
  - `sudo ssh -L 443:localhost:11443 username@poweredge` (`poweredge` is local server hostname)
  - Open https://localhost/ (ignore cert error)

### Set up nginx as a hostname-based proxy

- Install nginx: `sudo apt install -y nginx libnginx-mod-stream`
- Verify stream module: `nginx -V 2>&1 | grep with-stream`
- Create config file: `sudo vi /etc/nginx/nginx.conf`
- Add this entry to the top level of the config. NOTE: Later, you will replace 10.42.0.0 with the actual proxy ingress
  IP, but for now we are just getting nginx reverse proxy working for gitlab's port 443:
```
# TLS SNI routing - forward traffic based on hostname without decryption, to either gitlab, or gitlab workspaces proxy
# Get the proper IP for the gitlab workspaces proxy.
stream {
  map $ssl_preread_server_name $backend {
    gitlab.example.com                    127.0.0.1:11443;
    workspaces.gitlab.example.com         10.42.0.0:443;
    *.workspaces.gitlab.example.com       10.42.0.0:443;
  }

  server {
    listen 443;
    proxy_pass $backend;
    ssl_preread on;
  }
}
```
- Test the config: `sudo nginx -t`
- Reload nginx: `sudo systemctl reload nginx`

## Generate TLS certificates

- You need TLS (SSL) certs for the two domains above.
- Follow instructions in https://docs.gitlab.com/17.8/ee/user/workspace/set_up_workspaces_proxy.html#generate-tls-certificates,
  or else obtain your own certs from a cert provider.

## Configure GitLab as an OAuth 2.0 identity provider

- See https://docs.gitlab.com/17.8/ee/integration/oauth_provider.html#create-an-instance-wide-application
- On gitlab instance, Admin -> Applications -> New Application
- Set name to `GitLab Workspaces instance app`  
- Set the redirect URI to `https://workspaces.gitlab.example.com/auth/callback` (this is the same as `GITLAB_WORKSPACES_PROXY_DOMAIN` below)
- Select the **Trusted** checkbox.
- Set the scopes to `api`, `read_user`, `openid`, and `profile`.
- Create the app
- Save the `Application ID` and `Secret` somewhere safe, like 1Password.

### Create an SSH host key

- You should be logged in as root
- cd `~/.ssh` (or other path if you want to save the key somewhere else)  
- `ssh-keygen -f workspaces-gitlab-example-com-ssh-host-key -N '' -t rsa`
- `export SSH_HOST_KEY=$(pwd)/workspaces-gitlab-example-com-ssh-host-key`
- Save the host key to somewhere safe, like 1Password

## Set up the proxy on kubernetes

The following should all be done in one terminal, as the root user, so the exported environment vars are persisted.

You will need to be root user, via `sudo su -`

### Set additional environment variables

- Put this `~/export_proxy_env_vars.sh` for the root user
```shell
export GITLAB_WORKSPACES_PROXY_DOMAIN="workspaces.example.com"
export GITLAB_WORKSPACES_WILDCARD_DOMAIN="*.workspaces.example.com"
export WORKSPACES_DOMAIN_CERT="path_to_workspaces_cert"
export WORKSPACES_DOMAIN_KEY="path_to_workspaces_key"
export WILDCARD_DOMAIN_CERT="path_to_wildcard_cert"
export WILDCARD_DOMAIN_KEY="path_to_wildcard_key"
export GITLAB_URL="https://gitlab.example.com"
export CLIENT_ID="your_application_id"
export CLIENT_SECRET="your_application_secret"
export REDIRECT_URI="https://workspaces.example.com/auth/callback"
export SIGNING_KEY="this-is-my-signing-key-there-are-many-others-like-it-but-this-is-mine"
export SSH_HOST_KEY="/root/.ssh/workspaces-gitlab-example-com-ssh-host-key"
```
- `chmod +x ./export_proxy_env_vars.sh`
- `source ./export_proxy_env_vars.sh`

### Create kubernetes secrets

#### Create namespace

- Create namespace: `kubectl create namespace gitlab-workspaces`

#### Create config secret

- Create secret: 
```
kubectl create secret generic gitlab-workspaces-proxy-config \
  --namespace="gitlab-workspaces" \
  --from-literal="auth.client_id=${CLIENT_ID}" \
  --from-literal="auth.client_secret=${CLIENT_SECRET}" \
  --from-literal="auth.host=${GITLAB_URL}" \
  --from-literal="auth.redirect_uri=${REDIRECT_URI}" \
  --from-literal="auth.signing_key=${SIGNING_KEY}" \
  --from-literal="ssh.host_key=$(cat ${SSH_HOST_KEY})"
```
- Verify: `kubectl get secret gitlab-workspaces-proxy-config -n gitlab-workspaces -o yaml`

#### Create tls secret

- Create secret:
```
kubectl create secret tls gitlab-workspace-proxy-tls \
  --namespace="gitlab-workspaces" \
  --cert="${WORKSPACES_DOMAIN_CERT}" \
  --key="${WORKSPACES_DOMAIN_KEY}"
```
- Verify: `kubectl get secret gitlab-workspace-proxy-tls -n gitlab-workspaces -o yaml`

#### Create wildcard tls secret

- Create secret:
```
kubectl create secret tls gitlab-workspace-proxy-wildcard-tls \
  --namespace="gitlab-workspaces" \
  --cert="${WILDCARD_DOMAIN_CERT}" \
  --key="${WILDCARD_DOMAIN_KEY}"
```
- Verify: `kubectl get secret gitlab-workspace-proxy-wildcard-tls -n gitlab-workspaces -o yaml`

### Install the helm chart

- Ensure you are the root user and have sourced `./export_proxy_env_vars.sh`
- `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` (needed because you are root, not sure why kubectl works)
- `helm repo add gitlab-workspaces-proxy https://gitlab.com/api/v4/projects/gitlab-org%2fworkspaces%2fgitlab-workspaces-proxy/packages/helm/devel`
- `helm repo update`
- Run the helm upgrade command:
```
helm upgrade --install gitlab-workspaces-proxy \
  gitlab-workspaces-proxy/gitlab-workspaces-proxy \
  --version=0.1.16 \
  --namespace="gitlab-workspaces" \
  --set="ingress.enabled=true" \
  --set="ingress.hosts[0].host=${GITLAB_WORKSPACES_PROXY_DOMAIN}" \
  --set="ingress.hosts[0].paths[0].path=/" \
  --set="ingress.hosts[0].paths[0].pathType=ImplementationSpecific" \
  --set="ingress.hosts[1].host=${GITLAB_WORKSPACES_WILDCARD_DOMAIN}" \
  --set="ingress.hosts[1].paths[0].path=/" \
  --set="ingress.hosts[1].paths[0].pathType=ImplementationSpecific" \
  --set="ingress.tls[0].hosts[0]=${GITLAB_WORKSPACES_PROXY_DOMAIN}" \
  --set="ingress.tls[0].secretName=gitlab-workspace-proxy-tls" \
  --set="ingress.tls[1].hosts[0]=${GITLAB_WORKSPACES_WILDCARD_DOMAIN}" \
  --set="ingress.tls[1].secretName=gitlab-workspace-proxy-wildcard-tls" \
  --set="ingress.className=nginx"
```
- Verify `kubectl -n gitlab-workspaces get ingress`:
```
NAME                      CLASS   HOSTS                                                                       ADDRESS   PORTS     AGE
gitlab-workspaces-proxy   nginx   workspaces.gitlab.example.com,*.workspaces.gitlab.example.com             80, 443   58s
```
- Verify `kubectl -n gitlab-workspaces get pods`:
```
NAME                                       READY   STATUS    RESTARTS   AGE
gitlab-workspaces-proxy-6899f4bbbb-9lgjs   1/1     Running   0          109s
```


## Update the nginx reverse proxy to point to the actual gitlab workspaces proxy IP

- Get pod IP with `kubectl get pods -n gitlab-workspaces -o jsonpath='{.items[*].status.podIP}'`
- Edit nginx config file: `sudo vi /etc/nginx/nginx.conf`
- Change the workspaces IPs to the actual pod IP.
- Test the config: `sudo nginx -t`
- Reload nginx: `sudo systemctl reload nginx`

# Map agent to group

- Go to `workspaces-dogfooding` group
- Settings -> Workspaces
- All agents -> Allow
