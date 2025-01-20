# (optional) Eclipse Che setup

# Table of contents

[TOC]

# Overview

This doc contains instructions on setting up and using [Eclipse Che](https://eclipse.dev/che/) on the kubernetes cluster.

Che is the primary open-source example of using the [devfile standard](https://devfile.io/) in a Kubernetes environment,
and thus is a good comparison/contrast for the GitLab workspaces offering which also uses the devfile standard.

See [this issue](https://gitlab.com/gitlab-org/gitlab/-/issues/512722) for more context and background on the motivation
for comparison.

# Install chectl CLI on MacOs

- See https://eclipse.dev/che/docs/stable/administration-guide/installing-the-chectl-management-tool/#installing-the-chectl-management-tool-on-linux-or-macos
- `bash <(curl -sL  https://che-incubator.github.io/chectl/install.sh)`
- Verify: `chectl version`
```
chectl/7.97.0 darwin-arm64 node-v18.18.0
```

# Install Che

- See https://eclipse.dev/che/docs/stable/administration-guide/installing-che-locally/
  - **NOTE: these docs don't actually document the `--platform=k8s` option, but that's what we're doing**
- `chectl server:deploy --platform=k8s`
  - Say `y` or `n` to usage data collection
- 
