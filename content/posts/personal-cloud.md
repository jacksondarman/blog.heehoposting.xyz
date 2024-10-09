---
marp: true
---

+++
title = 'Personal Cloud'
date = 2024-10-08T11:07:20-04:00
draft = true
+++
# A Guide on Setting Up your Own File Server
## Introduction

Setting up a server might sound daunting, especially when you hear terms like "containerization," "Kubernetes/K8s," and "Docker." But don’t worry—owning and maintaining a server is easier than it sounds. In this guide, I’ll show you how to set up a fully functional Dropbox replacement using **Nextcloud**.

*Disclaimer*: This guide assumes that you have basic knowledge of how to use Linux terminal commands. If you need some help, [here's a quick cheatsheet](https://www.geeksforgeeks.org/linux-commands-cheat-sheet/).

## Preliminary Information

### What is Nextcloud?
Nextcloud is a powerful, free, and open-source solution for file storage and sharing, often used as a Google Drive or Dropbox replacement. If you want more control over your data without relying on big tech, Nextcloud is an excellent option.

Nextcloud can also be extended with various plugins from its marketplace. You can explore these [here](https://github.com/andreasjacobsen93/awesome-nextcloud). For now, I’ll focus on the file storage and sharing features, but feel free to explore—it’s your file server, after all!


### Servers: What are they and how do I get one?

To host something like Nextcloud, you need a server. But where do you start?

The key point: A server is defined by its role, not its hardware. It's simply a computer designed to serve data to a client. This means almost any computer you own can act as a server, whether it's a dedicated server from eBay[like this one](https://www.ebay.com/itm/115818567694?itmmeta=01J9PBEQ8E51X5QEEGDDVEC1DC&hash=item1af753180e:g:HgEAAOSwQh5k1YNe&itmprp=enc%3AAQAJAAAA4Mxmj%2BiGvOveHXEBClPb29gJukUG%2FflZbxcv0TMCvxAl3XvyZfd%2BRo30M37skgqzsM2mZYi0QZHgXQlf9VFpFBYSmXpvEPbZIcUQRUIpbYaRSAHxHn3uukGQlOosxVyFYhGXR5F1%2F%2BS%2FoGyA0XjJMhQJ4%2Fd8MrxbuIbTBVNQV0qHE9Iz9kcnGUPYR7uGM1gzniiYLzHVMWZoJRg6LyB%2B8HpGUcMLVzDRPpKrV2YcLth2VyvknQxObDRD30RUsXPGKo7isTbmX0H6p2bzLF9e3975EtUDzqXkAD7VtlA5mVlp%7Ctkp%3ABFBMqPS6y81k), or an old laptop gathering dust in your closet.

If you don’t have hardware ready, you can rent server space through companies like **Digital Ocean**, which offers a special deal for students through GitHub Education in which you get $100 of free credit. Regardless of the hardware or service, the following steps will be the same.

## Getting Started

### Installing your Operating System
Once you have your hardware, you need to pick an operating system to use as the base of your server. For the sake of this guide, I will recommend the latest LTS version of [Ubuntu Server](https://ubuntu.com/download/server), but realistically, most of the steps outlined here can work with any Linux distribution. Be sure to check the documentation of the solutions I've listed here for your specific usecase if you choose to use another distribution.

If you're using your own hardware, you need to create a bootable disk from the Ubuntu ISO. I recommend using something like [Balena Etcher](https://etcher.balena.io/).

### Connecting to your server remotely
Now that you have Ubuntu server up and running, you should attempt to connect to it remotely using SSH. If you're running your server locally, type `ip a` into your server's terminal to find it's internal IP address (it should start with something like `inet XXX.XXX.XXX.XXX`, and not be the `lo` (loopback) interface).

If you opted to use a VPS, use the public IP address that they gave to you.

To connect to the server using SSH, input the following into your terminal from another machine.

```bash
ssh username@ip-address
```

You should now be successfully connected to your server using SSH.

*Pro-tip: I highly recommend following the steps highlighted by DigitalOcean [here](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu) in order to ensure that your server is following security best practices. At the very least, you should not be using the root user to connect to your server.*

### Installing Docker and Docker Compose
In order to run Nextcloud, we're going to be using a containerization solution called Docker. Essentially, Docker isolates services that you run on your server into something called "containers", that basically act like their own mini-computers within your server. It's kind of like you have several smaller computers within your large server (make no mistake though, these containers can be very powerful).

In order to streamline running these Docker containers, we're going to use something called Docker Compose. Docker Compose allows you to define settings for the containers you want to run in a single config file, allowing for much easier deployment.

First, set up Docker's `apt` repository.

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
Then, install the latest version of Docker and Docker Compose.

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Finally, run these commands to ensure that everything is installed correctly.

```bash
docker -v #Checks the Docker version. If you see an output here, you installed it correctly.
docker compose #Checks to see if Docker Compose is installed. If it is, you'll see output.
sudo docker run hello-world #Spins up a sample "Hello world" Docker container to verify the runtime works.
```

By default, Docker is only runnable by the root user (or by using `sudo`). If this is not desired behavior for your environment, add your user account to the Docker group.

```bash
sudo usermod -aG docker $USER
```

Once everything looks good, you're free to move on through the guide!

## Setting Up the Nextcloud Container

We will be using Nextcloud's AIO image in order to spin up everything we need in one file. With Nextcloud AIO you get:

- Nextcloud's base features, including file storage and file transfer.
- Nextcloud Office, a replacement to Google Office/O365.
- Nextcloud Talk, a voice chat solution to replace something like Zoom.

Create a directory called `nextcloud` to store your config files, and then make a `docker-compose.yml` file.

```bash
mkdir nextcloud && cd nextcloud
touch docker-compose.yml
```

Here is a sample `docker-compose.yml` to get you started, directly from Nextcloud's GitHub:

```yml
services:
  nextcloud-aio-mastercontainer:
    image: nextcloud/all-in-one:latest
    init: true
    restart: always
    container_name: nextcloud-aio-mastercontainer # This line is not allowed to be changed as otherwise AIO will not work correctly
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config # This line is not allowed to be changed as otherwise the built-in backup solution will not work
      - /var/run/docker.sock:/var/run/docker.sock:ro # May be changed on macOS, Windows or docker rootless. See the applicable documentation. If adjusting, don't forget to also set 'WATCHTOWER_DOCKER_SOCKET_PATH'!
    network_mode: bridge # add to the same network as docker run would do
    ports:
      - 80:80 # Can be removed when running behind a web server or reverse proxy (like Apache, Nginx, Caddy, Cloudflare Tunnel and else). See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
      - 8080:8080
      - 8443:8443 # Can be removed when running behind a web server or reverse proxy (like Apache, Nginx, Caddy, Cloudflare Tunnel and else). See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
    # environment: # Is needed when using any of the options below
      # AIO_DISABLE_BACKUP_SECTION: false # Setting this to true allows to hide the backup section in the AIO interface. See https://github.com/nextcloud/all-in-one#how-to-disable-the-backup-section
      # APACHE_PORT: 11000 # Is needed when running behind a web server or reverse proxy (like Apache, Nginx, Caddy, Cloudflare Tunnel and else). See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
      # APACHE_IP_BINDING: 127.0.0.1 # Should be set when running behind a web server or reverse proxy (like Apache, Nginx, Caddy, Cloudflare Tunnel and else) that is running on the same host. See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
      # BORG_RETENTION_POLICY: --keep-within=7d --keep-weekly=4 --keep-monthly=6 # Allows to adjust borgs retention policy. See https://github.com/nextcloud/all-in-one#how-to-adjust-borgs-retention-policy
      # COLLABORA_SECCOMP_DISABLED: false # Setting this to true allows to disable Collabora's Seccomp feature. See https://github.com/nextcloud/all-in-one#how-to-disable-collaboras-seccomp-feature
      # NEXTCLOUD_DATADIR: /mnt/ncdata # Allows to set the host directory for Nextcloud's datadir. ⚠️⚠️⚠️ Warning: do not set or adjust this value after the initial Nextcloud installation is done! See https://github.com/nextcloud/all-in-one#how-to-change-the-default-location-of-nextclouds-datadir
      # NEXTCLOUD_MOUNT: /mnt/ # Allows the Nextcloud container to access the chosen directory on the host. See https://github.com/nextcloud/all-in-one#how-to-allow-the-nextcloud-container-to-access-directories-on-the-host
      # NEXTCLOUD_UPLOAD_LIMIT: 10G # Can be adjusted if you need more. See https://github.com/nextcloud/all-in-one#how-to-adjust-the-upload-limit-for-nextcloud
      # NEXTCLOUD_MAX_TIME: 3600 # Can be adjusted if you need more. See https://github.com/nextcloud/all-in-one#how-to-adjust-the-max-execution-time-for-nextcloud
      # NEXTCLOUD_MEMORY_LIMIT: 512M # Can be adjusted if you need more. See https://github.com/nextcloud/all-in-one#how-to-adjust-the-php-memory-limit-for-nextcloud
      # NEXTCLOUD_TRUSTED_CACERTS_DIR: /path/to/my/cacerts # CA certificates in this directory will be trusted by the OS of the nexcloud container (Useful e.g. for LDAPS) See See https://github.com/nextcloud/all-in-one#how-to-trust-user-defined-certification-authorities-ca
      # NEXTCLOUD_STARTUP_APPS: deck twofactor_totp tasks calendar contacts notes # Allows to modify the Nextcloud apps that are installed on starting AIO the first time. See https://github.com/nextcloud/all-in-one#how-to-change-the-nextcloud-apps-that-are-installed-on-the-first-startup
      # NEXTCLOUD_ADDITIONAL_APKS: imagemagick # This allows to add additional packages to the Nextcloud container permanently. Default is imagemagick but can be overwritten by modifying this value. See https://github.com/nextcloud/all-in-one#how-to-add-os-packages-permanently-to-the-nextcloud-container
      # NEXTCLOUD_ADDITIONAL_PHP_EXTENSIONS: imagick # This allows to add additional php extensions to the Nextcloud container permanently. Default is imagick but can be overwritten by modifying this value. See https://github.com/nextcloud/all-in-one#how-to-add-php-extensions-permanently-to-the-nextcloud-container
      # NEXTCLOUD_ENABLE_DRI_DEVICE: true # This allows to enable the /dev/dri device in the Nextcloud container. ⚠️⚠️⚠️ Warning: this only works if the '/dev/dri' device is present on the host! If it should not exist on your host, don't set this to true as otherwise the Nextcloud container will fail to start! See https://github.com/nextcloud/all-in-one#how-to-enable-hardware-transcoding-for-nextcloud
      # NEXTCLOUD_KEEP_DISABLED_APPS: false # Setting this to true will keep Nextcloud apps that are disabled in the AIO interface and not uninstall them if they should be installed. See https://github.com/nextcloud/all-in-one#how-to-keep-disabled-apps
      # TALK_PORT: 3478 # This allows to adjust the port that the talk container is using. See https://github.com/nextcloud/all-in-one#how-to-adjust-the-talk-port
      # WATCHTOWER_DOCKER_SOCKET_PATH: /var/run/docker.sock # Needs to be specified if the docker socket on the host is not located in the default '/var/run/docker.sock'. Otherwise mastercontainer updates will fail. For macos it needs to be '/var/run/docker.sock'
    # security_opt: ["label:disable"] # Is needed when using SELinux

  # # Optional: Caddy reverse proxy. See https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
  # # You can find further examples here: https://github.com/nextcloud/all-in-one/discussions/588
  # caddy:
  #   image: caddy:alpine
  #   restart: always
  #   container_name: caddy
  #   volumes:
  #     - ./Caddyfile:/etc/caddy/Caddyfile
  #     - ./certs:/certs
  #     - ./config:/config
  #     - ./data:/data
  #     - ./sites:/srv
  #   network_mode: "host"

volumes: # If you want to store the data on a different drive, see https://github.com/nextcloud/all-in-one#how-to-store-the-filesinstallation-on-a-separate-drive
  nextcloud_aio_mastercontainer:
    name: nextcloud_aio_mastercontainer # This line is not allowed to be changed as otherwise the built-in backup solution will not work

```


### A Brief Word on Port Forwarding

Port forwarding is a very messy topic to explain. Essentially, you are exposing ports on your home network so that you can access your Nextcloud instance remotely (which is desired behavior for a cloud environment). I will not be going into depth on how to expose ports to the Internet, as that varies so wildly depending on what router you have.

In general, you want to have the following ports exposed, or behind a reverse proxy (preferred).

```
# These ports are necessary to connect to the web portal
80/TCP, 80/UDP 
8443/TCP, 8443/UDP

```

