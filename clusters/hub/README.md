# Hub application examples

> Note: this builds on top of the example [Deploying Advanced Cluster Management and OpenShift Data Foundation for ARO Disaster Recovery](https://cloud.redhat.com/experts/aro/acm-odf-aro/) with applications to try out. I did not test out private cluster with VNET peering.

Hub GitOps is the app-of-apps chart in `values.yaml`. Managed clusters are `primary-cluster` (East US) and `secondary-cluster` (Central US), in ManagedClusterSet `aro-clusters`.

OpenShift GitOps reads PlacementDecisions only in `openshift-gitops` (`acm-placement`). Any Placement that an ApplicationSet follows has to live in that namespace.

Shared DR setup, used by the PVC examples:

| Piece | Role |
|---|---|
| `MirrorPeer` `mirrorpeer` | Pairs the two StorageClusters and, with `manageS3: true`, creates the Ramen metadata bucket. It does not replicate application buckets. |
| `DRPolicy` `drpolicy` | Names the two DRClusters and the 5m interval. |
| `DRPlacementControl` | Selects the running cluster and protects PVCs. `preferredCluster` is `primary-cluster`, `failoverCluster` is `secondary-cluster`. |

Docs: [Regional-DR for OpenShift Data Foundation 4.20](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.20/html/configuring_openshift_data_foundation_disaster_recovery_for_openshift_workloads/rdr-solution), [Application health in the console](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.20/html/configuring_openshift_data_foundation_disaster_recovery_for_openshift_workloads/monitoring_disaster_recovery_health).

The console Failover and Relocate actions edit `DRPlacementControl.spec.action`. These hub apps use selfHeal, so a console edit is overwritten unless the same field is in Git. Ramen sets `cluster.open-cluster-management.io/backup=ramen` on the DRPlacementControl; the apps ignore that label.

## stateless-sample and stateless-echo

No storage and no ODF. One Placement, `stateless-sample-placement` (`numberOfClusters: 1`), and two ApplicationSets that watch it. Changing the Placement moves both apps.

Failover:

1. In `components/stateless-sample/placement-stateless-sample.yaml`, set `matchLabels.name` to `secondary-cluster`.
2. Sync hub app `stateless-sample`.
3. ACM writes a new PlacementDecision. Argo CD prunes both apps on `primary-cluster` and creates them on `secondary-cluster`.

Move back:

1. Set `matchLabels.name` back to `primary-cluster`.
2. Sync hub app `stateless-sample`. Argo CD prunes both apps on `secondary-cluster` and creates them on `primary-cluster`.

Features: RHACM Placement, ManagedClusterSet, OpenShift GitOps `clusterDecisionResource`. There is no DRPlacementControl and no console Failover button.

Docs: [RHACM GitOps](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/applications/gitops-overview).

## stateless-replicas and replicas-echo

No storage. Placement `stateless-replicas-placement` keeps both clusters selected, including when one is unavailable, so the Applications are not pruned. Replica counts come from `components/stateless-replicas/cluster-replicas/`.

Failover:

1. In `components/stateless-replicas/cluster-replicas/`, set `replicas: 0` in `primary-cluster.yaml` and `replicas: 1` in `secondary-cluster.yaml`.
2. Sync hub app `stateless-replicas`. The Deployments stay on both clusters. Primary scales to 0 and secondary scales to 1.

Move back:

1. Set `replicas: 1` in `primary-cluster.yaml` and `replicas: 0` in `secondary-cluster.yaml`.
2. Sync hub app `stateless-replicas`.

The count is applied with a Kustomize JSON patch. `source.kustomize.replicas.count` on this GitOps version is rendered as 0.

Features: same Placement and ApplicationSet path as the stateless sample, plus an ApplicationSet merge of the PlacementDecision with the per-cluster Git files.

## busybox-sample

GitOps workload from `workloads/deployment/odr-regional-rbd` in `RamenDR/ocm-ramen-samples`. The PVC label is `appname: busybox`. Placement `busybox-placement` has scheduling disabled, so only the DRPlacementControl writes the PlacementDecision. Both objects are in `openshift-gitops`.

Failover:

1. Set `spec.action: Failover` on `components/busybox-sample/drplacementcontrol-busybox-drpc.yaml`.
2. Sync hub app `busybox-sample`. Ramen demotes the primary VolumeReplicationGroup, promotes the secondary, and moves the PlacementDecision to `secondary-cluster`. The ApplicationSet follows that decision.
3. When `status.phase` is `FailedOver`, the pod is on `secondary-cluster`.

Move back:

1. Set `spec.action: Relocate`. Relocate is the planned return to `preferredCluster`.
2. When `status.phase` is `Relocated`, delete `spec.action` and sync.

Clearing `action` while the app is still failed over does not move it back.

Features: MirrorPeer, DRPolicy, DRCluster, DRPlacementControl, RBD mirroring, Placement with `cluster.open-cluster-management.io/experimental-scheduling-disable`. VolSync covers CephFS; this sample is RBD.

## busybox-subscription

Same workload and the same `drpolicy`, deployed with an RHACM Subscription instead of an ApplicationSet. Channel, Subscription, PlacementRule (`schedulerName: ramen`), and DRPlacementControl all live in `busybox-subscription`. The PlacementRule and the DRPlacementControl must share that namespace.

Failover:

1. Set `spec.action: Failover` on `components/busybox-subscription/drplacementcontrol.yaml`.
2. Sync hub app `busybox-subscription`. Ramen moves the PlacementRule decision to `secondary-cluster`, and the Subscription deploys the workload there.
3. Wait until `status.phase` is `FailedOver`.

Move back:

1. Set `spec.action: Relocate`.
2. Sync hub app `busybox-subscription`. Ramen returns the workload to `primary-cluster`.
3. When `status.phase` is `Relocated`, delete `spec.action` and sync.

Clearing `action` while the phase is still `FailedOver` leaves the workload on `secondary-cluster`. There is no ApplicationSet.

Features: RHACM Application, Channel, Subscription, and PlacementRule, plus the same ODF DR objects as `busybox-sample`.

## obc-sample

Object buckets are not protected by DRPlacementControl. `MirrorPeer` `manageS3` is only the Ramen metadata bucket.

Each cluster has data bucket `obc-sample`. A namespace bucket on that NooBaa points at the other cluster's S3 route, and the data bucket's replication rule copies objects there. Both buckets stay in place. Only the writer moves.

Failover:

1. In `components/obc-sample/placement-writer.yaml`, set `matchLabels.name` to `secondary-cluster`.
2. Sync hub app `obc-sample`.
3. Argo CD prunes the writer on `primary-cluster` and starts it on `secondary-cluster`. It uses secondary's local `obc-sample` bucket. Objects already copied from primary are there. New writes on secondary are copied to primary by the rule whose destination is `obc-sample-to-primary`.

Move back:

1. Set `matchLabels.name` back to `primary-cluster`.
2. Sync hub app `obc-sample`. Argo CD prunes the writer on `secondary-cluster` and starts it on `primary-cluster`.
3. The writer uses primary's local bucket. Objects written while it ran on secondary are already being copied there. The copy is asynchronous, so the newest object can still be in flight for one replication pass.

Secret `obc-sample-peer-s3` in `openshift-storage` on each cluster holds the peer bucket keys. It is created from the peer ObjectBucketClaim secret and is not in Git.

Features: RHACM Placement and ApplicationSets for the buckets (both clusters) and the writer (one cluster). NooBaa NamespaceStore and [MCG bucket replication](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.20/html/managing_hybrid_and_multicloud_resources/multicloud_object_gateway_bucket_replication). No DRPolicy and no console Failover button.
