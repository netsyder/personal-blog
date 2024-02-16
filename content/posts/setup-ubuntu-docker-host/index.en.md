---
title: "How to setup Ubuntu as a Docker Host"
date: 2024-02-16T02:35:45.517Z
lastmod: {{ .Date }}
draft: false
author: "Bersayder"
authorLink: "https://netsyder.com"
description: "First of a series of tutorials about my homelab in no particular order " 
images: []
resources:
- name: "featured-image"
  src: "Ubuntu_Docker.png"

tags: ["Home Lab", "Linux", "Ubuntu", "Docker", "Portainer","Cockpit"]
categories: ["Home Lab"]

lightgallery: true
---
This article is part of a serie that will show how I have setup my own homelab after learning how to do it from multiple places on the internet. <!--more-->This serie won´t be in any particular order, so let's Begin.

{{<admonition warning "Warning: Work in Progress" true>}}
This article is incomplete, since I'm still learning Markdown and the features of this Hugo theme.
{{</admonition>}}

The steps are as follows:

1. [Fix Network Config for IP Address Reservation](#fix-network-config-for-ip-address-reservation)
2. [Setup Cockpit](#setup-cockpit)
3. [Setup Docker](#setup-docker)
4. [Setup Portainer](#setup-portainer)

---
## Fix Network Config for IP Address Reservation 

What do I mean by this? Basically Ubuntu uses a Client-ID that is not the MAC Address when sending a DHCP request. This means that if you create an IP reservation on your DHCP server, *maybe* your Ubuntu server will receive a different IP address instead of said reservation. This depends entirely on the behavior of the DHCP server, but to prevent this issue we can force our server to use its MAC Address as the Client-ID.

To set Ubuntu to use it's MAC as the Client-ID we need to edit the network configuration file located in `/etc/netplan/`. For that we issue the command:

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

Here we ensure Ubuntu uses its MAC address when requesting DHCP, we manually set the DNS servers and make sure it uses Network Manager as the network config renderer.

### Change DNS Behavior (Optional, but needed to host Pi-Hole)

By default Ubuntu uses resolvd for DNS resolution, which means it points all DNS request to the loopback IP 127.0.0.53. This is not an issue by itself, but it means that Ubuntu also keeps listening on port 53, which doesn't allows to host any service listening on this port. If you want to host Pi-Hole (Or any DNS server) on your Ubuntu instance, then we need to change this behavior by disabling resolved:

```bash
sudo systemctl disable systemd-resolved.service && sudo systemctl stop systemd-resolved
```

Set your DNS server in `/etc/resolv.conf` to your prefered DNS server instead of 127.0.0.53.

---

## Setup Cockpit

Cockpit is a web-based GUI for management servers that typically ships with RHEL-based distributions such as Red Hat Enterprise Linux, CentOS Stream, Rocky Linux and AlmaLinux. It’s a great way to keep tabs on your servers, manage users/groups/storage/services, update software, view logs and so much more.

Although Cockpit does come pre-installed with some of the RHEL-based Linux distributions, it is not found on Ubuntu Server out of the box. Fortunately, the process for installing Cockpit on Ubuntu Server isn’t all that challenging.

Let’s do just that.

### Install Cockpit

Log into your Ubuntu Server instance and issue the command:

```bash
`sudo apt-get install cockpit -y`
```

Once the installation completes, start and enable Cockpit with:

```bash
`sudo systemctl enable --now cockpit.socket`
```

Now that Cockpit is installed and running, you can log in.

### Log into Cockpit

Open a web browser and point it to https://SERVER:9090. You should be greeted by the login screen (Figure A).

<!---This is a placeholder for an image--->
---

## Setup Docker

### Install Docker from the repository

1. Set up Docker's `apt` repository.

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

2. Install the Docker Packages:

```bash
sudo apt-get install \
  docker-ce docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

3. Verify that the Docker Engine installation is successful by running the `hello-world` image.

```bash
sudo docker run hello-world
```

### Manage Docker as a non-root user

The Docker daemon binds to a Unix socket, not a TCP port. By default it's the root user that owns the Unix socket, and other users can only access it using sudo. The Docker daemon always runs as the root user.

If you don't want to preface the docker command with sudo, create a Unix group called docker and add users to it. When the Docker daemon starts, it creates a Unix socket accessible by members of the docker group. On some Linux distributions, the system automatically creates this group when installing Docker Engine using a package manager. In that case, there is no need for you to manually create the group.

To create the `docker` group and add your user:

1. Create the `docker` group:

```bash
sudo groupadd docker
```

2. Add your user to the docker group:

```bash
sudo usermod -aG docker $USER
```

3. Log out and log back in so that your group membership is re-evaluated. You can also run the following command to activate the changes to groups:

```bash
newgrp docker
```

4. Verify that you can run `docker` commands without `sudo`:

```bash
docker run hello-world
```

### Configure Docker to start on boot with systemd

On Debian and Ubuntu, the Docker service starts on boot by default. To automatically start Docker and containerd on boot for other Linux distributions using systemd, run the following commands:

```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

---

## Setup Portainer

First, create the volume that Portainer Server will use to store its database:

```bash
docker volume create portainer_data
```

Then, download and install the Portainer Server container:

```bash
docker run -d -p 8000:8000 -p 9443:9443 \
 --name portainer --restart=always \
 -v /var/run/docker.sock:/var/run/docker.sock \
 -v portainer_data:/data portainer/portainer-ce:latest
```

Portainer Server has now been installed. You can check to see whether the Portainer Server container has started by running `docker ps`:

Now that the installation is complete, you can log into your Portainer Server instance by opening a web browser and going to:

```bash
https://localhost:9443
```

Replace localhost with the relevant IP address or FQDN if needed, and adjust the port if you changed it earlier.

You will be presented with the initial setup page for Portainer Server.