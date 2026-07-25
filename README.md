# Is Your Kubernetes Disaster Recovery Actually Ready?

Demo lab from the KubeCon + CloudNativeCon Japan 2026 session by
Saiyam Pathak and Saloni.

Everything runs locally: two kind clusters, SeaweedFS, and Gitea in Docker.
After the one-time online setup, the demo acts run with no internet at all.

This repository contains only:

- Kubernetes YAML manifests
- the manual commands and setup steps
- this short entry point

Start here:

- [`demo/MANUAL-DEMO-GUIDE.md`](demo/MANUAL-DEMO-GUIDE.md) - one-time setup, end to end
- [`demo/DEMO-STEPS.md`](demo/DEMO-STEPS.md) - the commands for each demo, step by step

The guide covers:

1. Creating the prod and recovery kind clusters.
2. Installing CSI snapshots and VolumeGroupSnapshot.
3. Installing Velero with a local S3 vault (SeaweedFS).
4. Configuring local Gitea and Argo CD.
5. Running the three main demo acts manually:
   - **Act 1** - Velero backup, delete the namespace, restore, data is back.
   - **Act 2** - lose production, GitOps goes green with an empty database,
     restore the real data into the recovery cluster.
   - **Act 3** - two individual snapshots tear a two-volume app;
     one VolumeGroupSnapshot keeps it consistent.
6. Running the optional vind rehearsal demo.
7. Preparing and validating the environment for an offline conference.

There is also a tested variant where production runs on
[kiac](https://github.com/saiyam1814/kiac) (Kubernetes in Apple Containers),
so the disaster kills a real VM and the restore crosses substrates:
[`demo/KIAC-PROD-VARIANT.md`](demo/KIAC-PROD-VARIANT.md)

Automation scripts, credentials, caches, recordings, slide files, research notes,
PDF, and PPTX artifacts are intentionally excluded from Git.
