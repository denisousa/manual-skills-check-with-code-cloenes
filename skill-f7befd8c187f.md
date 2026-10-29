<!-- sawrus/agent-guides:areas/devops/ci-cd/skills/gitlab-ci-patterns/SKILL.md | https://github.com/sawrus/agent-guides/blob/HEAD/areas/devops/ci-cd/skills/gitlab-ci-patterns/SKILL.md -->

---
name: gitlab-ci-patterns
type: skill
description: GitLab CI/CD pipelines — include templates, environments, OIDC auth, caching, protected runners, deployment gates.
related-rules:
  - pipeline-standards.md
  - quality-gates.md
allowed-tools: Read, Write, Edit
---

# Skill: GitLab CI Patterns

> **Expertise:** GitLab CI YAML, include/extends, environments, DAST, protected runners, Kubernetes deploy.

## When to load

When creating or reviewing `.gitlab-ci.yml` files for build, test, or deployment pipelines.

## Standard Pipeline Structure

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - build
  - scan
  - deploy-staging
  - smoke-test
  - deploy-production

variables:
  IMAGE_NAME: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  DOCKER_BUILDKIT: "1"

# ── Validate ───────────────────────────────────────────
lint:
  stage: validate
  image: python:3.12-slim
  cache:
    key: pip-$CI_COMMIT_REF_SLUG
    paths: [.cache/pip]
  script:
    - pip install ruff mypy --cache-dir .cache/pip
    - ruff check src/ tests/
    - mypy src/ --strict

test:
  stage: validate
  image: python:3.12-slim
  cache:
    key: pip-$CI_COMMIT_REF_SLUG
    paths: [.cache/pip]
  script:
    - pip install -r requirements.txt -r requirements-dev.txt --cache-dir .cache/pip
    - pytest tests/ --cov=src --cov-report=xml --cov-fail-under=80
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

# ── Build ──────────────────────────────────────────────
build-image:
  stage: build
  image: docker:24
  services: [docker:24-dind]
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build --cache-from $CI_REGISTRY_IMAGE:cache
        --build-arg BUILDKIT_INLINE_CACHE=1
        -t $IMAGE_NAME
        -t $CI_REGISTRY_IMAGE:cache .
    - docker push $IMAGE_NAME
    - docker push $CI_REGISTRY_IMAGE:cache
  only: [main, tags]

# ── Scan ───────────────────────────────────────────────
container-scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE_NAME
  needs: [build-image]

sast:
  stage: scan
  include:
    - template: Security/SAST.gitlab-ci.yml

# ── Deploy Staging ─────────────────────────────────────
deploy-staging:
  stage: deploy-staging
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - helm upgrade --install my-service charts/my-service
        --set image.tag=$CI_COMMIT_SHA
        --namespace staging
        --atomic --timeout 5m
  only: [main]

# ── Smoke Test ─────────────────────────────────────────
smoke-staging:
  stage: smoke-test
  script:
    - curl -f https://staging.example.com/health
  needs: [deploy-staging]
  only: [main]

# ── Deploy Production ──────────────────────────────────
deploy-production:
  stage: deploy-production
  environment:
    name: production
    url: https://app.example.com
  when: manual                         # manual approval gate
  allow_failure: false
  script:
    - helm upgrade --install my-service charts/my-service
        --set image.tag=$CI_COMMIT_SHA
        --namespace production
        --atomic --timeout 5m
  only: [main]
  needs: [smoke-staging]
```

## Include & Extends (DRY pipelines)

```yaml
# Shared templates in infra repo
include:
  - project: 'infra/ci-templates'
    file: '/templates/docker-build.yml'
    ref: v1.2.0
  - template: Security/SAST.gitlab-ci.yml

# Extend base job
.base-deploy:
  image: bitnami/helm:3
  before_script:
    - echo $KUBECONFIG_B64 | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig

deploy-staging:
  extends: .base-deploy
  environment: staging
  script: helm upgrade --install ...
```

## Protected Runners (bare-metal / internal registry)

```yaml
# Tag jobs to run on specific runners
build-internal:
  tags:
    - self-hosted
    - bare-metal
    - docker
  script: ...
```

Configure in GitLab → Settings → CI/CD → Runners:
- Protected runners only run on protected branches (main, tags)
- Untagged jobs run on shared runners only
