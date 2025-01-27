# Basic server OS and application setup

# Table of contents

[TOC]

# Overview

This doc contains instructions starting from the point that Ubuntu is installed and booting successfully,
up to the point that basic server OS and applications are set up to support basic server access and
installation/operation of kubernetes and necessary kubernetes workloads.

## Make vim arrow and delete keys work

TODO: Maybe this works, maybe not? And it doesn't fix (or maybe breaks) delete. Need a vim expert to help...

- `vi ~/.vimrc`
- Add:
```
set nocompatible
set esckeys
```
- Do this for every user that uses vim.

## No-password sudo

- `sudo visudo`
- Change `%sudo ...` line to `%sudo ALL=(ALL) NOPASSWD: ALL`

## Package installation

- `sudo apt update`
- `sudo apt install -y net-tools` - for `ifconfig` and others
- `sudo apt install -y curl` - for downloading stuff

## sshd server setup

- `sudo apt update`
- `sudo apt install -y openssh-server` - for accessing server via ssh
- Verify `sudo systemctl status ssh`

### Add public ssh key

- From main development machine, copy public key: `cat ~/.ssh/id_rsa.pub | pbcopy`
- ssh to server
- `sudo vi ~/.ssh/authorized_keys`
- paste public key and save

## Nomachine NX

Allows access to server GUI from main development machine.

- Download
    - https://www.nomachine.com
    - download free edition
    - Linux DEB amd64
- Double click downloaded .deb, open with App Center, install
- On main development machine, install nomachine client, and access via hostname (TODO: document config)
