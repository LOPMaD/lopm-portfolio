# lopm-portfolio
<img width="1103" height="912" alt="image" src="https://github.com/user-attachments/assets/c8160ee5-f026-4b79-9625-e14927671554" />

**Live:** [lopm.space](https://lopm.space)

**Stack:** Docker, GitLab CI/CD, GitLab Runner, Nginx, Prometheus, Grafana

## Overview

Static files and CI/CD for my portfolio website. The pipeline is fully set up: a single push updates the whole site, with validation checks along the way. It also runs a monitoring stack.

The infrastructure that runs this site (server provisioning, Nginx, TLS, firewall) lives in a separate repository, [infra-lopm-portfolio](https://github.com/LOPMaD/infra-lopm-portfolio). This is intentional: splitting application code and infrastructure across two repositories is common practice, so developers work in this repo while the infrastructure lives on its own.

## Architecture

```
Developer push
   │
   ▼
GitLab CI (.gitlab-ci.yml)
   │  validate Prometheus config
   │  build images (site + projects)
   │  push to GitLab Container Registry
   │  docker compose up
   ▼
Docker containers on the VPS
   ├── site (nginx + static)
   ├── odin-eye (demo project)
   └── monitoring (Prometheus, Grafana, cAdvisor, node-exporter)
```

## Prerequisites

- A VPS with Docker installed (set up via the [infrastructure repo](https://github.com/LOPMaD/infra-lopm-portfolio)).
- A registered domain.
- An SSH key for the server.
- A GitLab account with Container Registry, CI/CD variables and a registered runner.
- Git installed locally.

## Project Structure

```
.
├── .gitlab-ci.yml            # CI/CD pipeline
├── .pre-commit-config.yaml   # gitleaks as a pre-commit hook
├── docker-compose.yaml       # site, projects and monitoring stack
├── Dockerfile                # nginx image serving the static site
├── grafana/
│   └── datasource.yml        # Grafana datasource (Prometheus)
├── prometheus/
│   └── prometheus.yml        # Prometheus scrape config
└── static/
    ├── index.html
    └── style.css
```

## How It Works

The GitLab Runner that executes the pipeline runs on the VPS itself (details in the [infrastructure repo](https://github.com/LOPMaD/infra-lopm-portfolio)), so Docker is already installed there.

The pipeline steps:

1. Log in to the GitLab Container Registry via Docker.
2. Validate the Prometheus config.
3. Build images for the site and for my other projects, so their work can be shown in the portfolio. The site itself is a small nginx-based image that just serves my static files; all containers are exposed through the Nginx reverse proxy configured in the [infrastructure repo](https://github.com/LOPMaD/infra-lopm-portfolio).
4. Run `docker compose up`, which brings up the site, the portfolio projects and the full monitoring stack.

For the Grafana datasource, the Prometheus address is specified. The Grafana admin password is passed through a GitLab CI/CD variable, substituted at runtime by the runner. A plain `.env` file excluded via `.gitignore` would also work, but since everything runs on the runner, the password has to reach it on each push without being exposed in the repo, which CI/CD variables handle cleanly.

Prometheus is given its config listing the exporters it scrapes: cAdvisor and node-exporter.

- **node-exporter** collects metrics about the VPS itself, to understand load, how the server is doing, and when it might need more resources.
- **cAdvisor** collects metrics per container, to see what happened to a container, what the load situation was, and whether a container may need more resources.

## Monitoring
<img width="1523" height="729" alt="image" src="https://github.com/user-attachments/assets/bb06cc60-231e-4991-8adb-89db2037eb58" />

The monitoring stack (Grafana, Prometheus, cAdvisor, node-exporter) runs in the same compose file. Prometheus scrapes the exporters and stores the metrics, Grafana visualizes them.

## Security

- **Gitleaks** as a pre-commit hook: scans commits and blocks them if a secret is found, warning about a potential token leak.
- **Containers are kept inside the VPS** with no internet exposure; they are reachable only locally and served through the Nginx proxy.
- **Grafana password via GitLab CI/CD variable**, substituted at runtime, so the secret never lives in the repository.

## Troubleshooting

**Registry login lost when splitting jobs.** When I first set up the pipeline I wanted to split it into several jobs, but ran into a problem: the registry login was lost between them, because environment state only lives within a single job:

```
echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
```

To make registry push and pull work correctly, I moved everything into a single job.
