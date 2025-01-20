# Ubuntu OS installation

# Table of contents

[TOC]

# Overview

This doc contains instructions starting from my clean bare-metal server, up to the point that Ubuntu is installed
and booting successfully.

# Install Ubuntu Desktop LTS

## Why Ubuntu desktop instead of server?

Because:

- I'm not an expert in Linux (even though I've installed and used it since it came out, including for a couple of years
  as my main development OS), and sometimes things are easier to do or debug with a GUI.
- Other people who may try to follow these instructions on their own may benefit from having access to a GUI
- Ubuntu desktop is (I assume) more likely to come with drivers/etc for my hardware, so I don't have to waste time futzing
  with those myself (been there, done that, lost countless hours of my life to it)
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
