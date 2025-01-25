# GitLab setup

# Table of contents

[TOC]

# Overview

This section contains instructions on installing GitLab on Ubuntu.

This is to provide a stable, high-uptime (read: not GDK) installation of GitLab on which Workspaces can be run.

# Prerequisites

- You should have some domain to use, and proper authorization to set it up according to
  https://docs.gitlab.com/omnibus/settings/dns. In these docs, we'll use `example.com` as an example.
- You need to manage SSL. You can let gitlab do it automatically via letsencrypt, but I purchased my own SSL cert.
  So these instructions use the manual approach: https://docs.gitlab.com/omnibus/settings/ssl/#configure-https-manually
  
# Ensure DNS resolves to server at standard port

See https://docs.gitlab.com/omnibus/settings/ssl/

Note that `gitlab.example.com` must resolve on standard ports (80/443) to work.

- Ensure your domain has an `A` record for `gitlab.example.com`
- Ensure that both port 80 and 443 resolve to `gitlab.example.com` in your router/firewall config.
- Purchase an SSL cert for the host and configure it. See the
  [configure HTTPS manually](#configure-https-manually) section below for details.

# Install Gitlab

- See https://about.gitlab.com/install/#ubuntu

## Install dependencies and prerequisites

- ssh to the server
- `sudo apt-get update`
- `sudo apt-get install -y curl openssh-server ca-certificates tzdata perl`
- `sudo apt-get install -y postfix`
  - Pick `Local only` option, we don't care about mail.
- Add GitLab package repo: `curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash`

## Set up DNS

- Follow https://docs.gitlab.com/omnibus/settings/dns
- If you want to access the server from your internal network,
  you can add its internal network IP to to the `/etc/hosts` from your local machine with the `gitlab.example.com` entry.

## Install gitlab-ee package

- `sudo EXTERNAL_URL="https://gitlab.example.com" apt-get install gitlab-ee`
- Pin the version to limit auto-updates: `sudo apt-mark hold gitlab-ee`
- Show what packages are held back: `sudo apt-mark showhold`

## Configure HTTPS manually

- See https://docs.gitlab.com/omnibus/settings/ssl/#configure-https-manually
- Edit `/etc/gitlab/gitlab.rb`, set `letsencrypt['enable'] = false`
- Create the /etc/gitlab/ssl directory and copy your key and certificate there:
```
sudo mkdir -p /etc/gitlab/ssl
sudo chmod 755 /etc/gitlab/ssl
sudo cp gitlab.example.com.key gitlab.example.com.crt /etc/gitlab/ssl/
```
- See
  [the "SSL cert" section in the server hardware and network info page](./server_hardware_and_network_info.md)
  for specific details.

## Configure KAS with fix for error

NOTE: this may be a bug that gets fixed...

- The following error came from `sudo tail -f /var/log/gitlab/gitlab-kas/current` (I think, forgot to note it originally):
```
2025-01-22_10:02:05.13902 {"time":"2025-01-22T02:02:05.138953742-08:00","level":"ERROR","msg":"Program aborted","error":"private API server: construct own private API multi URL: failed to parse OWN_PRIVATE_API_URL grpc://localhost:8155: ParseAddr(\"localhost\"): unable to parse IP"}
```
- Here's the fix - edit `/etc/gitlab/gitlab.rb`, change `OWN_PRIVATE_API_URL` to have `127.0.0.1` instead of `localhost`:
```
gitlab_kas['env'] = {
  'OWN_PRIVATE_API_URL' => 'grpc://127.0.0.1:8155'
}
```
- `sudo gitlab-ctl reconfigure`

# Initial configuration

## Initial root login

- Navigate to hostname for server: `https://gitlab.example.com`
- On the server console: `sudo cat /etc/gitlab/initial_root_password`
- Copy the password (make sure you get the trailing `=`)  
- Log in with `root` and that password
- Deactivate new signups from warning alert

## Reset root password

- Avatar -> Edit Profile -> Password
- Set new password, log in with it to test

## Add EE license

- See https://docs.gitlab.com/ee/administration/license_file.html
- Admin (bottom of left sidebar) -> Settings -> General -> Add License
- Enter license key
- Verify in: Admin -> Overview -> Dashboard -> License Overview (should be banner at top with license details)

## Add SSH key

- See https://docs.gitlab.com/17.8/ee/user/ssh.html

## Misc notes

- Reconfigure: `sudo gitlab-ctl reconfigure`
- Restart: `sudo gitlab-ctl restart`
- Status: `sudo gitlab-ctl status`
- Console: `sudo gitlab-rails console`
- Postgres console: `sudo gitlab-psql -d gitlabhq-production`
- View rails production log: `sudo tail -f /var/log/gitlab/gitlab-rails/production.log`
- View kas log: `sudo tail -f /var/log/gitlab/gitlab-kas/current`
- Vieew agent logs: `k logs -f -l="app.kubernetes.io/instance=workspaces-agent" -n gitlab-agent-workspaces-agent`
