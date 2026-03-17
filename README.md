# Engineering Journal

> How I think, build, and make decisions — documented as I go.

I'm a software developer transitioning into Cloud/DevOps engineering. This repo captures my engineering decisions, project write-ups, and learning journey. It exists because I believe **showing how you think** matters as much as showing what you build.

---

## Projects

### [FTIP — Fundamental Trading Intelligence Platform](projects/ftip/)
Multi-agent AI trading research desk. Real macro data from FRED + Alpha Vantage, FastAPI REST API, Redis Streams message bus, AI agents for currency scoring.
- **Repo:** [github.com/emmanuelhiss/ftip](https://github.com/emmanuelhiss/ftip)
- **Status:** Active development

### [NemonixCentral — Trading Analysis Platform](projects/nemonixcentral/)
Financial macro analysis terminal with AWS Bedrock AI integration. 17+ API endpoints processing real US Treasury data. Built for personal trading use.
- **Repo:** Private (proprietary trading logic — demo on request)
- **Status:** Evolved into FTIP

### [NexOps / devops-learning — Infrastructure Platform](projects/nexops/)
Hybrid infrastructure management built on homelab. Proxmox, Docker, OPNsense firewall, WireGuard VPN, network segmentation.
- **Repo:** [github.com/emmanuelhiss/devops-learning](https://github.com/emmanuelhiss/devops-learning)
- **Status:** Foundation skills carried into FTIP

### [MASE — Multi Agent Systems Engineering](projects/mase/)
Multi-agent AI platform built collaboratively. Google ADK, Neo4j, Google Cloud Run.
- **Status:** Ongoing team project

---

## Architecture Decisions

ADRs (Architecture Decision Records) explain **why** I chose specific tools and patterns. Each one is a short document: context, decision, reasoning.

| ADR | Project | Decision |
|-----|---------|----------|
| [001](decisions/001-why-fastapi.md) | FTIP | Why FastAPI over Django/Flask |
| [002](decisions/002-why-redis-streams.md) | FTIP | Why Redis Streams over Celery/RabbitMQ |
| [003](decisions/003-why-separate-ingestion.md) | FTIP | Why ingestion is a separate service from API |
| [004](decisions/004-why-uv.md) | FTIP | Why uv over pip/poetry |
| [005](decisions/005-why-async-everything.md) | FTIP | Why async across the entire stack |
| [006](decisions/006-why-multi-stage-docker.md) | NexOps/FTIP | Why multi-stage Docker builds |

---

## About Me

Emmanuel — Melbourne-based software developer building toward Cloud/DevOps engineering roles. I learn by building real systems, not following tutorials.

- **GitHub:** [github.com/emmanuelhiss](https://github.com/emmanuelhiss)
- **Target:** DevOps / Cloud / Platform Engineering — Australia
