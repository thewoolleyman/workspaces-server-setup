# workspaces-server-setup

# Table of contents

[TOC]

# What is this project?

[Chad's](https://gitlab.com/cwoolley-gitlab) notes and supporting files for setting up [GitLab Remote Development Workspaces](https://docs.gitlab.com/ee/user/workspace/) from scratch
on a bare-metal server in my local home network.

It also serves as a dumping ground for other Workspaces-related dogfooding and setup info,
such as [Eclipse Che Setup](./eclipse_che_setup.md).

# Overview

The approach I am taking is straightforward and manual. Automation is not a priority for now. For now, the goal is to 
ensure I document every step in detail for future reference (and potential automation).

Some of this is very specific to my particular server, home network setup, and ISP, as well as the versions of OS/software
used. But most of it should be generally applicable to kubernetes and workspaces setup on any bare-metal server
or Debian-based virtual machine environment which supports installing kubernetes.

## Skill assumptions

- "Basic" linux/unix/networking knowledge - "Basic" being whatever knowledge I already had
- Can use `vi`/`vim`
- YAML knowledge

## Order and dependencies

Even though this has been split up into sub-pages, each sub-page linked from this main page should be done in order,
and all steps on each sub-page should be done in order. This is because some steps depend on previous steps.

# Server hardware and network info

- [Server hardware and network info](./server_hardware_and_network_info.md)

# Dynamic DNS Raspberry Pi setup

- [Dynamic DNS Raspberry Pi setup](./dynamic_dns_raspberry_pi_setup.md)

# Ubuntu OS installation

- [Ubuntu OS installation](./ubuntu_os_installation.md)

# Basic server OS and application setup

- [Basic server OS and application setup](./basic_server_os_and_application_setup.md)

# Kubernetes client initial setup

- [Kubernetes client initial setup](./kubernetes_client_initial_setup.md)

# Kubernetes setup

There's two separate versions:

- [Kubernetes k3s setup](./kubernetes_k3s_setup.md) (simpler, the approach I am currently using)
- [Kubernetes full setup](./kubernetes_full_setup.md) (more complex)

# Helm setup

- [Helm setup](./helm_setup.md)

# GitLab setup

- [GitLab setup](./gitlab_setup.md)

# Workspaces agent setup

- [Workspaces agent setup](./workspaces_agent_setup.md)

# Other various workspaces-related setup documentation

## Eclipe Che setup

Eclipse Che is a primary open-source example of implementing the devfile specification which GitLab workspaces uses,
so it is useful to be familiar with its installation and behavior.

- [Ubuntu dual boot setup](./ubuntu_dual_boot_setup.md)
- [Eclipse Che setup](./eclipse_che_setup.md)
