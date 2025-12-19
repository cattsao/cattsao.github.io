---
layout: page
title: "Project 3- WireGuard Lab"
permalink: /wireguard
---

# Project 3- Wireguard Lab


**Step 1: Create a Digital Ocean Account**
I set up an account using https://m.do.co/c/d33d59113ab6.

**Step 2: Create an Ubuntu 24.04 Droplet**
I selected the Region and Datacenter closest to me, San Francisco.

I chose an Ubuntu 24.04 Image.

I selected a Basic Droplet Type, a Regular CPU with a regular SSD, and the $24/mo plan.

To authenticate, I chose to use a password.

**Step 3: Install Docker**
I installed Docker using the command `curl -sSL https://get.docker.com | sh`. 

**Step 4: Create Directory for Docker Compose File**
I created a directory to store the Docker Compose file to run WireGuard using `sudo mkdir -p /opt/stacks/wireguard` and changed directories into it.

**Step 5: Generate a WG-Easy Password Hash**
I wanted to use the WG-Easy container to manage my VPN through a web interface. To generate a hash for the password I intend to use to access the WG-Easy web interface, I ran the command `docker run --rm -it ghcr.io/wg-easy/wg-easy wgpw '<PASSWORD>'`, replacing `<PASSWORD>` with an actual password I created.

**Step 6: Create the Docker Compose File**
With nano, I wrote the `compose.yaml` file. I wrote in 
`services:`
`  wg-easy:`
   ` container_name: wg-easy`
    `image: ghcr.io/wg-easy/wg-easy`

    `environment:`
      `- <PASSWORD_HASH>`
      `- WG_HOST=<IPADDRESS>`

    `volumes:`
      `- ./config:/etc/wireguard`
      `- /lib/modules:/lib/modules`
    `ports:`
      `- "51820:51820/udp"`
      `- "51821:51821/tcp"`
    `restart: unless-stopped`
    `cap_add:`
      `- NET_ADMIN`
      `- SYS_MODULE`
    `sysctls:`
      `- net.ipv4.ip_forward=1`
      `- net.ipv4.conf.all.src_valid_mark=1`
In the place of `<PASSWORD_HASH>` I put my WG-Easy password hash with an extra $ after each instance of $. In the place of `<IPADDRESS>` I put the external IP address of my Ubuntu image.

**Step 8: Start up the WireGuard Container**
To start up WireGuard, I ran `docker compose up -d`.

**Step 9: Access the WireGuard Docker Container Web Interface**
To access the WG-Easy web interface, I went to `http://<IPADDRESS>:51821`, replacing `<IPADDRESS>` with the IP address of my Ubuntu image.

I logged into the web interface using the password I set earlier. Then I created a new client and named it.

**Testing on Mobile Device**
I downloaded the WireGuard app on my phone. Then I added a tunnel using the `Create from QR code` option. I scanned the QR code of my client.

Before I activated the VPN tunnel, my IP address was:
![[IMG_2094.jpeg]]

Once I activated it on the WireGuard application interface,
![[IMG_2096 1.jpeg]]

my IP address was:
![[IMG_2095.jpeg]]

**Testing on Laptop**
I downloaded the configuration files for my VPN. I installed WireGuard on my MacBook. Then I imported a tunnel from the configuration files.

Before I activated the VPN tunnel, my IP address was:
![[Screenshot 2025-12-18 at 6.55.54 PM.png]]

Once I activated it on the WireGuard application interface,
![[Screenshot 2025-12-18 at 7.04.28 PM 1.png]]

my IP address was:
![[Screenshot 2025-12-18 at 7.09.32 PM.png]]
