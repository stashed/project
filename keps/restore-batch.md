# Introduce RestoreBatch + Fix BackupBatch Issue

<!-- code_chunk_output -->

- [Introduce RestoreBatch + Fix BackupBatch Issue](#introduce-restorebatch--fix-backupbatch-issue)
  - [Motivation](#motivation)
    - [Goal](#goal)
  - [Fix BackupBatch Issue](#fix-backupbatch-issue)
    - [Introduce `alias` field in `target`](#introduce-alias-field-in-target)
    - [Changes in BackupConfiguration CRD](#changes-in-backupconfiguration-crd)
    - [Changes in BackupBatch CRD](#changes-in-backupbatch-crd)
    - [Changes in RestoreSession CRD](#changes-in-restoresession-crd)
  - [Introduce `RestoreBatch` CRD](#introduce-restorebatch-crd)
  - [Backup and restore order for `BackupBatch` and `RestoreBatch`](#backup-and-restore-order-for-backupbatch-and-restorebatch)

<!-- /code_chunk_output -->

## Motivation

Currently, we have `BackupBatch` CRD that allows the users to backup multiple targets under a single configuration. So, it is a logical step to provide a `RestoreBatch` CRD that will allow the users to restore the targets that were backed up by a `BackupBatch` under a single configuration. This doc proposes a possible structure of the `RestoreBatch` CRD. This doc also proposes a fix to the `BackupBatch` issue that happened due to not ensuring 1:1 mapping between `Repository` and `BackupBatch`.

### Goal

The goal of this KEPS can be summarized as below:

1. Ensure 1:1 Mapping between `Repository` and `BackupBatch` CRD.
2. Introduce `RestoreBatch` CRD.
3. Finalize backup and restore behavior for `BackupBatch` and `RestoreBatch`.
4. Discuss how to keep backward compatibility.

## Fix BackupBatch Issue

The underlying [restic](https://restic.net/) tools use `hostname` to identify the backed up data of a target. All targets under a single repository must have a unique `hostname`. Currently, we generate those `hostname` automatically based on the target types. In this approach, it is possible to generate the same `hostname` for different members of a `BackupBatch` CR. That's why it was not possible to use a single repository for the members of a `BackupBatch`. This results some issues like [stashed/stash#1039](https://github.com/stashed/stash/issues/1039).

In order to fix the issue, we have to to ensure 1:1 mapping between `Repository` and `BackupBatch`. This requires some changes in the `BackupBatch`, `BackupConfiguration`, and `RestoreSession` CRD. This section will discuss which change is necessary for those CRDs.

### Introduce `alias` field in `target`

Instead of auto-generating `hostname`, we can introduce `alias` field in the `target` section. We will use this `alias` value as the `hostname`. It will be the user's responsibility to provide unique `alias` for the members of a `BackupBatch`. We can also use validator to enforce it.

```yaml
target:
  alias: app # this will be used as hostname in the actual restic repository
  ref:
    apiVersion: apps/v1
    kind: Deployment
    name: wordpress
  volumeMounts:
    - name: wordpress-persistent-storage
      mountPath: /var/www/html
  paths:
    - /var/www/html
```

Now, the `hostname` generation will follow the following patterns:

1. For `Deployment`, `ReplicaSet`, `ReplicationController`, `DeploymentConfig` , and `AppBinding`,  the `alias` value will be used directly as the `hostname`.
2. For `StatefulSet` the `hostname` will be generated as `$alias-$POD_INDEX` for the individual pods.
3. For `DaemonSet` the `hostname` will be generated as `$alias-$NODE_NAME` for each daemon pod running in different nodes.

**Backward Compatibility:**

1. If the user doesn't provide this `alias` field, we can default it to the respective auto-generated `hostname`.
2. For existing `BackupConfiguration` or `BackupBatch` we can default this field to the respective auto-generated `hostname`.

### Changes in BackupConfiguration CRD

For `BackupConfiguration` only `alias` field will be added to the `target` section. No other changes will be necessary.

```yaml
apiVersion: stash.appscode.com/v1beta1
kind: BackupConfiguration
metadata:
  name: deployment-backup
  namespace: demo
spec:
  repository:
    name: gcs-repo
  schedule: "*/5 * * * *"
  target:
    alias: demo-app # this will be used as "hostname"
    ref:
      apiVersion: apps/v1
      kind: Deployment
      name: stash-demo
    volumeMounts:
    - name: source-data
      mountPath: /source/data
    paths:
    - /source/data
  retentionPolicy:
    name: 'keep-last-5'
    keepLast: 5
    prune: true
```

### Changes in BackupBatch CRD

The `target` field of the individual member will have `alias` field. The `alias` value for each individual target should be unique across all members of the `BackupBatch`.

```yaml
apiVersion: stash.appscode.com/v1beta1
kind: BackupBatch
metadata:
  name: deploy-backup-batch
  namespace: demo
spec:
  repository:
    name: gcs-repo
  schedule: "*/5 * * * *"
  members:
  - target:
      alias: db # this will be used as "hostname" for the database
      ref:
        apiVersion: apps/v1
        kind: AppBinding
        name: sample-mysql
    task:
      name: mysql-backup-8.0.14
  - target:
      alias: app # this will be used as "hostname" for the deployment
      ref:
        apiVersion: apps/v1
        kind: Deployment
        name: wordpress
      volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      paths:
        - /var/www/html
  retentionPolicy:
    name: 'keep-last-10'
    keepLast: 10
    prune: true
```

### Changes in RestoreSession CRD

For `RestoreSession` CRD, we have to make the following changes:

1. Move the `rules` section into `target` section and mark the current `rule` section deprecated. This is not related to this `BackupBatch` issue but it will be necessary for `RestoreBatch`.
2. Add `alias` field in the target section. This will be used to generate `sourceHost` and `targetHosts` if the user doesn't specify them. The `alias` must match with the backed up target `alias`.

```yaml
apiVersion: stash.appscode.com/v1beta1
kind: RestoreSession
metadata:
  name: statefulset-restore
  namespace: demo
spec:
  driver: Restic
  repository:
    name: gcs-repo
  target:
    alias: app # must match with the alias of the backed up target
    ref:
      apiVersion: apps/v1
      kind: StatefulSet
      name: recovered-statefulset
    volumeMounts:
    - mountPath: /source/data
      name: source-data
    rules:
    - targetHosts: ["app-3","app-4"] # "app-3" and "app-4" will have restored data of   backed up host "app-1"
      sourceHost: "app-1" # source host
      paths:
      - /source/data
    - targetHosts: [] # empty host match all hosts
      sourceHost: "" # no source host indicates that the hostname will be generated from the alias as "app-$POD_INDEX" for the respective pods.
      paths:
      - /source/data
```

**Backward Compatibility:**

1. If the user doesn't specify the `alias` field, we can default it with auto-generated `hostname` that matches the current behavior.
2. If the user provides `rules` in the deprecated section, we can mutate the `RestoreSession` to move it into the inside `target` section.
3. `RestoreSession` is a one-time object. Hence, it is not necessary to mutate already existing `RestoreSession` objects.

## Introduce `RestoreBatch` CRD

A `RestoreBatch` CRD should allow specify multiple members as `target`. Each  `target` should have its own restore `rules`, `task`, `hooks`, `runtimeSettings`. A sample `RestoreBatch` object should be look like as the following:

```yaml
apiVersion: stash.appscode.com/v1beta1
kind: RestoreBatch
metadata:
  name: wordpress-restore
  namespace: demo
spec:
  driver: Restic
  repository:
    name: gcs-repo
  members:
  - target:
      alias: db
      ref:
        apiVersion: apps/v1
        kind: AppBinding
        name: sample-mysql
    task:
      name: mysql-backup-8.0.14
    rules:
    - snapshots: [latest]
  - target:
      alias: app
      ref:
        apiVersion: apps/v1
        kind: Deployment
        name: wordpress
      volumeMounts:
      - name: wordpress-persistent-storage
        mountPath: /var/www/html
    rules:
    - paths:
      - /source/data
```

The `.status` section of the `RestoreBatch` should be similar to the `.status` section of the `BackupBatch`.

## Backup and restore order for `BackupBatch` and `RestoreBatch`

The underlying `restic` tool can backup and restore in parallel to and from a single repository for different `hosts`. So, the members of the `BackupBatch` or `RestoreBatch` can be backed up / restored in parallel. However, there might be a case where the user may need an order guarantee for the members. In this case, we can do the followings:

1. Introduce a top-level boolean field named `maintainOrder` in the `.spec` section of the `BackupBatch` or `RestoreBatch`. If the value of this field is `false` (which is the default value), we can run the backup/restore in parallel. This will match our current behavior. If it is `true` then we will run backup/restore in the same order as the targets are specified in `members` section.
2. We can always run the backup/restore maintaining the order. In this approach, the user may lose the performance benefits that were possible in parallel backup/restore.
