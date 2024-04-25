---
author:	Vance
date:	2023-08-09
title:	Setting up Home Assistant on a Raspberry Pi using Docker
---

## Install OS on the Pi

### Pi OS

Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/), and use it to install Pi OS on your SD Card.

Run the Imager then: 
~~~sh
Choose OS -> Raspberry Pi OS (other) -> Raspberry Pi OS Lite (64-bit)

Choose Storage -> <your SD Card>

Click the cog and fill in details for Wifi and SSH, then click `write`
~~~

Once the card has been written and verified put it into the Raspberry Pi and boot it.  First time boot can take a little while, but once booted you should be able to SSH in to the Pi using the credentials you set up in the Pi Imager.

#### Initial setup

Expand the file system to the full size of the SD card.

~~~ sh
sudo raspi-config

Advanced Options -> A1 Expand Filesystem
~~~

Then reboot

#### Update all software on the Pi

~~~sh
sudo apt update && sudo apt upgrade
~~~

## Base OS Software

### Docker

Guides:

* [Official Docker Guide](https://docs.docker.com/engine/install/debian/#install-using-the-repository)

**Watch Out!** Because you installed the 64-bit Pi OS, you need to follow the [Debian install](https://docs.docker.com/engine/install/debian/#install-using-the-repository) and not the [Raspberry Pi Install](https://docs.docker.com/engine/install/raspbian/) which only works for the 32-bit OS

Use the second option in this guide for installation, namely "Use Docker's apt repository"

### Nginx

Install `nginx` 

~~~sh
sudo apt install nginx
~~~

Set up proxy entries for access to all of the installed components web pages

#### Nginx config files

##### /etc/nginx/snippets/ssl.conf

~~~sh

~~~

##### /etc/nginx/sites-available/weyhill.geo-fun.org

~~~sh

~~~

#### CertBot SSL certificates

Install [certbot](https://snapcraft.io/install/certbot/raspbian) and [certbot-dns-route53](https://certbot-dns-route53.readthedocs.io/en/stable/) using `snapd` for both

~~~sh
certbot certonly --dns-route53 -d weyhill.geo-fun.org
~~~

Certbot sets up a renewal task automatically.  Check it with:

~~~sh
sudo systemctl list-timers
~~~

### Remote Access

#### DNS with updates

In order to access this Pi from outside the home network we need an IP address.  AWS Route53 is hosting my domain geo-fun.org, so we are going to add a server to this list and allow the Pi to update the IP as it changes.  DNS name for the Pi is `weyhill@geo-fun.org`

Install [`aws_cli`](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). Tweak `/root/bin/r53_weyhill.sh`

Create a crontab job to check for updated IP address.

~~~sh

~~~

#### Port forwarding

The only port that I intend to open is for wireguard.  Once we are connected to the wireguard VPN we should be able to access the rest of the network.

In order to contact the Pi behind the BT Hub is to configure the BT Hub to forward the necessary ports to the Pi.

At a minimum this will be the wireguard port `33187`

#### Wireguard

Install PiVPN using [this guide](https://adamtheautomator.com/wireguard-raspberry-pi/)

I changed the port away from the default to `33187`.  I also set up the dns name of the server to `weyhill.geo-fun.org`

#### SSH

Password authentication configured during the writing of the SD card.  To make access more secure I have uploaded my key and turned off password authentication in `/etc/ssh/sshd_config`

## Docker Containers

We are aiming for the following docker contaners:

* [Home Assistant](https://www.home-assistant.io/installation/raspberrypi#install-home-assistant-container)
* [Mosquitto](https://hub.docker.com/_/eclipse-mosquitto)
* [Zigbee2MQTT](https://hub.docker.com/r/koenkk/zigbee2mqtt/)
* [Appdaemon](https://appdaemon.readthedocs.io/en/latest/DOCKER_TUTORIAL.html)
* [Sunsync](https://github.com/kellerza/sunsynk)

### Docker Compose

The above containers are going to be configured in a `docker-compose.yaml` file so that all of the containers can be brought up at together with dependencies respected.

