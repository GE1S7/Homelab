# Homelab

## Description

This repository contains Infrastructure as Code (IaC) configuration 
for my home server running containerized services managed via Docker 
Compose. It also documents my process of setting up a one-node 
Kubernetes cluster using Minikube.

## Hardware
- ASUS X550VB
- CPU: Intel i5-3230M (4) @ 3.200GHz
- GPU: NVIDIA GeForce GT 740M 
- RAM: 8GB

## Software

- OS: Ubuntu Server
- Containerization: Docker, Docker Compose
- Automation: Cronjobs, Bash, Python
- Remote Access: Tailscale VPN for secure SSH
- Minikube (in progress)

## Services
### AdGuard Home
DNS server with network-wide ad blocking

### Media Center (mediactr)
Automated media management pipeline:
- **yt-dlp**: Video downloads
- **Transmission**: BitTorrent client
- **Jellyfin**: Media streaming server
- **letterboxd-stats**: Scrobbling integration
- **Automation**: Bash scripts + cronjobs for automatic processing

### PostgreSQL
Database for backend projects with persistent storage

## Kubernetes
Currently learning Kubernetes and setting up a one-node cluster 
with Minikube. Plan to migrate existing Docker Compose services 
to K8s deployments.



