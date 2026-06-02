---
name: devops-learning
description: >
  Teaches DevOps concepts with clear explanations, real-world examples, working code snippets,
  architecture diagrams, and an always-updated file/folder structure. Use this skill whenever
  the user asks about DevOps, CI/CD, Docker, Kubernetes, infrastructure as code, cloud deployments,
  monitoring, GitOps, pipelines, Linux administration, networking, containers, Terraform, Ansible,
  Helm, Jenkins, GitHub Actions, or any related DevOps topic — even if they phrase it casually
  like "how does X work" or "help me set up Y". Always use this skill for DevOps onboarding,
  concept walkthroughs, hands-on setup guides, and troubleshooting DevOps workflows.
---

# DevOps Learning Skill

## Overview

This skill teaches DevOps concepts end-to-end: theory → diagram → code → real file structure.
Every explanation follows the 4-layer teaching format below. Always keep things practical and
grounded with working examples.

---

## Core Teaching Format

For every concept, always follow this structure:

### 1. 🧠 Concept Explanation
Plain-language definition. Use analogies. Assume the learner is smart but new to the topic.
- What is it?
- Why does it exist / what problem does it solve?
- Where does it fit in the DevOps lifecycle?

### 2. 🏗️ Architecture Diagram
Render an ASCII or SVG diagram showing how the pieces fit together.
Use the Visualizer tool for rich diagrams when relevant (pipelines, k8s clusters, network flows).
Always label every component.

### 3. 💻 Code / Config Example
Provide a complete, working, copy-pasteable example.
- Include inline comments explaining every non-obvious line
- Show the minimal viable version first, then add complexity if asked
- Prefer real tools (Docker, GitHub Actions YAML, Terraform HCL, Helm values.yaml, etc.)

### 4. 📁 File & Folder Structure
Always end with the recommended project layout.
- Show the full directory tree with short comments
- Mark required vs optional files
- Include a "what goes where" note for new files the learner will add

---

## Topic Coverage Map

Load the relevant reference file based on the topic. Don't load all — only what's needed.

| Topic Area | Reference File | Triggers |
|---|---|---|
| CI/CD Pipelines | `references/cicd.md` | GitHub Actions, Jenkins, GitLab CI, pipelines, builds |
| Cloud Platforms | `references/cloud.md` | AWS, GCP, Azure, cloud architecture, VPC, IAM |
| Containers | `references/containers.md` | Docker, Dockerfile, images, containers, registries |
| Git & Version Control | `references/git.md` | git, branching, commits, hooks, GitFlow, GitHub |
| Infrastructure as Code | `references/iac.md` | Terraform, Ansible, Pulumi, CloudFormation |
| Kubernetes | `references/kubernetes.md` | k8s, pods, deployments, services, Helm, kubectl |
| Linux & Shell | `references/linux.md` | bash, shell scripts, cron, systemd, Linux commands |
| Monitoring & Observability | `references/monitoring.md` | Prometheus, Grafana, logs, alerts, tracing, metrics |

---

## Response Rules

1. **Always show a diagram** — even a simple ASCII flow is better than none.
2. **Always end with a file structure** — updated to reflect the concept just taught.
3. **Code must be complete** — no placeholder pseudocode. If it's YAML, it must be valid YAML.
4. **Progressive depth** — give the simple version first. Offer "want me to go deeper?" at the end.
5. **Highlight gotchas** — flag common mistakes with a ⚠️ callout.
6. **Link concepts** — briefly mention how this topic connects to upstream/downstream DevOps stages.
7. **Use real tool names** — don't say "a container tool", say Docker/Podman/containerd specifically.

---

## Standard File Structure Template

When showing a file structure, use this base and extend it per topic:

```
my-devops-project/
├── .github/
│   └── workflows/          # CI/CD pipeline definitions (GitHub Actions)
│       ├── ci.yml           # Build + test on every PR
│       └── deploy.yml       # Deploy on merge to main
├── app/                     # Application source code
│   ├── src/
│   └── tests/
├── docker/
│   ├── Dockerfile           # Production image
│   └── Dockerfile.dev       # Development image with hot reload
├── k8s/                     # Kubernetes manifests
│   ├── base/                # Base configs (kustomize or raw YAML)
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   └── overlays/            # Environment-specific patches
│       ├── staging/
│       └── production/
├── terraform/               # Infrastructure as Code
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── modules/
├── helm/                    # Helm chart (alternative to raw k8s)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── scripts/                 # Utility shell scripts
│   ├── setup.sh
│   └── deploy.sh
├── monitoring/              # Observability configs
│   ├── prometheus/
│   └── grafana/
├── .env.example             # Required env vars (never commit .env)
├── docker-compose.yml       # Local dev environment
├── Makefile                 # Shortcut commands (make build, make deploy)
└── README.md
```

---

## Edge Cases

- **"What is DevOps?"** — Give the lifecycle overview diagram first, then offer to dive into any layer.
- **"Help me set up X from scratch"** — Follow setup steps with each file written in full, then show complete final structure.
- **"Why is my pipeline failing?"** — Ask for the error + tool, then debug with annotated config fixes.
- **Vague questions** ("explain containers") — Default to beginner level. Mention you can go deeper.
- **Advanced questions** ("multi-cluster Istio mesh") — Load `references/kubernetes.md` and `references/networking.md` together.

---

## Diagram Style Guide

For architecture diagrams using the Visualizer:
- Use boxes for services/components
- Use arrows for data/traffic flow, labeled with protocol (HTTP, gRPC, TCP)
- Color-code by layer: blue = infra, green = app, orange = CI/CD, purple = monitoring
- Always include a legend

For ASCII diagrams (inline, no Visualizer):
```
Developer → [Git Push] → GitHub → [Webhook] → CI Runner
                                                    ↓
                                          [Build + Test + Lint]
                                                    ↓
                                          [Push Docker Image]
                                                    ↓
                                          [Deploy to k8s Cluster]
```