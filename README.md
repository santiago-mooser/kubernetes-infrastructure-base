# Kubernetes Infrastructure Base

Mono-repo to manage Argo CD Applications and Helm charts for a homelab stack (e.g., Immich, Radarr, Paperless, Authelia, Cloudflared). Uses the App-of-Apps pattern: a single bootstrap Helm chart renders Argo CD Projects and per-app Applications sourced from this repo.

Repository layout
- apps/
  - bootstrap/
    - Chart.yaml
    - values.yaml
    - templates/
      - applications.yaml   # Renders Argo CD Applications from .Values.apps
      - projects.yaml       # Renders Argo CD Projects from .Values.projects
- charts/
  - immich/
  - radarr/
  - paperless/
  - authelia/
  - cloudflared/
- environments/
  - home/
    - apps/
      - immich.yaml
      - radarr.yaml
      - paperless.yaml
      - authelia.yaml
      - cloudflared.yaml
- bootstrap/
  - install-home.yaml       # Root Argo CD Application pointing to apps/bootstrap
- LICENSE
- README.md

Argo CD strategy
- App-of-Apps: install one root Argo CD Application (bootstrap/install-home.yaml).
- The bootstrap Helm chart creates:
  - Argo CD Projects: media, productivity, security, networking.
  - One Argo CD Application per app in .Values.apps, each pointing to charts/<app> with env-specific values files under environments/<env>/apps.

Bootstrap flow
1) Prereqs:
   - A running Kubernetes cluster
   - Argo CD installed in namespace argocd
   - Cluster access via kubectl

2) Apply the root Application (env=home by default):
```bash
kubectl apply -n argocd -f bootstrap/install-home.yaml
```

3) Argo CD will sync the `apps/bootstrap` chart, which will:
   - Create AppProjects
   - Create per-app Argo CD Applications
   - Sync the apps using values from `environments/home/apps/*.yaml`

Conventions
- One namespace per app (e.g., immich, radarr).
- Git as the single source of truth. Changes flow via PRs.
- Helm chart per app with minimal templates (deployment, service, optional ingress/config).
- Environment overlays under environments/<env>/apps/<app>.yaml contain app-specific values.
- Automated sync with prune + self-heal enabled by default.
- Sync options include CreateNamespace and ServerSideApply for clean first installs.

Add a new app (example)
1) Create a new chart under charts/<newapp>/ with:
   - Chart.yaml, values.yaml, templates/* (at minimum deployment + service)
2) Add an env overlay file:
   - environments/home/apps/<newapp>.yaml
3) Register the app in apps/bootstrap/values.yaml:
   - Add an entry under .Values.apps with:
     - name, project, namespace, chartPath, valueFiles
4) Commit and push. Argo CD will detect changes and reconcile.

Security and secrets
- Secret management is intentionally out of scope for the initial scaffold.
- Consider adding SOPS/External Secrets later (e.g., .sops.yaml + external-secrets operator).
- For now, keep any sensitive data out of Git or use placeholders and sealed secrets as needed.

Notes
- The repoURL and targetRevision default to this repository and the main branch.
- Ingress, storage, and app-specific configuration should live in the env overlays per app.
- Adjust projects, namespaces, and policies as desired in apps/bootstrap/values.yaml.

Troubleshooting
- If Applications do not appear, check the Argo CD Application events and controller logs:
```bash
kubectl -n argocd get applications
kubectl -n argocd describe application homelab
kubectl -n argocd logs deploy/argocd-repo-server
kubectl -n argocd logs deploy/argocd-application-controller
```

License
See LICENSE.
