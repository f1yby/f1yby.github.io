---
title: "Installing Fedora Silverblue"
date: 2022-07-28
last_modified_at: 2022-08-03
categories:
  - blog
tags:
  - Silverblue
  - Environment Setup
  - Nvidia
---
Installing Fedora Silverblue develop environment including Nvidia and IDEs.

## Attention

1. [snap](https://snapcraft.io/) is **NOT** available on Silverblue by default, you may refer [this page](https://github.com/coreos/rpm-ostree/issues/1711) for further details.
2. System Files are immutable. You may need to pay **MORE** effort to set up the dev env.
3. Secure Boot is **NOT** officially supported now. You are required to disable it (conflicting with win11 dual boot), or use community solution to install Nvidia Driver.

## Installation

1. Download Image and USB Writer on [Fedora Silverblue Download](https://silverblue.fedoraproject.org/download)
2. Boot on USB and follow the installation guide.
3. Reboot and set network, user name, password. *Be careful at this stage, any error could cause the installation to fail and require a re-installation.*

## Nvidia Driver

Although **SECURE BOOT** can easily be configure on Fedora, the situation on Silverblue is different. Follow [this guide](https://github.com/CheariX/silverblue-akmods-keys), or simply disable the SECURE BOOT.

1. [Configure](https://github.com/CheariX/silverblue-akmods-keys) or disable Secure Boot.
2. [Add RPM Fusion Repository](https://rpmfusion.org/Configuration). Use relative Fedora version repository. e.g. Silverblue Fedora 36 user can use Fedora 36 Repository.
3. [install nvidia driver](https://rpmfusion.org/Howto/NVIDIA#Silverblue)

## Use Nvidia in toolbox

Install kernel module by `sudo dnf install akmod-nvidia` inside toolbox.

If Secure Boot is enabled, I have no idea  whether the kernel module will be signed automatically. Maybe [Secure Boot Setup](https://rpmfusion.org/Howto/Secure%20Boot) is needed inside toolbox.

## Enable Flatpak apps to visit toolbox containers

1. [Set up Flatpak remote repository](https://flatpak.org/setup/Fedora) if not set.
2. Install [FlatSeal](https://flathub.org/apps/details/com.github.tchx84.Flatseal).
3. Start podman.socket `systemctl --user enable --now podman.socket`
4. Grant app access to `podman.socket`
    1. Open `FlatSeal`
    2. Find the app
    3. Add`xdg-run/podman` to `FileSytem/OtherFiles`
5. Use `podman-remote` instead of `podman` inside flatpak apps.

## Set up GUI forwarding in toolbox

In my experience, most apps inside toolbox can correctly communicate with host desktop Graphic interface. But some app can't find the display device.

1. Grant container with X Server access, by [xhost command](https://www.ibm.com/docs/en/aix/7.2?topic=x-xhost-command), or simpliy type `xhost +` which will allow anyone to connect to host's X Server.
2. Set the environment variable `DISPLAY` inside container to `127.0.0.1$HOST_DISPLAY`(`$HOST_DISPLAY` is the value of `DISPLAY` on host). e.g If `DISPLAY` on host is `:0`, execute `export DISPLAY=127.0.0.1:0` inside container.
