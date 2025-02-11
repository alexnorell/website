---
title: "IoT Village Virtual CTF"
date: 2021-05-01T10:31:33-07:00
draft: false
tags:
  - security
  - ctf
  - iot
categories:
  - security
images:
  - images/rack.jpg
  - images/results.png
---

Last week, [IoTVillage](https://iotvillage.org) hosted a virtual Capture the Flag (CTF) event. The purpose of the event is to use known exploits on devices to retrieve information from within the device. This information is known as a flag and can really be anything; from the MD5 of a particular function, to the contents of a file in the system.

The CTF is set up as a 3 tiered network with vulnerable devices in each network. You connect into the network via a VPN and it drops you into level one. There are no instructions other than the name of product and what we need to retrieve from the device. These devices are all IoT devices with known vulnerabilities: Network Attached Storage Devices, IP Cameras, Consumer Routers, etc.

{{< img src="images/rack.webp" alt="IoT Village SOHOplessly Broken Rack" >}}

Myself and a friend worked through most of level one over the two day event. We had already attempted this CTF last year DEFCON 28. Some of the challenges were the same and we were able to use the same exploits to grab new flags. We also did a much better job of some of the static analysis flags. In the end, we ended up tying for 2nd place, which is much better than our 14th place finish at DEFCON 28.

{{< img src="images/results.webp" alt="weslack team results" >}}

For more information about what we did during the CTF, refer to our write-ups website, [weslack.team](https://weslack.team).