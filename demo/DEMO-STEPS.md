# Demo steps

Commands to run each demo, with a one-line explanation per step. Assumes the
one-time setup from `MANUAL-DEMO-GUIDE.md` is complete. Run from `demo/`.

Set once:

```bash
export PROD_CTX=kind-dr-prod   # or kiac-drprod when using KIAC-PROD-VARIANT.md
export DR_CTX=kind-dr-dr
```

---

## Demo 1 - Velero backup and restore

Shows that Velero can back up a stateful app (objects + PV data) and restore
it, data included.

```bash
# Find the newest completed backup (created during setup, stored in SeaweedFS)
export BACKUP=$(kubectl --context "$PROD_CTX" -n velero get backups.velero.io -o json |
  jq -r '[.items[] | select(.status.phase=="Completed"
    and (.metadata.name | startswith("guestbook-rehearsal")))]
    | sort_by(.metadata.creationTimestamp) | last | .metadata.name')
echo "$BACKUP"

# Show the current production data
kubectl --context "$PROD_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c 'select * from attendees;'

# Show that the PV bytes were copied out of the cluster to object storage
kubectl --context "$PROD_CTX" -n velero get datauploads \
  -l "velero.io/backup-name=$BACKUP" \
  -o custom-columns='NAME:.metadata.name,PHASE:.status.phase,BYTES:.status.progress.bytesDone'

# Delete the namespace, PVC included
kubectl --context "$PROD_CTX" delete namespace guestbook --wait

# Restore from the backup
velero restore create --kubecontext "$PROD_CTX" --from-backup "$BACKUP" --wait

# Wait for Postgres and confirm the same rows are back
kubectl --context "$PROD_CTX" -n guestbook rollout status statefulset/postgres --timeout 300s
kubectl --context "$PROD_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c 'select * from attendees;'
```

---

## Demo 2 - cluster loss, GitOps sync, restore into the recovery cluster

Shows that GitOps recreates the declared objects but not the stored data, and
that the Velero recovery point restored in the right order completes the
recovery. Uses `$BACKUP` from Demo 1 (both clusters see the same bucket).

```bash
# Record the start time
date

# Stop the production control plane
docker stop dr-prod-control-plane            # kind prod
# container stop kiac-drprod-control-plane   # kiac variant

# Confirm the production API no longer answers
kubectl --context "$PROD_CTX" get nodes --request-timeout=4s || true

# Trigger the Argo CD sync in the recovery cluster
kubectl --context "$DR_CTX" -n argocd patch application guestbook --type merge \
  -p '{"operation":{"initiatedBy":{"username":"on-call"},"sync":{"revision":"HEAD"}}}'

# Wait until Argo reports Synced and Postgres is Ready
kubectl --context "$DR_CTX" -n argocd wait application/guestbook \
  --for=jsonpath='{.status.sync.status}'=Synced --timeout=300s
kubectl --context "$DR_CTX" -n guestbook rollout status statefulset/postgres --timeout 300s

# Query the data: the table does not exist, the volume is new and empty
kubectl --context "$DR_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c 'select * from attendees;' || true

# Remove the empty resources and restore the recovery point instead
kubectl --context "$DR_CTX" delete namespace guestbook --wait
velero restore create --kubecontext "$DR_CTX" --from-backup "$BACKUP" --wait

# Confirm the data is present in the recovery cluster
kubectl --context "$DR_CTX" -n guestbook rollout status statefulset/postgres --timeout 300s
kubectl --context "$DR_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c 'select * from attendees;'

# Record the end time
date
```

---

## Demo 3 - individual snapshots vs one VolumeGroupSnapshot

Shows that snapshots of two volumes taken at different times restore into a
state that never existed (payments without orders), and that one
VolumeGroupSnapshot produces a consistent recovery point. The ledger app
writes matched ORDER/PAYMENT pairs to two PVCs; a `.pause` file pauses the
writer so timing is deterministic.

```bash
# Fresh run id and a clean ledger
export RUN_ID=$(date +%H%M%S)
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- touch /orders/.pause
sleep 1
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- \
  rm -f /orders/orders.log /payments/payments.log /orders/.pause

# Snapshot the orders volume at t=0
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- touch /orders/.pause
sleep 1
kubectl --context "$DR_CTX" -n ledger apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: {name: torn-orders-${RUN_ID}}
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source: {persistentVolumeClaimName: orders-pvc}
EOF
kubectl --context "$DR_CTX" -n ledger wait volumesnapshot/torn-orders-"$RUN_ID" \
  --for=jsonpath='{.status.readyToUse}'=true --timeout=120s

# Let the writer run for five seconds, then snapshot the payments volume
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- rm -f /orders/.pause
sleep 5
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- touch /orders/.pause
sleep 1
kubectl --context "$DR_CTX" -n ledger apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: {name: torn-payments-${RUN_ID}}
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source: {persistentVolumeClaimName: payments-pvc}
EOF
kubectl --context "$DR_CTX" -n ledger wait volumesnapshot/torn-payments-"$RUN_ID" \
  --for=jsonpath='{.status.readyToUse}'=true --timeout=120s
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- rm -f /orders/.pause

# Restore both individual snapshots into new PVCs
for item in orders payments; do
kubectl --context "$DR_CTX" -n ledger apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: restored-torn-${item}-${RUN_ID}}
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: csi-hostpath-sc
  resources: {requests: {storage: 100Mi}}
  dataSource: {apiGroup: snapshot.storage.k8s.io, kind: VolumeSnapshot, name: torn-${item}-${RUN_ID}}
EOF
done

# Run the verifier: expect FAIL, payments exist without matching orders
export JOB_NAME="verify-torn-${RUN_ID}" ORDERS_PVC="restored-torn-orders-${RUN_ID}" PAYMENTS_PVC="restored-torn-payments-${RUN_ID}"
envsubst '${JOB_NAME} ${ORDERS_PVC} ${PAYMENTS_PVC}' < manifests/ledger/verify-job.tmpl.yaml |
  kubectl --context "$DR_CTX" apply -f -
kubectl --context "$DR_CTX" -n ledger wait job/"$JOB_NAME" --for=condition=complete --timeout=120s || true
kubectl --context "$DR_CTX" -n ledger logs job/"$JOB_NAME"

# Take one VolumeGroupSnapshot across both PVCs (selected by label)
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- touch /orders/.pause
sleep 1
kubectl --context "$DR_CTX" -n ledger apply -f - <<EOF
apiVersion: groupsnapshot.storage.k8s.io/v1
kind: VolumeGroupSnapshot
metadata: {name: group-${RUN_ID}}
spec:
  volumeGroupSnapshotClassName: csi-hostpath-groupsnapclass
  source: {selector: {matchLabels: {group: ledger}}}
EOF
kubectl --context "$DR_CTX" -n ledger wait volumegroupsnapshot/group-"$RUN_ID" \
  --for=jsonpath='{.status.readyToUse}'=true --timeout=120s
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- rm -f /orders/.pause

# Find the two member snapshots the group created
export ORDERS_SNAPSHOT=$(kubectl --context "$DR_CTX" -n ledger get volumesnapshot -o json |
  jq -r --arg g "group-${RUN_ID}" '.items[] | select(.status.volumeGroupSnapshotName==$g)
    | select(.spec.source.persistentVolumeClaimName=="orders-pvc") | .metadata.name')
export PAYMENTS_SNAPSHOT=$(kubectl --context "$DR_CTX" -n ledger get volumesnapshot -o json |
  jq -r --arg g "group-${RUN_ID}" '.items[] | select(.status.volumeGroupSnapshotName==$g)
    | select(.spec.source.persistentVolumeClaimName=="payments-pvc") | .metadata.name')

# Restore the member snapshots the same way
for item in orders payments; do
  if [ "$item" = orders ]; then SNAP="$ORDERS_SNAPSHOT"; else SNAP="$PAYMENTS_SNAPSHOT"; fi
kubectl --context "$DR_CTX" -n ledger apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: restored-group-${item}-${RUN_ID}}
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: csi-hostpath-sc
  resources: {requests: {storage: 100Mi}}
  dataSource: {apiGroup: snapshot.storage.k8s.io, kind: VolumeSnapshot, name: ${SNAP}}
EOF
done

# Run the verifier again: expect OK, every payment has a matching order
export JOB_NAME="verify-group-${RUN_ID}" ORDERS_PVC="restored-group-orders-${RUN_ID}" PAYMENTS_PVC="restored-group-payments-${RUN_ID}"
envsubst '${JOB_NAME} ${ORDERS_PVC} ${PAYMENTS_PVC}' < manifests/ledger/verify-job.tmpl.yaml |
  kubectl --context "$DR_CTX" apply -f -
kubectl --context "$DR_CTX" -n ledger wait job/"$JOB_NAME" --for=condition=complete --timeout=120s
kubectl --context "$DR_CTX" -n ledger logs job/"$JOB_NAME"
```

The result is crash-consistent across volumes, not application-consistent.

---

## Demo 4 (optional) - vind cluster snapshot and restore

Shows a whole local cluster deleted and restored from one archive file, with
the original data intact.

```bash
export VCLUSTER_SKIP_VERSION_CHECK=true

# Show the file written when the cluster was first created
vcluster connect drill
kubectl -n prod-data exec deployment/drill-app -- cat /data/birth.txt

# Delete the entire cluster
vcluster delete drill

# Restore it from the archive and confirm the same timestamp
vcluster restore drill .local/drill-snapshot.tar.gz
vcluster connect drill
kubectl -n prod-data rollout status deployment/drill-app --timeout 180s
kubectl -n prod-data exec deployment/drill-app -- cat /data/birth.txt
```

---

## Reset between runs

```bash
docker start dr-prod-control-plane            # kind prod back
# kiac resume cluster --name drprod           # kiac variant instead
kubectl --context "$DR_CTX" delete namespace guestbook --ignore-not-found
kubectl --context "$DR_CTX" delete namespace ledger --ignore-not-found
kubectl --context "$DR_CTX" apply -f manifests/ledger/ledger.yaml
kubectl --context "$DR_CTX" -n ledger rollout status deployment/ledger-writer --timeout 180s
```
