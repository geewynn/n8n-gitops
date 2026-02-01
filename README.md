# n8n GitOps Deployment

GitOps repository for n8n multi-tenant deployment on Azure Kubernetes Service (AKS) using Flux CD.

## Repository Structure
```
.
├── apps/                    # Application workloads
│   ├── base/               # Base manifests (templates)
│   │   └── customer1/      # Per-customer n8n deployment
│   └── staging/            # Staging environment overlays
├── infrastructure/          # Cluster infrastructure
│   ├── configs/            # Infrastructure configurations
│   │   ├── base/
│   │   └── staging/
│   └── controllers/        # Infrastructure controllers (Helm releases)
│       ├── base/
│       │   ├── cert-manager/
│       │   ├── cnpg/
│       │   └── traefik/
│       └── staging/
├── monitoring/              # Observability stack
│   ├── configs/            # Alerting rules and dashboards
│   │   └── staging/
│   │       └── grafana/
│   └── controllers/        # Monitoring controllers
│       ├── base/
│       │   └── kube-prometheus-stack/
│       └── staging/
└── README.md
```

## Components

### Infrastructure Controllers

| Component | Description | Namespace |
|-----------|-------------|-----------|
| Traefik | Ingress controller and load balancer | `traefik` |
| cert-manager | TLS certificate management with Let's Encrypt | `cert-manager` |
| CNPG | CloudNativePG operator for PostgreSQL databases | `cnpg-system` |

### Monitoring Stack

| Component | Description | Namespace |
|-----------|-------------|-----------|
| kube-prometheus-stack | Prometheus, Grafana, and Alertmanager | `monitoring` |

### Applications

| Component | Description | Namespace |
|-----------|-------------|-----------|
| n8n | Workflow automation platform (per-customer) | `customer-<name>` |

## Environments

| Environment | Branch | Path Suffix |
|-------------|--------|-------------|
| Staging | `main` | `/staging` |
| Production | `main` | `/production` |

## Flux Kustomization Dependencies
```
infra-controllers
       │
       ├── infra-configs
       │        │
       │        └── apps
       │
       └── monitoring-controllers
                │
                └── monitoring-configs
```

## Adding a New Customer

1. Copy the base customer template:
```bash
   cp -r apps/base/customer1 apps/base/<customer-name>
```

2. Update the namespace and resource names in the new directory.

3. Create the staging overlay:
```bash
   mkdir -p apps/staging/<customer-name>
```

4. Create `apps/staging/<customer-name>/kustomization.yaml`:
```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - ../../base/<customer-name>
```

5. Add the customer to `apps/staging/kustomization.yaml`:
```yaml
   resources:
     - customer1
     - <customer-name>
```

6. Create required secrets in Azure Key Vault (via Terraform).

7. Commit and push. Flux will reconcile automatically.

## Adding a New Environment

1. Create the environment directories:
```bash
   mkdir -p apps/production
   mkdir -p infrastructure/configs/production
   mkdir -p infrastructure/controllers/production
   mkdir -p monitoring/configs/production
   mkdir -p monitoring/controllers/production
```

2. Create kustomization overlays referencing base manifests.

3. Update Terraform Flux configuration with new kustomizations.

## Secret Management

Secrets are managed via Azure Key Vault and synced to Kubernetes using the CSI Secrets Store Driver.

| Secret | Key Vault Name | Description |
|--------|---------------|-------------|
| Database credentials | `customer1-db-user`, `customer1-db-password` | PostgreSQL credentials |
| Backup SAS token | `customer1-blob-sas` | Azure Blob storage access |
| Grafana admin | `grafana-admin-user`, `grafana-admin-password` | Grafana credentials |
| Bot token | `bot-token`, `bot-chat-id` | Alerting notifications |

## Local Development

### Prerequisites

- `kubectl` configured for the cluster
- `flux` CLI installed
- Azure CLI authenticated

### Validate Manifests
```bash
# Validate all kustomizations
find . -name kustomization.yaml -execdir kubectl kustomize . \;

# Validate specific environment
kubectl kustomize apps/staging
```

### Force Reconciliation
```bash
# Reconcile all
flux reconcile kustomization flux-system --with-source

# Reconcile specific kustomization
flux reconcile kustomization apps
```

### Check Status
```bash
# Flux status
flux get all

# Kustomization status
flux get kustomizations

# Helm releases
flux get helmreleases -A
```

## Infrastructure Repository

The underlying AKS cluster and Azure resources are managed in a separate Terraform repository:

- **Repository**: `azure-k8s-deployment`
- **Resources**: AKS cluster, Key Vault, Storage Account, Flux configuration

## Troubleshooting

### Flux not syncing
```bash
flux logs --level=error
flux get sources git
```

### Helm release failed
```bash
flux get helmreleases -A
kubectl describe helmrelease <name> -n <namespace>
```

### Certificate issues
```bash
kubectl get certificates -A
kubectl describe certificate <name> -n <namespace>
kubectl get challenges -A
```

### Database issues
```bash
kubectl get clusters -A  # CNPG clusters
kubectl describe cluster <name> -n <namespace>
```

## Contributing

1. Create a feature branch
2. Make changes
3. Validate locally with `kubectl kustomize`
4. Create pull request
5. Merge to `main` after review
6. Flux auto-deploys changes

## References

- [Flux CD Documentation](https://fluxcd.io/docs/)
- [Kustomize Documentation](https://kustomize.io/)
- [CloudNativePG Documentation](https://cloudnative-pg.io/docs/)
- [n8n Documentation](https://docs.n8n.io/)