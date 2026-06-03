# CI/CD Pipelines Reference

## Table of Contents
1. Core Concepts
2. GitHub Actions (full example)
3. GitLab CI (full example)
4. Jenkins (Declarative Pipeline)
5. Pipeline Stages Explained
6. Secrets & Environment Variables
7. Common Patterns
8. File Structure

---

## 1. Core Concepts

**CI (Continuous Integration)**: Automatically build and test code on every commit.
**CD (Continuous Delivery)**: Automatically prepare a release — human approves deploy.
**CD (Continuous Deployment)**: Fully automated — no human needed to push to production.

Pipeline stages (in order):
```
[Trigger] → [Lint] → [Build] → [Test] → [Security Scan] → [Package] → [Deploy] → [Notify]
```

---

## 2. GitHub Actions — Complete Example

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-and-test:
    name: Lint & Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4

  build-and-push:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: lint-and-test          # Only runs if lint-and-test passes
    if: github.ref == 'refs/heads/main'

    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}  # Auto-provided by GitHub

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

  deploy:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build-and-push
    environment: staging           # Requires environment protection rules

    steps:
      - name: Deploy via kubectl
        run: |
          kubectl set image deployment/app \
            app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
```

---

## 3. GitLab CI — Complete Example

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - build
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

lint:
  stage: lint
  image: node:20-alpine
  script:
    - npm ci
    - npm run lint
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - node_modules/

test:
  stage: test
  image: node:20-alpine
  script:
    - npm ci
    - npm test
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
  only:
    - main

deploy_staging:
  stage: deploy
  script:
    - kubectl set image deployment/app app=$DOCKER_IMAGE
  environment:
    name: staging
  only:
    - main
```

---

## 4. Jenkins Declarative Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        APP_IMAGE = "${DOCKER_REGISTRY}/myapp:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh 'npm ci && npm test'
            }
            post {
                always {
                    junit 'test-results/**/*.xml'
                }
            }
        }

        stage('Build Image') {
            steps {
                sh "docker build -t ${APP_IMAGE} ."
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'registry-creds',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh """
                        docker login ${DOCKER_REGISTRY} -u $USER -p $PASS
                        docker push ${APP_IMAGE}
                    """
                }
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh "kubectl set image deployment/app app=${APP_IMAGE}"
            }
        }
    }

    post {
        failure {
            slackSend channel: '#alerts', message: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
```

---

## 5. Secrets & Environment Variables

⚠️ **Never commit secrets to Git.**

| Tool | Where to store secrets |
|---|---|
| GitHub Actions | Settings → Secrets and variables → Actions |
| GitLab CI | Settings → CI/CD → Variables |
| Jenkins | Manage Jenkins → Credentials |
| Local dev | `.env` file (gitignored) |

Pattern for `.env.example`:
```bash
# Copy to .env and fill in values — never commit .env
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
API_KEY=your_api_key_here
JWT_SECRET=your_jwt_secret_here
```

---

## 6. Common Patterns

**Matrix builds** (test across multiple versions):
```yaml
strategy:
  matrix:
    node: [18, 20, 22]
    os: [ubuntu-latest, windows-latest]
```

**Manual approval gate**:
```yaml
deploy-prod:
  environment:
    name: production    # Requires reviewer approval in GitHub UI
  needs: deploy-staging
```

**Caching dependencies**:
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

---

## 7. CI/CD File Structure

```
.github/
└── workflows/
    ├── ci.yml            # Lint + test on every PR
    ├── deploy.yml        # Build + deploy on merge to main
    ├── release.yml       # Tag-based production release
    └── scheduled.yml     # Nightly jobs (cleanup, scans)

.gitlab-ci.yml            # (If using GitLab)
Jenkinsfile               # (If using Jenkins)

scripts/
├── test.sh               # Reusable test runner
├── build.sh              # Docker build helper
└── deploy.sh             # kubectl deploy helper

.env.example              # Template for required env vars
```