# Variant: prod on kiac (Kubernetes in Apple Containers)

In this variant the **production cluster is a kiac cluster**, where the node is
its own lightweight VM instead of a Docker container. The recovery cluster stays
on kind, exactly as in the main guide. The result on stage:

- Act 2's disaster kills a **real VM with its own kernel**, not a container
  pretending to be a machine.
- Act 2's restore becomes a **cross-substrate recovery**: a backup taken on one
  kind of infrastructure, restored onto another. That is scorecard question 8
  in its strongest form.
- Act 3 is **unchanged**: it runs entirely on the kind recovery cluster.
- The optional vind appendix is **unchanged**.

Status: tested end to end once on 2026-07-25 (Apple silicon, macOS, 64 GB).
Backup, delete, restore on kiac; control-plane VM kill and resume; and
cross-substrate restore into the kind recovery cluster all passed. Rehearse the
full show at least three times before using it at a conference. If anything
fights you, the main guide's all-kind path remains fully valid.

## 1. What changes vs the main guide

| Piece | Main guide | This variant |
|---|---|---|
| Prod cluster | `kind-dr-prod` (container node) | `kiac-drprod` (VM node) |
| Recovery cluster | `kind-dr-dr` | `kind-dr-dr` (unchanged) |
| Backup vault | SeaweedFS on the `kind` Docker network | same container, reached via the published host port |
| Prod Velero S3 URL | `http://dr-seaweed:8333` | `http://192.168.64.1:9200` (host gateway) |
| DR Velero S3 URL | `http://dr-seaweed:8333` | `http://dr-seaweed:8333` (unchanged) |
| Act 2 disaster | `docker stop dr-prod-control-plane` | `container stop kiac-drprod-control-plane` |
| Bring prod back after | `docker start dr-prod-control-plane` | `kiac resume cluster --name drprod` |
| Gitea, Argo CD, ledger, Act 3, Act 4 | - | unchanged |

Why SeaweedFS: `minio/minio` was archived on GitHub in 2026. The pinned MinIO
image from the main guide keeps working offline forever, but this variant uses
SeaweedFS (Apache-2.0, actively maintained) so the slide does not advertise an
archived project. Velero only needs an S3-compatible endpoint; both work.

## 2. Extra prerequisites and tested versions

On top of the main guide's CLI list:

- Apple silicon Mac (the node VMs are arm64); 32 GB RAM or more recommended
- `container` (apple/container) 1.0.0
- `kiac` 0.4.0: `brew install saiyam1814/tap/kiac`
- SeaweedFS `chrislusf/seaweedfs:4.40@sha256:52194fba4fecd0083c842158b3a902ba6e04a63619b2b0efcd08007bdb6a4602`

Everything else matches the main guide's tested versions. The kiac cluster
boots the same `kindest/node` v1.36.1 image (same digest) as the kind
clusters, so the Kubernetes, CSI, and snapshotter behavior matches the tested
matrix.

## 3. Networking model (read once)

- kiac node VMs get IPs like `192.168.64.x` from apple/container's network.
  The Mac itself is reachable from inside the VMs at the gateway,
  `192.168.64.1`. Confirm your subnet with `container list` (node IP column);
  the gateway is `.1` on that subnet.
- The vault container joins the `kind` Docker network (so the kind DR cluster
  reaches it by name) **and** publishes host port 9200 (so kiac pods reach it
  via the gateway).
- A stopped kiac VM gets a **fresh IP** on next boot. `kiac resume cluster`
  heals the pinned control-plane address automatically. The gateway IP that
  Velero uses does not change.
- During offline preflight, verify the pod-to-gateway path with Wi-Fi off
  (section 8). Local interfaces normally survive, but prove it on your machine.

## 4. Create the kiac prod cluster

```bash
kiac create cluster --name drprod --cp-memory 6G
kubectl --context kiac-drprod get nodes
```

Single node, Kubernetes v1.36.1, ready in about 90 seconds.

## 5. The shared SeaweedFS vault

The main guide's section 8 already creates the `dr-seaweed` container with the
published host port 9200, so there is nothing new to start. Sanity checks:

```bash
curl -fsS -o /dev/null -w 'host port: %{http_code}\n' http://127.0.0.1:9200

kubectl --context kiac-drprod run nettest --restart=Never \
  --image='busybox:1.36.1@sha256:73aaf090f3d85aa34ee199857f03fa3a95c8ede2ffd4cc2cdb5b94e566b11662' \
  -- sh -c 'wget -q -S -O /dev/null http://192.168.64.1:9200 2>&1 | head -1'
kubectl --context kiac-drprod wait pod/nettest \
  --for=jsonpath='{.status.phase}'=Succeeded --timeout=90s
kubectl --context kiac-drprod logs nettest      # expect an HTTP status line
kubectl --context kiac-drprod delete pod nettest
```

## 6. Install the snapshot stack and Velero on the kiac cluster

Run the main guide's sections 6 and 7 (snapshot controller, CSI hostpath
driver, storage classes) once with:

```bash
export CTX=kiac-drprod
```

The commands are identical; they were tested unchanged on the kiac node.

Then Velero, pointing at the vault through the host gateway:

```bash
cat >/tmp/velero-minio-creds <<'EOF'
[default]
aws_access_key_id=minio
aws_secret_access_key=minio123
EOF

velero install \
  --kubecontext kiac-drprod \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.14.0 \
  --bucket velero \
  --secret-file /tmp/velero-minio-creds \
  --backup-location-config \
    'region=seaweed,s3ForcePathStyle=true,s3Url=http://192.168.64.1:9200' \
  --use-node-agent \
  --use-volume-snapshots=false \
  --features=EnableCSI \
  --wait

kubectl --context kiac-drprod -n velero get backupstoragelocation   # Available
```

On the **recovery cluster** nothing changes: the main guide's Velero install
already points at `http://dr-seaweed:8333`, so both clusters share the same
bucket and the DR side syncs the prod backups automatically (about a minute).

## 7. Seed prod and create the rehearsal backup

Run the main guide's section 10 with `PROD_CTX=kiac-drprod`. Verify the moved
bytes:

```bash
kubectl --context kiac-drprod -n velero get datauploads \
  -o custom-columns='NAME:.metadata.name,PHASE:.status.phase,BYTES:.status.progress.bytesDone'
```

Gitea and Argo CD setup (sections 11 and 12) are unchanged: they live on the
Docker side and only talk to the DR cluster.

## 8. Offline preflight additions

With Wi-Fi off, on top of the main guide's section 14:

```bash
kubectl --context kiac-drprod get --raw=/readyz
kubectl --context kiac-drprod -n velero get backupstoragelocation      # Available
curl -fsS -o /dev/null http://127.0.0.1:9200 && echo vault ok
```

If the kiac backup location is not Available offline, the pod-to-gateway path
is the suspect; do not go on stage with this variant until it passes.

## 9. Act deltas

**Act 1** - identical commands, `PROD_CTX=kiac-drprod`. The talking point
upgrades: the volume bytes crossed a hypervisor boundary to reach the vault.

**Act 2** - the disaster and the reset change; everything else is identical:

```bash
date
container stop kiac-drprod-control-plane
kubectl --context kiac-drprod get nodes --request-timeout=4s || true
```

You did not stop a container sharing your kernel. You powered off a machine.
The restore into the kind DR cluster is then a cross-substrate recovery:
different node runtime, different infrastructure, same application and data.

After the talk (or between rehearsals):

```bash
kiac resume cluster --name drprod
```

The VM boots with a fresh IP; `kiac resume` rewrites the pinned control-plane
address and the cluster returns Ready with its data intact (verified: the
guestbook rows and the Velero location survive the kill and resume).

**Act 3** - no changes. It runs on the kind recovery cluster.

**Act 4** - no changes.

## 10. Reset and teardown

```bash
# between rehearsals
kiac resume cluster --name drprod
kubectl --context kind-dr-dr delete namespace guestbook --ignore-not-found

# full teardown of variant-only pieces
kiac delete cluster --name drprod
docker rm -f dr-seaweed
docker volume rm dr-seaweed-data
```
