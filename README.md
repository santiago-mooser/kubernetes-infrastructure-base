# Kubernetes Infrastructure Base

This mono-repo is meant to be a base for managing Kubernetes infrastructure using GitOps principles with ArgoCD.

It contains the argocd applications definitions for different environments and namespaces, as well as the necessary helm charts and kustomize overlays to deploy common infrastructure components.

## Repository Structure

- `argocd-apps/`: Contains ArgoCD application definitions for different environments and namespaces.
  - `root/`: Root level applications that manage other applications.
  - `infrastructure/`: Applications related to infrastructure components.
  - `personal/`: Applications related to personal projects or namespaces.
- `charts/`: Contains Helm charts for deploying various infrastructure components.
- `kustomize/`: Contains Kustomize overlays for different environments and components.

