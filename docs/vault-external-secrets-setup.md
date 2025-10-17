# Vault + External Secrets + SOPS Integration Guide

## Overview

This guide covers the complete setup of HashiCorp Vault with External Secrets Operator and SOPS-encrypted secrets management using Terraform/Terragrunt.

## Architecture

```
SOPS-encrypted secrets (Git)
    ↓ (CI/CD decrypt)
Terraform/Terragrunt
    ↓ (push secrets)
Vault (HA with Consul backend)
    ↓ (Kubernetes auth)
External Secrets Operator
    ↓ (sync)
Kubernetes Secrets (per namespace)
```

## Phase 1: Vault & Consul Deployment (COMPLETED ✓)

### 1.1 Deploy Consul

```bash
# Update Helm dependencies
helm dependency update common/helm-charts/infra/consul

# Apply ArgoCD application
kubectl apply -f argocd-apps/nixos-prod/infrastructure/consul.yaml
```

Wait for Consul to be ready:
```bash
kubectl wait --for=condition=ready pod -l app=consul -n consul --timeout=300s
```

### 1.2 Deploy Vault

```bash
# Update Helm dependencies
helm dependency update common/helm-charts/infra/vault

# Apply ArgoCD application
kubectl apply -f argocd-apps/nixos-prod/infrastructure/vault.yaml
```

### 1.3 Initialize Vault

**IMPORTANT**: This is a ONE-TIME operation. Store the unseal keys and root token securely!

```bash
# Exec into first Vault pod
kubectl exec -it vault-0 -n vault -- /bin/sh

# Initialize Vault
vault operator init

# Output will show:
# Unseal Key 1: <key1>
# Unseal Key 2: <key2>
# Unseal Key 3: <key3>
# Unseal Key 4: <key4>
# Unseal Key 5: <key5>
# Initial Root Token: <root-token>

# Save these securely! You need 3 keys to unseal Vault
```

### 1.4 Unseal All Vault Pods

Repeat for each pod (vault-0, vault-1, vault-2):

```bash
kubectl exec -it vault-0 -n vault -- vault operator unseal <key1>
kubectl exec -it vault-0 -n vault -- vault operator unseal <key2>
kubectl exec -it vault-0 -n vault -- vault operator unseal <key3>

kubectl exec -it vault-1 -n vault -- vault operator unseal <key1>
kubectl exec -it vault-1 -n vault -- vault operator unseal <key2>
kubectl exec -it vault-1 -n vault -- vault operator unseal <key3>

kubectl exec -it vault-2 -n vault -- vault operator unseal <key1>
kubectl exec -it vault-2 -n vault -- vault operator unseal <key2>
kubectl exec -it vault-2 -n vault -- vault operator unseal <key3>
```

### 1.5 Access Vault UI

```bash
# Port-forward to access UI
kubectl port-forward -n vault svc/vault-ui 8200:8200

# Open http://localhost:8200
# Login with root token
```

## Phase 2: External Secrets Operator (TODO)

### 2.1 Create External Secrets Chart

Create the following files:
- `common/helm-charts/infra/external-secrets/Chart.yaml`
- `common/helm-charts/infra/external-secrets/values.yaml`
- `common/helm-charts/infra/external-secrets/config/nixos-prod.yaml`
- `common/helm-charts/infra/external-secrets/templates/clustersecretstore.yaml`

### 2.2 Deploy External Secrets Operator

```bash
helm dependency update common/helm-charts/infra/external-secrets
kubectl apply -f argocd-apps/nixos-prod/infrastructure/external-secrets.yaml
```

## Phase 3: Terraform Setup for Vault (TODO)

### 3.1 Terraform Module Structure

Create modules in `common/terraform-modules/`:

#### vault-init Module
```hcl
# common/terraform-modules/vault-init/main.tf
provider "vault" {
  address = var.vault_address
  token   = var.vault_token
}

# Enable KV v2 secrets engine
resource "vault_mount" "secret" {
  path = "secret"
  type = "kv-v2"
}
```

#### vault-k8s-auth Module
```hcl
# common/terraform-modules/vault-k8s-auth/main.tf
# Enable Kubernetes auth method
resource "vault_auth_backend" "kubernetes" {
  type = "kubernetes"
}

resource "vault_kubernetes_auth_backend_config" "config" {
  backend            = vault_auth_backend.kubernetes.path
  kubernetes_host    = var.kubernetes_host
  kubernetes_ca_cert = var.kubernetes_ca_cert
}
```

#### vault-policies Module
```hcl
# common/terraform-modules/vault-policies/main.tf
# Per-namespace policy
resource "vault_policy" "namespace_policy" {
  for_each = var.namespaces

  name   = "ns-${each.key}"
  policy = templatefile("${path.module}/policies/namespace.hcl", {
    environment = var.environment
    namespace   = each.key
  })
}

# Per-namespace Kubernetes auth role
resource "vault_kubernetes_auth_backend_role" "namespace_role" {
  for_each = var.namespaces

  backend                          = "kubernetes"
  role_name                        = "ns-${each.key}"
  bound_service_account_names      = ["external-secrets"]
  bound_service_account_namespaces = [each.key]
  token_policies                   = [vault_policy.namespace_policy[each.key].name]
  token_ttl                        = 3600
}
```

Policy template (`policies/namespace.hcl`):
```hcl
# Allow read access to namespace-specific secrets
path "secret/data/${environment}/${namespace}/*" {
  capabilities = ["read", "list"]
}
```

#### vault-secrets Module
```hcl
# common/terraform-modules/vault-secrets/main.tf
resource "vault_kv_secret_v2" "secrets" {
  for_each = var.secrets

  mount = "secret"
  name  = "${var.environment}/${var.namespace}/${each.key}"
  data_json = jsonencode(each.value)
}
```

### 3.2 Terragrunt Configuration

#### Root terragrunt.hcl
```hcl
# terragrunt/terragrunt.hcl
remote_state {
  backend = "s3"
  config = {
    bucket         = "your-terraform-state-bucket"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock-table"
  }
}
```

#### Environment config
```hcl
# terragrunt/nixos-prod/env.hcl
locals {
  environment = "nixos-prod"
  vault_address = "http://vault.vault.svc.cluster.local:8200"
}
```

#### Module configurations
```hcl
# terragrunt/nixos-prod/vault-init/terragrunt.hcl
terraform {
  source = "../../../common/terraform-modules//vault-init"
}

include "root" {
  path = find_in_parent_folders()
}

include "env" {
  path = find_in_parent_folders("env.hcl")
}

inputs = {
  vault_address = local.vault_address
  vault_token   = get_env("VAULT_TOKEN")
}
```

## Phase 4: SOPS Integration (TODO)

### 4.1 Setup SOPS Configuration

Create `.sops.yaml`:
```yaml
creation_rules:
  - path_regex: terragrunt/secrets/.*\.yaml$
    pgp: >-
      YOUR_GPG_FINGERPRINT
```

### 4.2 Create Encrypted Secrets

```bash
# Create a secret file
cat > terragrunt/secrets/nixos-prod/immich.sops.yaml <<EOF
namespace: immich
secrets:
  database-password: "changeme"
  redis-password: "changeme"
  jwt-secret: "changeme"
EOF

# Encrypt with SOPS
sops -e -i terragrunt/secrets/nixos-prod/immich.sops.yaml
```

### 4.3 Terraform Integration

```hcl
# terragrunt/nixos-prod/vault-secrets/immich/terragrunt.hcl
terraform {
  source = "../../../../common/terraform-modules//vault-secrets"
}

locals {
  # Load and decrypt SOPS file
  secrets_file = yamldecode(sops_decrypt_file("../../../secrets/nixos-prod/immich.sops.yaml"))
}

inputs = {
  environment = "nixos-prod"
  namespace   = local.secrets_file.namespace
  secrets     = local.secrets_file.secrets
}
```

### 4.4 CI/CD Pipeline

```yaml
# .github/workflows/vault-secrets.yaml
name: Deploy Vault Secrets

on:
  push:
    paths:
      - 'terragrunt/secrets/**'
      - 'terragrunt/nixos-prod/vault-secrets/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Terragrunt
        uses: autero1/action-terragrunt@v1

      - name: Import GPG key
        run: |
          echo "${{ secrets.SOPS_GPG_KEY }}" | gpg --import

      - name: Deploy secrets
        env:
          VAULT_TOKEN: ${{ secrets.VAULT_TOKEN }}
        run: |
          cd terragrunt/nixos-prod/vault-secrets
          terragrunt run-all apply --terragrunt-non-interactive
```

## Phase 5: Using Secrets in Applications

### 5.1 Create ExternalSecret Resource

```yaml
# Example: argocd-apps/nixos-prod/personal/immich.yaml (add to existing)
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: immich-secrets
  namespace: immich
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: immich-secrets
    creationPolicy: Owner
  data:
    - secretKey: database-password
      remoteRef:
        key: secret/data/nixos-prod/immich/database-password
    - secretKey: redis-password
      remoteRef:
        key: secret/data/nixos-prod/immich/redis-password
    - secretKey: jwt-secret
      remoteRef:
        key: secret/data/nixos-prod/immich/jwt-secret
```

### 5.2 Reference in Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: immich
spec:
  template:
    spec:
      containers:
      - name: immich
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: immich-secrets
              key: database-password
```

## Maintenance Operations

### Unsealing Vault After Restart

After pod restarts or cluster maintenance:

```bash
# Unseal each pod with 3 keys
for pod in vault-0 vault-1 vault-2; do
  kubectl exec -it $pod -n vault -- vault operator unseal <key1>
  kubectl exec -it $pod -n vault -- vault operator unseal <key2>
  kubectl exec -it $pod -n vault -- vault operator unseal <key3>
done
```

### Rotating Secrets

```bash
# 1. Update SOPS-encrypted file
sops terragrunt/secrets/nixos-prod/immich.sops.yaml

# 2. Apply with Terragrunt
cd terragrunt/nixos-prod/vault-secrets/immich
terragrunt apply

# 3. External Secrets will automatically sync within refreshInterval
```

### Backup Vault Data

Consul stores Vault's data. Backup Consul:

```bash
kubectl exec -it consul-server-0 -n consul -- consul snapshot save /tmp/backup.snap
kubectl cp consul/consul-server-0:/tmp/backup.snap ./consul-backup-$(date +%Y%m%d).snap
```

## Security Best Practices

1. **Never commit unseal keys or root token to Git**
2. **Store unseal keys in multiple secure locations** (password managers, HSMs)
3. **Use namespace-specific policies** - least privilege access
4. **Rotate secrets regularly**
5. **Enable audit logging** in Vault
6. **Use GPG keys stored securely** for SOPS encryption
7. **Limit Terraform state access** - use S3 bucket policies
8. **Use short-lived tokens** for CI/CD

## Troubleshooting

### Vault is sealed

```bash
kubectl logs -n vault vault-0
# If logs show "Vault is sealed", unseal it
```

### External Secrets not syncing

```bash
kubectl describe externalsecret <name> -n <namespace>
kubectl logs -n external-secrets deployment/external-secrets
```

### Terraform can't connect to Vault

```bash
# Port-forward Vault
kubectl port-forward -n vault svc/vault 8200:8200

# Test connection
export VAULT_ADDR=http://localhost:8200
export VAULT_TOKEN=<root-token>
vault status
```

## Next Steps

1. Deploy Consul and Vault (Phase 1 - COMPLETED)
2. Initialize and unseal Vault
3. Create External Secrets Operator chart (Phase 2)
4. Create Terraform modules (Phase 3)
5. Setup Terragrunt with S3 backend (Phase 3)
6. Configure SOPS with GPG (Phase 4)
7. Create encrypted secret files (Phase 4)
8. Setup CI/CD pipeline (Phase 4)
9. Migrate existing secrets to Vault (Phase 5)
10. Update applications to use ExternalSecrets (Phase 5)

## Files Created

### Phase 1 (Completed)
- ✓ `common/helm-charts/infra/consul/Chart.yaml`
- ✓ `common/helm-charts/infra/consul/values.yaml`
- ✓ `common/helm-charts/infra/consul/config/nixos-prod.yaml`
- ✓ `common/helm-charts/infra/vault/values.yaml` (updated)
- ✓ `common/helm-charts/infra/vault/config/nixos-prod.yaml`
- ✓ `argocd-apps/nixos-prod/infrastructure/consul.yaml`
- ✓ `argocd-apps/nixos-prod/infrastructure/vault.yaml`

### Phase 2-5 (TODO)
- See sections above for file structures
