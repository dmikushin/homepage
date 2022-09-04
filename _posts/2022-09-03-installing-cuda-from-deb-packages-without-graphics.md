---
layout: post
title: "Installing CUDA from DEB packages without graphics"
tags:
- CUDA
thumbnail_path: blog/2022-09-03-installing-cuda-from-deb-packages-without-graphics/nvidia-graphics.png
---

You may want to have your NVIDIA GPU not to be involved in any desktop rendering for many reasons. While this is the default on the headless servers, personal systems such as desktops and laptops need some further configuration. Traditional downloadable CUDA binary packages had a `--no-opengl-files` command line option, which simply skips the installation of any graphics-related components. The other installation method based on Debian packages is more favorable for system compatibility, but unfortunately lacks any option for graphics disablement. This tutorial shows how to disable desktop graphics after installing the Debian packages.

## Run the standard CUDA installation

(You may want to check for an up-to-date instruction at the [CUDA downloads page](https://developer.nvidia.com/cuda-downloads).)

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.0-1_all.deb
sudo dpkg -i cuda-keyring_1.0-1_all.deb
sudo apt-get update
sudo apt-get -y install cuda
```

Also install a package management tool called `equivs`, which we will need on a later step:

```bash
sudo apt install equivs
```

## Uninstall graphics packages, without uninstalling what depends on them

```bash
sudo dpkg -r --force-depends libnvidia-gl-515 libnvidia-gl-515:i386
sudo dpkg -r --force-depends xserver-xorg-video-nvidia-515
```

Unfortunately, such uninstallation would leave APT in a broken state:

```
You might want to run 'apt --fix-broken install' to correct these.
The following packages have unmet dependencies:
 cuda-drivers-515 : Depends: libnvidia-gl-515 (>= 515.65.01) but it is not going to be installed
                    Depends: xserver-xorg-video-nvidia-515 (>= 515.65.01) but it is not going to be installed
 nvidia-driver-515 : Depends: libnvidia-gl-515 (= 515.65.01-0ubuntu1) but it is not going to be installed
                     Depends: xserver-xorg-video-nvidia-515 (= 515.65.01-0ubuntu1) but it is not going to be installed
                     Recommends: libnvidia-gl-515:i386 (= 515.65.01-0ubuntu1)
E: Unmet dependencies. Try 'apt --fix-broken install' with no packages (or specify a solution).
```

We will fix this error in the next step.

## Create dummy packages that shall replace the uninstalled official packages

Create dummy packages configurations templates using `equivs` to unbreak the APT state and restore the consistency of dependencies:

```bash
equivs-control libnvidia-gl-515
equivs-control libnvidia-gl-515
```

Customize the templates using a text editor. Specifically, we need to set the packages names and versions. In this case, the version for both packages must be precisely `515.65.01-0ubuntu1`.

Finally, create the packages and install them on the system: 

```bash
equivs-build xserver-xorg-video-nvidia-515
equivs-build xserver-xorg-video-nvidia-515
sudo dpkg -i libnvidia-gl-515_515.65.01-0ubuntu1_all.deb
sudo dpkg -i xserver-xorg-video-nvidia-515_515.65.01-0ubuntu1_all.deb
```

Now APT is in a correct state, and desktop graphics rendering is disabled.

This method is based on [this great tutorial](https://www.brain-dump.org/blog/creating-dummy-packages-to-fulfil-dependencies-in-debian/).

