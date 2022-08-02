---
layout: post
title: "VPN tunnelling for ipv6 only providers"
tags:
- Networking
thumbnail_path: blog/2022-08-02-vpn-tunnelling-for-ipv6-only-providers/vpn.png
---

The number of providers having problems with ipv4 support is growing. Recently we came across an ISP, which offers only ipv6, and is able to connect only to ipv6 websites. I was able to follow this [excellent tutorial](https://wpyoga.github.io/blog/2021/10/30/oracle-cloud-instance-ipv6) and configure an Always-free Oracle Cloud VPS to enable ipv6. After it is all set, I was able to configure a private VPN tunnel on top of it, in order to get the normal ipv4 support in my location. If the OpenVPN server is proxied via Nginx, ipv6 must also be enabled in the Nginx configuration, as shown [here](https://unixcop.com/how-to-enable-ipv6-in-nginx/).
