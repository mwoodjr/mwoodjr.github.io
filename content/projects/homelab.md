---
title: "Raspberry Pi Homelab"
date: 2026-11-12T00:00:00-06:00
draft: false
author: "Miguel Wood Jr"
description: "Two Raspberry Pis running media, networking, and home automation, with a tested recovery path for Docker services that share a VPN."
---

I built this Linux homelab to run services I use at home and learn how to recover them when something breaks. A Raspberry Pi 5 handles media, storage, and monitoring. A Raspberry Pi 4 handles DNS, web routing, and home automation. Splitting the work across two hosts lets me maintain the media stack while keeping DNS and home automation available.

## Architecture

{{< homelab-map >}}

Pi-hole handles DNS and DHCP on the home network. For remote access, Tailscale routes my lab's DNS queries to Pi-hole while leaving ordinary lookups with the remote device's usual resolver. Nginx Proxy Manager routes HTTPS requests to the right web app, so DNS resolution and web routing are separate steps.

## What runs on it

### Media and storage · Pi 5

- **Media and automation.** Docker Compose manages the playback, library automation, and activity monitoring services.
- **Storage and file sharing.** An 8 TB drive holds the library. Containers access it through mounted directories, and local devices use a network share.
- **VPN networking.** Four containers share a network namespace with a dedicated VPN container. That centralizes the connection, but replacing the VPN container means the dependent apps must rejoin its network.

### Networking and automation · Pi 4

- **Pi-hole, Nginx Proxy Manager, and Tailscale** handle local DNS and DHCP, HTTPS routing, and remote access. Tailscale runs on both hosts.
- **Home Assistant and Mosquitto** handle automations and MQTT messages. **AirConnect** makes compatible network speakers available as AirPlay targets.
- **Glance** gives me a single dashboard for the services I use most, and **code-server** gives me a browser-based editor.

### Monitoring and recovery · Both hosts

- **Portainer** manages Docker on both Pis from the Pi 5 dashboard. **Beszel** agents collect host and container metrics on both machines and report to a central dashboard.
- I keep Docker Compose configuration in Git and pin deployed images by digest so a restart uses the same image. Both hosts make scheduled configuration backups and exchange copies. I rehearsed an isolated restore and checked the recovered databases and critical configuration files. Those backups cover application state and settings. The media library is outside their scope.

## Troubleshooting a VPN recovery

**What broke.** The replacement VPN container could connect, but four apps still shared the old network namespace. A healthy VPN indicator alone gave me an incomplete picture of whether the apps could recover.

**How I isolated it.** I compared the network namespace of each app with the namespace of the replacement VPN container. The mismatch showed why a healthy VPN could coexist with unreachable apps.

**What I changed.** I set up a systemd timer to check once a minute. Once the VPN container is healthy, it recreates running apps that are still attached to the old namespace. It skips apps that were intentionally stopped.

**How I verified it.** I replaced the VPN container during a controlled test. Within a minute of the VPN reconnecting, the timer recreated all four apps. I verified that all four shared the new namespace and that every HTTP endpoint responded. The VPN firewall stayed enabled throughout the test.
