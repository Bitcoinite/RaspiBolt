---
layout: default
title: ThunderHub
parent: + Lightning
grand_parent: Bonus Section
nav_exclude: true
has_toc: false
---

# Bonus Guide: Install ThunderHub
{: .no_toc }

---

[ThunderHub](https://github.com/apotdevin/thunderhub){:target="_blank"} is an open source LND node manager where you can manage and monitor your node on any device or browser. It allows you to take control of the lightning network with a simple and intuitive UX and the most up-to-date tech stack.

Difficulty: Medium
{: .label .label-yellow }

Status: Tested v3
{: .label .label-green }

![thunderhub](../../images/75_thunderhub_channel_view.png)

---

Table of contents
{: .text-delta }

1. TOC
{:toc}

---

## Requirements

* Node.js v14.15+
* NPM

---

## Preparations

### Check Node.js

* Node.js v16 should have been installed for the BTC RPC Explorer and RTL. We can check our version of Node.js with user "admin": 
  
  ```sh
  $ node -v
  > v16.13.2
  ```

* If the version is v14.15 or above, you can move to the next section. If Node.js is not installed, follow [this guide](https://raspibolt.org/btcrpcexplorer.html#install-nodejs){:target="_blank"} to install it.

### Firewall & reverse proxy

* Enable NGINX reverse proxy to route external encrypted HTTPS traffic internally to ThunderHub

  ```sh
  $ sudo nano /etc/nginx/streams-enabled/thunderhub-reverse-proxy.conf
  ```

  ```sh
  upstream rtl {
    server 127.0.0.1:3010;
  }
  server {
    listen 4002 ssl;
    proxy_pass rtl;
  }
  ```

* Test and reload NGINX configuration

  ```sh
  $ sudo nginx -t
  $ sudo systemctl reload nginx
  ```

* Configure firewall to allow incoming HTTPS requests

  ```sh
  $ sudo ufw allow 4002/tcp comment 'allow TunderHub SSL'
  $ sudo ufw status
  ```
  
---

## ThunderHub

### Data directory

We do not want to run ThunderHub alongside `bitcoind` and `lnd` because of security reasons.
For that we will create a separate user and we will be running the code as the new user.

* Create a new "thunderhub" user. The new user needs read-only access to the `tls.cert` and our `admin.macaroon`, 
  so we add him to the "lnd" group. Open a new session.

  ```sh
  $ sudo adduser --disabled-password --gecos "" thunderhub
  $ sudo adduser thunderhub lnd
  $ sudo su - thunderhub
  ```

* Download the source code directly from GitHub and install all dependencies using NPM.

  ```sh
  $ git clone https://github.com/apotdevin/thunderhub.git
  $ cd thunderhub
  ```

### Configuration

By default, ThunderHub is configured to favour privacy over functionality.

* Still with user "thunderhub", make a local copy of the configuration file that will be preserved during future updates.

  ```sh
  $ cp ~/thunderhub/.env ~/thunderhub/.env.local
  $ nano ~/thunderhub/.env.local
  ```

* By default, ThunderHub runs on port 3000 which is already allocated to Ride The Lightning. 
Add the following line at the end of the "Server Configs" section to use port 3010 instead.

  ```ini
  BASE_PATH = 'http://localhost:3010'
  ```

* Add the following line at the end of the URLs section to use our own self-hosted blockchain explorer. Save and exit.

  ```ini
  MEMPOOL_URL='https://raspibolt.local:4000/'
  ---

* 

### Installation

* Still with user "ThunderHub" and still inside `~/thunderhub`, install ThunderHub

  ```sh
  $ npm install
  $ npm run build
  ```

## First Start

Test starting ThunderHub manually first to make sure it works.

* Let's do a first start to make sure it's running as expected. 
  Make sure we are in the thunderhub directory and start the web server.

 ```
  $ cd ~/thunderhub
  $ npm run start -p 3010
  ```

* Now point your browser to `http://raspibolt.local:4002` (or whatever you chose as hostname) or the ip address (e.g. `http://192.168.0.20:4002`).
  You should see the home page of ThunderHub.

* If you see a lot of errors on the RaspiBolt command line, then you have to change file permissions maybe,  
  because thunderhub can't "access" the `.lnd` directory.

* Stop ThunderHub in the terminal with `Ctrl`-`C` and exit the "thunderhub" user session.

  ```
  $ exit
  ```

---

## Autostart on boot

Now we'll make sure ThunderHub starts as a service on the Raspberry Pi so it's always running.
In order to do that we create a systemd unit that starts the service on boot directly after LND.

* As user "admin", create the service file.

  ```sh
  $ sudo nano /etc/systemd/system/thunderhub.service
  ```

* Paste the following configuration. Save and exit.

  ```ini
  # RaspiBolt: systemd unit for Thunderhub
  # /etc/systemd/system/thunderhub.service

  [Unit]
  Description=Thunderhub
  Wants=lnd.service
  After=network.target lnd.service

  [Service]
  WorkingDirectory=/home/thunderhub/thunderhub
  ExecStart=/usr/bin/npm run start -p 3010
  User=thunderhub
  Restart=always
  TimeoutSec=120
  RestartSec=30
  StandardOutput=null
  StandardError=journal

  [Install]
  WantedBy=multi-user.target
  ```

* Enable the service, start it and check log logging output.

  ```sh
  $ sudo systemctl enable thunderhub.service
  $ sudo systemctl start thunderhub.service
  $ sudo journalctl -f -u thunderhub
  ```

* You can now access ThunderHub from within your local network by browsing to <http://raspibolt.local:3010> (or your equivalent ip address).

---

## Remote access over Tor (optional)

Do you want to access ThunderHub remotely?
You can easily do so by adding a Tor hidden service on the RaspiBolt and accessing ThunderHub with the Tor browser from any device.

* Add the following three lines in the section for "location-hidden services" in the `torrc` file.
  Save and exit.

 ```sh
  $ sudo nano /etc/tor/torrc
  ```

  ```ini
  HiddenServiceDir /var/lib/tor/thunderhub
  HiddenServiceVersion 3
  HiddenServicePort 80 127.0.0.1:3010
  ```

* Restart Tor and get your connection address.

  ```sh
  $ sudo systemctl restart tor
  $ sudo cat /var/lib/tor/thunderhub/hostname
  > abcdefg..............xyz.onion
  ```

* With the [Tor browser](https://www.torproject.org), you can access this onion address from any device.
  Please be aware that this access is not password protected and should not be shared widely.

**Congratulations!**
You now have the BTC RPC Explorer running to check the Bitcoin network information directly from your node.

---

## Upgrade

Updating to a [new release](https://github.com/apotdevin/thunderhub/releases) should be straight-forward.

* From user "admin", stop the service and open a "thunderhub" user session.

  ```sh
  $ sudo systemctl stop thunderhub
  $ sudo su - thunderhub
  ```

* Fetch the latest GitHub repository information and check out the new release:

  ```sh
  $ cd thunderhub
  $ git pull
  $ npm install
  $ npm run build
  $ exit
  ```

* Start the service again.

  ```sh
  $ sudo systemctl start thunderhub
  ```
  
  <br /><br />
  
---

<<Back: [+ Lightning](index.md)
