---
layout: post
title: "Use multiple mirrors in apt"
tags:
- Linux
- Ubuntu
thumbnail_path: blog/2022-11-16-use-multiple-mirrors-in-apt/ubuntu-logo.png
---

There is an interesting and not so well-known feature of APT package manager: the ability to automatically choose a download mirror for every individual operation.

By "mirror" I mean the URL in `/etc/apt/sources.list`, for example `http://ch.archive.ubuntu.com/ubuntu` stands for official Ubuntu mirror in Switzerland. You may make a choice of a mirror during the system installation, and then it remains static, regardless of your location. In fact, you can manually switch to another mirror for the duration of stay in another country. But actually, there is a way to let the package manager to do this automatically by specifying `mirror://mirrors.ubuntu.com/mirrors.txt` as the URL. The `mirror://` appears to be a special protocol, which is handled internally by the recent versions of APT. Upon the action, the URL is substituted by an real URL of some mirror chosen by the backend.

The "mirror://" could have been just a mirror picker, a simple redirection. But actually it's much cooler than that: it does sort of load balancing! So, when you update the packages lists, or install a package along with many dependencies, it will download different things from multiple mirrors simultaneously!

An ordinary `source.list` could be switch to `mirror://` with a simple command (we will do a backup, of course):

```
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo sed -i "s|http://.*\.archive\.ubuntu\.com/ubuntu/|mirror://mirrors.ubuntu.com/mirrors.txt|g" /etc/apt/sources.list
```

Now we can run `apt update`, and enjoy the balanced use of multiple mirrors:

Voila. In case one mirror experiences a momentum heavy usage, the operation could finish much faster.

Credits go to this [askubuntu post](https://askubuntu.com/a/37754/919956).
 
