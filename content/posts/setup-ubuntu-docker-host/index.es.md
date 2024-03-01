---
title: "Cómo configurar Ubuntu como un host de Docker"
date: 2024-02-16T02:35:45.517Z
lastmod: 
draft: false
author: "Bersayder"
authorLink: "https://netsyder.com"
description: "Primero de una serie de artículos sobre mi Homelab sin ningún orden en particular"
images: []
resources:
- name: "featured-image"
  src: "Ubuntu_Docker.png"

tags: ["Home Lab", "Linux", "Ubuntu", "Docker", "Portainer","Cockpit"]
categories: ["Home Lab"]

lightgallery: true
---
Este artículo es parte de una serie donde mostraré como tengo configurado mi Homelab. <!--more--> Esta serie no está en ningún orden en particular, así que vamos allá!

{{<admonition warning "Nota: Trabajo en progreso" true>}}
Este artículo esta incompleto, ya que todavía estoy aprendiendo a usar Markdown y las características de esta plantilla de Hugo.
{{</admonition>}}

Los pasos son los siguientes:

1. [Editar la configuración de red para porder reservar una IP](#edit-network-config)
2. [Instalar y configurar Cockpit](#setup-Cockpit)
3. [Instalar y configurar Docker](#setup-docker)
4. [Instalar y configurar Portainer](#setup-portainer)

---

## Editar la configuración de red para poder reservar una dirección IP {#edit-network-config}

¿Que significa esto? Básicamente Ubuntu usa un Client-ID que no es su dirección MAC cuando solicita DHCP. Esto significa que si creas una reservación para Ubuntu en tu servidor de DHCP, *lo más probable* sea que tu cliente reciba una dirección IP diferente a la que se reservó. Esto depende del comportamiento del servidor DHCP, así que para no dejarlo a la suerte podemos forzar Ubuntu a usar su dirección MAC cómo su Client-ID.

Para lograrlo, sólo debemos de editar el archivo de configuración de red ubicado en la ruta `/etc/netplan/`. Eso lo hacemos con el comando:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Vamos a editar nuestro archivo cómo sigue:

```bash
# This is the network config written by 'subiquity'
network:
  ethernets:
    eth0:
      dhcp4: true
      dhcp-identifier: mac
      nameservers:
        addresses: [1.1.1.1,8.8.8.8,9.9.9.9]
  version: 2
  renderer: NetworkManager
```

Aquí la tarjeta de red aparece cómo `eth0` (puede ser diferente en tu caso), y nos aseguramos de que Ubuntu use su MAC como su Client-ID al agregar la línea `dhcp-identifier: mac`. También agregamos nuestros servidores DNS favoritos con el bloque `nameservers` y nos aseguramos que use Network Manager como su renderizador de red por defecto con `renderer: NetworkManager`. Por favor ten en cuenta la jerarquía de las sangrías.

### Cambiar el comportamiento de DNS (Opcional, pero necesario si vas a instalar Pi-Hole)

De forma predeterminada, Ubuntu usa resolvd para la resolución de DNS, lo que significa que apunta todas las solicitudes de DNS a la IP 127.0.0.53. Esto no es un problema en sí mismo, pero significa que Ubuntu se mantiene escuchando en el puerto 53, lo que impide que ningún servicio pueda escuchar en este puerto. Si deseas instalar Pi-Hole (o cualquier otro servidor DNS) en tu instancia de Ubuntu, entonces necesitas cambiar este comportamiento deshabilitando resolvd:

```bash
sudo systemctl disable systemd-resolved.service && sudo systemctl stop systemd-resolved
```

También debes de especificar tu servidor de DNS preferido en el archivo `/etc/resolv.conf` en vez de 127.0.0.53.

---

## Instalar y configurar Cockpit {#setup-Cockpit}

Cockpit es una interfaz usuario que puedes abrir en tu navegador para administrar servidores Linux, y normalmente viene instalado por defecto en distribuciones basadas en RHEL, como Red Hat Enterprise Linux, CentOS Stream, Rocky Linux y AlmaLinux. Es una excelente manera de controlar sus servidores, administrar usuarios/grupos/almacenamiento/servicios, actualizar software, ver registros y mucho más.

Lamentablemente Cockpit no viene instalado en Ubuntu Server. Afortunadamente, el proceso de instalación de Cockpit en Ubuntu Server no es tan difícil, y vamos a hacer justamente eso.

### Install Cockpit

Conéctate a tu instancia de Ubuntu Server y ejecuta el siguiente comando:

```bash
`sudo apt-get install cockpit -y`
```

Una vez terminada la instalación, habilita e inicia Cockpit con el comando:

```bash
`sudo systemctl enable --now cockpit.socket`
```

Ahora ya puedes inicia sesióne en Cockpit.

### Iniciar sesión en Cockpit

Abre un navegador web y coloca la dirección https://<IP de tu servidor>:9090. Te debería de aparecer una pantalla parecida a la siguiente:

{{<image src="/images/cockpit-login-screen.png" caption="Pantalla de inicio de sesión de Cockpit" linked="false">}}

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