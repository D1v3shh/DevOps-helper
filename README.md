# DevOps Helper (DevOps Learning Skill)

This repository contains a comprehensive **AI Skill** designed to teach DevOps concepts end-to-end. By utilizing this skill definition, AI assistants can explain complex DevOps topics using a standardized, highly effective 4-layer teaching format:

1. 🧠 **Concept Explanation**
2. 🏗️ **Architecture Diagram**
3. 💻 **Code / Config Example**
4. 📁 **File & Folder Structure**

## 📂 Project Structure

- `SKILL.md`: The core system prompt and instruction set for the AI agent. It defines the teaching format, response rules, and standard templates.
- `references/`: A collection of markdown files providing detailed knowledge on specific DevOps topics. The AI can load these dynamically based on the user's questions.

## 📚 Topics Covered

The `references/` directory includes deep-dive material on the following areas:

- **CI/CD Pipelines** (`cicd.md`): GitHub Actions, Jenkins, GitLab CI, pipelines, builds.
- **Cloud Platforms** (`cloud.md`): AWS, GCP, Azure, cloud architecture, VPC, IAM.
- **Containers** (`containers.md`): Docker, Dockerfile, images, containers, registries.
- **Git & Version Control** (`git.md`): git, branching, commits, hooks, GitFlow, GitHub.
- **GitOps** (`gitops.md`): Infrastructure and application deployment via Git repositories.
- **Infrastructure as Code (IaC)** (`iac.md`): Terraform, Ansible, Pulumi, CloudFormation.
- **Kubernetes** (`kubernetes.md`): k8s, pods, deployments, services, Helm, kubectl.
- **Linux & Shell** (`linux.md`): bash, shell scripts, cron, systemd, Linux commands.
- **Monitoring & Observability** (`monitoring.md`): Prometheus, Grafana, logs, alerts, tracing, metrics.

## 🚀 How to Use

1. Provide the contents of `SKILL.md` to your AI agent as its core system prompt or persona definition.
2. Based on what topic you are learning or troubleshooting, ensure the agent has access to the relevant reference file in the `references/` folder.
3. Ask the AI questions like:
   - *"How does Kubernetes work?"*
   - *"Help me set up a GitHub Actions CI pipeline."*
   - *"Explain Docker to me like I'm a beginner."*
   - *"Why is my Terraform plan failing?"*

The AI will respond with clear explanations, text-based architecture diagrams, complete working code, and the recommended file structures.
