# Manual Demo Guide

This is the committed, step-by-step version of the demo. It contains no
automation dependency.

The main presentation uses one continuous demo in three acts. The vind drill is
an optional appendix.

## 1. What each local component represents

| Local component | Production meaning |
|---|---|
| `drprod` kiac cluster (each node its own VM) | Active cloud region or primary data center |
| `dr-dr` kind cluster | Recovery region, account, provider, or secondary data center |
| SeaweedFS (local S3) | Recovery vault or offsite object store |
| Gitea | Git and infrastructure configuration outside production |
| Stop the prod control-plane VM | Loss of the source failure domain |
| SQL row check | Business-level recovery validation |

Production runs on [kiac](https://github.com/saiyam1814/kiac): the node is a
real lightweight VM with its own kernel, so stopping it is a machine loss, not
a container stop. The recovery cluster runs on kind. The restore in Demo 2 is
therefore a cross-substrate recovery: a backup taken on one kind of
infrastructure, restored onto another. Both boot the same `kindest/node` image,
so the Kubernetes and CSI behavior is identical.

**No Apple silicon?** Use kind for prod too: create it with
`kind create cluster --name dr-prod --image "$KIND_IMAGE" --wait 120s`, set
`PROD_CTX=kind-dr-prod`, use the in-network vault URL
`http://dr-seaweed:8333` in the prod Velero install, and stop prod in Demo 2
with `docker stop dr-prod-control-plane`. Everything else is unchanged.

## 2. Required CLIs and tested versions

Install these before starting: `docker`, `kind`, `kubectl`, `velero`, `git`,
`curl`, `jq`, and `envsubst` (from gettext). For the kiac production cluster
(Apple silicon): `container` (apple/container) and
`brew install saiyam1814/tap/kiac`. The optional appendix also needs the
`vcluster` CLI.

Tested versions:

- Docker Desktop 29.5.2
- kind 0.32.0
- kiac 0.4.0 on apple/container 1.0.0 (Apple silicon)
- Kubernetes 1.36.1
- kubectl 1.36.3
- Velero 1.18.2
- Velero AWS plugin 1.14.0
- external-snapshotter 8.6.0
- csi-driver-host-path repository 1.17.1
- hostpath plugin 1.17.1
- Argo CD 3.4.5
- PostgreSQL 16.14
- BusyBox 1.36.1
- Gitea 1.24.7
- SeaweedFS 4.40 (S3-compatible vault; `minio/minio` is archived upstream)
- MinIO `mc` client `RELEASE.2025-08-13` (bucket creation only)
- vCluster CLI 0.36.0

All application and local service images in the manifests are pinned to an exact
version or digest.

## 3. Important offline rule

Initial setup requires internet access to download pinned images and manifests.
The conference acts do not.

Prepare everything days before the talk. Do not delete the kiac cluster, the
kind cluster, the Docker volumes, or local demo artifacts after preparation.

## 4. Set common values

Every block in this guide starts from the `demo/` directory; the first line
below puts you there from anywhere inside the repo checkout. Re-run this whole
block in every new terminal.

```bash
cd "$(git rev-parse --show-toplevel)/demo"

export PROD_CTX=kiac-drprod
export DR_CLUSTER=dr-dr
export DR_CTX=kind-dr-dr

export KIND_IMAGE='kindest/node:v1.36.1@sha256:3489c7674813ba5d8b1a9977baea8a6e553784dab7b84759d1014dbd78f7ebd5'
export SEAWEED_IMAGE='chrislusf/seaweedfs:4.40@sha256:52194fba4fecd0083c842158b3a902ba6e04a63619b2b0efcd08007bdb6a4602'
export MC_IMAGE='minio/mc:RELEASE.2025-08-13T08-35-41Z@sha256:a7fe349ef4bd8521fb8497f55c6042871b2ae640607cf99d9bede5e9bdf11727'
export GITEA_IMAGE='gitea/gitea:1.24.7@sha256:918955f16b1e91732af6c449bb2db3a34271748dbed1ccfbae48f8a2fb5480b8'
export POSTGRES_IMAGE='postgres:16.14-alpine@sha256:57c72fd2a128e416c7fcc499958864df5301e940bca0a56f58fddf30ffc07777'
export BUSYBOX_IMAGE='busybox:1.36.1@sha256:73aaf090f3d85aa34ee199857f03fa3a95c8ede2ffd4cc2cdb5b94e566b11662'
```

## 5. Create the two clusters

Production on kiac (single VM node, ~90 seconds):

```bash
kiac create cluster --name drprod --cp-memory 6G
kubectl --context "$PROD_CTX" get nodes
```

Recovery on kind:

```bash
kind create cluster --name "$DR_CLUSTER" --image "$KIND_IMAGE" --wait 120s
```

Skip a command if that cluster already exists.

Note the kiac networking model once: the node VM gets an IP like
`192.168.64.x`, and the Mac itself is reachable from inside the VM at the
subnet gateway, `192.168.64.1` (confirm your subnet in the IP column of
`container list`). The prod cluster reaches the vault through that gateway.
A stopped VM gets a fresh IP on the next boot; `kiac resume cluster --name
drprod` heals everything that pins the old address. The gateway IP itself
never changes.

## 6. Install VolumeSnapshot and VolumeGroupSnapshot APIs

Run these commands once for each context:

```bash
export CTX="$PROD_CTX"  # repeat later with CTX="$DR_CTX"

kubectl --context "$CTX" apply -k \
  'https://github.com/kubernetes-csi/external-snapshotter/client/config/crd?ref=v8.6.0'

kubectl --context "$CTX" apply -k \
  'https://github.com/kubernetes-csi/external-snapshotter/deploy/kubernetes/snapshot-controller?ref=v8.6.0'

kubectl --context "$CTX" -n kube-system set image deployment/snapshot-controller \
  snapshot-controller=registry.k8s.io/sig-storage/snapshot-controller:v8.6.0

kubectl --context "$CTX" -n kube-system patch deployment snapshot-controller \
  --type json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--feature-gates=CSIVolumeGroupSnapshot=true"}]'

kubectl --context "$CTX" -n kube-system rollout status \
  deployment/snapshot-controller --timeout 180s
```

The patch may report that the value already exists on a repeat run. That is safe.

## 7. Install the hostpath CSI test driver

Clone once while online:

```bash
cd "$(git rev-parse --show-toplevel)/demo"

git clone --depth 1 --branch v1.17.1 \
  https://github.com/kubernetes-csi/csi-driver-host-path.git
```

For each context:

```bash
cd "$(git rev-parse --show-toplevel)/demo"
export CTX="$PROD_CTX"  # repeat later with CTX="$DR_CTX"

kubectl config use-context "$CTX"
./csi-driver-host-path/deploy/kubernetes-latest/deploy.sh

kubectl --context "$CTX" set image statefulset/csi-hostpathplugin \
  hostpath=registry.k8s.io/sig-storage/hostpathplugin:v1.17.1 \
  csi-snapshotter=registry.k8s.io/sig-storage/csi-snapshotter:v8.6.0

kubectl --context "$CTX" set image statefulset/csi-hostpath-socat \
  socat=registry.k8s.io/sig-storage/hostpathplugin:v1.17.1

export SNAPSHOTTER_INDEX=$(
  kubectl --context "$CTX" get statefulset csi-hostpathplugin -o json |
  jq '[.spec.template.spec.containers[].name] | index("csi-snapshotter")'
)

kubectl --context "$CTX" patch statefulset csi-hostpathplugin --type json \
  -p="[{\"op\":\"add\",\"path\":\"/spec/template/spec/containers/${SNAPSHOTTER_INDEX}/args/-\",\"value\":\"--feature-gates=CSIVolumeGroupSnapshot=true\"}]"

kubectl --context "$CTX" rollout status statefulset/csi-hostpathplugin \
  --timeout 180s

kubectl --context "$CTX" apply -f manifests/storage/storageclass.yaml
kubectl --context "$CTX" apply -f manifests/storage/snapshotclasses.yaml
```

This driver is for the local test only. It is not a production storage
recommendation.

## 8. Start the persistent local S3 vault (SeaweedFS)

```bash
mkdir -p .local
cat > .local/s3.json <<'EOF'
{"identities":[{"name":"velero","credentials":[{"accessKey":"minio","secretKey":"minio123"}],"actions":["Admin","Read","Write","List","Tagging"]}]}
EOF

docker volume create dr-seaweed-data
docker rm -f dr-seaweed 2>/dev/null || true

docker run -d \
  --name dr-seaweed \
  --network kind \
  --restart unless-stopped \
  -p 0.0.0.0:9200:8333 \
  -v dr-seaweed-data:/data \
  -v "$PWD/.local/s3.json":/etc/seaweedfs/s3.json:ro \
  "$SEAWEED_IMAGE" server -dir=/data -s3 -s3.port=8333 -s3.config=/etc/seaweedfs/s3.json

docker run --pull=never --rm --network kind --entrypoint sh "$MC_IMAGE" -c '
  until mc alias set sw http://dr-seaweed:8333 minio minio123 2>/dev/null; do sleep 2; done
  mc mb --ignore-existing sw/velero
'
```

The vault lives outside both clusters and uses a persistent Docker volume. It
is reachable two ways, and both matter: containers on the `kind` network reach
it as `http://dr-seaweed:8333` (the DR cluster uses this), and the kiac VM
reaches it through the published host port as `http://192.168.64.1:9200` (the
prod cluster uses this).

Sanity-check both paths now:

```bash
curl -fsS -o /dev/null -w 'host port: %{http_code}\n' http://127.0.0.1:9200

kubectl --context "$PROD_CTX" run nettest --restart=Never \
  --image="$BUSYBOX_IMAGE" \
  -- sh -c 'wget -q -S -O /dev/null http://192.168.64.1:9200 2>&1 | head -1'
kubectl --context "$PROD_CTX" wait pod/nettest \
  --for=jsonpath='{.status.phase}'=Succeeded --timeout=90s
kubectl --context "$PROD_CTX" logs nettest     # expect an HTTP status line
kubectl --context "$PROD_CTX" delete pod nettest
```

## 9. Install Velero in both clusters

Create local credentials. Do not commit this file:

```bash
cat >/tmp/velero-minio-creds <<'EOF'
[default]
aws_access_key_id=minio
aws_secret_access_key=minio123
EOF
```

Install on the prod cluster (vault via the host gateway):

```bash
velero install \
  --kubecontext "$PROD_CTX" \
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
```

Install on the recovery cluster (vault via the kind network):

```bash
velero install \
  --kubecontext "$DR_CTX" \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.14.0 \
  --bucket velero \
  --secret-file /tmp/velero-minio-creds \
  --backup-location-config \
    'region=seaweed,s3ForcePathStyle=true,s3Url=http://dr-seaweed:8333' \
  --use-node-agent \
  --use-volume-snapshots=false \
  --features=EnableCSI \
  --wait
```

Same bucket, two routes: the DR cluster syncs every backup the prod cluster
writes (within about a minute), which is what Demo 2 relies on.

Verify:

```bash
kubectl --context "$PROD_CTX" -n velero get backupstoragelocation
kubectl --context "$DR_CTX" -n velero get backupstoragelocation
```

Both must show `Available`.

## 10. Deploy and seed the production guestbook

```bash
kubectl --context "$PROD_CTX" apply -f manifests/guestbook/namespace.yaml
kubectl --context "$PROD_CTX" apply -f manifests/guestbook/postgres.yaml
kubectl --context "$PROD_CTX" -n guestbook rollout status \
  statefulset/postgres --timeout 300s

kubectl --context "$PROD_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c \
  'CREATE TABLE IF NOT EXISTS attendees
   (id serial PRIMARY KEY, name text, note text);'

kubectl --context "$PROD_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c \
  "INSERT INTO attendees (name, note)
   SELECT * FROM (VALUES
     ('Priya',  'came for the demos'),
     ('Marcus', 'still calls it swarm'),
     ('Chen',   'has tested a restore. once.'),
     ('Fatima', 'owns the pager tonight')) v(name, note)
   WHERE NOT EXISTS (SELECT 1 FROM attendees);"
```

Create the pre-tested conference recovery point:

```bash
export BACKUP="guestbook-rehearsal-$(date +%Y%m%d%H%M%S)"

velero backup create "$BACKUP" \
  --kubecontext "$PROD_CTX" \
  --include-namespaces guestbook \
  --snapshot-move-data \
  --wait
```

Record the backup name.

## 11. Start local Gitea

```bash
docker volume create dr-gitea-data
docker rm -f dr-gitea 2>/dev/null || true

docker run -d \
  --name dr-gitea \
  --network kind \
  --restart unless-stopped \
  -v dr-gitea-data:/data \
  -p 127.0.0.1:3300:3000 \
  -e GITEA__security__INSTALL_LOCK=true \
  -e GITEA__server__HTTP_PORT=3000 \
  "$GITEA_IMAGE"
```

Wait until `http://127.0.0.1:3300/api/healthz` returns successfully.

Create the local user and repository:

```bash
docker exec dr-gitea su git -c \
  'gitea admin user create \
   --username demo \
   --password demo1234 \
   --email demo@example.com \
   --admin \
   --must-change-password=false' || true

curl -fsS -X POST http://127.0.0.1:3300/api/v1/user/repos \
  -u demo:demo1234 \
  -H 'Content-Type: application/json' \
  -d '{"name":"guestbook","auto_init":false}' || true
```

Push the guestbook manifests:

```bash
rm -rf /tmp/guestbook-demo-repo
mkdir -p /tmp/guestbook-demo-repo
cp manifests/guestbook/*.yaml /tmp/guestbook-demo-repo/

git -C /tmp/guestbook-demo-repo init -b main
git -C /tmp/guestbook-demo-repo \
  -c user.name=demo -c user.email=demo@example.com add .
git -C /tmp/guestbook-demo-repo \
  -c user.name=demo -c user.email=demo@example.com \
  commit -m 'guestbook app'
git -C /tmp/guestbook-demo-repo push --force \
  http://demo:demo1234@127.0.0.1:3300/demo/guestbook.git main
```

## 12. Install Argo CD in the DR cluster

```bash
kubectl --context "$DR_CTX" create namespace argocd \
  --dry-run=client -o yaml | kubectl --context "$DR_CTX" apply -f -

kubectl --context "$DR_CTX" -n argocd apply \
  --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.5/manifests/core-install.yaml

kubectl --context "$DR_CTX" -n argocd rollout status \
  deployment/argocd-repo-server --timeout 300s
```

Create the default project:

```bash
kubectl --context "$DR_CTX" apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
  namespace: argocd
spec:
  sourceRepos: ["*"]
  destinations:
    - namespace: "*"
      server: "*"
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"
EOF
```

Create the manual-sync Application:

```bash
export GITEA_URL='http://dr-gitea:3000'
envsubst '${GITEA_URL}' < manifests/argocd/guestbook-app.tmpl.yaml |
  kubectl --context "$DR_CTX" apply -f -
```

## 13. Deploy the ledger app

```bash
kubectl --context "$DR_CTX" apply -f manifests/ledger/ledger.yaml
kubectl --context "$DR_CTX" -n ledger rollout status \
  deployment/ledger-writer --timeout 180s
```

## 14. Offline stage checklist

Turn Wi-Fi off. Confirm all of these before the talk:

```bash
docker ps --format '{{.Names}}'

kubectl --context "$PROD_CTX" get --raw=/readyz
kubectl --context "$DR_CTX" get --raw=/readyz

kubectl --context "$PROD_CTX" -n velero get backupstoragelocation
kubectl --context "$DR_CTX" -n velero get backupstoragelocation

kubectl --context "$PROD_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -tAc 'select count(*) from attendees'

kubectl --context "$DR_CTX" -n ledger get deployment ledger-writer

curl -fsS http://127.0.0.1:3300/api/healthz
```

Expected:

- both Kubernetes APIs return `ok`
- both backup locations are `Available` - the prod one proves the
  VM-to-host-gateway vault path works without internet; do not present until
  this passes offline
- guestbook row count is `4`
- ledger writer is Ready
- `dr-seaweed` and `dr-gitea` are running

Do not run setup commands on stage.

# Main demo

## Act 1. Prove the recovery point and restore locally

Find the newest tested backup:

```bash
export BACKUP=$(
  kubectl --context "$PROD_CTX" -n velero get backups.velero.io -o json |
  jq -r '[.items[]
    | select(.status.phase=="Completed"
      and (.metadata.name | startswith("guestbook-rehearsal")))]
    | sort_by(.metadata.creationTimestamp)
    | last
    | .metadata.name'
)
echo "$BACKUP"
```

Show the original rows and copied volume bytes:

```bash
kubectl --context "$PROD_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c 'select * from attendees;'

velero backup get "$BACKUP" --kubecontext "$PROD_CTX"

kubectl --context "$PROD_CTX" -n velero get datauploads \
  -l "velero.io/backup-name=$BACKUP" \
  -o custom-columns='NAME:.metadata.name,PHASE:.status.phase,BYTES:.status.progress.bytesDone'
```

Delete and restore:

```bash
kubectl --context "$PROD_CTX" delete namespace guestbook --wait

velero restore create \
  --kubecontext "$PROD_CTX" \
  --from-backup "$BACKUP" \
  --wait

kubectl --context "$PROD_CTX" -n guestbook rollout status \
  statefulset/postgres --timeout 300s

kubectl --context "$PROD_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c 'select * from attendees;'
```

Expected: the same four rows return.

## Act 2. Lose prod, expose the GitOps gap, restore into DR

Start a timer, then stop the source cluster:

```bash
date
container stop kiac-drprod-control-plane    # kind-prod fallback: docker stop dr-prod-control-plane
kubectl --context "$PROD_CTX" get nodes --request-timeout=4s || true
```

Trigger Argo sync:

```bash
kubectl --context "$DR_CTX" -n argocd patch application guestbook \
  --type merge \
  -p '{"operation":{"initiatedBy":{"username":"on-call"},"sync":{"revision":"HEAD"}}}'

kubectl --context "$DR_CTX" -n argocd wait application/guestbook \
  --for=jsonpath='{.status.sync.status}'=Synced \
  --timeout=300s

kubectl --context "$DR_CTX" -n guestbook rollout status \
  statefulset/postgres --timeout 300s
```

Show that Healthy is not recovered:

```bash
kubectl --context "$DR_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c 'select * from attendees;' || true
```

Expected: `relation "attendees" does not exist`.

Remove the empty GitOps-created resources and restore:

```bash
kubectl --context "$DR_CTX" delete namespace guestbook --wait

velero restore create \
  --kubecontext "$DR_CTX" \
  --from-backup "$BACKUP" \
  --wait

kubectl --context "$DR_CTX" -n guestbook rollout status \
  statefulset/postgres --timeout 300s

kubectl --context "$DR_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c 'select * from attendees;'

date
```

Expected: the four rows return on the DR cluster.

The elapsed time is only the scripted demo segment. Production RTO also includes
detection, declaration, decision, traffic cutover, validation, and failback.

## Act 3. Show cross-volume timing skew

Use a run suffix:

```bash
export RUN_ID=$(date +%H%M%S)
```

Reset the small ledger:

```bash
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- \
  touch /orders/.pause
sleep 1
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- \
  rm -f /orders/orders.log /payments/payments.log
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- \
  rm -f /orders/.pause
```

Pause and snapshot orders:

```bash
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- \
  touch /orders/.pause
sleep 1

kubectl --context "$DR_CTX" -n ledger apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: torn-orders-${RUN_ID}
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: orders-pvc
EOF

kubectl --context "$DR_CTX" -n ledger wait \
  volumesnapshot/torn-orders-"$RUN_ID" \
  --for=jsonpath='{.status.readyToUse}'=true \
  --timeout=120s
```

Resume writes for five seconds, then pause and snapshot payments:

```bash
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- \
  rm -f /orders/.pause
sleep 5
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- \
  touch /orders/.pause
sleep 1

kubectl --context "$DR_CTX" -n ledger apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: torn-payments-${RUN_ID}
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: payments-pvc
EOF

kubectl --context "$DR_CTX" -n ledger wait \
  volumesnapshot/torn-payments-"$RUN_ID" \
  --for=jsonpath='{.status.readyToUse}'=true \
  --timeout=120s

kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- \
  rm -f /orders/.pause
```

Restore both individual snapshots:

```bash
for item in orders payments; do
  kubectl --context "$DR_CTX" -n ledger apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-torn-${item}-${RUN_ID}
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: csi-hostpath-sc
  resources:
    requests:
      storage: 100Mi
  dataSource:
    apiGroup: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: torn-${item}-${RUN_ID}
EOF
done
```

Create the verifier:

```bash
export JOB_NAME="verify-torn-${RUN_ID}"
export ORDERS_PVC="restored-torn-orders-${RUN_ID}"
export PAYMENTS_PVC="restored-torn-payments-${RUN_ID}"

envsubst '${JOB_NAME} ${ORDERS_PVC} ${PAYMENTS_PVC}' \
  < manifests/ledger/verify-job.tmpl.yaml |
  kubectl --context "$DR_CTX" apply -f -

kubectl --context "$DR_CTX" -n ledger wait \
  job/"$JOB_NAME" --for=condition=complete --timeout=120s || true

kubectl --context "$DR_CTX" -n ledger logs job/"$JOB_NAME"
```

Expected: payments exist without matching orders.

Now create one group recovery point:

```bash
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- \
  touch /orders/.pause
sleep 1

kubectl --context "$DR_CTX" -n ledger apply -f - <<EOF
apiVersion: groupsnapshot.storage.k8s.io/v1
kind: VolumeGroupSnapshot
metadata:
  name: group-${RUN_ID}
spec:
  volumeGroupSnapshotClassName: csi-hostpath-groupsnapclass
  source:
    selector:
      matchLabels:
        group: ledger
EOF

kubectl --context "$DR_CTX" -n ledger wait \
  volumegroupsnapshot/group-"$RUN_ID" \
  --for=jsonpath='{.status.readyToUse}'=true \
  --timeout=120s

kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- \
  rm -f /orders/.pause
```

Find the two generated member snapshots:

```bash
export ORDERS_SNAPSHOT=$(
  kubectl --context "$DR_CTX" -n ledger get volumesnapshot -o json |
  jq -r --arg group "group-${RUN_ID}" '
    .items[]
    | select(.status.volumeGroupSnapshotName==$group)
    | select(.spec.source.persistentVolumeClaimName=="orders-pvc")
    | .metadata.name'
)

export PAYMENTS_SNAPSHOT=$(
  kubectl --context "$DR_CTX" -n ledger get volumesnapshot -o json |
  jq -r --arg group "group-${RUN_ID}" '
    .items[]
    | select(.status.volumeGroupSnapshotName==$group)
    | select(.spec.source.persistentVolumeClaimName=="payments-pvc")
    | .metadata.name'
)
```

Create restored PVCs from those member snapshots:

```bash
for item in orders payments; do
  if [ "$item" = orders ]; then SNAPSHOT="$ORDERS_SNAPSHOT"; else SNAPSHOT="$PAYMENTS_SNAPSHOT"; fi
  kubectl --context "$DR_CTX" -n ledger apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-group-${item}-${RUN_ID}
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: csi-hostpath-sc
  resources:
    requests:
      storage: 100Mi
  dataSource:
    apiGroup: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: ${SNAPSHOT}
EOF
done
```

Run the verifier again:

```bash
export JOB_NAME="verify-group-${RUN_ID}"
export ORDERS_PVC="restored-group-orders-${RUN_ID}"
export PAYMENTS_PVC="restored-group-payments-${RUN_ID}"

envsubst '${JOB_NAME} ${ORDERS_PVC} ${PAYMENTS_PVC}' \
  < manifests/ledger/verify-job.tmpl.yaml |
  kubectl --context "$DR_CTX" apply -f -

kubectl --context "$DR_CTX" -n ledger wait \
  job/"$JOB_NAME" --for=condition=complete --timeout=120s

kubectl --context "$DR_CTX" -n ledger logs job/"$JOB_NAME"
```

Expected: every payment has a matching order.

This proves only cross-volume crash consistency. It does not make a database
application-consistent.

# Optional appendix: vind drill

Use only if there is extra time.

## Prepare online before the event

```bash
vcluster telemetry disable
export VCLUSTER_SKIP_VERSION_CHECK=true
vcluster use driver docker
vcluster delete drill || true
vcluster create drill --connect=true

kubectl -n local-path-storage patch configmap local-path-config \
  --type merge \
  --patch-file manifests/vind/local-path-config-patch.yaml

kubectl -n local-path-storage rollout restart \
  deployment/local-path-provisioner
kubectl -n local-path-storage rollout status \
  deployment/local-path-provisioner --timeout 180s

kubectl apply -f manifests/vind/drill.yaml
kubectl -n prod-data rollout status deployment/drill-app --timeout 180s
kubectl -n prod-data exec deployment/drill-app -- cat /data/birth.txt

mkdir -p .local
vcluster snapshot create drill .local/drill-snapshot.tar.gz
```

Record the original timestamp.

## Run the optional demo offline

```bash
export VCLUSTER_SKIP_VERSION_CHECK=true
vcluster connect drill
kubectl -n prod-data exec deployment/drill-app -- cat /data/birth.txt

vcluster delete drill
docker ps --format '{{.Names}}' | rg drill || echo 'cluster container is gone'

vcluster restore drill .local/drill-snapshot.tar.gz
vcluster connect drill
kubectl -n prod-data rollout status deployment/drill-app --timeout 180s
kubectl -n prod-data exec deployment/drill-app -- cat /data/birth.txt
```

Expected: the timestamp matches.

This is local drill tooling, not production DR. The configured PVC path is inside
the archived Docker node volume, and CPU architecture must match.

## Reset after rehearsal

```bash
kiac resume cluster --name drprod    # kind-prod fallback: docker start dr-prod-control-plane
kubectl --context "$DR_CTX" delete namespace guestbook --ignore-not-found
kubectl --context "$DR_CTX" delete namespace ledger --ignore-not-found
kubectl --context "$DR_CTX" apply -f manifests/ledger/ledger.yaml
kubectl --context "$DR_CTX" -n ledger rollout status \
  deployment/ledger-writer --timeout 180s
```

For simplicity, it is also safe to rebuild the local environment while online
before the next full rehearsal.
