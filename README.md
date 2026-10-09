# Radarr on k3s

Radarr runs as a single-instance Deployment in the `default` namespace. Its
configuration uses the existing dedicated Synology iSCSI LUN; movie and
download data is mounted from the existing `synology-data` claim at `/data`.

## Placement and image

The pod is pinned to `k3s-worker-02` using the hostname, iSCSI, and internet
egress labels. Radarr needs the worker's approved internet access for metadata
lookups. The workload image is pinned to an immutable digest in the private
registry and uses the `image-registry-credentials` pull secret. Worker-02 must
retain private-registry CA trust and access to TCP/32050.

- Upstream: `docker.io/linuxserver/radarr@sha256:7dfd049e79c00b16fbc29c3f5d96a9e7b9e73a23930b4c5b3c4541d60b366814`
- Internal workload image: `192.168.50.222:32050/radarr/radarr@sha256:7dfd049e79c00b16fbc29c3f5d96a9e7b9e73a23930b4c5b3c4541d60b366814`

## Storage and backups

`radarr-existing-lun-pv.yaml` statically binds the dedicated 5 GiB LUN named
`k8s-csi-pvc-b68ef9fe-333d-4fb4-a930-102f3622189e` using DSM's internal UUID
`9238a561-1bca-4116-97e5-6f7b378394e7`. The PV reclaim policy is `Retain`.
The previous Radarr configuration is stored in the LUN's `radarr/config`
subdirectory, which the Deployment mounts at `/config`. Do not mount the LUN
root instead, or Radarr will initialize a separate empty configuration. Do not
delete or recreate the PV/PVC or LUN as a troubleshooting shortcut; `/config`
contains Radarr's database, settings, and credentials. Back up this
configuration separately from these manifests.

The shared `synology-data` PVC is an external dependency and is not managed in
this repository. It must remain mounted at `/data` for Radarr to access the
movie and download folders. Back up media independently.

## Access and HAProxy

The Service exposes container HTTP port `7878` through NodePort `30787`.
Configure the internal HAProxy backend to use `k3s-worker-02.home:30787` over
HTTP. For backend health checks, send `GET /ping` and expect HTTP 200. The
application's in-cluster startup, readiness, and liveness probes use the same
path on container port `7878`.

The normal browser URL is `https://radarr.styxut.net` when the HAProxy backend
is configured. The direct diagnostic/backend URL is
`http://k3s-worker-02.home:30787`.

## Apply and verify

Validate and apply from this directory:

```sh
kubectl apply --dry-run=server -k .
kubectl apply -k .
kubectl rollout status deployment/radarr -n default --timeout=300s
kubectl get pods,svc,endpoints -n default -l app=radarr -o wide
kubectl logs -n default deployment/radarr
```

Radarr does not expose a native Prometheus metrics endpoint; `/ping` is a health
check, not a metrics endpoint, so this deployment does not create a
ServiceMonitor.
