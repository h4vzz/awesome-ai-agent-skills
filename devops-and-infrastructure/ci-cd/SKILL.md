---
name: ci-cd
description: Set up a continuous integration and continuous delivery (CI/CD) pipeline for a software project, automating builds, tests, and deployments across environments.
license: MIT
metadata:
  author: awesome-ai-agent-skills
  version: 1.0.0
---

# CI/CD Pipeline Setup

This skill enables the agent to design, configure, and maintain CI/CD pipelines that automate the entire software delivery lifecycle. The agent can set up pipeline stages including linting, testing, building, deploying, and notifying stakeholders, ensuring that every code change is validated and delivered reliably. The agent understands secrets management, caching strategies, matrix builds, and deployment strategies such as blue/green and canary releases.

## Workflow

1. **Assess the Project and Choose a Platform:** The agent analyzes the project's language, framework, hosting environment, and team preferences to recommend a CI/CD platform. Options include GitHub Actions, GitLab CI/CD, Jenkins, CircleCI, and Azure DevOps. The agent considers factors like repository hosting, cost, plugin ecosystem, and integration with existing tools before making a recommendation.

2. **Define Pipeline Stages:** The agent structures the pipeline into discrete stages: lint (static analysis and code style), test (unit, integration, and end-to-end), build (compilation, bundling, Docker image creation), deploy (staging and production), and notify (Slack, email, or webhook alerts). Each stage has clearly defined inputs, outputs, and failure conditions so the pipeline fails fast on errors.

3. **Configure Secrets and Environment Variables:** The agent sets up secure storage for API keys, database credentials, cloud provider tokens, and other sensitive values using the platform's native secrets manager (e.g., GitHub Secrets, GitLab CI/CD Variables, or Jenkins Credentials). Secrets are never hardcoded in pipeline files and are scoped to the appropriate environment.

4. **Implement Caching and Optimization:** The agent configures dependency caching (npm, pip, Maven) and build artifact caching to reduce pipeline execution time. Matrix builds are used to test across multiple language versions or operating systems in parallel. The agent also sets up conditional execution so that expensive stages like end-to-end tests only run on relevant branches.

5. **Configure Deployment Strategies:** The agent implements the appropriate deployment strategy based on the project's risk tolerance and infrastructure. Options include rolling updates, blue/green deployments (two identical environments swapped at the load balancer), and canary releases (gradual traffic shifting). The agent also configures rollback procedures in case a deployment fails health checks.

6. **Set Up Notifications and Monitoring:** The agent configures post-pipeline notifications to inform the team of build status via Slack, Microsoft Teams, email, or custom webhooks. Deployment events are logged, and the agent can integrate with monitoring tools to verify application health after each deployment.

## Supported Technologies

- **Platforms:** GitHub Actions, GitLab CI/CD, Jenkins, CircleCI, Azure DevOps, Bitbucket Pipelines, Travis CI
- **Languages:** Node.js, Python, Java, Go, Rust, Ruby, .NET, PHP
- **Containerization:** Docker, Podman, Buildah
- **Cloud Providers:** AWS (ECS, EKS, Lambda), GCP (Cloud Run, GKE), Azure (App Service, AKS)
- **Artifact Registries:** Docker Hub, GitHub Container Registry, AWS ECR, Google Artifact Registry

## Usage

Provide the agent with your project's language, framework, repository host, target deployment environment, and any specific requirements such as testing frameworks or deployment strategies.

**Example prompt:**

```
Set up a CI/CD pipeline for my Node.js Express app hosted on GitHub.
- Run ESLint and Prettier checks, then Jest unit tests
- Build a Docker image and push to GitHub Container Registry
- Deploy to AWS ECS staging on push to develop, production on push to main
- Send Slack notifications on failure
```

## Examples

### Example 1: GitHub Actions Workflow for a Node.js Application

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run format:check

  test:
    runs-on: ubuntu-latest
    needs: lint
    strategy:
      matrix:
        node-version: [18, 20, 22]
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage
        env:
          DATABASE_URL: postgres://postgres:testpass@localhost:5432/testdb
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-${{ matrix.node-version }}
          path: coverage/

  build-and-push:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push'
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/develop'
    environment: staging
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      - run: |
          aws ecs update-service --cluster staging-cluster \
            --service my-app --force-new-deployment

  deploy-production:
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      - run: |
          aws ecs update-service --cluster production-cluster \
            --service my-app --force-new-deployment

  notify:
    runs-on: ubuntu-latest
    needs: [deploy-staging, deploy-production]
    if: always() && contains(needs.*.result, 'failure')
    steps:
      - uses: slackapi/slack-github-action@v1.25.0
        with:
          payload: |
            {"text": "Pipeline failed for ${{ github.repository }} on ${{ github.ref_name }}"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Example 2: GitLab CI Pipeline for a Python Application with Docker

```yaml
stages:
  - lint
  - test
  - build
  - deploy

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.pip-cache"
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

cache:
  paths:
    - .pip-cache/
    - .venv/

lint:
  stage: lint
  image: python:3.12-slim
  script:
    - pip install ruff mypy
    - ruff check src/
    - mypy src/ --ignore-missing-imports

test:
  stage: test
  image: python:3.12-slim
  services:
    - postgres:16
  variables:
    POSTGRES_DB: testdb
    POSTGRES_PASSWORD: testpass
    DATABASE_URL: "postgresql://postgres:testpass@postgres:5432/testdb"
  script:
    - python -m venv .venv
    - source .venv/bin/activate
    - pip install -r requirements.txt -r requirements-dev.txt
    - pytest tests/ --cov=src --cov-report=xml
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE -t $CI_REGISTRY_IMAGE:latest .
    - docker push $DOCKER_IMAGE
    - docker push $CI_REGISTRY_IMAGE:latest

deploy_production:
  stage: deploy
  image: alpine:latest
  only:
    - main
  environment:
    name: production
    url: https://myapp.example.com
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
  script:
    - ssh deploy@production-server "docker pull $DOCKER_IMAGE && docker-compose up -d"
```

## Best Practices

- **Fail fast:** Order pipeline stages so that quick checks (linting, formatting) run first, preventing wasted compute on code that will fail review anyway.
- **Pin action and image versions:** Always use specific version tags for CI actions and Docker images (e.g., `actions/checkout@v4`, `python:3.12-slim`) to ensure reproducible builds and avoid supply-chain attacks.
- **Scope secrets tightly:** Restrict secrets to the environments and branches that need them. Use environment-level protection rules to require manual approval before production deployments.
- **Cache aggressively:** Cache dependency installations, Docker layers, and build artifacts between pipeline runs. This can reduce pipeline duration by 50% or more on typical projects.
- **Use matrix builds for compatibility:** Test across multiple language versions and operating systems in parallel to catch compatibility issues early without increasing pipeline wall-clock time.
- **Implement deployment gates:** Use manual approval steps, health check verifications, or canary analysis before promoting a release to production to reduce the blast radius of bad deployments.

## Edge Cases

- **Flaky tests:** Tests that pass intermittently can block pipelines. Implement retry logic for known flaky tests and track flakiness metrics to prioritize fixes. Most CI platforms support a `retry` directive for individual jobs.
- **Monorepo pipelines:** In monorepos, changes to one service should not trigger pipelines for unrelated services. Use path-based filters (e.g., GitHub Actions `paths:` or GitLab `changes:`) to scope pipeline triggers.
- **Rate limits and quotas:** Docker Hub, npm, and cloud APIs impose rate limits. Use authenticated pulls, private mirrors, or caching proxies to avoid pipeline failures due to throttling.
- **Long-running pipelines:** Pipelines exceeding platform time limits (e.g., GitHub Actions' 6-hour job limit) need to be split into smaller jobs or use self-hosted runners with higher limits.
- **Branch protection conflicts:** If the CI pipeline writes back to the repo (e.g., auto-formatting commits), it may conflict with branch protection rules. Use a dedicated bot token or app-level authentication to push changes.
