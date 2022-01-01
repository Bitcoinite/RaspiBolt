---
layout: default
title: UPS
parent: + Raspberry Pi
grand_parent: Bonus Section
nav_exclude: true
has_toc: false
---

# Bonus guide: UPS
{: .no_toc }

This guide shows how to set up a UPS (Uninterruptible Power Supply) and connect the node to it to initiate a graceful shutdown when the battery is almost empty. This set up will prevent data corruption in case of power issues.

Difficulty: Medium
{: .label .label-yellow }

Status: Tested v3
{: .label .label-green }

![ups](../../ups.png)

---

## UPS installation

### Selecting

We want the UPS to avoid being offline during short power cuts and to gives enough time to do a garceful shutdown. Say we would like the UPS to power our node for up to 1 hour.  

First, we need to calculate an estimation of how much power the node uses:
* Raspberry Pi 4 = ~ 5 W ([source](https://www.pidramble.com/wiki/benchmarks/power-consumption):target="_blank"})
* SSD drive = ~ 5 W ([source](https://www.anandtech.com/show/8216/samsung-ssd-850-pro-128gb-256gb-1tb-review-enter-the-3d-era/12){:target="_blank"})
* Internet router = ~ 5 W ([source](https://generatorist.com/power-consumption-of-household-appliances){:target="_blank"})
* TOTAL = ~ 15 W

For a power draw of 15 W for 1 hour, the UPS needs to be **> 350 VA**.

### Installing

* Turn off your node and router gracefully
* Unplug the Pi and router from their sockets
* Plug them into the UPS on the side that is protected by the battery
* Plug the UPS into the wall socket

---

## Node-UPS connection

TBD

<br /><br />

------

<< Back: [+ Raspberry Pi](index.md)
