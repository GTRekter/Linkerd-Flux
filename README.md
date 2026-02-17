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

```
vault (in-cluster, dev mode)
    |
external-secrets (ESO)
    |
cert-manager
    ├── cert-manager (v1.11.5)
    └── trust-manager (v0.4.0)
    |
linkerd-buoyant (v0.30)
    |
linkerd
    ├── linkerd-enterprise-crds (v2.15.2)
    └── linkerd-enterprise-control-plane (v2.15.2)
```

Vault runs inside the cluster in dev mode. The External Secrets Operator connects to it via the in-cluster service (`vault.vault.svc.cluster.local:8200`) using a static root token, and syncs secrets into Kubernetes Secrets. No plaintext credentials are stored in Git.

## Repository Structure

```
.
├── base/
│   ├── cert-manager/          # cert-manager + trust-manager
│   ├── external-secrets/      # External Secrets Operator
│   ├── linkerd/               # Linkerd Enterprise CRDs + control plane
│   ├── linkerd-buoyant/       # Buoyant Cloud operator
│   └── vault/                 # HashiCorp Vault (dev mode)
└── overlays/
    ├── cert-manager/          # Env-specific patches
    ├── external-secrets/      # ClusterSecretStore + Vault token
    ├── linkerd/               # Certificates, Flux Kustomization, patches
    │   └── certificates/      # CA, identity issuer, trust bundle, webhooks
    ├── linkerd-buoyant/       # Version override, agent name
    └── vault/                 # Vault overlay
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

This installs Flux and configures it to reconcile all manifests from the `overlays/` directory. Flux will deploy Vault, ESO, cert-manager, and Linkerd in the correct order based on their `dependsOn` chains.

### 3. Wait for Vault to be ready

Vault is deployed automatically by Flux. Wait for the pod to be running:

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=vault -n vault --timeout=120s
```

### 4. Seed secrets into Vault

Once Vault is running, use `kubectl exec` to populate it with the required secrets:

```bash
kubectl exec -n vault vault-0 -- vault kv put secret/linkerd \
  license="<YOUR_LINKERD_LICENSE>"

kubectl exec -n vault vault-0 -- vault kv put secret/linkerd-buoyant \
  client_id="<YOUR_BUOYANT_CLIENT_ID>" \
  client_secret="<YOUR_BUOYANT_CLIENT_SECRET>"
```

After seeding, trigger ESO to reconcile the ExternalSecrets:

```bash
kubectl annotate externalsecret -n linkerd linkerd-license force-sync=$(date +%s) --overwrite
kubectl annotate externalsecret -n linkerd-buoyant linkerd-license force-sync=$(date +%s) --overwrite
kubectl annotate externalsecret -n linkerd-buoyant buoyant-cloud-credentials force-sync=$(date +%s) --overwrite
```

### 5. Verify the deployment

Watch Flux reconcile the resources:

```bash
flux get kustomizations --watch
```

Check individual components:

```bash
# Vault
kubectl get pods -n vault

# External Secrets Operator
kubectl get pods -n external-secrets
kubectl get externalsecrets --all-namespaces

# cert-manager
kubectl get pods -n cert-manager
kubectl get certificates --all-namespaces

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

## Vault Secret Paths

| Vault Path | Keys | Used By |
|---|---|---|
| `secret/data/linkerd` | `license` | Linkerd control plane, Buoyant operator |
| `secret/data/linkerd-buoyant` | `client_id`, `client_secret` | Buoyant Cloud registration |

## Certificate Chain

cert-manager manages the full Linkerd certificate lifecycle:

- **Trust anchor** (`ca.yaml`) -- self-signed root CA, 365-day validity, auto-rotated
- **Identity issuer** (`identity.yaml`) -- intermediate CA for mTLS identity, 48h validity
- **Trust bundle** (`bundle.yaml`) -- distributes the root CA to control plane namespaces
- **Webhook certificates** (`webhook.yaml`) -- short-lived certs (24h) for admission webhooks (policy-validator, proxy-injector, sp-validator)

## Cleanup

```bash
k3d cluster delete linkerd
```
