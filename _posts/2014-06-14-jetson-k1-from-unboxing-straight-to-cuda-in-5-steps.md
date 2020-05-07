---
layout: post
title: "Jetson K1: from unboxing straight to CUDA in 5 steps"
tags:
  - Software Engineering
thumbnail_path: blog/2014-06-14-jetson-k1-from-unboxing-straight-to-cuda-in-5-steps/jetson-k1-box.jpg
---

We finally got the most wanted Jetson K1 board in the house! In this post we show how to turn a just unboxed tiny board into fully-functional CUDA development node.

![alt text](\assets\img\blog\2014-06-14-jetson-k1-from-unboxing-straight-to-cuda-in-5-steps/jetson-k1-box.jpg "Logo Title Text 1")

From now on our training center also offers [CUDA developer certification on Jetson K1](https://parallel-computing.pro/index.php/certification).

## 1. Connect to board remotely over SSH

SSH is enabled by default, so simply connect ethernet and power cables, power on and search for the board in your local network, e.g. using `fping`. I use another Ubuntu laptop to login to Jetson K1:
`$ ssh ubuntu@192.168.0.18`
So no display is really needed, unless you really want it. X server does not work out of the box though (binary driver is missing, see step 3). Note the display port is HDMI-only (no D-Sub). In case you'd want to connect keyboard/mouse/usb stick - note there is only one USB port available.

Some earlier Jetson K1 shipments reportedly had Ubuntu 13 onboard, ours (received 12 June) has fresh Ubuntu 14.04 LTS (April 2014, Long-Time Support release):
`$ lsb_release -a`
No LSB modules are available.
Distributor ID: Ubuntu
Description: Ubuntu 14.04 LTS
Release: 14.04
Codename: trusty

## 2. Turn on software repositories

By default you will not be able to install almost any extra packages. In order to solve this, go to sources list and uncomment everything:

```
$ sudo vi /etc/apt/sources.list
$ sudo apt-get update
$ sudo apt-get install mc
$ mc
```

So now we have midnight commander, sweet!
![alt text](\assets\img\blog\2014-06-14-jetson-k1-from-unboxing-straight-to-cuda-in-5-steps/jetson-k1-mc.png "Logo Title Text 1")

## 3. Install NVIDIA GPU compute/graphics driver

First thing to note in mc is ~/NVIDIA-INSTALLER

```
$ sudo ./installer.sh
Extracting the BSP...
/home/ubuntu/NVIDIA-INSTALLER
Installing NVIDIA binaries
Using rootfs directory of: /
Extracting the NVIDIA user space components to /
Extracting the NVIDIA gst test applications to /
Extracting the configuration files for the supplied root filesystem to /
Creating a symbolic link nvgstplayer pointing to nvgstplayer-0.10
Adding symlink libcuda.so --> libcuda.so.1.1
Extracting the firmwares and kernel modules to /
Installing the board *.dtb files into //boot
Success!
/home/ubuntu/NVIDIA-INSTALLER
Removing installation files...
Configuring the desktop, please reboot to take effect
SUCCESS!
```

## 4. Install CUDA

Download and install CUDA for L4T (Linux for Tegra) from here: [](https://developer.nvidia.com/rdp/assets/cuda-l4t-r192) (requires Registered Developer login/password):

```
$ sudo dpkg -i ./cuda-repo-l4t-r19.2_6.0-42_armhf.deb
$ sudo apt-get update
\$ sudo apt-get install cuda-toolkit-6.0
```

CUDA 6.0 toolkit will be installed to `/usr/local/cuda-6.0`, but still not connected to the command line. So, we need to:

- Create file `/etc/profile.d/cuda.sh` (as admin) with: export `PATH=\$PATH:/usr/local/cuda/bin`
- Create file `/etc/ld.so.conf.d/cuda.conf`
- Reboot

Now nvcc is in command line!

```
\$ nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2013 NVIDIA Corporation
Built on Sat_Mar_15_02:05:29_PDT_2014
Cuda compilation tools, release 6.0, V6.0.1
```

## 5. Compile and run deviceQuery sample

```
$ cd /usr/local/cuda
$ sudo chmod o+w samples/ -R
$ cd samples/1_Utilities/deviceQuery
$ make
$ ../../bin/armv7l/linux/release/gnueabihf/deviceQuery
```

**Here's how deviceQuery output compares to laptop graphics and Tesla card:**

|                                             | **GK20A**              | **GeForce GTX 680M**   | **Tesla K20c**         |
| ------------------------------------------- | ---------------------- | ---------------------- | ---------------------- |
| CUDA Capability Major/Minor version number  | 3.2                    | 3.0                    | 3.5                    |
| **Total amount of global memory**           | **1746 Mbytes**        | **2048 Mbytes**        | **4800 Mbytes**        |
| **Multiprocessors**                         | **1**                  | **7**                  | **13**                 |
| CUDA Cores/MP                               | 192                    | 192                    | 192                    |
| **GPU Clock rate**                          | **852 MHz (0.85 GHz)** | **758 MHz (0.76 GHz)** | **706 MHz (0.71 GHz)** |
| **Memory Clock rate**                       | **924 Mhz**            | **1800 Mhz**           | **2600 Mhz**           |
| **Memory Bus Width**                        | **64-bit**             | **256-bit**            | **320-bit**            |
| **L2 Cache Size**                           | **131072 bytes**       | **524288 bytes**       | **1310720 bytes**      |
| Maximum Texture Dimension Size (x,y,z)      | 1D=(65536)             | 1D=(65536)             | 1D=(65536)             |
|                                             | 2D=(65536, 65536)      | 2D=(65536, 65536)      | 2D=(65536, 65536)      |
|                                             | 3D=(4096, 4096, 4096)  | 3D=(4096, 4096, 4096)  | 3D=(4096, 4096, 4096)  |
| Maximum Layered 1D Texture Size, (num)      | 1D=(16384), 2048       | 1D=(16384), 2048       | 1D=(16384), 2048       |
| layers                                      |                        |                        |                        |
| Maximum Layered 2D Texture Size, (num)      | 2D=(16384, 16384),     | 2D=(16384, 16384),     | 2D=(16384, 16384),     |
| layers                                      | 2048                   | 2048                   | 2048                   |
| Total amount of constant memory             | 65536 bytes            | 65536 bytes            | 65536 bytes            |
| Total amount of shared memory per block     | 49152 bytes            | 49152 bytes            | 49152 bytes            |
| **Total number of registers available per** | **32768**              | **65536**              | **65536**              |
| **block**                                   |                        |                        |                        |
| Warp size                                   | 32                     | 32                     | 32                     |
| Maximum number of threads per               | 2048                   | 2048                   | 2048                   |
| multiprocessor                              |                        |                        |                        |
| Maximum number of threads per block         | 1024                   | 1024                   | 1024                   |
| Max dimension size of a thread block        | (1024, 1024, 64)       | (1024, 1024, 64)       | (1024, 1024, 64)       |
| (x,y,z)                                     |                        |                        |                        |
| Max dimension size of a grid size (x,y,z)   | (2147483647, 65535,    | (2147483647, 65535,    | (2147483647,65535,     |
|                                             | 65535)                 | 65535)                 | 65535)                 |
| Maximum memory pitch                        | 2147483647 bytes       | 2147483647 bytes       | 2147483647 bytes       |
| Texture alignment                           | 512 bytes              | 512 bytes              | 512 bytes              |
| **Concurrent copy and kernel execution**    | **Yes with 1 copy**    | **Yes with 1 copy**    | **Yes with 2 copy**    |
|                                             | **engine(s)**          | **engine(s)**          | **engine(s)**          |
| Run time limit on kernels                   | No                     | No                     | No                     |
| Integrated GPU sharing Host Memory          | Yes                    | No                     | No                     |
| Support host page-locked memory mapping     | Yes                    | Yes                    | Yes                    |
| Alignment requirement for Surfaces          | Yes                    | Yes                    | Yes                    |
| Device has ECC support                      | Disabled               | Disabled               | Enabled                |
| Device supports Unified Addressing (UVA)    | Yes                    | Yes                    | Yes                    |

**In our next post we will show how typical data processing applications perform on Jetson K1 GPU.**
