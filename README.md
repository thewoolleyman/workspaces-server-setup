# workspaces-server-setup

# Table of contents

[TOC]

# What is this project?

[Chad's](https://gitlab.com/cwoolley-gitlab) notes and supporting files for setting up [GitLab Remote Development Workspaces](https://docs.gitlab.com/ee/user/workspace/) from scratch.

For now, it only contains notes on setting up my home bare-metal server.

# Overview

The approach I am taking is simple and manual. Automation is not a priority for now. For now, the goal is to ensure I document every step in detail for future reference (and potential automation).

Some of this is very specific to my particular server, home network setup, and ISP, as well as the versions of OS/software used.

## Skill assumptions

- "Basic" linux/unix/networking knowledge - "Basic" being whatever knowledge I already had
- Can use `vi`/`vim`
- YAML knowledge

# Server hardware info

- https://www.theserverstore.com/Dell-PowerEdge-R630-36-CORE-Virtualization-Server-192GB-3x-800GB-SSD-H730p_p_988.html
- https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://i.dell.com/sites/doccontent/shared-content/data-sheets/en/documents/dell-poweredge-r630-spec-sheet.pdf&ved=2ahUKEwiOmZmr_bCEAxW3FjQIHexrAh8QFnoECCYQAQ&usg=AOvVaw0GT_z0LYLwK9Rt-9iKSgBz
- https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://i.dell.com/sites/doccontent/shared-content/data-sheets/en/Documents/Dell-PowerEdge-R630-Technical-Guide-v1-6.pdf&ved=2ahUKEwiOmZmr_bCEAxW3FjQIHexrAh8QFnoECCgQAQ&usg=AOvVaw0UcLDArCKOOhUeVQuWJ_oZ
- https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://www.dell.com/support/manuals/en-us/poweredge-r630-dsms/r630_om_pub/dell-poweredge-r630-system-overview%3Fguid%3Dguid-a743b271-3784-4599-8c90-0fbca035aeb0%26lang%3Den-us&ved=2ahUKEwiO2JzZ_rCEAxXVHjQIHdIyBgwQFnoECA4QAQ&usg=AOvVaw14gsicB1V6nroCDHgMTogF
- https://www.dell.com/support/manuals/en-us/poweredge-r630-dsms/r630_om_pub/dell-poweredge-r630-system-overview?guid=guid-a743b271-3784-4599-8c90-0fbca035aeb0&lang=en-us

# Network info

## Basic network and machine IP info

- Home routers and subnets:
    - External modem/router subnet: `10.0.0.0` (XFinity model CGM4981COM modem/router - admin webpage http://10.0.0.1/)
    - Internal router subnet (all home machines): `192.168.1.0` (TP-Link Archer - admin webpage http://192.168.1.1/webpages/login.html)
    - All admin passwords in 1password
- Main development machine `192.168.1.111`
- `poweredge` server: `192.168.1.200`, connected to `eno1`, MAC `14:18:77:57:75:36`
- Router DHCP configured to assign static IPs based on MAC addresses

## Dynamic DNS Raspberry PI setup

- Raspberry PI Zero 2 W
- DNS provider namecheap.com
    - https://www.namecheap.com/support/knowledgebase/article.aspx/595/11/how-do-i-enable-dynamic-dns-for-a-domain/
    - https://www.namecheap.com/support/knowledgebase/article.aspx/5/11/are-there-any-alternate-dynamic-dns-clients/
- TODO: finish setup, test, and document    

## Add hosts entry to main development machine

- Add `192.168.1.200 poweredge` to `/etc/hosts`

# Install Ubuntu Desktop LTS

## Why Ubuntu desktop instead of server?

Because:

- I'm not an expert in Linux (even though I've installed and used it since it came out, including for a couple of years as my main development OS), and sometimes things are easier to do or debug with a GUI.
- Other people who may try to follow these instructions on their own may benefit from having access to a GUI
- Ubuntu desktop is (I assume) more likely to come with drivers/etc for my hardware, so I don't have to waste time futzing with those myself (been there, done that, lost countless hours of my life to it)
- I can use other useful GUI tools if desired, e.g. Rancher Desktop.

If this were a production server, that would be a different story.

## Download Ubuntu, burn to installation media, and start install

- https://ubuntu.com/download/desktop
- Follow instructions to burn to DVD or USB drive (using balenaEtcher - MacOS Sequoia and Windows 11 don't seem to even recognize the Ubuntu ISO image). I used a WD 120GB USB portable drive.
- Plug in USB to server
- F11 to get boot menu in BIOS
- Boot to USB

## Ubuntu initial install

Basic steps like language, keyboard layout, etc are omitted. All defaults are assumed, only changes from defaults are documented.

- Installer asked to auto-update, I did it, then re-launched from desktop
- Type of installation: Interactive
- Applications: Extended selection
- Optimise your computer: Check "Install third-party software..."
- Disk setup: Manual installation (I had issues with grub customization and boot-from-lan when choosing automatic disk setup on a previous install)
    - Delete any existing partitions on existing drive (`sda`)
    - Pick `sda` for bootloader installation (creates ~1GB FAT32 partition)
    - Create 1000GB ext4 partition with mount point of `/` (leaving ~900 GB free for future partitions)
- Create account
    - Username: `cwoolley` (same as my main dev laptop, so I don't have to specify username for SSH, just host. If your username is different you can configure it in your ssh config)
    - Computer name: `poweredge`
    - So, wherever it says `ssh to server` in instructions, I'm typing `ssh poweredge`
- Wait for install to finish, then restart, and wait for reboot and welcome screen
- Take all defaults and finish
- From "Show Apps", run `Software Updater` (NOT `Software & Updates`) and do all updates

## Dual boot setup (optional)

Since I ran into issues with [kubernetes full setup](./kubernetes_full_setup.md), I set up a dual boot with a separate OS install to test k3s seetup. This section gives instructions on how I set up the dual boot

- Boot server
- Hold down F2 until you get into System Setup (BIOS)
- System BIOS -> Boot Settings -> UEFI Boot Settings -> Enter on UEFI Boot Sequence -> Ensure "Disk connected to back USB" is first (before Integrated RAID Controller). This will allow you to boot from USB to install a second OS.
- Back -> Finish -> Save Changes -> Finish -> Exit and Reboot
- From Ubuntu setup, follow same instructions as above, but "How do you want to install Ubuntu", do not pick "Install alongside Ubuntu", pick "Manual installation"
-  in Manual partitioning:
    - resize the existing `sda2` partition smaller, from 1TB to 200G
    - Create a second `sda3` partition for the second OS, with 500G, `ext4` and mount point `/`. ONLY check `format` for this partition.
- Continue with same install instructions as above  
- After install reboots, the default should be the new OS.
- To boot to old OS, hold down F11 to get boot menu, and pick the sda2 drive to boot into the initial (full-kubernetes) OS
- TODO: add instructions on how to change default OS. EFI? grub?

# Basic OS and application setup

## No-password sudo

- `sudo visudo`
- Change `%sudo ...` line to `%sudo ALL=(ALL) NOPASSWD: ALL`

## Package installation

- `sudo apt update`
- `sudo apt install -y net-tools` - for `ifconfig` and others
- `sudo apt install -y curl` - for downloading stuff

## sshd server setup

- `sudo apt update`
- `sudo apt install -y openssh-server`
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

# Install basic kubernetes tools on MacOS client machine

## Install kubernetes tools

Run all these commands on the main development machine, i.e. the client normally used to administer the cluster.

- `brew install kubernetes-cli` - kubernetes cli tools
  - Verify with `kubectl version`
```
Client Version: v1.32.0
Kustomize Version: v5.5.0
```
  - **_IMPORTANT NOTE!!! - If the CLI tool versions such as kubectl are incompatible with the server, you can get obscure errors such as "No route to host".
    This can happen even if the minor versions are close or perhaps even the same. If you have problems accessing the cluster via CLI tools, ensure you have
    downloaded and installed the latest (or matching) versions of the CLI tools, AND are actually using that version and not a different one on your path._**
- `brew install k9s`:  text-based kubernetes management interface
  - Verify with `k9s` (`ctrl-C` to quit)
- `brew install kubectx`: gives `kubens` command line to change kubernetes namespaces
  - Verify with `kubens --help`

The remaining client setup will be done after the kubernetes cluster setup is completed, in the corresponding section
"Set up MacOS development machine to administer cluster"

# Kubernetes setup

There's two separate versions:

- [kubernetes k3s setup](./kubernetes_k3s_setup.md) (simpler, the approach I am currently using)
- [kubernetes full setup](./kubernetes_full_setup.md) (more complex, incomplete, abandoned due to issues)
