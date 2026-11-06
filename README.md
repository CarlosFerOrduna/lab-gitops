# lab-gitops

The desired state of the lab clusters. ArgoCD applies whatever is in this repository: a change here is a change in the cluster.

## Status

Today:

- `system/sealed-secrets/base/`: the Sealed Secrets controller v0.41.2, in `kube-system`.
- `system/cert-manager/base/`: cert-manager v1.21.3, in `cert-manager`.
- `system/cert-manager/issuers/`: the lab's own certificate authority: a root CA (`lab-root-ca`), the `lab-ca` ClusterIssuer, which signs with it, and a second root, `kafka-clients-ca`, with its ClusterIssuer, only for Kafka's clients.
- `system/strimzi/base/`: the Strimzi 1.3.1 cluster operator, in `strimzi`, watching only `platform`.
- `platform/kafka/base/`: the Kafka cluster `events` (KRaft, one node), its TLS listener certificate, the producers' topics, and one `KafkaUser` per app with its ACLs. Each app requests its client certificate in its own namespace (`kafka-client.certificate.yaml` in its base).
- `namespaces/platform/`: the `platform` namespace, apart from the apps in it, so deleting or pruning an app never deletes the namespace and the volumes of its databases.
- `platform/processor/base/`: the processor (Deployment with 2 replicas and Service `processor`, port 80), its PostgreSQL (StatefulSet with a 1 Gi volume, headless Service), its configuration (ConfigMap) and its password (SealedSecret). The image tag in `kustomization.yaml` is the tree hash of the lab-processor commit it was built from. Each replica runs the migrations in an init container, under a PostgreSQL advisory lock.
- `namespaces/producer-a/` and `producer-a/base/`: producer-a's API (Deployment with 2 replicas, Service `producer-a`), its worker (Deployment), and its queue: Valkey in a StatefulSet with a 1 Gi volume (AOF, `noeviction`), its configuration (ConfigMap) and its password (SealedSecret). The worker publishes to `http://processor.platform`.
- `namespaces/producer-b/` and `producer-b/base/`: producer-b's API (Deployment with 2 replicas and an init container for the migrations, Service `producer-b`), its worker, its PostgreSQL (StatefulSet), its queue (Valkey StatefulSet, as producer-a's) and its cache (Valkey Deployment without a volume), each with its own SealedSecret.

Until ArgoCD arrives, changes are applied by hand from the bastion, always from a commit of this repository on GitHub, never from a local copy:

```bash
ssh lab-bastion kubectl apply -k 'https://github.com/CarlosFerOrduna/lab-gitops//system/sealed-secrets/base?ref=main'
```

cert-manager goes in two steps: its issuers need its CRDs and its webhook, so they are applied once the webhook is available:

```bash
ssh lab-bastion kubectl apply -k 'https://github.com/CarlosFerOrduna/lab-gitops//system/cert-manager/base?ref=main'
ssh lab-bastion kubectl -n cert-manager wait --for=condition=Available deployment --all --timeout=180s
ssh lab-bastion kubectl apply -k 'https://github.com/CarlosFerOrduna/lab-gitops//system/cert-manager/issuers?ref=main'
```

Strimzi's CRDs are larger than the annotation a client-side apply keeps, so Strimzi is always applied, and diffed, server-side:

```bash
ssh lab-bastion kubectl apply --server-side -k 'https://github.com/CarlosFerOrduna/lab-gitops//system/strimzi/base?ref=main'
ssh lab-bastion kubectl diff --server-side -k 'https://github.com/CarlosFerOrduna/lab-gitops//system/strimzi/base?ref=main'
```

Apply a namespace before the apps in it:

```bash
ssh lab-bastion kubectl apply -k 'https://github.com/CarlosFerOrduna/lab-gitops//namespaces/platform?ref=main'
ssh lab-bastion kubectl apply -k 'https://github.com/CarlosFerOrduna/lab-gitops//platform/processor/base?ref=main'
ssh lab-bastion kubectl apply -k 'https://github.com/CarlosFerOrduna/lab-gitops//platform/kafka/base?ref=main'
ssh lab-bastion kubectl apply -k 'https://github.com/CarlosFerOrduna/lab-gitops//namespaces/producer-a?ref=main'
ssh lab-bastion kubectl apply -k 'https://github.com/CarlosFerOrduna/lab-gitops//producer-a/base?ref=main'
ssh lab-bastion kubectl apply -k 'https://github.com/CarlosFerOrduna/lab-gitops//namespaces/producer-b?ref=main'
ssh lab-bastion kubectl apply -k 'https://github.com/CarlosFerOrduna/lab-gitops//producer-b/base?ref=main'
```

### The processor's database

- PostgreSQL 18 keeps its data under `/var/lib/postgresql` (`PGDATA` is `/var/lib/postgresql/18/docker`): that is where the volume is mounted.
- The StatefulSet keeps its volume when it is deleted or scaled down (`persistentVolumeClaimRetentionPolicy: Retain`). Deleting the namespace still deletes it.
- The volume asks for 1 Gi, but `local-path` does not enforce that size: it is a directory on the node's disk, and it cannot be resized.

The password was generated and sealed on the bastion, so its plain value never left it:

```bash
ssh lab-bastion 'openssl rand -base64 32 | tr -d "\n" \
  | kubectl create secret generic processor-db --namespace platform --from-file=password=/dev/stdin --dry-run=client -o yaml \
  | kubeseal --format yaml' > platform/processor/base/processor-db.sealedsecret.yaml
```

To read it, from the bastion (it prints the password: do it only when needed, and never paste it anywhere):

```bash
ssh lab-bastion "kubectl -n platform get secret processor-db -o jsonpath='{.data.password}' | base64 -d; echo"
```

PostgreSQL reads `POSTGRES_PASSWORD` only when it initializes an empty volume: changing the Secret afterwards does not change the database password. Rotating it needs its own procedure (`ALTER ROLE`, then a new SealedSecret), not written yet.

### The producers' queues

Each producer has its own Valkey for BullMQ: a StatefulSet with AOF (`appendfsync everysec`) and `noeviction`, `maxmemory 128mb`, and a memory limit of 256 Mi, about twice `maxmemory`, because rewriting the AOF forks the process. It runs as its own user (999) from the start.

The password is one SealedSecret with two keys made from the same random value on the bastion: `auth.conf` (`requirepass …`), mounted as a file and loaded with `include`, and `password`, for the apps (`QUEUE_PASSWORD`) and for the probes (`REDISCLI_AUTH`). It never appears in a command line.

```bash
ssh lab-bastion 'pw=$(openssl rand -hex 24); kubectl create secret generic valkey-queue-auth --namespace producer-a \
  --from-literal=password="$pw" --from-literal=auth.conf="requirepass $pw" --dry-run=client -o yaml \
  | kubeseal --format yaml' > producer-a/base/valkey-queue-auth.sealedsecret.yaml
```

### producer-b's database and cache

producer-b's PostgreSQL follows the processor's: its own StatefulSet, volume and SealedSecret (`producer-b-db`), and the migrations in an init container of the API.

Its data came from the PostgreSQL in producer-b's compose, once: the cluster is now the source of truth, and the compose keeps its PostgreSQL only for local development, with data that is no longer kept in sync. The copy ran with producer-b and its worker stopped in the compose, so nothing wrote during it. Only the rows of `item`: the schema is created by the migrations, and `--on-conflict-do-nothing` makes a second run insert nothing.

```bash
docker compose stop producer-b producer-b-worker   # in lab-producer-b; only postgres keeps running
docker compose exec -T postgres pg_dump -U producer_b -d producer_b --data-only --table=item --inserts --on-conflict-do-nothing \
  | ssh lab-bastion kubectl -n producer-b exec -i producer-b-postgres-0 -- psql -U producer_b -d producer_b -v ON_ERROR_STOP=1
```

Both sides are compared with a count and an md5 over `id`, `name`, `quantity` and `extract(epoch from created_at)`: a `timestamptz` printed as text depends on the session's time zone, its epoch does not.

The cache is a Valkey Deployment without a volume: no persistence, `allkeys-lru`, `maxmemory 64mb`, a memory limit of 128 Mi, and its own password (`valkey-cache-auth`). The API never depends on it: any cache error is a miss, and readiness does not check it.

Readiness checks the database and the queue. With the queue down, both replicas leave the Service and reads (`GET /items`) fail too, although they would not need the queue. This is an accepted decision: an API that cannot queue the events of new items is not ready, and splitting reads and writes into separate Services is not worth it here.

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

### The lab's certificate authority

cert-manager issues every certificate of the lab from its own root, without anything copied by hand:

- `lab-selfsigned` (ClusterIssuer) signs only the lab's two roots, each with its own key.
- `lab-root-ca` (Certificate in `cert-manager`, `isCA`) is the root: ECDSA P-256, 10 years, renewed one year before it expires. Its key is never rotated on renewal (`rotationPolicy: Never`), so the certificates issued before still validate against the renewed root. Replacing the key is a rotation, with its own procedure (stage 11).
- `lab-ca` (ClusterIssuer) signs the lab's certificates with the root. Each certificate it issues carries the root as `ca.crt`, so whoever holds one can verify the others.
- `kafka-clients-ca` (Certificate in `cert-manager`, `isCA`, self-signed; 10 years, `rotationPolicy: Never`) is a second root, and its ClusterIssuer signs only Kafka's client certificates. It is a root and not an intermediate of `lab-root-ca` because the brokers' truststore must hold it alone (see Kafka below), and the `ca.crt` that cert-manager writes is always the root of the chain. The brokers trust only this root, so no other certificate of the lab (the ingress's, for one) is a valid Kafka client.

The roots' keys live only in their Secrets, in the `cert-manager` namespace. Each certificate is requested by a `Certificate` in the namespace that uses it, and its key is created there and never leaves it.

### The Strimzi operator

`strimzi-cluster-operator-1.3.1.yaml` is the upstream manifest, unchanged; everything specific to the lab is a patch in `kustomization.yaml`:

- The operator runs in `strimzi` and watches only `platform` (`STRIMZI_NAMESPACE`). Its rights over the Kafka cluster are RoleBindings in `platform`; its leader election stays in `strimzi`.
- The upstream bindings name the placeholder namespace `myproject` in their subjects, which Kustomize's `namespace` field does not rewrite: the patches point them at `strimzi`. That field is not used at all, because it runs after the patches and would move the `platform` RoleBindings back to `strimzi`.
- The operator image, and the images it starts (Kafka 4.3.2, topic and user operators, init), are pinned by digest. `STRIMZI_KAFKA_IMAGES` keeps the whole upstream list, because the operator refuses to start unless every supported Kafka version has an image; only 4.3.2, the version the lab runs, is pinned by digest. Moving to another version means pinning its image there first.

Watching a new namespace means adding it to `STRIMZI_NAMESPACE` (comma-separated) and the three RoleBindings in it.

### Kafka

`events` runs in `platform`, managed by the Strimzi operator: one KRaft node that is both controller and broker (`KafkaNodePool` `dual-role`), with a 512 MB heap, 1 Gi of memory (request = limit) and its log on a `local-path` volume kept when the cluster is deleted (`deleteClaim: false`). With one broker every replication factor is 1, and `acks=all` is the same as `acks=1`. Topics are created only from `KafkaTopic` resources (`auto.create.topics.enable: false`).

The apps reach it at `events-kafka-bootstrap.platform.svc:9093`, its only listener: TLS, with mutual authentication.

- The brokers present `events-kafka-listener-tls`, issued by `lab-ca` for the bootstrap service and every broker. Kafka loads its key as a PEM keystore, so it is PKCS#8. When cert-manager renews it, Strimzi rolls the broker: with one node, that is a short outage.
- Each app requests, in its own namespace, `kafka-client` (Secret `kafka-client-tls`) from `kafka-clients-ca`, with exactly `CN=<app>` as subject; the key is created there and never leaves it. It verifies the brokers with the `ca.crt` of `lab-root-ca-ref`, a certificate from `lab-ca` whose only use is to carry the lab's root into the namespace. Strimzi's internal cluster CA stays in `platform`.
- The brokers trust only certificates issued by `kafka-clients-ca` (`clientsCa.type: cert-manager`, Early Access in Strimzi 1.3), read from the `ca.crt` of `kafka-clients-ca-ref`, a certificate in `platform` whose only use is to carry that root. Strimzi never holds the key of either root. That `ca.crt` must hold the client root alone: Java takes every certificate in a truststore as a trust anchor, so with `lab-root-ca` in it any certificate of the lab would be a valid client.
- The roots renew with `rotationPolicy: Never`: a renewed root keeps its key and subject, so the old copy in Kafka's truststore, and in each `lab-root-ca-ref`, keeps validating until it expires. What is pending is making sure the `-ref` certificates that carry a root (`kafka-clients-ca-ref` for `kafka-clients-ca`, every `lab-root-ca-ref` for `lab-root-ca`) are renewed before the old copy expires (`cmctl renew`). Replacing a root's key is a rotation, with its own procedure (stage 11).
- The `KafkaUser` of each app (`tls-external`) holds only its ACLs, for the principal `CN=<app>`. Each producer can write only to the topics that start with its name; the processor can read both producers' topics and commit as the group `processor`.

Any namespace can request a certificate from `kafka-clients-ca` with any CN: what keeps an app from claiming another app's identity is the review of this repository, not the cluster. Stage 08 enforces it with cert-manager's approver-policy, one `CertificateRequestPolicy` per namespace (in `producer-b`, only `CN=producer-b` and only from `kafka-clients-ca`). That needs cert-manager's default approver turned off (`--controllers=-certificaterequests-approver` in the `cert-manager` Deployment's args): otherwise it approves every request before the policy is evaluated.

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
