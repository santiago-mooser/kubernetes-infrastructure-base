# Tailscale Kubernetes Operator

This Helm chart deploys the Tailscale Kubernetes Operator to enable secure remote access to your Kubernetes cluster services via Tailscale.

## Overview

The Tailscale Kubernetes Operator allows you to:
- Expose Kubernetes services to your Tailnet securely
- Access services remotely without exposing them to the public internet
- Use Tailscale's built-in authentication and access controls
- Automatically provision TLS certificates for HTTPS

## Prerequisites

### 1. Tailscale Account Setup

1. Create a Tailscale account at [https://login.tailscale.com](https://login.tailscale.com)

2. Update your [tailnet policy file](https://login.tailscale.com/admin/acls):
   ```json
   {
     "tagOwners": {
       "tag:k8s-operator": [],
       "tag:k8s": ["tag:k8s-operator"],
       "tag:k8s-ingress": ["tag:k8s-operator"],
       "tag:nixos-prod": ["tag:k8s-operator"]
     }
   }
   ```

3. Create an OAuth client in [Tailscale OAuth settings](https://login.tailscale.com/admin/settings/oauth):
   - Select **Write** for both **Devices** and **Auth Keys** scopes
   - Add tag: `tag:k8s-operator`
   - Save the **Client ID** and **Client Secret**

### 2. Store OAuth Credentials

Store the OAuth credentials in your secrets manager (AWS Secrets Manager, Vault, etc.):

```json
{
  "client_id": "your-oauth-client-id",
  "client_secret": "your-oauth-client-secret"
}
```

Update `config/nixos-prod.yaml` with the correct secret paths for your secrets manager.

### 3. External Secrets Operator

Ensure External Secrets Operator is deployed and configured with a `ClusterSecretStore` named `cluster-secret-store` (or update the name in `config/nixos-prod.yaml`).

## Installation

### Via ArgoCD (Recommended)

1. Update Helm dependencies:
   ```bash
   helm dependency update common/helm-charts/infra/tailscale-operator
   ```

2. Apply the ArgoCD Application:
   ```bash
   kubectl apply -f argocd-apps/nixos-prod/infrastructure/tailscale-operator.yaml
   ```

3. ArgoCD will automatically deploy the operator to the `tailscale` namespace.

### Manual Installation

```bash
helm install tailscale-operator ./common/helm-charts/infra/tailscale-operator \
  --namespace tailscale \
  --create-namespace \
  -f common/helm-charts/infra/tailscale-operator/config/nixos-prod.yaml
```

## Exposing Services via Tailscale

### Method 1: Using Ingress Annotation

Add the annotation to your existing Ingress resource:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-service
  annotations:
    tailscale.com/expose: "true"
    tailscale.com/hostname: "my-service"
    tailscale.com/tags: "tag:k8s-ingress,tag:nixos-prod"
spec:
  ingressClassName: tailscale
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

### Method 2: Using Service Annotation

Annotate your Service directly:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    tailscale.com/expose: "true"
    tailscale.com/hostname: "my-service"
    tailscale.com/tags: "tag:k8s-ingress,tag:nixos-prod"
spec:
  type: LoadBalancer
  loadBalancerClass: tailscale
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: my-service
```

### Method 3: Using ProxyClass (Advanced)

For more control, use a ProxyClass:

```yaml
apiVersion: tailscale.com/v1alpha1
kind: ProxyClass
metadata:
  name: custom-ingress
spec:
  statefulSet:
    labels:
      app: my-custom-proxy
    annotations:
      custom: annotation
  tailscale:
    tags:
      - "tag:k8s-ingress"
      - "tag:my-custom-tag"
```

## Accessing Services

Once exposed, services will be accessible at:
- `https://<hostname>.<tailnet-name>.ts.net`
- Example: `https://longhorn.my-tailnet.ts.net`

Services are automatically secured with:
- Tailscale's end-to-end encryption
- Built-in HTTPS with automatic certificate management
- Tailnet access controls and ACLs

## Configuration

### Key Values

- `tailscale-operator.oauth`: OAuth client credentials (provided via ExternalSecret)
- `tailscale-operator.operatorConfig.hostname`: Default hostname prefix for the operator
- `tailscale-operator.defaultTags`: Default tags applied to all devices
- `proxyClass`: Configuration for ingress proxy pods

### Environment-Specific Config

Environment-specific settings are in `config/nixos-prod.yaml`:
- Resource limits
- Tags
- Proxy configurations

## Tag-Based Access Control

Configure fine-grained access in your Tailscale ACL policy:

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["group:admins"],
      "dst": ["tag:k8s-ingress:*"]
    },
    {
      "action": "accept",
      "src": ["group:developers"],
      "dst": ["tag:k8s-ingress:80,443"]
    }
  ]
}
```

## Troubleshooting

### Check Operator Status

```bash
kubectl get pods -n tailscale
kubectl logs -n tailscale deployment/tailscale-operator
```

### Verify OAuth Secret

```bash
kubectl get secret operator-oauth -n tailscale
kubectl describe externalsecret tailscale-operator-oauth -n tailscale
```

### Check Tailscale Devices

Visit [Tailscale Machines](https://login.tailscale.com/admin/machines) to see:
- `tailscale-operator` (the operator itself)
- Proxy devices for each exposed service

### Common Issues

1. **Operator not joining Tailnet**: Check OAuth credentials
2. **Services not exposed**: Verify annotations and ProxyClass configuration
3. **Access denied**: Review Tailscale ACL policy for proper tag permissions

## Migration from nginx-internal

To migrate existing services from nginx-internal to Tailscale:

1. Add Tailscale annotations to your Ingress/Service
2. Change `ingressClassName` from `nginx-internal` to `tailscale`
3. Update DNS/access patterns to use Tailscale hostnames
4. Remove public-facing ingress once verified

## Resources

- [Tailscale Kubernetes Operator Documentation](https://tailscale.com/kb/1236/kubernetes-operator)
- [Tailscale Ingress Guide](https://tailscale.com/kb/1439/kubernetes-operator-cluster-ingress)
- [Tailscale ACL Documentation](https://tailscale.com/kb/1018/acls)
- [External Secrets Operator](https://external-secrets.io)

## Support

For issues related to:
- Tailscale Operator: [GitHub Issues](https://github.com/tailscale/tailscale/issues)
- This chart: Contact your platform team
