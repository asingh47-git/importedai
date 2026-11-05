# AKS Deployment Runbook

Status as of 2026-11-04: the AKS cluster and deployment foundation are configured, but a successful application rollout has not yet been verified.

## Environment

| Resource | Value |
|---|---|
| Azure subscription | `11ffc202-6d6e-4537-8d53-3700d463b679` |
| AKS cluster | `aksaikubernetes` |
| AKS resource group | `aksaiproject` |
| ACR registry | `aks8997` |
| ACR login server | `aks8997-encecwbjexh2gsgu.azurecr.io` |
| Kubernetes namespace | `legallens` |
| GitHub Actions environment | `production` |

## Deployment Flow

`.github/workflows/ci.yml` validates backend, frontend and Docker builds. After a successful push-to-`main` CI run, `.github/workflows/CD.yml` builds and publishes backend and frontend images to ACR, tagged `latest` and `sha-<commit>`. The deploy job then applies `k8s/app.yaml`, updates both Deployments to the immutable commit tags, and waits for rollout completion.

The frontend and backend Services are `ClusterIP`; neither is exposed publicly. NGINX proxies `/api`, `/healthz`, and `/readyz` to the backend Service. The backend runs one replica with SQLite and encrypted blob data on a 16 GiB `managed-csi` persistent volume. Its Deployment uses `Recreate` to avoid two processes writing the same SQLite database during an update.

## Azure Identity and Registry Access

GitHub Actions authenticates to Azure using OIDC, not a client secret. The app registration has a federated credential with:

- Issuer: `https://token.actions.githubusercontent.com`
- Subject: `repo:asingh47-git/importedai:environment:production`
- Audience: `api://AzureADTokenExchange`

The workflow expects `AZURE_CLIENT_ID` and `AZURE_TENANT_ID` in the GitHub `production` environment. Its identity has `Container Registry Repository Writer` access conditioned to the repositories `legallens-backend` and `legallens-frontend`. This registry uses ABAC repository permissions; legacy `AcrPush` is not honored.

The AKS kubelet identity has `Container Registry Repository Reader` access, likewise conditioned to those two repositories.

## GitHub Environment Secrets

Configure these secrets at **GitHub repository → Settings → Environments → production**:

| Secret | Purpose |
|---|---|
| `AZURE_CLIENT_ID` | OIDC app registration client ID |
| `AZURE_TENANT_ID` | Microsoft Entra tenant ID |
| `KUBE_CONFIG` | Kubeconfig for the namespace-scoped `github-actions-deployer` service account |
| `LL_BLOB_KEY` | Stable 32-byte hex key for encrypted files, extracted text and page images |
| `LL_SECRET_KEY` | Stable signing key for sessions and signed read URLs |

Generate the two application keys separately and store them in a secure password manager before adding them to GitHub. Do not rotate `LL_BLOB_KEY` after documents have been stored unless you also implement a data re-encryption procedure; losing it makes existing encrypted data unreadable.

The local deployer kubeconfig is at `~/.kube/legallens-github-kubeconfig` with file mode `0600`. Add its contents as the `KUBE_CONFIG` environment secret; never commit it or paste it into chat. It contains a long-lived Kubernetes service-account token. Its RBAC is limited to the `legallens` namespace, but the token should still be treated as a credential and rotated if exposed. A future improvement is short-lived OIDC-based Kubernetes authentication with Entra-integrated AKS RBAC.

## First Rollout

1. Confirm all five environment secrets exist in GitHub's `production` environment.
2. Push a commit to `main`. CI must pass before the CD workflow starts.
3. In GitHub Actions, inspect the CD run. Confirm both ACR image builds complete, then verify the `Deploy to AKS` job reports both Deployments rolled out.
4. Check the cluster directly:

   ```bash
   KUBECONFIG=/tmp/legallens-aks-kubeconfig kubectl get deployments,pods,services,pvc -n legallens
   KUBECONFIG=/tmp/legallens-aks-kubeconfig kubectl rollout status deployment/legallens-backend -n legallens
   KUBECONFIG=/tmp/legallens-aks-kubeconfig kubectl rollout status deployment/legallens-frontend -n legallens
   ```

5. For private local access, run:

   ```bash
   KUBECONFIG=/tmp/legallens-aks-kubeconfig kubectl port-forward -n legallens svc/legallens-frontend 8080:80
   ```

   Then open `http://localhost:8080`.

## Operational Gaps

- The app Services are private. Public production access still needs an ingress or gateway, DNS, TLS, and `LL_COOKIE_SECURE=true`.
- The database is empty on first install. The application has no production registration flow. Do not run `python -m app.seed` in production: it creates a known demo account and fixture corpus. A secure initial-user provisioning path is still required.
- SQLite and encrypted blobs share one persistent volume and are limited to a single backend replica. Before scaling or requiring high availability, move to managed PostgreSQL and durable object storage with a key-management/rotation plan.
- The workflow is not considered deployed until its first successful CD run and live health checks are confirmed.

## Troubleshooting

- **Missing secret:** the deploy job reports which required environment secret is empty. Add it to the `production` environment, not only repository-level variables.
- **Azure login fails:** verify the OIDC subject exactly matches `repo:asingh47-git/importedai:environment:production`, the workflow uses the `production` environment, and the federation is attached to the app identified by `AZURE_CLIENT_ID`.
- **Image push denied:** verify the GitHub service principal's ABAC-conditioned `Container Registry Repository Writer` role for both repository names.
- **`ImagePullBackOff`:** inspect pod events and verify the AKS kubelet's ABAC-conditioned `Container Registry Repository Reader` role and the exact ACR login server.
- **PVC pending:** inspect `kubectl describe pvc legallens-data -n legallens` and the AKS `managed-csi` storage class.
- **Backend crash on startup:** check that both `LL_BLOB_KEY` and `LL_SECRET_KEY` exist and that the PVC is mounted and writable by UID/GID 10001.
- **Workflow can't apply resources:** verify `KUBE_CONFIG` is the namespace-scoped deployer kubeconfig and that its token has not expired or been rotated.