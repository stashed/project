# Support Multiple Repositories and Schedules


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Support Multiple Repositories and Schedules](#support-multiple-repositories-and-schedules)
  - [Motivation](#motivation)
    - [Goal](#goal)
  - [Proposal](#proposal)
    - [Changes in CRDs](#changes-in-crds)
      - [Changes in `BackupConfiguration` CRD](#changes-in-backupconfiguration-crd)
      - [Changes in `Repository` CRD](#changes-in-repository-crd)
      - [Changes in `BackupSession` CRD](#changes-in-backupsession-crd)
    - [How multiple schedules will work](#how-multiple-schedules-will-work)
    - [How multiple repositories will work](#how-multiple-repositories-will-work)
    - [How Auto-Backup will work](#how-auto-backup-will-work)
  - [Backward Compatibility](#backward-compatibility)
    - [Migration of `BackupConfiguration` CRD](#migration-of-backupconfiguration-crd)
    - [Migration of `Repository` CRD](#migration-of-repository-crd)
    - [Migration of `BackupBlueprint` CRD](#migration-of-backupblueprint-crd)
  - [Possible Problems and Solution](#possible-problems-and-solution)
      - [If multiple schedules has overlap, whom to chose?](#if-multiple-schedules-has-overlap-whom-to-chose)
      - [If some backup succeeded for some backend but not for all backends?](#if-some-backup-succeeded-for-some-backend-but-not-for-all-backends)
      - [How repository initialization will work?](#how-repository-initialization-will-work)
      - [How retention policy will be applied?](#how-retention-policy-will-be-applied)
  - [Backlogs](#backlogs)

<!-- /code_chunk_output -->

## Motivation

Currently, Stash allows only taking backup of a target into a single Repository with a single schedule. However, this is not sufficient in some use-cases. For example, a user want to schedule a daily backup into a `GCS` bucket and a weekly backup into a `S3` bucket. Although, currently this goal can be achieved by creating two different `BackupConfiguration` for the target. However, those `BackupConfiguration` won't co-ordinate with each others. As a result, the user might end up with running two different backups of a target at the same time. Also, this does not work well with the Auto-Backup process. Hence, it is necessary to have built-in support for this use-case from Stash operator.

### Goal

The goal of this proposal can be summarized as bellow:

- Support multiple schedules for a single target.
- Support multiple repositories per schedule.
- Support different retention policy for different repositories.
- Keep backward compatibility with v1beta1 API.

## Proposal

We can introduce a `session` field in `BackupConfiguration` CRD that will allow specifying multiple session schedule and their backend repositories.

### Changes in CRDs
 
In order to support this use-case, some CRD specification need to be changed. Respective changes has been outlined below.

#### Changes in `BackupConfiguration` CRD

Proposed `BackupConfiguration` YAML should be similar as below,

```yaml
apiVersion: stash.appscode.com/v1beta2
kind: BackupConfiguration
metadata:
  name: sts-backup
  namespace: demo
spec:
  driver: Restic
  sessions:
  - name: daily-backup
    type: full-backup
    schedule: "0 0 * * *" # At 00:00 of everyday
    repositories:
    - name: gcs-repo
    - name: minio-repo
  - name: weekly-backup
    type: full-backup
    schedule: "0 0 * * 5" # At 00:00 of every Friday
    repositories:
    - name: s3-repo
  target:
    alias: sample-sts
    ref:
      apiVersion: apps/v1
      kind: StatefulSet
      name: sample-sts
status:
  nextSessions:
  - name: daily-backup
    scheduledAt: "2020-10-14 Wed 12:00:00"
  - name: daily-backup
    scheduledAt: "2020-10-15 Thu 12:00:00"
  - name: daily-backup
    scheduledAt: "2020-10-16 Fri 12:00:00"
  - name: weekly-backup
    scheduledAt: "2020-10-16 Fri 12:00:00"
```

The following changes is being proposed in the `BackupConfiguration` crd,

- Introduce `spec.sessions` section. This section may have the following fields:
  - `name` specifies the name of the session. User should give a meaningful name of the session. This name must be unique within a `BackupConfiguration`.
  - `type` specifies the type of the backup this session will perform. It can be either `full-backup` or `incremental-backup`. We may introduce other types in future if necessary.
  - `schedule` specifies a CronExpression that determine when this session should be invoked.
  - `repositories` specifies a list of repository where the backed up data of this session should be stored.
- Remove `spec.schedule` field as it now has been moved under `sessions` section.
- Remove `spec.repository` section as new `repositories` section has been introduced in `sessions`section.
- Move `spec.retentionPolicy` into the `Repository` crd.
- Add `status.nextSessions` section which will give the users and indication of when the next session will run.

#### Changes in `Repository` CRD

Since, we are supporting multiple repositories per `BackupConfiguration`, the `retentionPolicy` should be moved under the `Repository` crd. This will allow to use different `retentionPolicy` for different repositories.

The sample YAML of the new `Repository` crd is given below,

```yaml
apiVersion: stash.appscode.com/v1beta2
kind: Repository
metadata:
  name: gcs-repo
  namespace: demo
spec:
  backend:
    gcs:
      bucket: stash-backup
      prefix: demo
    storageSecretName: gcs-secret
  retentionPolicy:
    name: "keep-last-5"
    keepLast: 5
    prune: true
```

#### Changes in `BackupSession` CRD

Since, now there could be multiple session for each `BackupConfiguration`, the `BackupSession` must uniquely point to the respective session.

Here is the sample YAML of proposed `BackupSession` CRD,

```yaml
apiVersion: stash.appscode.com/v1beta2
kind: BackupSession
metadata:
  labels:
    stash.appscode.com/invoker-name: sts-backup
    stash.appscode.com/invoker-type: BackupConfiguration
  name: sts-backup-1578458376
  namespace: demo
spec:
  type: full-backup
  sessionName: daily-backup
  invoker:
    apiGroup: stash.appscode.com
    kind: BackupConfiguration
    name: mariadb-backup
status:
  phase: Succeeded
  sessionDuration: 22.575920065s
  targets:
  - phase: Succeeded
    ref:
      apiVersion: apps/v1
      kind: StatefulSet
      name: sample-mariadb
    repositories:
    - name: gcs-repo
      phase: Succeeded
    - name: minio-repo
      phase: Succeeded
    totalHosts: 1
```

Here we have introduced the following fields

- `spec.type` specifies the type of backup. Respective backup executor will use this information to differentiate between `full-backup` and `incremental-backup`.
- `spec.sessionName` specifies the name of the session. Respective backup executor will use this information to identify the repositories for this BackupSession.
- `status.target[*].repositories` specifies the backup status of this target in different repositories.

### How multiple schedules will work

Stash will create backup triggering CronJob for each schedules. The CronJobs will create `BackupSession` for triggering the respective session. If `BackupSession` with same `type` and same `sessionName` already exist and haven't completed its backup, the CronJobs will not create any new `BackupSession` until the existing one completes.

Stash operator will make sure that no two `BackupSession` for a particular BackupConfiguration runs simultaneously. It will keep one of them in `Pending` state. When the other complete its backup, Stash operator will then trigger backup for the next `BackupSession`.

### How multiple repositories will work

For filepath backup, Stash will run the backup process against each each Repository. However, for database backup where Stash dump the database and pipe the output into `restic`, we must split/duplicate the piped data into different `restic` process that are running against the individual repository.

### How Auto-Backup will work

Since, `BackupConfiguration` and `Repository` CRDs are changing, we must also update the `BackupBlueprint` to support the new functionalities.

Here is the proposed `BackupBlueprint` YAML structure:

```yaml
apiVersion: stash.appscode.com/v1beta2
kind: BackupBlueprint
metadata:
  name: workload-backup-blueprint
spec:
  # ============== Blueprint for Repository ==========================
  repoTemplates:
  - name: gcs
    backend:
      gcs:
        bucket: appscode-qa
        prefix: stash-backup/${TARGET_NAMESPACE}/${TARGET_KIND}/${TARGET_NAME}
      storageSecretName: gcs-secret
    retentionPolicy:
      name: 'keep-last-5'
      keepLast: 5
      prune: true
  - name: s3
    backend:
      s3:
        bucket: appscode-qa
        prefix: stash-backup/${TARGET_NAMESPACE}/${TARGET_KIND}/${TARGET_NAME}
      storageSecretName: s3-secret
    retentionPolicy:
      name: 'keep-last-10'
      keepLast: 10
      prune: true
  # ============== Blueprint for BackupConfiguration =================
  sessions:
  - name: daily-full-backup
    type: full-backup
    schedule: "0 0 * * *"
    repositories:
    - gcs
  - name: daily-incremental-backup
    type: incremental-backup
    schedule: "*/15 * * * *"
    repositories:
    - gcs
  - name: weekly-backup
    type: full-backup
    schedule: "0 0 * * 5"
    repositories:
    - s3
```
Here, `repoTemplates[*].name`  will be used as prefix of the auto-generated Repositories. Stash will add additional target specific suffix to the repositories. Stash will also update the `sessions[*].repositories` field to accurately point to the respective Repository objects.

Then we can support the following configurations via annotations:

- Use only particular sessions for a target. By default Stash will create BackupConfiguration with all sessions.
```yaml
sessions.stash.appscode.com/names: "daily-full-backup,weekly-backup"
```

- Customize schedule for a session

```yaml
schedule.stash.appscode.com/daily-full-backup: "30 11 * * *"
schedule.stash.appscode.com/weekly-backup: "00 12 * * *"
```
- Customize repositories for a session

```yaml
repositories.stash.appscode.com/daily-full-backup: "gcs,nfs"
repositories.stash.appscode.com/weekly-backup: "s3"
```

## Backward Compatibility

In order to keep backward compatibility with v1beta1 APIs, we must run migration on `BackupConfiguration`, `Repository`, and `BackupBlueprint` objects.

### Migration of `BackupConfiguration` CRD

The following migration is required for `BackupConfiguration` objects:

- Add a session field with information from `spec.schedule`, and `spec.repository` field and remove the info from there.
- Find the respective `Repository` for the `BackupConfiguration` and move the `retentionPolicy` into the `Repository`. Then, remove it from `BackupConfiguration`.

### Migration of `Repository` CRD

The retention policy now will be properties of the Repository CRD. Hence, we must find the respective `BackupConfiguration` and move its `retentionPolicy` into the `Repository`. However, we don't need to perform this migration since the repository will updated during migration of `BackupConfiguration`.

### Migration of `BackupBlueprint` CRD

We must migrate the existing `BackupBlueprint` to comply with the new structure. We must ensure that after migration newly auto-generated `BackupConfiguration` and `Repository` name matches with the previously generated `BackupConfiguration` and `Repository` name. Otherwise, a target may have two different BackupConfigurations pointing to the same backend and scheduled at the same time.

We must also consider supporting existing ` stash.appscode.com/schedule: <Cron Expression>` annotation. Otherwise, we have to migrate the respective workloads too.

## Possible Problems and Solution

We may face the following issues for this features.

#### If multiple schedules has overlap, whom to chose?

The CronJobs will ensure that there is no two uncompleted `BackupSession` of same `type` and same `sessionName` . If there is two `BackupSession` of different `type` or different `sessionName`, Stash operator will chose one of them (maybe full-backup type can have higher priority) and keep the other in `Pending` state. Eventually, both of the session will take their respective backup.

#### If some backup succeeded for some backend but not for all backends?

Overall `BackupSession` phase should be `Failed`. The status section of `BackupSession` session will have phase entry for each backend.

#### How repository initialization will work?

Currently, Stash operator assign pre-backup action to one of the backup executor. The backup executor initialize repository if it is not initialized already. Now, same executor will initialize repository for all backend for this session.

#### How retention policy will be applied?

Same as current. However, this time instead of applying retention policy into a single repository, now the process will apply for every repositories.

## Backlogs

Todo:
- [ ] Do a POC of splitting piped data into different `restic` process running against different backend.

Ref:
- https://www.zupzup.org/io-pipe-go/
- https://rodaine.com/2015/04/async-split-io-reader-in-golang/

