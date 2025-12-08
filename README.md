RHCL Demo Lab — Argo CD, Gateway API, and External Secrets on OpenShift/Kubernetes

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.27%2B-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![OpenShift](https://img.shields.io/badge/OpenShift-4.x-E00?logo=redhatopenshift&logoColor=white)](https://www.redhat.com/en/technologies/cloud-computing/openshift)
[![Argo%20CD](https://img.shields.io/badge/Argo%20CD-GitOps-ef7b4d?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/)
[![Gateway%20API](https://img.shields.io/badge/Gateway%20API-v1-4c9aff)](https://gateway-api.sigs.k8s.io/)
[![External%20Secrets](https://img.shields.io/badge/External%20Secrets-Operator-4CAF50)](https://external-secrets.io/)
[![License](https://img.shields.io/badge/License-Apache--2.0-blue.svg)](./LICENSE)
[![Last Updated](https://img.shields.io/badge/Last%20Updated-2025--12--02_00%3A13_localtime-informational)](#)

Welcome! This repo is a minimal yet production‑flavored GitOps demo that shows how to ship two microservices behind Gateway API with secrets sourced from Vault — all continuously reconciled by Argo CD.

Quick links: [Quickstart](#quickstart-90-seconds) • [Architecture](#architecture) • [Deploy](#deploy-options) • [Verify](#verification) • [Troubleshoot](#troubleshooting) • [Customize](#customization)

Table of Contents

- [Quickstart (90 seconds)](#quickstart-90-seconds)
- [Repository Structure](#repository-structure)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
  - [Cluster/Operators](#clusteroperators)
  - [Access & DNS](#access--dns)
  - [Container Registry Access](#container-registry-access)
  - [Secrets in Vault](#secrets-in-vault)
- [Deploy Options](#deploy-options)
  - [Option 1 — Argo CD Applications](#option-1--argo-cd-applications)
  - [Option 2 — Argo CD ApplicationSet](#option-2--argo-cd-applicationset)
  - [Manual Apply (without Argo CD)](#manual-apply-without-argo-cd)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Customization](#customization)
- [Cleanup](#cleanup)
- [License](#license)

Quickstart (90 seconds)

1) Fork or use this repo as‑is. Ensure the required operators/CRDs are ready. See [Prerequisites](#prerequisites).
2) Apply via Argo CD (recommended):

```bash
kubectl apply -n openshift-gitops -f argocd/applications.yaml
```

3) Wait for sync, then verify:

```bash
kubectl get ns rhcl-demo-lab -L istio.io/rev -L msm3-discovery
kubectl -n rhcl-demo-lab get deploy,svc,pods
kubectl -n rhcl-demo-lab get externalsecret,secret db-creds -o wide
kubectl -n rhcl-demo-lab get gateway -o wide
```

4) Test routes (tweak domains if you changed hostnames):

```bash
curl -i http://orders.int.rhcl.arencloud.com/api/v1/orders/healthz
curl -i http://inventory.int.rhcl.arencloud.com/api/v1/inventory/healthz
```

Repository Structure

- `LICENSE` — License for this repository
- `app/`
  - `namespace.yaml` — Creates the `rhcl-demo-lab` namespace and applies service mesh labels
  - `inventory.yaml` — `Deployment` + `Service` for the Inventory microservice
  - `orders.yaml` — `Deployment` + `Service` for the Orders microservice
  - `secrets.yaml` — External Secrets Operator resource to pull values from Vault into a `Secret`
- `argocd/`
  - `applications.yaml` — Two Argo CD `Application` resources (microservices and gateway)
  - `applicationset.yaml` — An Argo CD `ApplicationSet` templating the same apps
- `gw/lan/`
  - `gateway.yaml` — Gateway API `Gateway` configured for hostname `*.int.rhcl.arencloud.com`
  - `httproute-inventory.yaml` — Routes `/api/v1/inventory` to the `inventory` service
  - `httproute-orders.yaml` — Routes `/api/v1/orders` to the `orders` service

Architecture

At a glance:
- Namespace: `rhcl-demo-lab`, labeled for Service Mesh control plane revision (`istio.io/rev: msm3-v1-27-3`) and discovery (`msm3-discovery: enabled`).
- Microservices: `orders` and `inventory` Deployments with HTTP readiness probes on `/healthz` port 8080.
- Ingress: Gateway API `Gateway` (gatewayClass `istio`) with `HTTPRoute`s per service and hostnames `orders.int.rhcl.arencloud.com` and `inventory.int.rhcl.arencloud.com`.
- Secrets: External Secrets Operator creates `Secret db-creds` from Vault `ClusterSecretStore` named `vault`, path/key `rhcl_demo_lab` with properties for DB URLs, Keycloak issuer/client, and role names.
- GitOps: Argo CD Applications or an ApplicationSet reconcile resources from this repo.

```mermaid
flowchart LR
  subgraph Client[Clients / cURL]
    o[orders.int.rhcl.arencloud.com]
    i[inventory.int.rhcl.arencloud.com]
  end

  %% Define Gateway node first (older Mermaid on GitHub is stricter with inline node creation)
  GW{{Gateway (istio)}}

  %% Ingress traffic from hostnames to Gateway
  o -->|HTTP:80 /api/v1/orders| GW
  i -->|HTTP:80 /api/v1/inventory| GW

  GW -->|HTTPRoute orders| SVC_O[(Service: orders:8080)]
  GW -->|HTTPRoute inventory| SVC_I[(Service: inventory:8080)]

  SVC_O --> POD_O1[(Deployment orders)]
  SVC_I --> POD_I1[(Deployment inventory)]

  subgraph Secrets[Secrets]
    V[(Vault: rhcl_demo_lab)] --> ESO[External Secrets Operator]
    ESO --> K8S[(Secret: db-creds)]
  end

  K8S -. env vars .-> POD_O1
  K8S -. env vars .-> POD_I1
```

Prerequisites

Cluster/Operators

- [ ] Argo CD installed (e.g., OpenShift GitOps Operator) and the `openshift-gitops` namespace available
- [ ] Istio or Red Hat Service Mesh with Gateway API integration and a `gatewayClassName: istio`
- [ ] Gateway API CRDs installed (v1 for `Gateway`/`HTTPRoute`)
- [ ] External Secrets Operator with a `ClusterSecretStore` named `vault` able to read from Vault
- [ ] Load balancer integration (e.g., MetalLB on bare metal). The Gateway annotations expect `metallb.io` pool `ip-addresspool`.

Access & DNS

- A DNS domain you control, with records for:
  - `orders.int.rhcl.arencloud.com`
  - `inventory.int.rhcl.arencloud.com`
  pointing to the external IP of the Gateway’s LoadBalancer service.
- If you do not control `*.int.rhcl.arencloud.com`, replace hostnames in `gw/lan/gateway.yaml` and `gw/lan/httproute-*.yaml`.

Container Registry Access

- Images used:
  - `quay.io/rh-ee-egevorky/rhcl/orders:0.1.0`
  - `quay.io/rh-ee-egevorky/rhcl/inventory:0.1.0`
- Ensure the cluster can pull from Quay (public images assumed). If private, add pull secrets.

Secrets in Vault

The `app/secrets.yaml` expects the following key/value properties at Vault path `rhcl_demo_lab` (via `ClusterSecretStore` `vault`):

- `INVENTORY_DB_URL`
- `ORDERS_DB_URL`
- `KEYCLOAK_ISSUER`
- `KEYCLOAK_CLIENT_ID`
- `INVENTORY_READ_ROLE`
- `INVENTORY_WRITE_ROLE`
- `INVENTORY_DELETE_ROLE`
- `ORDERS_READ_ROLE`
- `ORDERS_WRITE_ROLE`
- `ORDERS_DELETE_ROLE`

These values become a Kubernetes `Secret` named `db-creds` in the `rhcl-demo-lab` namespace and are consumed by the microservices as environment variables.

Deploy Options

Option 1 — Argo CD Applications

Apply the Argo CD Applications into the Argo CD control namespace (default: `openshift-gitops` on OpenShift):

```bash
kubectl apply -n openshift-gitops -f argocd/applications.yaml
```

What it does:

- `rhcl-microservices` syncs everything under `app/` (namespace, services, deployments, externalsecret)
- `rhcl-kuadrant` syncs everything under `gw/lan/` (gateway and routes)

Option 2 — Argo CD ApplicationSet

Apply the ApplicationSet:

```bash
kubectl apply -n openshift-gitops -f argocd/applicationset.yaml
```

What it does:

- Generates two Applications from a list generator:
  - `rhcl-microservices` → path `app`
  - `rhcl-kuadrant` → path `gw/lan`

Note: Both options set `syncPolicy.automated.prune: true` and `.selfHeal: true`, and include `syncOptions: [CreateNamespace=true]` so the target namespace will be created if missing.

Manual Apply (without Argo CD)

While GitOps is preferred, you can apply resources manually in the following order:

```bash
kubectl apply -f app/namespace.yaml
kubectl apply -f app/secrets.yaml
kubectl apply -f app/inventory.yaml
kubectl apply -f app/orders.yaml
kubectl apply -f gw/lan/gateway.yaml
kubectl apply -f gw/lan/httproute-inventory.yaml
kubectl apply -f gw/lan/httproute-orders.yaml
```

Verification

After Argo CD syncs or manual apply:

- Check namespace and workloads

```bash
kubectl get ns rhcl-demo-lab -L istio.io/rev -L msm3-discovery
kubectl -n rhcl-demo-lab get deploy,svc,pods
```

- Confirm `ExternalSecret` synced and Secret materialized

```bash
kubectl -n rhcl-demo-lab get externalsecret,secret db-creds -o wide
```

- Inspect Gateway and IP assignment

```bash
kubectl -n rhcl-demo-lab get gateway -o wide
kubectl get svc -A | grep -i loadbalancer
```

- Test routes (replace domains if customized)

```bash
curl -i http://orders.int.rhcl.arencloud.com/api/v1/orders/healthz
curl -i http://inventory.int.rhcl.arencloud.com/api/v1/inventory/healthz
```

Troubleshooting

<details>
<summary>CRD not found / IDE schema warnings</summary>

Some files use CRDs from Argo CD, Gateway API, and External Secrets. If your IDE flags unknown kinds like `Application`, `ApplicationSet`, `Gateway`, `HTTPRoute`, or `ExternalSecret`, ensure the corresponding CRDs/operators are installed in the cluster. These warnings do not affect GitOps if operators are present.

</details>

<details>
<summary>Gateway has no external IP</summary>

Ensure a load balancer is available. On bare metal clusters, install and configure MetalLB. Verify `metallb.io` annotations reference a valid address pool.

</details>

<details>
<summary>DNS resolution fails</summary>

Make sure `orders.int.rhcl.arencloud.com` and `inventory.int.rhcl.arencloud.com` resolve to the Gateway’s external IP. Update the route hostnames and DNS records to match your domain.

</details>

<details>
<summary>Pods can’t start due to missing secrets</summary>

Verify the External Secrets Operator is running and the `ClusterSecretStore` named `vault` is configured and authorized to read path `rhcl_demo_lab`.

</details>

<details>
<summary>Traffic not reaching pods</summary>

Confirm the namespace has correct mesh labels and that the `gatewayClassName: istio` exists. Check `HTTPRoute` `parentRefs` and path prefixes.

</details>

Customization

- Namespace and mesh labels: edit `app/namespace.yaml` to change name or mesh revision labels.
- Images and replicas: edit `app/inventory.yaml` and `app/orders.yaml`.
- Hostnames and paths: edit `gw/lan/gateway.yaml` and `gw/lan/httproute-*.yaml`.
- Git repo and branch: edit `argocd/applications*.yaml` to point to your fork and desired `targetRevision`.

Cleanup

If deployed via Argo CD Applications:

```bash
kubectl delete -n openshift-gitops -f argocd/applications.yaml
```

If deployed via ApplicationSet:

```bash
kubectl delete -n openshift-gitops -f argocd/applicationset.yaml
```

Then remove namespace (will also remove workloads/secrets/gateway/routes):

```bash
kubectl delete ns rhcl-demo-lab
```

License

This project is licensed under the terms of the `LICENSE` file included in this repository.
