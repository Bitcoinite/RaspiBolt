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

* Install [`smartmontools`](https://www.smartmontools.org/){:target="_blank"}, a drive health monitoring tool, and [`debsums`](https://manpages.org/debsums){:target="_blank"}, a Debian package integrity check tool.

  ```sh
  $ sudo apt update
  $ sudo apt install smartmontools debsums
  ```

---

## Drive health tests

We configure `smartd`, the `smartmontools` daemon, to run short and long drive health tests at regular intervals. 

* Check that your drive has SMART capability and that it is enabled

  ```sh
  $ sudo smartctl -i /dev/sda
  > [...]
  > SMART support is: Available - device has SMART capability.
  > SMART support is: Enabled
  ```

* Open the daemon configuration file

  ```sh
  $ sudo nano /etc/smartd.conf
  ```

* Comment out the following line

  ```ini
  #DEVICESCAN -d removable -n standby -m root -M exec /usr/share/smartmontools/smartd-runner
  ```

* Add the following line at the end of the file. Save and exit.

  ```ini
  # Short test every day between 5-6am; long test on the 21st of each month, starting between 1-2 am
  /dev/sda -a -s (S/../.././5|L/../21/./1)
  ```

* Restart the daemon

  ```sh
  $ sudo systemctl restart smartd
  ```

---

## Packages integrity test

`debsums` checks the MD5 sums of installed Debian packages and output a list of corrupted packages. 
The can take a few minutes and cannot be run each time we log in to the node. 
Instead, we will set up a cron job to run the test once a day and export the results in a log file.

* Run the command manually. It should return nothing.

  ```sh
  $ sudo debsums -c
  ```

* Edit (option -e) the `crontab` file of the "root" user. 
If asked, select the `/bin/nano` text editor (type `1` and `Enter`)

  ```sh
  $ sudo crontab -e
  ```

* At the end of the file, paste the following lines. Save and exit.

  ```ini
  ##########################################
  # 1 - Debian package integrity test #
  ##########################################

  # Run a debsums test once a day, at 4:21am. Log the results in the /tmp/raspibolt_debsums_test.log log file
  21 4 * * * debsums -c > /var/log/raspibolt_debsums_test.log 2>&1; date >> /var/log/raspibolt_debsums_test.log 
  ```

---

## Script

To prevent drive failures, we need to be alerted of drive issues as early as possible. 
Now that we configured the drive health tests, we need a way to automatically see the results on a regular basis. 

### MOTD script

We will use the 'Message Of The Day' functionality of Linux, which displays a message in the terminal when a SSH session is opened.

* Deactivate the default MOTD messages
  
  ```sh
  $ sudo mv /etc/motd /etc/motd.bak
  $ sudo chmod -x /etc/update-motd.d/10-uname
  ```

* Create a new MOTD script and paste the following lines at the end. Save and exit.

  ```sh
  $ sudo nano /etc/update-motd.d/21-disk-health
  ```

  ```ini
  #!/bin/sh
  
  ###################################
  # Disk and data health monitoring #
  ###################################
  
  # Safety bash script options
  # -e causes a bash script to exit immediately when a command fails
  # -u causes the bash shell to treat unset variables as an error and exit immediately.
  set -eu
  
  # Specify colors
  color_red='\033[0;31m'
  color_grey='\033[0;37m'
  color_green='\033[0;32m'
  
  # Obtain test results
  overall=$(sudo smartctl -d sat -H /dev/sda | grep -i overall)
  error=$(sudo smartctl -d sat -l error /dev/sda | tail -2 | head -n 1)
  test=$(sudo smartctl -l selftest /dev/sda | grep "# 1")
  debsums=$(cat /var/log/raspibolt_debsums_test.log)
  
  if [[ -n $sums ]]
  then
      debsums="!!You have corrupted Debian packages!! Run 'sudo debsums -c' to get a list of damaged packages."
  else
      debsums="All good! There are no corrupted packages."
  fi
  
  # Display results
  printf "
  ${color_grey}
  ${color_grey}##########################################################
  ${color_grey}#              SSD DRIVE HEALTH REPORT                   #
  ${color_grey}##########################################################
  ${color_grey}
  ${color_grey}1. %-50s
  ${color_grey}
  ${color_grey}2. Logged errors:
  ${color_green}   %-20s
  ${color_grey}
  ${color_grey}3. Result of the last short test:
  ${color_green}   %-20s
  ${color_green}
  ${color_grey}4. Latest Debian package integrity test:
  ${color_red}   %-20s
  ${color_grey}##########################################################
  " \
  "${overall}" \
  "${error}" \
  "${test}" \
  "${debsums}"
  ```

* Make the script executable
  
  ```sh
  $ sudo chmod +x /etc/update-motd.d/05-disk-health
  ```








