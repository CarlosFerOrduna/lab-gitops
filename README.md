# lab-gitops

The desired state of the lab clusters. ArgoCD applies whatever is in this repository: a change here is a change in the cluster.

## Status

Today: only this README.

Planned layout:

```text
system/              cluster-wide components
  argocd/
  cert-manager/
  metallb/
  strimzi/
  sealed-secrets/
platform/            shared services
  kafka/
  processor/
producer-a/          producer-a namespace
producer-b/          producer-b namespace
```

Each component has a Kustomize `base/` and one overlay per environment:

```text
<component>/
  base/
  overlays/
    dev/
    stage/
    prod/
```

## Rules

- Configuration that differs between environments lives only in the overlays. The base holds what every environment shares.
- Secrets are committed only as SealedSecrets. A plain Kubernetes `Secret` never lands in this repository.

## How it ties in

- The service repositories build one image per tree hash. Their pipeline updates the image tag in the matching overlay here.
- ArgoCD syncs dev and stage automatically. Prod waits for a manual sync.

## Lab repositories

| Repository                                                              | Role                                                                                                         |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [lab-infra](https://github.com/CarlosFerOrduna/lab-infra)               | Builds the infrastructure: Hyper-V VMs today, Ansible and k3s next. Holds the repository settings checklist. |
| [lab-gitops](https://github.com/CarlosFerOrduna/lab-gitops)             | Desired state of the clusters. ArgoCD applies whatever is in this repo.                                      |
| [lab-ci-workflows](https://github.com/CarlosFerOrduna/lab-ci-workflows) | Reusable GitHub Actions workflows shared by every lab repo.                                                  |
| [lab-processor](https://github.com/CarlosFerOrduna/lab-processor)       | Shared processor service, namespace `platform`.                                                              |
| [lab-producer-a](https://github.com/CarlosFerOrduna/lab-producer-a)     | Producer service, namespace `producer-a`.                                                                    |
| [lab-producer-b](https://github.com/CarlosFerOrduna/lab-producer-b)     | Producer service, namespace `producer-b`.                                                                    |
