# Hikma Health Server Helm Chart

Helm chart for deploying Hikma Health Server to a DigitalOcean Kubernetes cluster with GitHub Actions CI/CD.

## Prerequisites

- DigitalOcean Kubernetes cluster with Gateway API installed
- GitHub repository access for `hikmahealth/hikma-health-server`
- GitHub Packages (GHCR) access
- PostgreSQL database (DigitalOcean Managed Database recommended)

## Quick Start

### 1. Configure GitHub Secrets

Set up the following secrets in your `gliax/helm-hikma` repository:

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:***@host:5432/dbname?sslmode=require` |
| `KUBE_CONFIG` | Base64-encoded Kubernetes kubeconfig | See below |
| `DIGITALOCEAN_TOKEN` | DO API token with cluster access | See below |
| `GHCR_PAT` | GitHub PAT for pulling images from GHCR | See below |
| `HIKMA_REPO_TOKEN` | GitHub token with access to hikma-health-server | Personal Access Token (classic) with `repo` scope |
| `GATEWAY_HOST` (optional) | Domain for Gateway API | `hikma.yourdomain.com` |

#### Getting your kubeconfig:

```bash
# Get your DO kubeconfig
doctl kubernetes cluster kubeconfig save your-cluster-name

# Base64 encode it for GitHub secret
cat ~/.kube/config | base64 -w 0
```

Copy the output and add it as the `KUBE_CONFIG` secret.

#### Getting DIGITALOCEAN_TOKEN:

1. Go to **DigitalOcean Control Panel** → **API** → **Generate New Token**
2. Name it something like "GitHub Actions - helm-hikma"
3. **Select these scopes:**
   - ✅ **access cluster** - View and download Kubernetes cluster credentials (REQUIRED)
   - ✅ **read** - View Kubernetes clusters (optional, for troubleshooting)
4. Click **Generate Token**
5. Copy the token and add it as `DIGITALOCEAN_TOKEN` secret

**Important:** The `access cluster` scope is required for `doctl` to authenticate with your Kubernetes cluster.

#### Getting HIKMA_REPO_TOKEN:

1. Go to GitHub Settings > Developer Settings > Personal access tokens > Tokens (classic)
2. Generate new token with `repo` scope
3. Add it as `HIKMA_REPO_TOKEN` secret

### 2. Configure PostgreSQL Database

Create a PostgreSQL database on DigitalOcean Managed Databases:

1. Go to DigitalOcean Control Panel > Databases
2. Create new PostgreSQL database
3. Note the connection string (DATABASE_URL format):
   ```
   postgresql://doadmin:password@host:25060/defaultdb?sslmode=require
   ```
4. Add this as the `DATABASE_URL` GitHub secret

### 3. Deploy

#### Option A: Automatic (via GitHub Actions)

Simply push to the `main` branch of this repository. The workflow will:
1. Clone hikma-health-server source
2. Build Docker image
3. Push to GHCR
4. Deploy to your Kubernetes cluster

#### Option B: Manual Deployment

```bash
# Clone this repo
git clone https://github.com/gliax/helm-hikma.git
cd helm-hikma

# Create namespace
kubectl create namespace hikma

# Create database secret
kubectl create secret generic hikma-db-credentials \
  --from-literal=DATABASE_URL="postgresql://user:password@host:5432/db?sslmode=require" \
  -n hikma

# Create image pull secret for GHCR
kubectl create secret docker-registry ghcr-credentials \
  --docker-server=ghcr.io \
  --docker-username=gliax \
  --docker-password=$YOUR_GITHUB_TOKEN \
  -n hikma

# Deploy with Helm
helm upgrade --install hikma . \
  --namespace hikma \
  --set gateway.host=hikma.yourdomain.com \
  --wait
```

## Configuration

### values.yaml Options

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.registry` | Container registry | `ghcr.io` |
| `image.repository` | Image repository | `gliax/hikma-health-server` |
| `image.tag` | Image tag | `latest` |
| `replicaCount` | Number of replicas | `1` |
| `service.port` | Service port | `3000` |
| `gateway.enabled` | Enable Gateway API HTTPRoute | `true` |
| `gateway.host` | Domain for Gateway API | `hikma.example.com` |
| `resources.limits.memory` | Memory limit | `6Gi` |
| `resources.limits.cpu` | CPU limit | `2000m` |
| `database.secretName` | Kubernetes secret name for DB | `hikma-db-credentials` |
| `namespace` | Kubernetes namespace | `hikma` |

### Custom Values

Create a `values-custom.yaml`:

```yaml
replicaCount: 2
resources:
  limits:
    memory: 8Gi
    cpu: 4000m
gateway:
  host: hikma.production.com
```

Deploy with:
```bash
helm upgrade --install hikma . -f values-custom.yaml -n hikma
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DigitalOcean K8s                          │
│                                                              │
│  ┌──────────────┐         ┌──────────────────────────────┐  │
│  │   Gateway    │────────▶│    hikma-hikma Service       │  │
│  │  (Public)    │         │         (ClusterIP)          │  │
│  └──────────────┘         └──────────────┬───────────────┘  │
│                                          │                   │
│                                          ▼                   │
│                              ┌───────────────────────┐       │
│                              │  Deployment (Pods)    │       │
│                              │  - Hikma Health App   │       │
│                              │  - Port 3000          │       │
│                              └──────────┬────────────┘       │
│                                         │                   │
│                                         ▼                   │
│                              ┌──────────────────────┐       │
│                              │  Image: GHCR         │       │
│                              │  ghcr.io/gliax/      │       │
│                              │  hikma-health-server │       │
│                              └──────────────────────┘       │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  DigitalOcean        │
                    │  Managed PostgreSQL  │
                    └──────────────────────┘
```

## Troubleshooting

### Check Deployment Status

```bash
# Check pods
kubectl get pods -n hikma

# Check deployment
kubectl get deployment hikma-hikma -n hikma

# View logs
kubectl logs -n hikma -l app.kubernetes.io/name=hikma -f

# Describe pod for events
kubectl describe pod -n hikma -l app.kubernetes.io/name=hikma
```

### Common Issues

**Issue: ImagePullBackOff**
```bash
# Verify image pull secret
kubectl get secret ghcr-credentials -n hikma -o yaml

# Check image tag exists
docker pull ghcr.io/gliax/hikma-health-server:latest
```

**Issue: CrashLoopBackOff**
```bash
# Check logs for database connection errors
kubectl logs -n hikma -l app.kubernetes.io/name=hikma

# Verify DATABASE_URL secret
kubectl get secret hikma-db-credentials -n hikma -o yaml
```

**Issue: Gateway not routing**
```bash
# Check HTTPRoute
kubectl get httproute -n hikma
kubectl describe httproute hikma-hikma -n hikma

# Check Gateway
kubectl get gateway gateway-public
```

### Uninstall

```bash
# Uninstall release
helm uninstall hikma -n hikma

# Delete namespace (optional)
kubectl delete namespace hikma
```

## CI/CD Workflow

The GitHub Actions workflow (`build-and-deploy.yaml`) handles:

1. **Build Phase**: Clones hikma-health-server, builds Docker image, pushes to GHCR
2. **Deploy Phase**: Updates Helm values, deploys to Kubernetes via helm upgrade

### Workflow Triggers

- **Push to main**: Full build and deploy
- **Pull requests**: Build only (no deploy)

### Manual Workflow Run

Go to Actions tab > Build and Deploy Hikma Health Server > Run workflow

## Security Notes

- Never commit DATABASE_URL or kubeconfig to the repository
- Use GitHub secrets for all sensitive values
- Consider using a dedicated GitHub token with minimal required scopes
- Rotate database credentials regularly
- Enable GitHub repository protection rules for main branch

## License

See the hikma-health-server repository for licensing information.
