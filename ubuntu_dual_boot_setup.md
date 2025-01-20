# Ubuntu dual boot setup

# Table of contents

[TOC]

## Overview

Setting up Ubuntu dual boot on a GeekOM mini PC, which already has Windows 11 installed.

# PC specs

```
Mini IT11 11th Gen Intel Core i7-i5 - i7-11390H 32GB RAM+2TB SSD
SIZE:
i7-11390H 32GB RAM+2TB SSD
```

Use `del` to enter BIOS (from https://service.geekompc.com/faq/update-bios-ec/)

# Dual boot setup

- Following https://linuxconfig.org/how-to-install-ubuntu-alongside-windows-11-dual-boot
- Using Ubuntu 20.04 on USB drive
- Enter BIOS with `del`, set USB to boot first
- Install Ubuntu alongside Windows using instructions from article above (choose "Install Ubuntu alongside Windows Boot Manager",
  and it will give you options to resize the Windows partition)
- Re-enter BIOS, and set first boot order to `ubuntu ...`

# Install Ubuntu

- Follow general instructions from [Basic server OS and application setup](./basic_server_os_and_application_setup.md) to get Ubuntu set up
