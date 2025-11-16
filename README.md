# GitOps Demo - Hello World Application

This directory contains a sample application structured for GitOps deployment with Argo CD.

## Directory Structure

```
gitops/
├── manifests/                # Argo CD Application manifests
│   ├── app1-dev.yaml
│   ├── app1-staging.yaml
│   └── app1-prod.yaml
├── environments/             # Helm values per environment, per app
│   ├── dev/
│   │   └── app1.yaml
│   ├── staging/
│   │   └── app1.yaml
│   └── production/
│       └── app1.yaml
└── app1-app/                 # Helm chart metadata
    ├── Chart.yaml
    └── templates/            # Templates; per app basis
        ├── deployment.yaml
        ├── service.yaml
        ├── ingress.yaml
        └── configmap.yaml
```


## Environment Differences

| Environment | Replicas | Domain | Cert Issuer | Purpose |
|-------------|----------|--------|-------------|---------|
| Development | 1 | app1-dev.* | staging | Testing new features |
| Staging | 2 | app1-staging.* | staging | Pre-production validation |
| Production | 3 | app1-prod.* | production | Live user traffic |

## Making Changes

### Update the Application

1. Edit the values file for your environment
2. Commit and push to Git
3. Argo CD automatically detects and syncs (if auto-sync enabled)

**Example - Scale development:**

```bash
# Edit values file
nano environments/dev/app1.yaml
# Change replicas: 3

# Commit and push
git commit -am "Scale dev environment to 3 replicas"
git push origin main

# Watch Argo CD sync
argocd app get app1-dev
```

### Promote Between Environments

Typical flow: dev → staging → production

```bash
# Test in dev
git checkout -b feature/new-message
# Edit environments/dev/values.yaml
git commit -am "Update message in dev"
git push origin feature/new-message

# After testing, promote to staging
# Edit environments/staging/values.yaml with same changes
git commit -am "Promote to staging"
git push origin feature/new-message

# After staging validation, merge to main for production
git checkout main
git merge feature/new-message
# Edit environments/production/values.yaml
git commit -am "Promote to production"
git push origin main
```

## Testing Locally

Before pushing to Git, test your Helm chart locally:

```bash
# Test dev environment
helm template app1-app -f environments/dev/app1.yaml .

# Install locally to test namespace
helm install test-dev app1-app -f environments/dev/app1.yaml -n test-dev --create-namespace

# Cleanup
helm uninstall test-dev -n test-dev
kubectl delete namespace test-dev
```

## Troubleshooting

### Verify Helm Chart Syntax

```bash
helm lint app1-app
```

### Check Generated Manifests

```bash
helm template app1-app -f environments/dev/app1.yaml .
```

### Validate Values

```bash
# Check what values will be used
helm show values app1-app
```
