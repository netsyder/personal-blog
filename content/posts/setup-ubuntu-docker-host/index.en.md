---
title: "How to setup Ubuntu as a Docker Host"
date: 2024-02-16T02:35:45.517Z
lastmod: 
draft: false
author: "Bersayder"
authorLink: "https://netsyder.com"
description: "First of a series of tutorials about my homelab in no particular order " 
images: []
resources:
- name: "featured-image"
  src: "Ubuntu_Docker.png"

tags: ["Home Lab", "Linux", "Ubuntu", "Docker", "Automation", "Portainer","Cockpit"]
categories: ["Home Lab"]

lightgallery: true
---
This article is part of a serie that will show how I have setup my own homelab after learning how to do it from multiple places on the internet. <!--more-->This serie won´t be in any particular order, so let's Begin.

{{<admonition warning "Warning: Work in Progress" true>}}
This article is incomplete, since I'm still learning Markdown and the features of this Hugo theme.
{{</admonition>}}

The steps are as follows:

1. Fix Network Config for IP Address Reservation
1. Fix DNS Behavior (Optional but needed if going to host Pi-Hole)
1. Install cockpit for webgui administration
1. Install docker
1. Install portainer

#Fix Network Config for IP Address Reservation

What do I mean by this? Basically Ubuntu uses a Client-ID that is not the MAC Address when sending a DHCP request. This means that if you create an IP reservation on your DHCP server, *maybe* your Ubuntu server will receive a different IP address instead of said reservation. This depends entirely on the behavior of the DHCP server, but to prevent this issue we can force our server to use its MAC Address as the Client-ID.

To set Ubuntu to use it's MAC as the Client-ID we need to edit the netplan config file located in `/etc/netplan/`

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```
We are going to set our file as follows:

```bash
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens3:
      dhcp4: true
      dhcp-identifier: mac
      nameservers:
        addresses: [1.1.1.1,8.8.8.8,9.9.9.9]
  version: 2
  renderer: NetworkManager
```
---