# Backup Verification


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Backup Verification](#backup-verification)
  - [Motivation](#motivation)
    - [Goal](#goal)
  - [Proposal](#proposal)
    - [Changes in CRD](#changes-in-crd)
      - [Changes in `BackupConfiguration` CRD](#changes-in-backupconfiguration-crd)
      - [New `BackupVerificationSession` CRD](#new-backupverificationsession-crd)
  - [How it is going to work](#how-it-is-going-to-work)
      - [When to run backup verification](#when-to-run-backup-verification)
      - [If multiple verification schedule overlap, which one should be given higher priority](#if-multiple-verification-schedule-overlap-which-one-should-be-given-higher-priority)
      - [How to schedule verification after each backup](#how-to-schedule-verification-after-each-backup)
      - [How to schedule verification after few backup](#how-to-schedule-verification-after-few-backup)
      - [How to trigger backup verification manually](#how-to-trigger-backup-verification-manually)
      - [How `BackupVerificationSession` will be cleaned up](#how-backupverificationsession-will-be-cleaned-up)

<!-- /code_chunk_output -->

## Motivation

Currently, Stash does not verify the backed up data. It just only check if the repository is corrupted or not after each backup. However, for mission critical data, user must be sure that the backed up data is in consistent state and he can restore his application from it. It will be highly appreciated if such verification can be done automatically by Stash. In this proposal, we are going to discuss about a possible implementation of how Stash can automatically verify the backed up data.

### Goal

The goal of this proposal can be summarized as bellow:

- Support automatic verification of backed up data.
- Support verification of subset of the sessions.
- Support verification of after every few backups.
- Support triggering backup verification manually.
- Support application specific verification logic.

## Proposal

For backup verification, we can follow the existing addon mechanism. Application specific verification process will be defined in terms of `Function` and `Task` CRDs. On a verification schedule, Stash will resolve those `Function` and `Task` and create a job to run verification.

### Changes in CRD

In order to support backup verification, we have to update `BackupConfiguration` crd and add new `BackupVerificationSession` CRD.

#### Changes in `BackupConfiguration` CRD

We have to add a `verifier` section under `spec.sessions`. This field will allow to specify the verification schedule and verifier etc.

The sample YAML of proposed `BackupConfiguration` is shown below:

```yaml
apiVersion: stash.appscode.com/v1beta2
kind: BackupConfiguration
metadata:
  name: mariadb-backup
  namespace: demo
spec:
  driver: Restic
  target:
    alias: sample-mariadb
    ref:
      apiVersion: apps/v1
      kind: StatefulSet
      name: sample-mariadb
  sessions:
  - name: full-backup-to-gcs-s3
    type: full-backup
    schedule: "0 0 * * *"
    repositories:
    - gcs-repo
    - s3-repo
    verify:
      schedule: "0 0 * * *"
      verifier: maria-backup-verify-full
      params:
      - name: releaseName
        value: sample-mariadb
  - name: incremental-backup-to-gcs
    type: incremental-backup
    schedule: "*/15 * * *"
    repositories:
    - gcs-repo
    verify:
      schedule: "*/45 * * * *"
      verifier: maria-backup-verify-quick
      params:
      - name: releaseName
        value: sample-mariadb
```

Here,
 - `verify.schedule` is a CronExpression that specifies when to run a verification.
 - `verify.verifier` specifies the `Task` name that will be used to run verification.
 - `verify.params` specifies some optional parameters that will be used to resolve the verifier task.

#### New `BackupVerificationSession` CRD

We are introducing a new CRD called `BackupVerificationSession`. Like the `RestoreSession` this is one time object. This represent a single run of backup verification.

The sample YAML of the proposed crd is give below,

```yaml
apiVersion: stash.appscode.com/v1beta2
kind: BackupVerificationSession
metadata:
  name: mariadb-backup-verification-123456
  namespace: demo
spec:
  sessionType: full-backup
  sessionName: full-backup-to-gcs-s3
  invoker:
    apiGroup: stash.appscode.com
    kind: BackupConfiguration
    name: mariadb-backup
  verifier: maria-backup-verify-full
  params:
  - name: releaseName
    value: sample-mariadb
  repositories:
  - gcs-repo
  - s3-repo
status:
  phase: Succeeded
  repositories:
  - name: gcs-repo
    phase: Succeeded
  - name: s3-repo
    phase: Succeeded
```

Here,
- `spec.sessionType` is an optional field that specifies the type of the session that is responsible for this verification session. For manual verification, user won't have to provide this field. 
- `spec.sessionName` is an optional field that specifies the name of the session that is responsible for this verification session. For manual verification, user won't have to provide this field.
- `spec.invoker` is an optional field that specify the respective backup invoker which is responsible for this verification session. For manual verification, user won't have to provide this field.
- `spec.verifier` is an required field that specify the name of Task to use for backup verification.
- `spec.params` is an optional field that can be used to pass optional parameters to the verifier Task.
- `spec.repositories` is required field that specify the list of repositories that the verifier should test against.

## How it is going to work

In each verification schedule, Stash will resolve the verifier `Task` and create a Job. The Job will first deploy an test application with similar configuration as the target. Then, it will restore the data from the backend. Then, it will run some test. Finally, it wil delete the test application and remove its resources.

#### When to run backup verification

Stash will not create any CronJob with the schedule specified in `verify` section. Instead, when a `BackupSession` successfully completes its backup, Stash will check whether there is a verification session between current backup session an next backup session. If there any, Stash will create a `BackupVerificationSession` to run verification immediately.

#### If multiple verification schedule overlap, which one should be given higher priority

Stash operator will ensure that there is no two `BackupVerificationSession` for same `sessionType` and same `sessionName`. If there are two `BackupVerificationSession` with different `sessionName`, Stash will run both of them. The `restore` process is read-only operation. So, there is no problem in running them in parallel.

#### How to schedule verification after each backup

In order to run verification after each backup, make verification schedule same as the session's schedule.

#### How to schedule verification after few backup

In order to run verification after few backup (for example, after every 3 backup), make verification schedule multiple of session's schedule. For example, if your session schedule run backup in each 15 minutes, make verification schedule to run at each 45 minutes to run verification after each 3 backup.

#### How to trigger backup verification manually

In order to trigger a backup verification manually, just create the `BackupVerificationSession`. You don't have to provide the optional session specific information. For example, you can just create the below `BackupVerificationSession` to trigger a verification manually,

```yaml
apiVersion: stash.appscode.com/v1beta2
kind: BackupVerificationSession
metadata:
  name: mariadb-backup-verification-123456
  namespace: demo
spec:
  verifier: maria-backup-verify-full
  params:
  - name: releaseName
    value: sample-mariadb
  repositories:
  - gcs-repo
  - s3-repo
```

#### How `BackupVerificationSession` will be cleaned up

Automatically created `BackupVerificationSession` will have the respective `BackupSession` as their owner. As a result, they will be cleaned up with the `BackupSession` according to `backupHistoryLimit`. For manually created `BackupVerificationSession`, its user's responsibility to cleanup them.
