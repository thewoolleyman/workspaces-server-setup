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
