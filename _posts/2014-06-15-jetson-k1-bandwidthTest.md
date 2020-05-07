---
layout: post
title: "Jetson K1: bandwidthTest"
tags:
  - Software Engineering
thumbnail_path: blog/2014-06-15-jetson-k1-bandwidthTest/jetson-k1-bandwidthTest.png
---

Chart on the left shows the bandwidths of memory transfers on Jetson K1 ([Click](\assets\img\blog\2014-06-15-jetson-k1-bandwidthTest\jetson-k1-bandwidthTest.pdf) to enlarge). For the baseline we also added GTX680M's host-device and device-host (peak device-device is ~90K -- too large for this chart).

![alt text](\assets\img\blog\2014-06-15-jetson-k1-bandwidthTest\jetson-k1-bandwidthTest.png "Logo Title Text 1")

Summary of Jetson K1's bandwidthTest results, two of which are quite unusual:

- peak device-device is ~**7.5x** smaller than GTX680M's
- peak pinned host-device is ~**12x** smaller than GTX680M's
- peak pinned device-host is only ~**2x** smaller than GTX680M's (!)
- peak pageable host-device is ~**1.5x** higher than pinned host-device (!)

[Download](\assets\img\blog\2014-06-15-jetson-k1-bandwidthTest\jetson-k1-bandwidthTest.ods) raw data table in ODS format.
