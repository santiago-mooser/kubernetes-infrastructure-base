# Kubernetes Infrastructure Base

This mono-repo is meant to be a base for managing Kubernetes infrastructure using GitOps principles with ArgoCD.

It contains the argocd applications definitions for different environments and namespaces, as well as the necessary helm charts and kustomize overlays to deploy common infrastructure components.

## Repository Structure

- `argocd-apps/`: Contains ArgoCD application definitions for different environments and namespaces.
  - `nixos-prod/`: Applications for the production environment on NixOS.
    - `root/`: Root level applications that manage other applications.
    - `infrastructure/`: Applications related to infrastructure components.
    - `personal/`: Applications related to personal projects or namespaces.
- `common/`: Contains common Helm charts and terraform modules used across different environments.
  - `charts/`: Contains Helm charts for deploying various infrastructure components.
  - `terraform-modules/`: Contains terraform modules for managing cloud resources.
- `terragrunt/`: Contains terraform configurations for setting up the necessary cloud infrastructure.

