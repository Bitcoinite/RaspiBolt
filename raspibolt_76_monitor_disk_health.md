---
layout: default
title: Monitor disk health
parent: Bonus Section
nav_order: 45
has_toc: false
---
## Bonus guide: Automatic monitoring of the external disk health
*Difficulty: easy*

[Smartmontools](https://www.smartmontools.org/) is a package that contains the `smartctl` and `smartd` utilities. It allows to monitor modern storage devices using their built-in Self-monitoring, Analysis and Reporting Technology System (SMART).
This guide explains how to install and use the `smartctl` utility and how to set up regular checks and issue notifications using the `smartd` daemon.

#### *Risk : none* 

#### *Requirements: none*

## Installation and preparations

* Install and then check the version

```bash
$ sudo apt install smartmontools
$ smartctl -V
```

* Let's check some information about our device. 
If you've followed the guide, your drive should be at `/dev/sda`. 
We can check this by listing all the devices connected to the Pi

```bash
$ lsblk
>NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
>sda           8:0    0 931.5G  0 disk
>__sda1        8:1    0 931.5G  0 part /mnt/ext
>mmcblk0     179:0    0 119.3G  0 disk
>__blk0p1 179:1    0   256M  0 part /boot
>__blk0p2 179:2    0   119G  0 part /
```

* We can now get some information about this device using the --info (or -i) option. 
The report should tell us that the device supports SMART capability and that it is enabled

```bash
$ sudo smartctl -i /dev/sda
>...
>SMART support is: Available - device has SMART capability.
>SMART support is: Enabled
```

* We need to know the device type, for this we can use the scanning tool
It scans for devices and prints each device name, device type and protocol
The device type will be between brackets, it will probably be [SAT] (for SATA)

```bash
$ smartctl --scan
>/dev/sda -d sat # /dev/sda [SAT], ATA device
```

* We can now do a quick health info. 
--device (or -d) specifies the type of device as found above, so sat for SATA; 
--health (or -H)  checks the device to report its SMART 'Health' status.
The test should show the result PASSED.

```bash
$ sudo smartctl -d sat -H /dev/sda
>=== START OF READ SMART DATA SECTION ===
>SMART overall-health self-assessment test result: PASSED
```

* We can get more information of individual attributes used for the test. 
--all (or -a) will list various attributes and give them a score based on online tests. 
The attribute to look at are #5 Reallocated_Sector_Ct, #187 Reported_Uncorrest, and #233 Media_Wearout_Indicator.
VALUES go from a score of 0 (worst) to 100 (best). We want RAW_VALUES for #5 and 187 to be 0.
```bash
$ sudo smartctl -a /dev/sda
> [...]
> ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
> 5 Reallocated_Sector_Ct   0x0032   100   100   000    Old_age   Always       -       0
> [...]
> 187 Reported_Uncorrect      0x0032   100   100   000    Old_age   Always       -       0
> [...]
> 233 Media_Wearout_Indicator 0x0032   100   100   ---    Old_age   Always       -       1141
> SMART Error Log Version: 1
> No Errors Logged
> [...]

* The test above is simply based on the device characteristics (power on hours, temperature, etc) and on reported errors for the past day-to-day activity (reallocated sectors etc). However, tests can be run that specifically checks the electrical and mechanical properties of the disk and also some amount of disk read testing and data verification. The tests can be either a short test (maximum two minutes) or, if the read/verify test is done on the entire disk rather than a small portion of the disk, a long test (several hours).
To check the estimated duration of each test we can use the --capabilities (or -c) option

```bash
$ sudo smartctl -c /dev/sda
> [...]
> Short self-test routine 
recommended polling time:        (   2) minutes.
> Extended self-test routine
recommended polling time:        ( 182) minutes.
```

* We can now do a short test that should last 2 minutes or less

```bash
$ sudo smartctl -t short /dev/sda
```

* Wait at least two minutes and then check the results.
If all goes well, we should see a 'Completed without error' status.

```bash
$ sudo smartctl -l selftest /dev/sda
> smartctl 6.6 2017-11-05 r4594 [aarch64-linux-5.10.52-v8+] (local build)
> Copyright (C) 2002-17, Bruce Allen, Christian Franke, www.smartmontools.org
>
> === START OF READ SMART DATA SECTION ===
> SMART Self-test log structure revision number 1
> Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
> # 1  Short offline       Completed without error       00%      1698         -
```

* If you want to read more about Smartmontools utilities and their options, you can read the manual: `$ man smartctl`

## Regular checks using the smartd daemon

`smartd` is a daemon that runs SMART tests on devices very 30 minutes (by default). Here, we will set up `smartd` to run every 30 minutes and an email notification if an issue is detected. 

### Setting up the mailbox and email utility

To send an email notification we will use `msmtp`, a light SMTP client that is able to send emails to a third-party SMTP server.  

* Install msmtp

```bash
$ sudo apt-get update
$ sudo apt-get install msmtp
```

We will use a configuration file specific to the admin user and paste the following settings

```bash
$ cd ~/
$ nano .msmtprc
```

```ini
# Set default values
defaults
auth on
tls on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile /var/log/msmtp.log

# Gmail configuration
account gmail
host smtp.gmail.com
port 587
from <your-username>@gmail.com
user <your_username>
passwordeval app-specific-password

# Set a default account
account default : gmail
```

* We need to setup the app-specfic password in our dedicated gmail account.
* Follow this link: [https://myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
* Log in
* Security > Less secure app access > Turn off

* 
* Open the smartd configuration file

```bash
$ sudo nano /etc/smartd.conf
```

* By default, everything is commented out except one line starting with DEVICESCAN.
With this keyword, the daemon scans for all existing ATA and SCSI devices, ignoring the rest of the configuration.
Instead, we want to only look at our sda device, so we comment the line starting with DEVICESCAN and uncomment the line starting with .

```ini
[...]
#DEVICESCAN -d removable -n standby -m root -M exec /usr/share/smartmontools/smartd-runner
[...]


# the script exits when a command fails
set -o errexit
# the script exits when a variable is undeclared
set -o nounset

# we create a log file named ssd_smartmon-$1.log, with $1 the name of the device we want to check (sda in our case)
## the log will start with a timestamp
date > /tmp/ssd_smartmon-$1.log
# the second line is the overall result of the SMART Health chech
sudo smartctl -d sat -H /dev/$1 | grep -i overall >> /tmp/ssd_smartmon-$1.log
# the next four lines of the log are key attributes from a SMART check
sudo smartctl -d sat -a /dev/$1 | grep -i -E 'attribute_name|reallocated|reported|wearout'>> /tmp/ssd_smartmon-$1.log
# the last six lines show the error log version and the logged errors
sudo smartctl -d sat -l error /dev/$1 >> /tmp/ssd_smartmon-$1.log
```
* We now need to make the file executable. 
Once executable, the file name should appear with a green color.

```bash
$ chmod +x ssd_smartmon.sh
$ ls -la
```

* Let's test our script is working by executing it and looking at the log. We execute the script with the argument `sda`.

```bash
$ ./ssd_smartmon.sh sda
$ cat /tmp/ssd_smartmon_log-sda
>Thu  7 Oct 12:27:09 BST 2021
>SMART overall-health self-assessment test result: PASSED
>ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
>5 Reallocated_Sector_Ct   0x0032   100   100   000    Old_age   Always       -       0
>187 Reported_Uncorrect      0x0032   100   100   000    Old_age   Always       -       0
>233 Media_Wearout_Indicator 0x0032   100   100   ---    Old_age   Always       -       1123
>smartctl 6.6 2017-11-05 r4594 [aarch64-linux-5.10.52-v8+] (local build)
>Copyright (C) 2002-17, Bruce Allen, Christian Franke, www.smartmontools.org
>
>=== START OF READ SMART DATA SECTION ===
>SMART Error Log Version: 1
>No Errors Logged
```

## Regular script execution

Now we can edit the admin user crontab to run the script every hours for example.

* Open the admin user crontab and add the following lines at the end of the file

```bash
$ crontab -e
```

```ini
# simple smart checks everyday
0 0 * * * /home/admin/smartmonitor sda
```

## Notifications of issues

TBD
