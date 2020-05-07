---
layout: post
title: "How to break Ubuntu 13.04/14.04 with vanilla CUDA driver and unbreak it back"
tags:
  - Software Engineering
thumbnail_path: blog/2014-06-01-how-to-break-ubuntu-13-04-14-04-with-vanilla-cuda-driver-and-unbreak-it-back/8.png
---

After installing CUDA driver from NVIDIA website, Ubuntu 13.04/14.04 window manager decorations (Unity, via Compiz) may stop working properly on Optimus machines (primary low-end Intel GPU + secondary high-end NVIDIA GPU).

![alt text](/assets/img/blog/2014-06-01-how-to-break-ubuntu-13-04-14-04-with-vanilla-cuda-driver-and-unbreak-it-back/8.png "Logo Title Text 1")

This tutorial explains how to bring back window manager decorations.

(Also available as [PDF](https://parallel-computing.pro/images/articles/ubuntu-cuda/Ubuntu13.04-CUDA.pdf))

Purpose of CUDA driver install:

- We want to install CUDA driver to enable the use of NVIDIA GPU in compute applications
- We want to have the newest CUDA development driver from NVIDIA website
- We do NOT want to do desktop acceleration with NVIDIA GPU – desktop shall still be managed by Intel GPU
- If you want to do NVIDIA desktop acceleration, this tutorial is NOT for you. Look into bumblebee instead.

## Breaking by installing vanilla CUDA driver:

![alt text](/assets/img/blog/2014-06-01-how-to-break-ubuntu-13-04-14-04-with-vanilla-cuda-driver-and-unbreak-it-back/1.png "Logo Title Text 1")

1. Switch to console-mode tty: `Ctrl+Alt+F2`

2. Disable Nouveau:

   ```
   $ cd Downloads
   $ sudo service lightdm stop
   $ chmod +x cuda_6.0.37_linux_64.run
   $ sudo ./cuda_6.0.37_linux_64.run
   $ sudo reboot
   ```

3. Install driver, CUDA toolkit and samples:

   ```
   $ cd Downloads
   $ sudo service lightdm stop
   $ sudo ./cuda_6.0.37_linux_64.run
   $ sudo reboot
   ```

After reboot and login, decorations and main main menu may disappear:

![alt text](/assets/img/blog/2014-06-01-how-to-break-ubuntu-13-04-14-04-with-vanilla-cuda-driver-and-unbreak-it-back/2.png "Logo Title Text 1")

## The following steps describe how to bring back the normal desktop:

1. Press `Ctrl+Alt+T` to open terminal

2. Install mc: `$ sudo apt-get install mc`

3. Execute mc under admin: `$ sudo mc`

4. Delete files in `/usr/lib/xorg/modules/extensions` as shown below:
   ![alt text](/assets/img/blog/2014-06-01-how-to-break-ubuntu-13-04-14-04-with-vanilla-cuda-driver-and-unbreak-it-back/3.png "Logo Title Text 1")

5. Delete libGL* files in `/usr/lib`, and libGL* and libGLES\* files in `/usr/lib32`
   ![alt text](/assets/img/blog/2014-06-01-how-to-break-ubuntu-13-04-14-04-with-vanilla-cuda-driver-and-unbreak-it-back/4.png "Logo Title Text 1")
   ![alt text](/assets/img/blog/2014-06-01-how-to-break-ubuntu-13-04-14-04-with-vanilla-cuda-driver-and-unbreak-it-back/5.png "Logo Title Text 1")

6. Reinstall GLX module: `$ sudo apt-get install --reinstall libgl1-mesa-glx`
   ![alt text](/assets/img/blog/2014-06-01-how-to-break-ubuntu-13-04-14-04-with-vanilla-cuda-driver-and-unbreak-it-back/6.png "Logo Title Text 1")

7. Reinstall xserver: `$ sudo apt-get install --reinstall xserver-org-core`
   ![alt text](/assets/img/blog/2014-06-01-how-to-break-ubuntu-13-04-14-04-with-vanilla-cuda-driver-and-unbreak-it-back/7.png "Logo Title Text 1")

8. Reinstall xorg: `$ sudo apt-get install --reinstall xorg`
   ![alt text](/assets/img/blog/2014-06-01-how-to-break-ubuntu-13-04-14-04-with-vanilla-cuda-driver-and-unbreak-it-back/8.png "Logo Title Text 1")

9. Reset and restart compiz:

   ```
   $ dconf reset -f /org/compiz/
   $ compiz –replace
   ```

   ![alt text](/assets/img/blog/2014-06-01-how-to-break-ubuntu-13-04-14-04-with-vanilla-cuda-driver-and-unbreak-it-back/9.png "Logo Title Text 1")

10. You should now see decorator and main menu running normally:
    ![alt text](/assets/img/blog/2014-06-01-how-to-break-ubuntu-13-04-14-04-with-vanilla-cuda-driver-and-unbreak-it-back/10.png "Logo Title Text 1")

## How to avoid this problem in future: **do not install OpenGL files**:

- Install graphics driver separately w/o OpenGL (--no-opengl-files):

```
Ctrl+Alt+F2
$ cd Downloads
$ ./cuda_6.0.37_linux_64.run --extract=$HOME/Downloads/cuda
$ cd cuda
$ sudo service lightdm stop
$ sudo ./NVIDIA-Linux-x86_64-331.62.run --no-opengl-files
```

- Then install CUDA and CUDA samples:

```
$ cd ..
$ sudo ./cuda_6.0.37_linux_64.run
Do you accept the previously read EULA? (accept/decline/quit): accept
Install NVIDIA Accelerated Graphics Driver for Linux-x86_64 331.62? ((y)es/(n)o/(q)uit): n
Install the CUDA 6.0 Toolkit? ((y)es/(n)o/(q)uit): y
Enter Toolkit Location [ default is /usr/local/cuda-6.0 ]: /opt/cuda
Do you want to install a symbolic link at /usr/local/cuda? ((y)es/(n)o/(q)uit): y
Install the CUDA 6.0 Samples? ((y)es/(n)o/(q)uit): y
$ sudo reboot
```
