<!-- freetison/kilocode-toolbox:skills/kubernetes-gitops/SKILL.md | https://github.com/freetison/kilocode-toolbox/blob/HEAD/skills/kubernetes-gitops/SKILL.md -->

---
name: "kubernetes-gitops"
description: "GitOps with ArgoCD and Flux: Application CRD, sync policies, multi-cluster, reconciliation, kustomize overlays"
mode: all
permission:
  edit: allow
  bash: allow
---

# Kubernetes GitOps

Practical guide for GitOps workflows with ArgoCD and Flux.

## ArgoCD Setup

### Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### ArgoCD Helm Install

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace \
  --set server.extraArgs[0]="--insecure" \
  --set configs.params."server\.rbac\.policy\.default"="role:readonly"
```

## ArgoCD Application CRD

### Basic Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-manifests.git
    targetRevision: main
    path: apps/api-server/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Helm Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/helm-charts.git
    targetRevision: main
    path: charts/api-server
    helm:
      releaseName: api-server
      valueFiles:
        - values.yaml
        - values-production.yaml
      parameters:
        - name: image.tag
          value: "1.2.0"
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Kustomize Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-manifests.git
    targetRevision: main
    path: apps/api-server/overlays/production
    kustomize:
      namePrefix: prod-
      images:
        - api-server=ghcr.io/myorg/api-server:1.2.0
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

## ArgoCD AppProject

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  namespace: argocd
spec:
  description: Production workloads
  sourceRepos:
    - 'https://github.com/myorg/*'
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
    - namespace: monitoring
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
  orphanedResources:
    warn: true
  syncWindows:
    - kind: deny
      schedule: '0 0 * * 5'
      duration: 48h
      applications:
        - '*'
      manualSync: true
```

## ApplicationSet (Multi-Cluster / Multi-Env)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/myorg/k8s-manifests.git
        revision: main
        directories:
          - path: apps/api-server/overlays/*
  template:
    metadata:
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/k8s-manifests.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### Cluster Generator (Multi-Cluster)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
  template:
    metadata:
    spec:
      project: production
      source:
        repoURL: https://github.com/myorg/k8s-manifests.git
        targetRevision: main
        path: platform/base
      destination:
        server: '{{server}}'
        namespace: platform
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## Kustomize Overlays

### Directory Structure

```
apps/
└── api-server/
    ├── base/
    │   ├── kustomization.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── configmap.yaml
    └── overlays/
        ├── staging/
        │   ├── kustomization.yaml
        │   ├── replica-patch.yaml
        │   └── resource-patch.yaml
        └── production/
            ├── kustomization.yaml
            ├── replica-patch.yaml
            ├── resource-patch.yaml
            └── ingress.yaml
```

### Base kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app.kubernetes.io/name: api-server
  app.kubernetes.io/part-of: platform

commonAnnotations:
  app.kubernetes.io/managed-by: argocd
```

### Overlay kustomization.yaml (production)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - ingress.yaml

namespace: production

namePrefix: prod-

patches:
  - path: replica-patch.yaml
  - path: resource-patch.yaml
  - target:
      kind: Deployment
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: ghcr.io/myorg/api-server:1.2.0

images:
  - name: ghcr.io/myorg/api-server
    newTag: "1.2.0"

replicas:
  - name: api-server
    count: 5
```

### Patch Files

```yaml
# resource-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
spec:
  template:
    spec:
      containers:
        - name: api-server
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
```

## Flux Setup

### Bootstrap Flux

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Check prerequisites
flux check --pre

# Bootstrap Flux on cluster
flux bootstrap github \
  --owner=myorg \
  --repository=k8s-manifests \
  --branch=main \
  --path=clusters/production \
  --personal=false
```

### Flux Directory Structure

```
clusters/
└── production/
    ├── flux-system/
    │   ├── gotk-components.yaml
    │   ├── gotk-sync.yaml
    │   └── kustomization.yaml
    ├── infrastructure.yaml
    └── apps.yaml
infrastructure/
├── controllers/
│   ├── cert-manager.yaml
│   └── ingress-nginx.yaml
└── base/
    └── namespaces.yaml
apps/
├── api-server/
│   ├── release.yaml
│   └── kustomization.yaml
└── frontend/
    ├── release.yaml
    └── kustomization.yaml
```

### Flux Kustomization

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
  path: ./apps/api-server
  prune: true
  wait: true
  timeout: 5m0s
  dependsOn:
    - name: infrastructure
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      namespace: production
  patches:
    - patch: |-
        apiVersion: apps/v1
        kind: Deployment
        metadata:
        spec:
          replicas: 5
      target:
        kind: Deployment
```

### Flux HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  namespace: production
spec:
  interval: 30m
  chart:
    spec:
      chart: api-server
      version: "1.x.x"
      sourceRef:
        kind: HelmRepository
        namespace: flux-system
      interval: 12h
  values:
    replicaCount: 3
    image:
      tag: "1.2.0"
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
  driftDetection:
    mode: enabled
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
```

### Flux HelmRepository

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  namespace: flux-system
spec:
  interval: 1h
  type: oci
  url: oci://ghcr.io/myorg/charts
```

### Flux Image Automation

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  namespace: flux-system
spec:
  image: ghcr.io/myorg/api-server
  interval: 5m
  secretRef:
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  namespace: flux-system
spec:
  imageRepositoryRef:
  policy:
    semver:
      range: '>=1.0.0 <2.0.0'
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: flux@myorg.com
      messageTemplate: '{{range .Updated.Images}}{{println .}}{{end}}'
    push:
      branch: main
  update:
    path: ./apps/api-server
    strategy: Setters
```

## Sync Policies and Strategies

### Progressive Delivery with Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 30
        - pause: { duration: 5m }
        - setWeight: 50
        - analysis:
            templates:
              - templateName: success-rate
            args:
              - name: service-name
                value: api-server
        - setWeight: 100
      canaryService: api-server-canary
      stableService: api-server-stable
      trafficRouting:
        nginx:
          stableIngress: api-server
```

## Best Practices

- Use separate Git repos for manifests and application code (or separate paths)
- Never use `latest` tag — always pin to specific versions or semver ranges
- Use ArgoCD AppProject to restrict which repos/namespaces each team can deploy to
- Enable `selfHeal` to detect and revert manual cluster changes
- Use `prune: true` to remove resources no longer in Git
- Implement sync windows to prevent deployments during off-hours
- Use ApplicationSets to reduce duplication across environments
- Store secrets outside Git using External Secrets Operator or Sealed Secrets
