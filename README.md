# Linkerd-Flux

GitOps deployment of Linkerd Enterprise with Flux, cert-manager, and External Secrets Operator backed by an in-cluster HashiCorp Vault.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [k3d](https://k3d.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [flux](https://fluxcd.io/flux/installation/#install-the-flux-cli)
- A valid Linkerd Enterprise license
- Buoyant Cloud credentials (`client_id`, `client_secret`)
- A GitHub personal access token with `repo` scope

## Architecture

Flux deploys components in stages using Kustomizations with `dependsOn` chains:

```
vault → external-secrets → external-secrets-config ─┬─→ linkerd-buoyant
                                                    └─→ linkerd
cert-manager → linkerd-certs ─────────────────────────→ linkerd
```

`linkerd-buoyant` and `linkerd` deploy in parallel at the Flux level. The HelmRelease-level `dependsOn` inside `base/linkerd/helmrelease.yaml` ensures the correct ordering between Helm charts (e.g. `linkerd-crds` waits for `linkerd-buoyant` and `trust-manager`).

| Stage | Component | Chart Version |
|---|---|---|
| vault | HashiCorp Vault (dev mode) | 0.28.1 |
| external-secrets | External Secrets Operator | 0.9.13 |
| external-secrets-config | ClusterSecretStore + Vault token | -- |
| cert-manager | cert-manager + trust-manager | 1.11.5 / 0.4.0 |
| linkerd-certs | CA, identity issuer, trust bundle, webhooks | -- |
| linkerd-buoyant | Buoyant Cloud operator | 0.30 |
| linkerd | Linkerd Enterprise CRDs + control plane | 2.15.2 |

Secrets are managed via the External Secrets Operator, which pulls them from the in-cluster Vault (`vault.vault.svc.cluster.local:8200`) into Kubernetes Secrets at runtime. No plaintext credentials are stored in Git.

## Repository Structure

```
.
├── base/
│   ├── cert-manager/              # cert-manager + trust-manager
│   ├── external-secrets/          # External Secrets Operator
│   ├── linkerd/                   # Linkerd Enterprise CRDs + control plane
│   ├── linkerd-buoyant/           # Buoyant Cloud operator
│   └── vault/                     # HashiCorp Vault (dev mode)
└── overlays/
    ├── flux-kustomizations.yaml   # Staged Flux Kustomizations with dependsOn
    ├── kustomization.yaml         # Top-level entry point
    ├── cert-manager/              # Env-specific patches (node selectors, tolerations)
    ├── external-secrets/
    │   └── config/                # ClusterSecretStore + Vault token
    ├── linkerd/
    │   └── certificates/          # CA, identity issuer, trust bundle, webhooks
    ├── linkerd-buoyant/           # Version override, agent name
    └── vault/                     # Vault overlay
```

## Setup

### 1. Create the k3d cluster

```bash
k3d cluster create linkerd \
  --k3s-arg "--disable=traefik@server:*" \
  -p "80:80@loadbalancer"
```

Verify the cluster is running:

```bash
kubectl cluster-info
kubectl get nodes
```

### 2. Bootstrap Flux

Export your GitHub credentials:

```bash
export GITHUB_TOKEN=<YOUR_GITHUB_PAT>
export GITHUB_USER=<YOUR_GITHUB_USERNAME>
export GITHUB_REPO=<YOUR_REPO_NAME>
```

Bootstrap Flux on the cluster:

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=$GITHUB_REPO \
  --branch=main \
  --path=./overlays \
  --personal
```

This installs Flux and configures it to reconcile the staged Kustomizations defined in `overlays/flux-kustomizations.yaml`. Each stage waits for its dependencies to be healthy before proceeding.

Flux will immediately begin deploying all components. The `linkerd-buoyant` and `linkerd` stages will fail initially because the Vault secrets don't exist yet — this is expected. Both are configured with unlimited retries (`remediation.retries: -1`) and will automatically recover once you seed the secrets in the next step.

### 3. Seed secrets into Vault

Wait for the Vault pod to be ready:

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=vault -n vault --timeout=120s
```

Populate Vault with the required secrets:

```bash
kubectl exec -n vault vault-0 -- vault kv put secret/linkerd \
  license="<YOUR_LINKERD_LICENSE>"

kubectl exec -n vault vault-0 -- vault kv put secret/linkerd-buoyant \
  client_id="<YOUR_BUOYANT_CLIENT_ID>" \
  client_secret="<YOUR_BUOYANT_CLIENT_SECRET>"
```

The ExternalSecrets refresh every hour by default. To sync immediately:

```bash
kubectl annotate externalsecret -n linkerd linkerd-license force-sync=$(date +%s) --overwrite
kubectl annotate externalsecret -n linkerd-buoyant linkerd-license force-sync=$(date +%s) --overwrite
kubectl annotate externalsecret -n linkerd-buoyant buoyant-cloud-credentials force-sync=$(date +%s) --overwrite
```

The HelmReleases will automatically retry and succeed once the secrets are available.

### 4. Verify the deployment

Watch Flux reconcile all stages:

```bash
flux get kustomizations --watch
```

All stages should show `Ready: True`:

```
NAME                    READY   MESSAGE
vault                   True    Applied revision: main@sha1:...
external-secrets        True    Applied revision: main@sha1:...
external-secrets-config True    Applied revision: main@sha1:...
cert-manager            True    Applied revision: main@sha1:...
linkerd-certs           True    Applied revision: main@sha1:...
linkerd-buoyant         True    Applied revision: main@sha1:...
linkerd                 True    Applied revision: main@sha1:...
```

Check individual components:

```bash
# Vault
kubectl get pods -n vault

# External Secrets Operator
kubectl get pods -n external-secrets
kubectl get externalsecrets --all-namespaces

# cert-manager + certificates
kubectl get pods -n cert-manager
kubectl get certificates --all-namespaces
kubectl get bundle linkerd-identity-trust-roots

# Linkerd
kubectl get pods -n linkerd-buoyant
kubectl get pods -n linkerd
linkerd check
```

Verify secrets were synced from Vault:

```bash
kubectl get secrets -n linkerd vss-linkerd-license
kubectl get secrets -n linkerd-buoyant vss-linkerd-license
kubectl get secrets -n linkerd-buoyant buoyant-cloud-org-credentials
```

Verify the trust bundle ConfigMap was distributed:

```bash
kubectl get configmap linkerd-identity-trust-roots -n linkerd
```

## Vault Secret Paths

| Vault Path | Keys | Used By |
|---|---|---|
| `secret/data/linkerd` | `license` | Linkerd control plane, Buoyant operator |
| `secret/data/linkerd-buoyant` | `client_id`, `client_secret` | Buoyant Cloud registration |

## Certificate Chain

cert-manager manages the full Linkerd certificate lifecycle:

- **Trust anchor** (`ca.yaml`) -- self-signed root CA, 365-day validity, auto-rotated
- **Identity issuer** (`identity.yaml`) -- intermediate CA for mTLS identity, 48h validity
- **Trust bundle** (`bundle.yaml`) -- distributes the root CA to namespaces labeled `linkerd.io/is-control-plane: "true"`
- **Webhook CA** (`webhook.yaml`) -- self-signed CA for admission webhooks
- **Webhook certificates** (`webhook.yaml`) -- short-lived certs (24h) for policy-validator, proxy-injector, and sp-validator

## Troubleshooting

**linkerd-control-plane HelmRelease stuck in Failed state**

If the control plane install times out (e.g. due to slow image pulls on first deploy), reset the retry counter:

```bash
flux suspend helmrelease linkerd-control-plane -n linkerd
flux resume helmrelease linkerd-control-plane -n linkerd
```

**Pods stuck in Init or CrashLoopBackOff after trust bundle fix**

If pods were created before the `linkerd-identity-trust-roots` ConfigMap existed, restart them:

```bash
kubectl rollout restart deployment -n linkerd linkerd-identity linkerd-destination linkerd-proxy-injector
```

**ExternalSecrets not syncing**

Check that Vault has the secrets and the ClusterSecretStore is healthy:

```bash
kubectl get clustersecretstore vault-backend
kubectl get externalsecrets --all-namespaces
```

## Cleanup

```bash
k3d cluster delete linkerd
```
