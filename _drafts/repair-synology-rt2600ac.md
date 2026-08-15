---
layout: post
title: Trying to repair a blocked Synology RT2600ac
---

This article is abount trying to fix a blocked Synology RT2600ac.

# Situation
The LED for 2.4G does not light up after the boot process.
* STATUS is solid orange
* 2.4G is off
* 5G is solid green
When a computer is wired to an Ethernet port, the corresponding LED (WAN1, WAN2[^wan2], 1, 2, 3, 4) is not blinking either.

According to Synology Knowldge Center[^synology-led-indicators], the solid orange status indicates the router is booting up, but this state does not change even after hours of patience.

There is no clear information if this is due to a hardware or software problem with the Wi-Fi interface for 2.4G, or something else.

Eventhough the LEDs for wired are not blinking, the computer is assigned an IP address on the network 192.168.1.0/24.
The first time install loads, but blocks when setting the password, despite meeting complexity requirements.

Also, no wireless network is broadcasted on the 5G band.

The router offers 2x USB-A ports and 1x SD card slot. However, no solution is documented to perform an update update via them without requiring the web interface.

[^wan2]: WAN2 can be configured on the RJ45 port 1.
[^synology-led-indicators]: [What do the LED indicators on my Synology Router mean?](https://kb.synology.com/en-global/SRM/tutorial/What_do_LED_indicators_mean_MR2200ac)

# Action plan
As the status indicates the boot sequence is not complete, but the router already seems to have features started, like wired networking and first time installation.
Trying to force completion of the first time installation is a good first path to explore, with a goal to update the router with a newer software version.
