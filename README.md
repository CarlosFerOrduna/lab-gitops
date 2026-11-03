# lab-gitops

The desired state of the lab clusters. ArgoCD applies whatever is in this repository: a change here is a change in the cluster.

## Status

Today:

- `system/sealed-secrets/base/`: the Sealed Secrets controller v0.41.2, in `kube-system`.

Until ArgoCD arrives, changes are applied by hand from the bastion, always from a commit of this repository on GitHub, never from a local copy:

```bash
ssh lab-bastion kubectl apply -k 'https://github.com/CarlosFerOrduna/lab-gitops//system/sealed-secrets/base?ref=main'
```

### Sealed Secrets

`controller.yaml` is the upstream manifest of the release, copied unchanged after checking its SHA256 against the release's checksums file, and `kustomization.yaml` pins the controller image by digest. To upgrade:

1. Download `controller.yaml` and `sealed-secrets_<version>_checksums.txt` from the release, and check the SHA256.
2. Replace `controller.yaml`, update the version, the SHA256 and the image digest in `kustomization.yaml`.
3. Upgrade `kubeseal` on the bastion to the same version (`kubeseal_version` in lab-infra's `bastion` role).

The controller's private key lives only in the cluster. Recreating the cluster creates a new key, so secrets sealed before can no longer be decrypted: backing up and restoring that key is planned for a later stage.

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
