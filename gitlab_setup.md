# GitLab setup

# Table of contents

[TOC]

# Overview

This section contains instructions on installing GitLab on Ubuntu.

This is to provide a stable, high-uptime (read: not GDK) installation of GitLab on which Workspaces can be run.

# Prerequisites

- You should have some domain to use, and proper authorization to set it up according to
  https://docs.gitlab.com/omnibus/settings/dns. In these docs, we'll use `example.com` as an example.
  
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
- Note this is required for the initial LetsEncrypt certificate to work during installation
- However, if you just want to access the server from your internal network,
  you can add it to the `/etc/hosts` from your local machine.

## Install gitlab-ee package

- `sudo EXTERNAL_URL="https://gitlab.example.com" apt-get install gitlab-ee`
- Pin the version to limit auto-updates: `sudo apt-mark hold gitlab-ee`
- Show what packages are held back: `sudo apt-mark showhold`

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

## Misc notes

- Restart: `sudo gitlab-ctl restart`
- Status: `sudo gitlab-ctl status`
- Console: `sudo gitlab-rails console`
- Postgres console: `sudo gitlab-psql -d gitlabhq-production`
