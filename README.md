# infra-btrfs-csi

Argo CD deployment of the btrfs CSI driver for the Talos workers.

The cluster's ApplicationSet discovers this repository through its `argocd`
GitHub topic, renders the Helm chart in `argocd/`, and deploys it into
`infra-btrfs-csi`. Automated sync, pruning, and self-healing are configured
by the ApplicationSet.

## Storage

The host's `/var/mnt` is mounted at `/var/lib/btrfs-csi` inside the driver.
The worker filesystems must already be formatted as btrfs and mounted by Talos.
This deployment does not format disks or change the Talos machine configuration.

| Talos Mount | Pool | VolumeSnapshotClass |
| --- | --- | --- |
| `/var/mnt/storage-hot` | `storage-hot` | `btrfs-hot` |
| `/var/mnt/storage-archive` | `storage-archive` | `btrfs-archive` |

| StorageClass | Pool | Reclaim Policy | Copy-on-Write |
| --- | --- | --- | --- |
| `hot-retain-cow` | `storage-hot` | Retain | Enabled |
| `hot-retain-inplace` | `storage-hot` | Retain | Disabled |
| `hot-delete-cow` | `storage-hot` | Delete | Enabled |
| `hot-delete-inplace` | `storage-hot` | Delete | Disabled |
| `archive-retain-cow` | `storage-archive` | Retain | Enabled |
| `archive-retain-inplace` | `storage-archive` | Retain | Disabled |
| `archive-delete-cow` | `storage-archive` | Delete | Enabled |
| `archive-delete-inplace` | `storage-archive` | Delete | Disabled |

No StorageClass is the default. Volumes are node-local and use
`WaitForFirstConsumer` binding. All classes allow expansion. The `inplace`
variants set the driver's `cow` parameter to `"false"`; the `cow` variants
set it to `"true"`. With `Retain`, deleting a PVC leaves its PV and data for
manual reclamation. With `Delete`, deleting a PVC also reclaims its PV and data.
StorageClasses are configured in `argocd/values.yaml`.

## Deployment

The chart includes the privileged driver DaemonSet, CSI sidecars and RBAC,
snapshot CRDs, a two-replica distributed snapshot controller, and pool classes.
All namespaced resources use the Helm release namespace. CSI registration,
CRDs, cluster RBAC, and storage/snapshot classes are cluster-scoped.

Sync waves install the namespace and CRDs first, then the snapshot controller,
then the driver, and finally the pool classes. Snapshot CRD pruning and deletion
require Argo CD confirmation to avoid accidentally deleting snapshot metadata.

```bash
helm lint argocd
helm template infra-btrfs-csi argocd \
  --namespace infra-btrfs-csi --include-crds
kubectl -n infra-argocd get application infra-btrfs-csi
kubectl -n infra-btrfs-csi get pods
```

## Sources

Driver manifests are vendored from the `klodzack/btrfs-csi` fork at commit
`ea0b5c45a8ec3c5203e258c64383d7a62ccf5c7a`, with the Talos mount mapping and
pool classes used in the manual deployment. The driver image is pinned in
`argocd/values.yaml` to the fork's `latest` image digest, not the upstream
`mporrato` image. Renovate updates and automerges driver digest changes;
Argo CD then deploys the updated pin. These merges do not require CI checks
because this repository currently has none. Other images are not auto-updated.

Snapshot CRDs, RBAC, and controller manifests are vendored from
`kubernetes-csi/external-snapshotter` tag `v8.0.0`. The controller image is also
`v8.0.0`, with distributed snapshotting enabled.
