# JetBrains Gateway setup

# Table of contents

[TOC]

# Overview

This is a guide to setting up JetBrains Gateway to run on a GitLab Workspaces workspace.

# Build a workspaces container image which supports Jetbrains gateway

## Prerequisites

The minimum container prerequisites for JetBrains Gateway usage are:

- sshd installed and configured for workspaces
- wget or curl installed

## Container build and publish steps

- The project at https://gitlab.com/gitlab-org/workspaces/examples/example-jetbrains-gateway
  contains a Dockerfile which can be used to build a container image
  that meets the minimum prerequisites for JetBrains Gateway usage in a workspace.
- All the container build and publish steps are in
  [that repo's `README.md`](https://gitlab.com/gitlab-org/workspaces/examples/example-jetbrains-gateway/-/blob/master/README.md?ref_type=heads).

# Start a workspace with the Jetbrains Gateway container image

- Copy the `.devfile.yaml` from the example-jetbrains-gateway project to the project you want to use with
  JetBrains Gateway. Here is the devfile:
```
schemaVersion: 2.2.0
components:
  - name: tooling-container
    attributes:
      gl/inject-editor: true
    container:
      # NOTE: THIS IMAGE EXISTS ONLY FOR DEMO PURPOSES AND WILL NOT BE MAINTAINED
      image: registry.gitlab.com/gitlab-org/workspaces/examples/example-jetbrains-gateway:latest
```  
- Start the workspace.
- [Verify you can connect to workspace via SSH](./workspace_ssh_connection.md#verify-you-can-connect-to-workspace-via-ssh).

# Connect with JetBrains Gateway

- Use JetBrains toolbox to download and install JetBrains Gateway
- Run JetBrains Gateway  
- Create an SSH connection, using same credentials as you tested with in the above section:
  - Username is the name of the workspace
  - Password is the workspace token
  - Port is the workspace SSH port (`4222`)
  - Choose to save token permanently, if you want
- Gateway should now download the backend on the workspace, and start a client to connect to it.  
