# Show Flow - exactly what to run on stage, in order

This is the on-stage script. All setup (guide sections 1-13) is already done
days before. Each act states the one claim it proves, then the commands in
order, each with a one-line reason. Run everything from `demo/`.

Set once in the stage terminal:

```bash
export PROD_CTX=kind-dr-prod   # or kiac-drprod if using KIAC-PROD-VARIANT.md
export DR_CTX=kind-dr-dr
```

Before walking on stage (30 seconds, not shown to the audience):

```bash
kubectl --context "$PROD_CTX" get --raw=/readyz                      # prod alive
kubectl --context "$DR_CTX" -n velero get backupstoragelocation     # Available
kubectl --context "$PROD_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -tAc 'select count(*) from attendees'  # 4
```

Rule for every act: if something hangs for more than ~30 seconds, switch to the
fallback recording and keep talking. Never debug on stage.

---

## ACT 1 - "Backup works. Watch." (~60s)

**The claim: Velero can back up a stateful app and bring it back, data included.**
This act builds trust before we break things. Velero is the star.

```bash
# 1. Find the pre-tested backup (created during prep, lives in SeaweedFS)
export BACKUP=$(kubectl --context "$PROD_CTX" -n velero get backups.velero.io -o json |
  jq -r '[.items[] | select(.status.phase=="Completed"
    and (.metadata.name | startswith("guestbook-rehearsal")))]
    | sort_by(.metadata.creationTimestamp) | last | .metadata.name')
echo "$BACKUP"

# 2. Show production data: four real rows, this is what we must not lose
kubectl --context "$PROD_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c 'select * from attendees;'

# 3. Prove the recovery point exists OUTSIDE the cluster: PV bytes moved to S3
kubectl --context "$PROD_CTX" -n velero get datauploads \
  -l "velero.io/backup-name=$BACKUP" \
  -o custom-columns='NAME:.metadata.name,PHASE:.status.phase,BYTES:.status.progress.bytesDone'

# 4. Be the disaster: delete the whole namespace, volume included
kubectl --context "$PROD_CTX" delete namespace guestbook --wait

# 5. One command brings it back
velero restore create --kubecontext "$PROD_CTX" --from-backup "$BACKUP" --wait

# 6. Wait for Postgres, then show the SAME four rows
kubectl --context "$PROD_CTX" -n guestbook rollout status statefulset/postgres --timeout 300s
kubectl --context "$PROD_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c 'select * from attendees;'
```

Say: "Useful. Real. And it is only the first quarter of disaster recovery."

---

## ACT 2 - "Let's lose production" (~3 min)

**The claim: GitOps rebuilds your intent, not your data. Recovery is Velero
plus GitOps in the right order.** Velero is still the star; Argo is the trap.

```bash
# 1. Start the clock: real RTO is measured, not guessed
date

# 2. THE DISASTER - the production control plane dies
docker stop dr-prod-control-plane            # kind prod
# container stop kiac-drprod-control-plane   # kiac variant: powers off a real VM

# 3. Prove prod is gone: the API does not answer
kubectl --context "$PROD_CTX" get nodes --request-timeout=4s || true

# 4. On-call reflex: sync the app into the DR cluster from Git
kubectl --context "$DR_CTX" -n argocd patch application guestbook --type merge \
  -p '{"operation":{"initiatedBy":{"username":"on-call"},"sync":{"revision":"HEAD"}}}'

# 5. Watch Argo go green: Synced, then Postgres becomes Ready
kubectl --context "$DR_CTX" -n argocd wait application/guestbook \
  --for=jsonpath='{.status.sync.status}'=Synced --timeout=300s
kubectl --context "$DR_CTX" -n guestbook rollout status statefulset/postgres --timeout 300s

# 6. THE TRAP - everything is green, so check the data...
kubectl --context "$DR_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c 'select * from attendees;' || true
#    -> "relation attendees does not exist". Green is not recovered.

# 7. Real recovery: clear the empty objects, restore the recovery point
kubectl --context "$DR_CTX" delete namespace guestbook --wait
velero restore create --kubecontext "$DR_CTX" --from-backup "$BACKUP" --wait

# 8. Validate like a user would: the data is there
kubectl --context "$DR_CTX" -n guestbook rollout status statefulset/postgres --timeout 300s
kubectl --context "$DR_CTX" -n guestbook exec statefulset/postgres -- \
  psql -U postgres -d guestbook -c 'select * from attendees;'

# 9. Stop the clock
date
```

Say: "The timer covers only this scripted slice. Production RTO adds detection,
decision, traffic cutover, and failback around it."

`$BACKUP` was set in Act 1. If the variable is lost, resolve it from the DR
side (both clusters share the vault):
`kubectl --context "$DR_CTX" -n velero get backups.velero.io`

---

## ACT 3 - "The snapshot gap" (~4 min)

**The claim: two individually perfect snapshots can produce one corrupt
application, and VolumeGroupSnapshot (Kubernetes 1.36) closes that gap.**
No Velero in this act, on purpose: this is a storage-layer problem that exists
below every backup tool. The ledger app writes matched ORDER/PAYMENT pairs to
two PVCs; the writer pauses via a `.pause` file so timing is deterministic.

```bash
# 0. Fresh run id and a clean ledger
export RUN_ID=$(date +%H%M%S)
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- touch /orders/.pause
sleep 1
kubectl --context "$DR_CTX" -n ledger exec deployment/ledger-writer -- \
  rm -f /orders/orders.log /payments/payments.log /orders/.pause

# 1. Snapshot volume A (orders) at t=0
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

# 2. Let the app keep writing for five seconds, then snapshot volume B (payments)
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

# 3. Restore both "perfect" snapshots into new PVCs
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

# 4. Verify: payments exist with NO matching order -> [FAIL]
export JOB_NAME="verify-torn-${RUN_ID}" ORDERS_PVC="restored-torn-orders-${RUN_ID}" PAYMENTS_PVC="restored-torn-payments-${RUN_ID}"
envsubst '${JOB_NAME} ${ORDERS_PVC} ${PAYMENTS_PVC}' < manifests/ledger/verify-job.tmpl.yaml |
  kubectl --context "$DR_CTX" apply -f -
kubectl --context "$DR_CTX" -n ledger wait job/"$JOB_NAME" --for=condition=complete --timeout=120s || true
kubectl --context "$DR_CTX" -n ledger logs job/"$JOB_NAME"

# 5. Now the fix: ONE VolumeGroupSnapshot cuts both volumes at the same instant
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

# 6. Find the two auto-generated member snapshots of the group
export ORDERS_SNAPSHOT=$(kubectl --context "$DR_CTX" -n ledger get volumesnapshot -o json |
  jq -r --arg g "group-${RUN_ID}" '.items[] | select(.status.volumeGroupSnapshotName==$g)
    | select(.spec.source.persistentVolumeClaimName=="orders-pvc") | .metadata.name')
export PAYMENTS_SNAPSHOT=$(kubectl --context "$DR_CTX" -n ledger get volumesnapshot -o json |
  jq -r --arg g "group-${RUN_ID}" '.items[] | select(.status.volumeGroupSnapshotName==$g)
    | select(.spec.source.persistentVolumeClaimName=="payments-pvc") | .metadata.name')

# 7. Restore the group members the same way
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

# 8. Verify again: every payment has its order -> [OK]
export JOB_NAME="verify-group-${RUN_ID}" ORDERS_PVC="restored-group-orders-${RUN_ID}" PAYMENTS_PVC="restored-group-payments-${RUN_ID}"
envsubst '${JOB_NAME} ${ORDERS_PVC} ${PAYMENTS_PVC}' < manifests/ledger/verify-job.tmpl.yaml |
  kubectl --context "$DR_CTX" apply -f -
kubectl --context "$DR_CTX" -n ledger wait job/"$JOB_NAME" --for=condition=complete --timeout=120s
kubectl --context "$DR_CTX" -n ledger logs job/"$JOB_NAME"
```

Say: "Crash-consistent across volumes, not application-consistent. The API is
GA; the driver support and the integration are still yours to own."

---

## OPTIONAL ACT 4 - vind drill (~60-90s, only if time)

**The claim: destructive recovery drills should be cheap enough to run weekly.**

```bash
export VCLUSTER_SKIP_VERSION_CHECK=true

# 1. Show the data born with the cluster
vcluster connect drill
kubectl -n prod-data exec deployment/drill-app -- cat /data/birth.txt

# 2. Delete the ENTIRE cluster
vcluster delete drill

# 3. Restore it from one archive file, then prove the same timestamp survived
vcluster restore drill .local/drill-snapshot.tar.gz
vcluster connect drill
kubectl -n prod-data rollout status deployment/drill-app --timeout 180s
kubectl -n prod-data exec deployment/drill-app -- cat /data/birth.txt
```

---

## After the talk / between rehearsals

```bash
docker start dr-prod-control-plane            # kind prod back
# kiac resume cluster --name drprod           # kiac variant instead
kubectl --context "$DR_CTX" delete namespace guestbook --ignore-not-found
kubectl --context "$DR_CTX" delete namespace ledger --ignore-not-found
kubectl --context "$DR_CTX" apply -f manifests/ledger/ledger.yaml
kubectl --context "$DR_CTX" -n ledger rollout status deployment/ledger-writer --timeout 180s
```

Note: `kiac resume` / `docker start` are **reset steps, never demo steps**. The
talk never brings prod back on stage; DR is about serving users from somewhere
else while prod is still smoking.
