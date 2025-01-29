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
