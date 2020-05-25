# kube-plex

kube-plex is a scalable Plex Media Server solution for Kubernetes. It
distributes transcode jobs by creating pods in a Kubernetes cluster to perform
transcodes, instead of running transcodes on the Plex Media Server instance
itself.

[For tips on how to set it up on a Raspberry Pi, jump to this section](#additional-raspberry-pi-requirements)

## How it works

kube-plex works by replacing the Plex Transcoder program on the main PMS
instance with our own little shim. This shim intercepts calls to Plex
Transcoder, and creates Kubernetes pods to perform the work instead. These
pods use shared persistent volumes to store the results of the transcode (and
read your media!).

## Prerequisites

* A persistent volume type that supports ReadWriteMany volumes (e.g. NFS,
Amazon EFS)
* Your Plex Media Server *must* be configured to allow connections from
unauthorized users for your pod network, else the transcode job is unable to
report information back to Plex about the state of the transcode job. At some
point in the future this may change, but it is a required step in order to make
transcodes work right now.

## Setup

This guide will go through setting up a Plex Media Server instance on a
Kubernetes cluster, configured to launch transcode jobs on the same cluster
in pods created in the same 'plex' namespace.

1) Obtain a Plex Claim Token by visiting [plex.tv/claim](https://plex.tv/claim).
This will be used to bind your new PMS instance to your own user account
automatically.

2) Deploy the Helm chart included in this repository using the claim token
obtained in step 1. If you have pre-existing persistent volume claims for your
media, you can specify its name with `--set persistence.data.claimName`. If not
specified, a persistent volume will be automatically provisioned for you.

```bash
➜  helm install plex ./charts/kube-plex \
    --namespace plex \
    --set claimToken=[insert claim token here] \
    --set persistence.data.claimName=existing-pms-data-pvc \
    --set ingress.enabled=true
```

This will deploy a scalable Plex Media Server instance that uses Kubernetes as
a backend for executing transcode jobs.

3) Access the Plex dashboard, either using `kubectl port-forward`, or using
the services LoadBalancer IP (via `kubectl get service`), or alternatively use
the ingress provisioned in the previous step (with `--set ingress.enabled=true`).

4) Visit Settings->Server->Network and add your pod network subnet to the
`List of IP addresses and networks that are allowed without auth` (near the
bottom). For example, `10.100.0.0/16` is the subnet that pods in my cluster are
assigned IPs from, so I enter `10.100.0.0/16` in the box.

You should now be able to play media from your PMS instance - pods will be
created to handle transcodes, and data automatically mounted in appropriately:

```bash
➜  kubectl get po -n plex
NAME                              READY     STATUS    RESTARTS   AGE
plex-kube-plex-75b96cdcb4-skrxr   1/1       Running   0          14m
pms-elastic-transcoder-7wnqk      1/1       Running   0          8m
```

## More Setup for Raspberry Pi

If you want to expand on this deployment with helm like this:

```bash
helm install plex kube-plex/charts/kube-plex/ \
  --values media.plex.values.yml \
  --namespace plex
```

Your `media.plex.values.yml` file can look like the following:

```yaml
# media.plex.values.yml

claimToken: "<CLAIM_TOKEN>" # Replace `<CLAIM_TOKEN>` by the token obtained previously. https://www.plex.tv/claim/

image:
  repository: linuxserver/plex
  tag: arm32v7-latest
  pullPolicy: IfNotPresent

kubePlex:
  enabled: false # kubePlex (transcoder job) is disabled because not available on ARM. The transcoding will be performed by the main Plex instance instead of a separate Job.

timezone: America/New_York # Replace with your own timezone based on the TZ Database Name value: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones

service:
  type: LoadBalancer # We will use a LoadBalancer to obtain a virtual IP that can be exposed to Plex Media via our router
  port: 32400 # Port to expose Plex

rbac:
  create: true

nodeSelector:
  beta.kubernetes.io/arch: arm # This is here so it can work on the Raspberry Pi Kubernetes Cluster

persistence:
  soloVolume: # Use this object only if you want to mount everything onto a single volume/volumeClaim
    enabled: true
    volumeName: "<VOLUME_NAME_HERE>"
    claimName: "<VOLUME_CLAIM_NAME_HERE>"
  transcode:
    subPath: "plex/transcode"
    storageClass: "manual"
  data:
    subPath: "plex/data"
    storageClass: "manual"
  config:
    subPath: "plex/config"
    storageClass: "manual"

proxy:
  enable: false
```

### Additional Raspberry Pi requirements

The following MUST be set in a configuration setting of your choice. This is to ensure that your Plex application is not looping on `CreatingContainer` endlessly since your Kubernetes Master Node is unable to match against the default value of `nodeSelector: beta.kubernetes.io/arch: amd64`.

```yaml
nodeSelector:
    beta.kubernetes.io/arch: arm # Set to arm so it works on a Raspberry Pi
```

### volumeName addition

This fork is different from the parent where the following options are passed down to the `deployment.yaml` file:

```yaml
persistence:
  soloVolume:
    enabled: true
    volumeName: "<VOLUME_MOUNT_NAME_HERE>"
    claimName: "<VOLUME_MOUNT_NAME_HERE>"
```

Before the Volume Mounts would be defined by default to their respective parent designations of `transcode`, `data`, and `config`. But I didn't want to have separate PVC/PV's set-up for each Plex element.

If:
  - `soloVolume.enabled` is set to `true`
  - `soloVolume.volumeName` and `soloVolume.claimName` are defined

All 3 (`transcode`/`data`/`config`) should now be sharing the same `volumeName` and `claimName`.

I was able to create a Persistent Volume:
```yaml
# media.persistentvolume.yml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "media-ssd"
  labels:
    type: "local"
spec:
  storageClassName: "manual"
  capacity:
    storage: "250Gi"
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/mnt/ssd/media"
---
```

and a Persistent Volume Claim:
```yaml
# media.persistentvolumeclaim.yml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: "media"
  name: "media-ssd"
spec:
  storageClassName: "manual"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: "250Gi"
---
```

and use 
```yaml
persistent:
  soloVolume:
    volumeName: media-ssd
    claimName: media-ssd
```

it was able to populate within the PV as expected under `/mnt/ssd/media/<plex-directory-here>`.