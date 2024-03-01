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
Este artículo es parte de una serie donde mostraré como tengo configurado mi Homelab. <!--more-->

Los pasos son los siguientes:

* [Editar la configuración de red para porder reservar una IP](#edit-network-config)
* [Instalar y configurar Cockpit](#setup-Cockpit)
* [Instalar y configurar Docker](#setup-docker)
* [Instalar y configurar Portainer](#setup-portainer)

---

## Editar la configuración de red para poder reservar una dirección IP {#edit-network-config}

¿Que significa esto? Básicamente Ubuntu usa un Client-ID que no es su dirección MAC cuando solicita DHCP. Esto significa que si creas una reservación para Ubuntu en tu servidor de DHCP, *lo más probable* sea que tu cliente reciba una dirección IP diferente a la que se reservó. Esto depende del comportamiento del servidor DHCP, así que para no dejarlo a la suerte podemos forzar Ubuntu a usar su dirección MAC cómo su Client-ID.

Para lograrlo, sólo debemos de editar el archivo de configuración de red ubicado en la ruta `/etc/netplan/`. Eso lo hacemos con el comando:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Vamos a editar nuestro archivo cómo sigue:

```bash
# Esta es la configuración de red escrita por 'subiquity'
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

### Instalar Cockpit

Conéctate a tu instancia de Ubuntu Server y ejecuta el siguiente comando:

```bash
`sudo apt-get install cockpit -y`
```

Una vez terminada la instalación, habilita e inicia Cockpit con el comando:

```bash
`sudo systemctl enable --now cockpit.socket`
```

Ahora ya puedes iniciar sesión en Cockpit.

### Iniciar sesión en Cockpit

Abre un navegador web y coloca la dirección https://ip_del_servidor:9090. Te debería de aparecer una pantalla parecida a la siguiente:

{{<image src="/images/cockpit-login-screen.png" caption="Pantalla de inicio de sesión de Cockpit" linked="false">}}

---

## Instalar y configurar Docker {#setup-docker}

Es perfectamente posible y valido instalar Docker desde los repositorios de Ubuntu con el comando `sudo apt install docker.io -y`. Sin embargo, prefiero instalarlo directamente de los repositorios de Docker ya que son actualizados más frecuentemente y personalmente me gusta estar al día con el software que utilizo. La siguiente sección describe este proceso.

### Instalar Docker desde su repositorio

1. Configuramos el repositorio `apt` de Docker con los siguientes comandos:

```bash
# Añadimos la llave GPG oficial de Docker:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Añadimos el repositorio a las fuentes Apt:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

2. Instalamos los paquetes de Docker:

```bash
sudo apt-get install \
  docker-ce docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

3. Verificamos que la instalación de Docker fué exitosa corriendo la imagen `hello-world` (Hola Mundo):

```bash
sudo docker run hello-world
```

### Administrar Docker sin root

El daemon de Docker se conecta a un socket Unix, no a un puerto TCP. De forma predeterminada, el usuario root es el propietario de dicho socket, y otros usuarios solo pueden acceder a él usando `sudo`. El daemon de Docker siempre se ejecuta como usuario root.

Si no quieres anteponer el comando `docker` con `sudo`, crea un grupo llamado docker y agréguele usuarios. Cuando el daemon de Docker inicia, se crea un socket Unix al que pueden acceder los miembros del grupo de Docker. En algunas distribuciones de Linux, el sistema crea automáticamente este grupo al instalar Docker mediante un administrador de paquetes. En ese caso, no es necesario crear manualmente el grupo.

Para crear el grupo `docker` y agregar tu usuario, usa los siguientes comandos:

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```
Luego cierra la sesión en inicia nuevamente para que se actualize la lista de grupos a las que pertenece el usuario, o activa los cambios con el comando:

```bash
newgrp docker
```
 Verifica que puedes correr los comandos de `docker` sin `sudo`:

```bash
docker run hello-world
```

### Configurar Docker para que inicie en el encendido con systemd

En Debian y Ubuntu, el servicio de Docker por defecto inicia con el encendido. Si por alguna razón no es tu caso, puedes iniciar Docker y containerd de manera automática en el encendido con los comandos siguientes:

```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

---

## Instalar y configurar Portainer {#setup-portainer}

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