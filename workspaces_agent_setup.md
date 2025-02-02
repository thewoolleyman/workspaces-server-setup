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
- NOTE: If your gitlab server is not connected to the internet, the helm command needs some special config in `extraArgs`
  (NOTE: not fully tested, I subsequently connected my server to the internet):
```
helm upgrade --install workspaces-agent gitlab/gitlab-agent \
    --namespace gitlab-agent-workspaces-agent \
    --create-namespace \
    --set image.tag=v17.8.0 \
    --set config.token=YOUR_TOKEN_VALUE \
    --set config.kasAddress=wss://192.168.1.200/-/kubernetes-agent/ \
    --set "extraArgs={--kas-tls-server-name=gitlab.example.com,--kas-insecure-skip-tls-verify}"
```

- Verify logs for the agent: `kubectl logs -f -l="app.kubernetes.io/instance=workspaces-agent" -n gitlab-agent-workspaces-agent`

# Setup GitLab Workspaces proxy

- See https://docs.gitlab.com/17.8/ee/user/workspace/set_up_workspaces_proxy.html

## Install an ingress controller in k3s

- We previously had to disable the default k3s traefik ingress controller, so we need to set up a
  an nginx ingress controller
- Add the nginx ingress helm repository: `helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx`
- Update helm repos: `helm repo update`
- Install nginx ingress controller:
```
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.ports.https=31443 \
  --set controller.service.externalTrafficPolicy=Local \
  --set controller.service.enableHttp=false
```
- Verify ingress controller pods are running: `kubectl get pods -n ingress-nginx`
- Get pod IP with `kubectl get svc -n ingress-nginx`:
```
NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
ingress-nginx-controller             LoadBalancer   10.43.221.2    <pending>     31443:30606/TCP   8s
ingress-nginx-controller-admission   ClusterIP      10.43.190.97   <none>        443/TCP           8s
```
- `curl -k https://localhost:30443` (should be a 404)
- If you need to uninstall: `helm uninstall ingress-nginx --namespace ingress-nginx`

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

### Set up nginx as a hostname-based reverse proxy

- Install nginx: `sudo apt install -y nginx libnginx-mod-stream`
- Verify stream module: `nginx -V 2>&1 | grep with-stream`
- Create config file: `sudo vi /etc/nginx/nginx.conf`
- Add this entry to the top level of the config:
```
# TLS SNI routing - forward traffic based on hostname without decryption, to either gitlab, or gitlab workspaces proxy
# Get the proper IP for the gitlab workspaces proxy.
# Note that `hostnames` is required for the wildcard domain to work!!!
stream {
  map $ssl_preread_server_name $backend {
    hostnames;
    gitlab.example.com                    127.0.0.1:11443;
    workspaces.gitlab.example.com         127.0.0.1:31443;
    *.workspaces.gitlab.example.com       127.0.0.1:31443;
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
- Tail logs via journalctl: `journalctl -u nginx.service -f`
- Tail logs directly: `tail -f /var/log/nginx/*.log`

## Generate TLS certificates

- You need TLS (SSL) certs for the two domains above.
- Follow instructions in https://docs.gitlab.com/17.8/ee/user/workspace/set_up_workspaces_proxy.html#generate-tls-certificates,
  or else obtain your own certs from a cert provider.
- If you are obtaining your own certs: 
  - Here's an example of generating a CSR (first step of requesting a cert):
```
sudo openssl req -new -newkey rsa:2048 -nodes -keyout wildcard_workspaces_gitlab_example_com.key -out wildcard_workspaces_gitlab_example_com.csr -config san.conf
```
  - Here's an example `san.conf` used as input to the above command:
```
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C=US
ST=California
L=San Francisco
O=example.com
OU=Ministry of Silly Walks
CN=*.workspaces.gitlab.example.com
emailAddress=your-email@example.com

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = *.workspaces.gitlab.example.com
```

## Configure GitLab as an OAuth 2.0 identity provider

- See https://docs.gitlab.com/17.8/ee/integration/oauth_provider.html#create-an-instance-wide-application
- On gitlab instance, Admin -> Applications -> New Application
- Set name to `GitLab Workspaces instance app`  
- Set the redirect URI to `https://workspaces.gitlab.example.com/auth/callback` (this is the same as `GITLAB_WORKSPACES_PROXY_DOMAIN` below)
- Select the **Trusted** checkbox.
- Set the scopes to `api`, `read_user`, `openid`, and `profile`.
- Create the app
- Save the `Application ID` and `Secret` somewhere safe, like 1Password.

## Create an SSH host key

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
export SSH_PORT="4222"
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
  --from-literal="ssh.port=${SSH_PORT}" \
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
- Add `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` to `.bashrc`, and source it.
- `helm repo add gitlab-workspaces-proxy https://gitlab.com/api/v4/projects/gitlab-org%2fworkspaces%2fgitlab-workspaces-proxy/packages/helm/devel`
- `helm repo update`
- Run the helm upgrade command. Note we have to run ssh daemon at a nonstandard port `4222` otherwise it will
  override the real ssh service on the host server, and we won't be able to log in via ssh anymore
  (notice this by ssh host keys and auth failing and `sudo tail -f /var/log/auth.log` showing no activity)
```
helm upgrade --install gitlab-workspaces-proxy \
  gitlab-workspaces-proxy/gitlab-workspaces-proxy \
  --version=0.1.16 \
  --namespace=gitlab-workspaces \
  --create-namespace \
  --set="sshService.port=${SSH_PORT}" \
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
- Verify `kubectl get svc -n gitlab-workspaces`:
```
NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
gitlab-workspaces-proxy       ClusterIP      10.43.214.49   <none>          80/TCP           2m54s
gitlab-workspaces-proxy-ssh   LoadBalancer   10.43.78.230   192.168.1.200   4222:32068/TCP   2m54s
```
- Verify `kubectl -n gitlab-workspaces get ingress`:
```
NAMESPACE           NAME                      CLASS   HOSTS                                                                       ADDRESS         PORTS     AGE
gitlab-workspaces   gitlab-workspaces-proxy   nginx   workspaces.gitlab.example.com,*.workspaces.gitlab.example.com   192.168.1.200   80, 443   68s
```
- Verify `kubectl -n gitlab-workspaces get pods`:
```
NAME                                       READY   STATUS    RESTARTS   AGE
gitlab-workspaces-proxy-6899f4bbbb-9lgjs   1/1     Running   0          109s
```
- Verify `kubectl get pods -n gitlab-workspaces`
```
NAME                                       READY   STATUS    RESTARTS      AGE
gitlab-workspaces-proxy-6899f4bbbb-5rxbq   1/1     Running   0             111s
```
- If you need to uninstall: `helm uninstall gitlab-workspaces-proxy -n gitlab-workspaces`

# Map agent to group

- Go to `workspaces-dogfooding` group
- Settings -> Workspaces
- All agents -> Allow

# Create a project for workspace

- Go to `workspaces-dogfooding` group
- Create a project `example-devfiles`.
- Create a file `devfile-example-sshd-http-app.yaml` with this content:
```
schemaVersion: 2.2.0
components:
  - name: tooling-container
    attributes:
      gl/inject-editor: true
    container:
      image: registry.gitlab.com/gitlab-org/workspaces/examples/example-sshd-http-app:latest
      endpoints:
        - name: http-8000
          targetPort: 8000
```

# Create a workspace and verify you can connect to it

## Create workspace

- See https://docs.gitlab.com/17.8/ee/user/workspace/configuration.html#create-a-workspace
- Go to https://gitlab.example.com/-/remote_development/workspaces/
- In left nav, click "Search or go to -> Your work -> Workspaces"   
- Create a workspace for the `workspaces-dogfooding/example-devfiles` project
- The `workspaces-agent` agent should be available and auto-selected
- Enter devfile location as `devfile-example-sshd-http-app.yaml`
- Click `Create workspace` and wait for it to be running
  
## Verify you can connect to workspace via HTTPS

- Click `Open workspace`
- The automatic oauth should happen properly (using `workspaces.example.com`) and the workspace
  should come up with a valid SSL cert (using `*.workspaces.example.com`).

## Verify you can connect to workspace via SSH

- See: https://docs.gitlab.com/17.8/ee/user/workspace/configuration.html#connect-to-a-workspace-with-ssh
- At https://gitlab.example.com/-/user_settings/personal_access_tokens, create a personal access token
  with scope `read_api`. This is your workspaces SSH password. Save it in 1Password.   
- Get the name of the workspace from the workspaces list accessed above (or the URL of the workspace)
- The workspaces SSH host is `<workspace_name>@<ssh_proxy_IP_address>`
- From the server, run `kubectl -n gitlab-workspaces get service gitlab-workspaces-proxy-ssh`
```
NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
gitlab-workspaces-proxy-ssh   LoadBalancer   10.43.78.230   192.168.1.200   4222:32068/TCP   50m
```  
- Get the external IP (e.g. `192.168.1.200`) and `sshService` port. The port is the one to the left of the colon,
  which you used for `SSH_PORT` above, e.g. `4222`  
- From a client terminal, run an SSH command: `ssh workspace-1-1-w7mqhh@192.168.1.200 -p 4222`
- Enter your access token as the password
- See that you are successfully connected:
```
gitlab-workspaces@workspace-1-1-w7mqhh-87c456b9b-k654x:~$ hostname
workspace-1-1-w7mqhh-87c456b9b-k654x
```
- If you have [opened port `4222`](./server_hardware_and_network_info.md#internet-port-access) externally, you can
  connect to the externally bound hostname, e.g.: `ssh workspace-1-1-w7mqhh@workspaces.gitlab.example.com -p 4222`

# Make agent usable from gitlab.com

TODO: Still not working, SSL certs still not working properly

## Create separate certs for usage on gitlab.com

Since the gitlab-workspaces-proxy does oauth with the gitlab instance, and gitlab.com is a separate instance
than gitlab.example.com, we need a separate instance of the proxy running on a separate domain with separate
certs.

Follow the instructions above in [Generate TLS certificates](#generate-tls-certificates) to generate certs for
`*.gitlab-com-workspaces.gitlab.com`.

IMPORTANT NOTE: For this case, we are NOT going to have a separate cert for the OAuth
(e.g. `workspaces.gitlab.example.com`), but instead we will just use the same wildcard cert and domain for both
the OAuth and the workspace proxy domains.

## Configure GitLab as an OAuth 2.0 identity provider on gitlab.com

- These are similar instructions as above in [Configure GitLab as an OAuth 2.0 identity provider](#configure-gitlab-as-an-oauth-20-identity-provider),
  but we will make it a user-level application, not an instance-wide application.
- On gitlab.com, User Settings -> Applications -> Add New Application
- Set name to `GitLab Workspaces instance app for gitlab.com`
- Set the redirect URI to `https://gitlab-workspaces-proxy.gitlab-com-workspaces.gitlab.example.com/auth/callback` (this is the same as `GITLAB_WORKSPACES_PROXY_DOMAIN` below)
- Set the scopes to `api`, `read_user`, `openid`, and `profile`.
- Create the app
- Save the `Application ID` and `Secret` somewhere safe, like 1Password.

## Reuse the same SSH host key

- Since the SSH host key is not tied to the domain, we can reuse the same key for the gitlab.com proxy that we created
  above in [Create an SSH host key](#create-an-ssh-host-key).

## Set up the proxy on kubernetes for gitlab.com

The following should all be done in one terminal, as the root user, so the exported environment vars are persisted.

You will need to be root user, via `sudo su -`

### Make a separate script to set envioronment vars for gitlab.com

- `cp ~/export_proxy_env_vars.sh ~/export_proxy_env_vars_gitlab_com.sh` 
- Edit the file and change the all the necessary entries. `SSH_HOST_KEY` is the only one to leave as-is:
```shell
export GITLAB_WORKSPACES_PROXY_DOMAIN="gitlab-workspaces-proxy.gitlab-com-workspaces.gitlab.example.com"
export GITLAB_WORKSPACES_WILDCARD_DOMAIN="*.gitlab-com-workspaces.gitlab.example.com"
export WORKSPACES_DOMAIN_CERT="/etc/gitlab/ssl/wildcard_gitlab-com-workspaces_gitlab_example_com/combined_wildcard_gitlab-com-workspaces_gitlab_example_com.crt"
export WORKSPACES_DOMAIN_KEY="/etc/gitlab/ssl/wildcard_gitlab-com-workspaces_gitlab_example_com/wildcard_gitlab-com-workspaces_gitlab_example_com.key"
export WILDCARD_DOMAIN_CERT="/etc/gitlab/ssl/wildcard_gitlab-com-workspaces_gitlab_example_com/combined_wildcard_gitlab-com-workspaces_gitlab_example_com.crt"
export WILDCARD_DOMAIN_KEY="/etc/gitlab/ssl/wildcard_gitlab-com-workspaces_gitlab_example_com/wildcard_gitlab-com-workspaces_gitlab_example_com.key"
export GITLAB_URL="https://gitlab.com"
export CLIENT_ID="your_application_id"
export CLIENT_SECRET="your_application_secret"
export REDIRECT_URI="https://gitlab-workspaces-proxy.gitlab-com-workspaces.gitlab.example.com/auth/callback"
export SIGNING_KEY="whatever-signing-key-you-want-to-use-for-workspaces-proxy"
export SSH_HOST_KEY="/root/.ssh/workspaces-gitlab-example-com-ssh-host-key"
export SSH_PORT="5222"
```
- `source ./export_proxy_env_vars.sh`

### Create kubernetes secrets for gitlab.com

#### Create namespace for gitlab.com

- Create namespace: `kubectl create namespace gitlab-com-gitlab-workspaces`

#### Create config secret for gitlab.com

- Create secret: 
```
kubectl create secret generic gitlab-com-gitlab-workspaces-proxy-config \
  --namespace="gitlab-com-gitlab-workspaces" \
  --from-literal="ssh.port=${SSH_PORT}" \
  --from-literal="auth.client_id=${CLIENT_ID}" \
  --from-literal="auth.client_secret=${CLIENT_SECRET}" \
  --from-literal="auth.host=${GITLAB_URL}" \
  --from-literal="auth.redirect_uri=${REDIRECT_URI}" \
  --from-literal="auth.signing_key=${SIGNING_KEY}" \
  --from-literal="ssh.host_key=$(cat ${SSH_HOST_KEY})"
```

#### Create tls secret for gitlab.com

- Create secret:
```
kubectl create secret tls gitlab-com-gitlab-workspace-proxy-tls \
  --namespace="gitlab-com-gitlab-workspaces" \
  --cert="${WORKSPACES_DOMAIN_CERT}" \
  --key="${WORKSPACES_DOMAIN_KEY}"
```

#### Create wildcard tls secret for gitlab.com

- Create secret:
```
kubectl create secret tls gitlab-com-gitlab-workspace-proxy-wildcard-tls \
  --namespace="gitlab-com-gitlab-workspaces" \
  --cert="${WILDCARD_DOMAIN_CERT}" \
  --key="${WILDCARD_DOMAIN_KEY}"
```

### Install the helm chart for gitlab.com

- Ensure you are the root user and have sourced `./export_proxy_env_vars_gitlab_com.sh`
- Run the helm upgrade command:
```
helm upgrade --install gitlab-com-gitlab-workspaces-proxy \
  gitlab-workspaces-proxy/gitlab-workspaces-proxy \
  --version=0.1.16 \
  --namespace=gitlab-com-gitlab-workspaces \
  --create-namespace \
  --set="sshService.port=${SSH_PORT}" \
  --set="ingress.enabled=true" \
  --set="ingress.hosts[0].host=${GITLAB_WORKSPACES_PROXY_DOMAIN}" \
  --set="ingress.hosts[0].paths[0].path=/" \
  --set="ingress.hosts[0].paths[0].pathType=ImplementationSpecific" \
  --set="ingress.hosts[1].host=${GITLAB_WORKSPACES_WILDCARD_DOMAIN}" \
  --set="ingress.hosts[1].paths[0].path=/" \
  --set="ingress.hosts[1].paths[0].pathType=ImplementationSpecific" \
  --set="ingress.tls[0].hosts[0]=${GITLAB_WORKSPACES_PROXY_DOMAIN}" \
  --set="ingress.tls[0].secretName=gitlab-com-gitlab-workspace-proxy-tls" \
  --set="ingress.tls[1].hosts[0]=${GITLAB_WORKSPACES_WILDCARD_DOMAIN}" \
  --set="ingress.tls[1].secretName=gitlab-com-gitlab-workspace-proxy-wildcard-tls" \
  --set="configSecretName=gitlab-com-gitlab-workspaces-proxy-config" \
  --set="ingress.className=nginx"
```
- Verify `kubectl get svc -n gitlab-com-gitlab-workspaces`:
```
NAME                                     TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
gitlab-com-gitlab-workspaces-proxy       ClusterIP      10.43.122.22   <none>          80/TCP           40s
gitlab-com-gitlab-workspaces-proxy-ssh   LoadBalancer   10.43.81.144   192.168.1.200   5222:30755/TCP   40s
```
- Verify `kubectl -n gitlab-com-gitlab-workspaces get ingress`:
```
NAME                                 CLASS   HOSTS                                                                                                                     ADDRESS         PORTS     AGE
gitlab-com-gitlab-workspaces-proxy   nginx   gitlab-workspaces-proxy.gitlab-com-workspaces.gitlab.example.com,*.gitlab-com-workspaces.gitlab.example.com   192.168.1.200   80, 443   58s
```
- Verify `kubectl get pods -n gitlab-com-gitlab-workspaces`
```
NAME                                                 READY   STATUS    RESTARTS   AGE
gitlab-com-gitlab-workspaces-proxy-9d5ff8c44-b4fpv   1/1     Running   0          108s
```

## Set up required DNS records

- Add DNS records for workspaces proxy domain and wildcard domain:
  - `gitlab-workspaces-proxy.workspaces.gitlab.example.com`
    - Type: CNAME Record
    - Host: `gitlab-com-workspaces.gitlab`
    - Value: `gitlab.example.com`
  - `*.workspaces.gitlab.example.com`
    - Type: CNAME Record
    - Host: `*.gitlab-com-workspaces.gitlab`
    - Value: `gitlab.example.com`

## Set up nginx as a hostname-based reverse proxy for gitlab.com

- Add the following hostnames to the `stream` -> `map` section of the nginx config:
```
    gitlab-workspaces-proxy.gitlab-com-workspaces.gitlab.example.com         127.0.0.1:31443;
    *.gitlab-com-workspaces.gitlab.example.com                               127.0.0.1:31443;
```
- At this point, if you try this config you will get this error:
```
$ sudo nginx -t
2025/02/01 22:47:34 [emerg] 106317#106317: could not build map_hash, you should increase map_hash_bucket_size: 64
nginx: configuration file /etc/nginx/nginx.conf test failed
```
- You need to add this inside the `stream` directive as the first line:
```
  map_hash_bucket_size 128;
```
- Test the config: `sudo nginx -t`
- Reload nginx: `sudo systemctl reload nginx`


## Create agent for gitlab.com

- Go to https://gitlab.com/gitlab-org/workspaces/testing/gitlab-agent-configurations
- Create an agent using the same steps above in [create agent](#create-agent), but with these changes:
  - Create the agent config file in
    https://gitlab.com/gitlab-org/workspaces/testing/gitlab-agent-configurations/-/tree/main/.gitlab/agents?ref_type=heads
  - Name the config file `z-please-do-not-use-cwoolley-dogfooding-server/config.yaml`  
    - Note 1: that the `z-please-do-not-use-` prefix is a hack to make it show up at the bottom of the list,
      because the current agent authorization scheme does not support restricting access to a mapped agent
      for users who inherit `Developer` access, if I ever map this agent from the parent group (`gitlab-org` in this case),
      so that it is usable for all my projects under `gitlab-org` group.
  - In the agent config file:
    - Substitute the `dns_zone` with `gitlab-com-workspaces.gitlab.com`
  - In the agent helm chart:
    - Substitute the `namespace` with `gitlab-com-workspaces-agent`
    - Substitute the `config.token` with the new token you created for the new agent
    - Substitute the `config.kasAddress` with `wss://kas.gitlab.com/-/kubernetes-agent/`
- Verify it is running: `kubectl logs -f -l="app.kubernetes.io/instance=workspaces-agent" -n gitlab-com-workspaces-agent`

## Map agent on gitlab.com

- Go to group: https://gitlab.com/gitlab-org/workspaces
- Settings -> Workspaces -> All agents
- Allow the `z-please-do-not-use-cwoolley-dogfooding-server` agent

## Test agent on gitlab.com

Since I don't have authorization to the `gitlab-org` group to map the agent, I need to create the workspace from a group
or a subgroup where the agent is mapped, which is at the `gitlab-org/workspaces` group level.

### Test with example-jetbrains-gateway project

- Go to https://gitlab.com/gitlab-org/workspaces/examples/example-jetbrains-gateway
- Pick `Edit -> New Workspace`
- In the workspace creation dialog, select the `z-please-do-not-use-cwoolley-dogfooding-server` agent
- Create the workspace

### Test with gitlab project

- Create a new private group `cwoolley-testing` under `gitlab-org/workspaces/testing`
- Fork https://gitlab.com/gitlab-org/workspaces/examples/example-jetbrains-gateway project under
  https://gitlab.com/gitlab-org/workspaces/testing/cwoolley-testing  
- In the new forked project at https://gitlab.com/gitlab-org/workspaces/testing/cwoolley-testing/example-jetbrains-gateway,
  pick `Edit -> New Workspace`
- In the workspace creation dialog, select the `z-please-do-not-use-cwoolley-dogfooding-server` agent
- Create the workspace
