---
layout: default
title: Monitoring
nav_order: 60
parent: Raspberry Pi
---
<!-- markdownlint-disable MD014 MD022 MD025 MD033 MD040 -->
{% include_relative include_metatags.md %}

# Monitoring
{: .no_toc }

We set up a simple welcome script to monitor the disk health and data integrity

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Monitoring data integrity and hardware health can help prevents catastrophic failures and the loss of funds or time.

## Install packages

* Install smartmontools and debsums

  ```sh
  $ sudo apt update
  $ sudo apt install smartmontools debsums
  ```

---

## Configure smartd

* Check that your drive has SMART capability and that it is enabled

  ```sh
  $ sudo smartctl -i /dev/sda
  > [...]
  > SMART support is: Available - device has SMART capability.
  > SMART support is: Enabled
  ```

---

## Welcome script

* Open the MOTD script and paste the following lines at the end. Save and exit.

  ```sh
  $ sudo nano /etc/update-motd.d/10-uname
  ```
    ```ini
  > ###################################
  > # Disk and data health monitoring #
  > ###################################
  >
  > # Specify colors
  > color_grey='\033[0;37m'
  > color_green='\033[0;32m'
  >
  > # Obtain test results
  > overall=$(sudo smartctl -d sat -H /dev/sda | grep -i overall)
  > error=$(sudo smartctl -d sat -l error /dev/sda | tail -2 | head -n 1)
  > test=$(sudo smartctl -l selftest /dev/sda | grep "# 1")
  >
  > # Display results
  > printf "${color_green}%-50s
  > ${color_grey}Logged erros: ${color_green}%-50s
  > ${color_grey}Result of the last short test: ${color_green}%-50s   
  > " \
  > "${overall}" \
  > "${error}" \
  > "${test}"
  ```ini
  
  > # Disk and data health monitoring
  > overall=$(sudo smartctl -d sat -H /dev/sda | grep -i overall)
  > error=$(sudo smartctl -d sat -l error /dev/sda | tail -2 | head -n 1)
  > test=$(sudo smartctl -l selftest /dev/sda | grep "# 1")
  > 
  ```








