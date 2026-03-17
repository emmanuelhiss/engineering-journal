# NexOps / devops-learning — Infrastructure Platform

## One-Liner
> Hybrid infrastructure management platform built on a homelab with Proxmox, Docker, OPNsense firewall, and WireGuard VPN. Defence-in-depth networking with three isolated network segments.

## What It Demonstrates
- Network segmentation (3 isolated networks: home, lab, VPN)
- OPNsense firewall configuration and traffic control
- WireGuard VPN for secure remote access
- Docker multi-stage builds with non-root containers
- Docker Compose orchestration with healthchecks
- Proxmox API integration with least-privilege service accounts

## Key Interview Talking Points
- "I built a production-grade network with defence-in-depth — OPNsense for segmentation, WireGuard for admin access, Cloudflare Tunnel for public services. Nothing is directly exposed."
- "All containers run as non-root users. When I hit a Celery Beat permission error, I traced it to the PID file path and fixed it by understanding Linux file permissions."
- "My Proxmox service account has read-only API access — principle of least privilege."

## Why It Paused
The Australian job market values cloud experience (AWS/Azure) over homelab-only projects. The infrastructure skills carry forward into FTIP, which adds cloud deployment, CI/CD, and AI.

## Links
- **Repo:** [github.com/emmanuelhiss/devops-learning](https://github.com/emmanuelhiss/devops-learning)
